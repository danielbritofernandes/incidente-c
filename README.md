# Incidente C — Split-brain depois de um ataque de rede

## 1. Identificação

| Campo | Valor |
| --- | --- |
| Nome | Daniel Brito |
| Formato | Individual |
| Cenário | C — Split-brain depois de um ataque de rede |
| Caminho | A (Docker Compose). Primário publicado em `5442`, réplica em `5443` |
| Data | 16/11/2026 |
| Imagem | `postgres:16` |

O relógio do host está em UTC−3. O relógio do PostgreSQL está em UTC, três horas à frente. Os horários da linha do tempo são os do host. Os `timestamptz` copiados do banco estão em UTC.

## 2. Resumo do incidente

Durante o plantão, a réplica foi promovida com `pg_promote()` enquanto o primário da porta 5442 continuava no ar e aceitando commit. Os dois nós passaram a ser primários, cada um na sua timeline: a antiga permaneceu na timeline 1 e a promovida abriu a timeline 2. Em seguida cada lado gravou linhas em `eventos` que o outro nunca recebeu. A replicação, que era assíncrona, não impede isso: `pg_promote()` só tira a réplica de recovery; não desliga o primário antigo.

A correção foi registrar a divergência, isolar o primário antigo e ficar com um único escritor na porta 5443. No bônus, o disco antigo foi apagado e reconstruído com `pg_basebackup` a partir do primário novo. A porta 5442 voltou com `pg_is_in_recovery() = true` e passou a recusar `INSERT`.

## 3. Linha do tempo

| Horário (UTC−3) | Ação | Resultado |
| --- | --- | --- |
| 14:59:11 | Conferir os papéis | 5442 fora de recovery. 5443 em recovery. `pg_stat_replication`: `streaming`, `async` |
| 14:59:12 | Criar `clientes` (1000 linhas) e `eventos` | A réplica já enxergava 1000 clientes |
| 14:59:12 | Testar a sentença do enunciado dentro de `ROLLBACK` | `INSERT 0 2` revertido. A sequence não volta: os ids 1 e 2 foram consumidos |
| 14:59:46 | **T0 / T_promocao.** `SELECT pg_promote()` na réplica, com o primário ligado | `pg_promote = t`. Epoch do host: `1794851986` |
| 14:59:47 | Confirmar os dois lados | `pg_is_in_recovery() = false` na 5442 e na 5443. Zero réplicas ligadas ao primário antigo. Log: `selected new timeline ID: 2` |
| 14:59:48 | `INSERT` de 10 linhas na 5442 e de 7 na 5443 | `INSERT 0 10` e `INSERT 0 7` |
| 14:59:49 | Comparar `max(id), count(*)` pelas portas publicadas | 5442: 10 linhas, max id 12. 5443: 7 linhas, max id 40. Os dois fora de recovery |
| 15:01:10 | Exportar as linhas e ler timeline e sequence | WAL `00000001…` no antigo e `00000002…` no novo. CSVs em `evidencias/` |
| 15:02:28 | Isolar o primário antigo: `docker compose stop primary` | Porta 5442: `Connection refused` |
| 15:02:29 | **T1.** Escrita só na 5443 | `prova_escrita` id 1 confirmado. 1000 clientes e 7 eventos preservados. Epoch: `1794852149` |
| 15:03:25 | Apagar o data directory do primário antigo | Volume `lab1_primary_data` vazio |
| 15:03:26 | `pg_basebackup` a partir do primário novo, slot `oldprimary_slot` | 23500 kB, `standby.signal` presente |
| 15:03:28 | Subir o container antigo | `pg_is_in_recovery() = true`. Streaming na timeline 2. 7 eventos, os mesmos ids 34–40 |
| 15:04:06 | `INSERT` da prova id 2 na 5443 | A linha apareceu na 5442 |
| 15:04:08 | `INSERT` na 5442 | `ERROR: cannot execute INSERT in a read-only transaction` |

As saídas literais, com o horário de cada comando, estão em `evidencias/` e a lista ordenada está em `comandos.md`.

## 4. Medições

### Failover

Quem foi promovido: a réplica (container `lab1-replica-1`, porta do host **5443**).

Quando: o comando começou às 14:59:46. O log da réplica registra `received promote request` e `selected new timeline ID: 2` às 17:59:47 UTC, isto é, 14:59:47 no host.

Como foi confirmado: `SELECT pg_is_in_recovery()` na 5443 devolveu `false`, e o arquivo de WAL passou a ser `000000020000000000000003`. A mesma consulta na 5442 também devolveu `false`. O failover aconteceu, e ao mesmo tempo deixou dois primários. Esse segundo fato é o incidente, não a recuperação.

### RTO

Método. O problema começa quando passam a existir dois primários (T0 = T_promocao). Termina quando resta um único primário consistente, já aceitando escrita, e o outro nó não aceita mais conexão (T1).

```text
T0 = 2026-11-16 14:59:46 -03 = 1794851986
T1 = 2026-11-16 15:02:29 -03 = 1794852149
RTO = 1794852149 − 1794851986 = 163 segundos = 2 min 43 s
```

O que esse número inclui: o registro obrigatório das linhas divergentes (contagem, amostra e CSV) antes de desligar o antigo, como pede a correção. O `docker compose stop` em si terminou no mesmo segundo em que foi chamado (15:02:28). A confirmação da escrita na 5443 fechou em 15:02:29.

O que esse número não inclui: detecção por um alarme (aqui o incidente foi provocado de propósito), troca de DNS, pool de conexões da aplicação e o tempo de os clientes passarem a usar a porta 5443. Não havia aplicação no ambiente. Um RTO de produção seria maior por esses passos, ou menor se o fencing fosse automático e não esperasse a coleta manual das linhas.

### RPO

Método. As duas histórias foram copiadas por completo antes de desligar qualquer nó (`evidencias/eventos_primario_antigo.csv` e `evidencias/eventos_novo_primario.csv`). A comparação é por conjunto de linhas, com `count(*)` e a lista de `id`. `max(id)` não mede perda neste incidente: as sequences divergiram e os intervalos de id não se encontram.

| Lado | Linhas | ids | `max(id)` | timestamp (UTC) |
| --- | --- | --- | --- | --- |
| Primário antigo, porta 5442 | 10 | 3 a 12 | 12 | 17:59:48.315781+00 |
| Primário novo, porta 5443 | 7 | 34 a 40 | 40 | 17:59:48.492399+00 |

As 17 linhas têm `ts` posterior a T_promocao (`2026-11-16 17:59:46.564105+00`). A interseção dos ids é vazia. Nenhuma linha de `eventos` existe nos dois lados.

A decisão de recuperação ficou com o primário promovido (timeline 2, porta 5443) e descartou o primário antigo. O RPO efetivo é o conjunto confirmado só no nó descartado:

```text
RPO = 10 linhas de eventos
```

Se a decisão inversa tivesse sido tomada (ficar com a 5442 e descartar a 5443), o RPO seria de 7 linhas. Os 1000 clientes já tinham sido replicados antes da promoção e continuaram nos dois lados; a perda é só de `eventos`.

Por que os ids não continuaram em 13. O teste com `ROLLBACK`, às 14:59:12, chamou `nextval` duas vezes. Sequence não participa do rollback, então o primário antigo seguiu do id 3 até o 12. No momento da comparação, `eventos_id_seq` nesse nó estava em `last_value = 12`, `log_cnt = 21`. O PostgreSQL grava sequence no WAL em bloco, não a cada `nextval`. A réplica promovida recuperou a página já avançada no WAL e o primeiro id que ela emitiu foi 34 (depois do insert, `last_value = 40`, `log_cnt = 26`). Subtrair os máximos (`40 − 12 = 28`, ou o contrário) não é quantidade de linhas perdidas.

## 5. Causa raiz

A promoção foi feita sem fencing e sem quórum. O ambiente tem dois nós e replicação física assíncrona (`sync_state = async`). Nesse desenho, `pg_promote()` é um comando local da réplica: ela sai de recovery, escolhe a timeline 2 e passa a aceitar escrita. O primário da timeline 1 não é avisado para parar. Como a rede do laboratório continuava inteira, os dois ficaram alcançáveis e os dois confirmaram commit.

O ataque de rede do enunciado produz o mesmo estado. O plantão deixa de alcançar o primário, conclui que ele caiu e promove a réplica. O primário, isolado do plantão e da réplica, segue confirmando escrita para quem ainda chega nele. Quando a rede volta, existem dois históricos incompatíveis. Não há um terceiro nó para desempatar, nem um lock de líder, nem o desligamento do processo antigo.

## 6. Correção aplicada e a prova

Regra de reconciliação: a única fonte da verdade depois de T_promocao é o primário promovido na timeline 2 (porta 5443); as 10 linhas gravadas só no primário antigo são divergentes e foram descartadas, e esse disco só volta como réplica reconstruída por `pg_basebackup`.

Passos:

1. Antes de desligar, a contagem e a amostra foram gravadas. `SELECT * FROM eventos WHERE ts > '2026-11-16 17:59:46.564105+00' ORDER BY id LIMIT 5` devolveu ids 3–7 na 5442 e ids 34–38 na 5443. Os CSVs guardam o conjunto inteiro, não só as cinco linhas.
2. O primário antigo foi isolado com `docker compose stop primary`.
3. A regra acima é a regra que foi aplicada: nada foi copiado de volta para a 5443, porque os ids já pertenciam a outra história.

Prova, antes e depois:

| Verificação | Antes (14:59:49) | Depois do isolamento (15:02:29) | Depois da reconstrução (15:04:08) |
| --- | --- | --- | --- |
| 5442 `pg_is_in_recovery()` | `false` | conexão recusada | `true` |
| 5443 `pg_is_in_recovery()` | `false` | `false` | `false` |
| `eventos` na 5442 | 10 linhas (ids 3–12) | indisponível | 7 linhas (ids 34–40) |
| `eventos` na 5443 | 7 linhas (ids 34–40) | 7 linhas (ids 34–40) | 7 linhas (ids 34–40) |
| Escrita | os dois aceitaram `INSERT` | só a 5443; `prova_escrita` id 1 | id 2 replicou da 5443 para a 5442; o `INSERT` na 5442 falhou com transação somente leitura |
| Réplica ligada ao escritor | nenhuma | nenhuma | `streaming` / `async`, timeline 2 |

### Bônus

O primário antigo foi reconstruído como réplica do novo. O enunciado mostra os comandos de `pg_ctl` do Caminho B; aqui o mesmo efeito foi feito no volume Docker, com o container parado:

```bash
pg_basebackup -h replica -p 5432 -U replicator \
  -D /var/lib/postgresql/data -R -X stream -C -S oldprimary_slot
```

Na 5442, `pg_is_in_recovery()` ficou `true`, `primary_slot_name` ficou `oldprimary_slot` e `primary_conninfo` aponta para `host=replica`. O log do container registra `entering standby mode` e `started streaming WAL from primary ... on timeline 2`. As 10 linhas da timeline 1 não voltaram: elas existem apenas nos CSVs de evidência. Isso é o RPO de 10 linhas materializado.

## 7. Comparação com um serviço gerenciado

Em um RDS Multi-AZ, ou no Aurora, o cliente fala com um endpoint de escritor só. A promoção da standby acontece depois do fencing do primário antigo: o nó antigo perde a capacidade de confirmar commit (o armazenamento ou o processo é isolado) e só então o DNS do endpoint passa a apontar para o novo escritor. Não há janela em que os dois aceitam escrita. A cópia entre zonas no Multi-AZ é síncrona no nível do armazenamento, então o RPO das transações já confirmadas ao cliente é zero. O RTO típico publicado pela AWS para failover Multi-AZ fica na casa de 60 a 120 segundos; no Aurora, com armazenamento compartilhado, o failover costuma ficar abaixo de 30 segundos. O RTO de 163 s deste laboratório é da mesma ordem de grandeza, mas o conteúdo é outro: aqui os 163 s incluem dois escritores ativos e uma coleta manual. No serviço gerenciado esse intervalo é detecção, fencing e troca do endpoint, com um escritor de cada vez.

PITR não é a ferramenta que evita este incidente. Ele reconstrói um nó até um instante escolhido e serviria, depois do fato, para obter uma cópia da timeline 1 até T_promocao. Não impede que dois primários confirmem commits diferentes, nem escolhe sozinho qual timeline a aplicação deve seguir. O recurso que trata o caso é o failover automático com fencing. Um Patroni com etcd ou Consul faz o equivalente fora da nuvem: a réplica só promove se conquistar o lock distribuído, e o primário que perde o lock é derrubado pelo agente ou pelo watchdog.

## 8. Prevenção e lições

- Não promover réplica enquanto o primário antigo puder aceitar conexão. A ordem é isolar primeiro, promover depois.
- Um par primário/réplica não tem voto de desempate. O lock de líder precisa de um conjunto de coordenação com quórum.
- A aplicação precisa de um único destino de escrita. Duas connection strings, uma para cada porta, reproduzem o split-brain no cliente mesmo que o banco depois seja consertado.
- Alarme quando `pg_stat_replication` fica vazio e quando mais de um nó responde `pg_is_in_recovery() = false`.
- Replicação síncrona reduz o RPO: o primário não confirma o commit sem a réplica. Ela não substitui o fencing. Numa partição, o primário síncrono trava a escrita em vez de divergir, e alguém ainda precisa decidir quem é o líder.
- O nó antigo não volta a ser ligado com o diretório de dados da timeline 1. Ou ele é refeito por `pg_basebackup` na timeline sobrevivente, ou permanece desligado.
- Antes de apagar o disco perdedor, exportar as linhas posteriores à promoção. Sem o CSV, as 10 linhas deste laboratório teriam sumido sem rastro.

## 9. Respostas às perguntas do cenário

**O que é fencing e por que o serviço gerenciado o faz antes de promover?**

Fencing é impedir, de forma que o próprio nó não consiga desfazer, que o primário antigo confirme mais escrita: parar o processo, cortar o disco ou bloquear a rede dele. O serviço gerenciado faz isso antes da promoção porque a réplica, ao virar primário, passa a dizer "commit" para outros clientes. Se o antigo ainda diz "commit" ao mesmo tempo, as duas respostas são verdadeiras e irreconciliáveis. Foi o que a 5442 e a 5443 fizeram às 14:59:48. Isolar antes da promoção deixa um único histórico.

**Por que um número ímpar de nós (quórum) evita split-brain?**

Com 3 ou 5 nós de coordenação, uma partição de rede deixa a maioria de um lado só. A regra é que apenas o lado com maioria pode ser líder. Os dois lados não conseguem, juntos, acreditar que têm maioria. Com dois nós, como neste laboratório, qualquer metade que o operador enxergue pode promover o nó que está na frente dele. Não existe terceiro voto para dizer que o primário da 5442 ainda estava vivo.
