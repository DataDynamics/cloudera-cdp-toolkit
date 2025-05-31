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

| 항목                                   | 위치                | 포맷  | 역할                      | 보안 수준 |
| ------------------------------------ | ----------------- | --- | ----------------------- | ----- |
| `nifi.sensitive.props.key`           | `nifi.properties` | 평문  | 민감정보 암호화 키              | ❌ 낮음  |
| `nifi.sensitive.props.key.protected` | `nifi.properties` | 암호문 | 암호화된 키                  | ✅ 중간  |
| `nifi.bootstrap.sensitive.key`       | `bootstrap.conf`  | 평문  | protected key 복호화용 루트 키 | ✅ 높음  |



### Impala StateStore 옵션

| 항목                                   | 위치                | 포맷  | 역할                      | 보안 수준 |
| ------------------------------------ | ----------------- | --- | ----------------------- | ----- |
| `nifi.sensitive.props.key`           | `nifi.properties` | 평문  | 민감정보 암호화 키              | ❌ 낮음  |
| `nifi.sensitive.props.key.protected` | `nifi.properties` | 암호문 | 암호화된 키                  | ✅ 중간  |
| `nifi.bootstrap.sensitive.key`       | `bootstrap.conf`  | 평문  | protected key 복호화용 루트 키 | ✅ 높음  |



### Impala Catalog 옵션

| 항목                                     | 위치       | 포맷            |
| ---------------------------------------- | --------- | --------------- |
| `--catalog_topic_mode`                   | `minimal` |                 |
| `--num_metadata_loading_threads`         | `64`      |                 |
| `--invalidate_tables_on_memory_pressure` | `true`    |                 |
| `--invalidate_tables_timeout_s`          | `86400`   |                 |

