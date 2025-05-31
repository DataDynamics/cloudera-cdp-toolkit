# Impala

## Coordinator Role

* CM > Impala > Instances > Role Groups
  * Role Group 생성
    * Create Role Group > IMPALA_COORDINATOR_ONLY
    * Create Role Group > IMPALA_EXECUTOR_ONLY
  * 좌측 IMPALA DAEMON
    * IMPALA_COORDINATORY_ONLY에 Coordinator로 사용할 호스트를 모두 추가
    * IMPALA_EXECUTOR_ONLY에 Executor로 사용할 호스트를 모두 추가
* CM > Impala > Configuration > Impala Daemon Specialization
  * IMPALA_COORDINATOR_ONLY --> COORDINATOR_ONLY
  * IMPALA_EXECTOR_ONLY --> EXECUTOR_ONLY

## 최적화 옵션

### Max Client Connection

* CM > Impala > Configuration > Impala Daemon Max Client Connections
  * IMPALA_COORDINATOR_ONLY --> 1024

### Query 옵션

* CM > Impala > Configuration > Impala Daemon Default Query Options
  * `OPTIMIZE_SIMPLE_LIMIT` = TRUE
  * `EXEC_TIME_LIMIT_S` = 600
  * `BATCH_SIZE` = 65536
  * `SPOOL_QUERY_RESULTS` = TRUE
  * `MAX_RESULT_SPOOLING_MEM` = 1073741824

### Impala Daemon 옵션

### Impala StateStore 옵션

### Impala Catalog 옵션
