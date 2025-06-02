# Impala

## Coordinator Role

* CM ＞ Impala ＞ Instances ＞ Role Groups
  * Role Group 생성
    * Create Role Group ＞ IMPALA_COORDINATOR_ONLY
    * Create Role Group ＞ IMPALA_EXECUTOR_ONLY
  * 좌측 IMPALA DAEMON
    * IMPALA_COORDINATORY_ONLY에 Coordinator로 사용할 호스트를 모두 추가
    * IMPALA_EXECUTOR_ONLY에 Executor로 사용할 호스트를 모두 추가
* CM ＞ Impala ＞ Configuration ＞ Impala Daemon Specialization
  * IMPALA_COORDINATOR_ONLY → COORDINATOR_ONLY
  * IMPALA_EXECTOR_ONLY → EXECUTOR_ONLY

## 최적화 옵션

### Max Client Connection

* CM ＞ Impala ＞ Configuration ＞ Impala Daemon Max Client Connections
  * IMPALA_COORDINATOR_ONLY → 1024

### Query 옵션

* CM ＞ Impala ＞ Configuration ＞ Impala Daemon Default Query Options
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

## Kudu 관련 옵션

| 항목                                     | 기본값      | 설명            |
| ---------------------------------------- | ---------- | --------------- |
| `--kudu_operation_timeout_ms`            | `180000`   |  Timeout (milliseconds) set for all Kudu operations. This must be a positive value, and there is no way to disable timeouts. |
| `--kudu_error_buffer_size`               | `10485760` |  The size (bytes) of the Kudu client buffer for returning errors, with a min of 1KB.If the actual errors exceed this size the query will fail. |
| `--kudu_mutation_buffer_size`            | `10485760` |  The size (bytes) of the Kudu client buffer for mutations. |
| `--kudu_scanner_keep_alive_period_sec`   | `15`       |  The period at which Kudu Scanners should send keep-alive requests to the tablet server to ensure that scanners do not time out. |
| `--kudu_max_row_batches`                 | `0`        |  The maximum size of the row batch queue, for Kudu scanners.  |

## 트러블 슈팅

### max allowed lag is 720000ms

쿼리 실행 시 임팔라 worker노드에서 쿼리 실행 리포트를 coordinator에 720000ms내에 보내지 않았을 경우에 발생하는 에러

```
Failed with message "Query 1a4dd6112f7f2908:4afb942600000000 cancelled due to unresponsive backend: 10.99.12.103:27000 has not sent a report in 732187ms (max allowed lag is 720000ms)
```

CM > Impala > Configuration > Impala Command Line Argument Advanced Configuration Snippet (Safety Valve)에서 다음을 설정하여 coordinator가 더 대기하도록 설정.

```
--status_report_cancellation_padding=30
--status_report_max_retry_s=800
```

위와 관련된 설정에 대해서 Impala 소스코드에는 다음과 같이 정의되어 있음

```
DEFINE_int32(status_report_interval_ms, 5000, "(Advanced) Interval between profile "
    "reports in milliseconds. If set to <= 0, periodic reporting is disabled and only "
    "the final report is sent.");
DEFINE_int32(status_report_max_retry_s, 600, "(Advanced) Max amount of time in seconds "
    "for a backend to attempt to send a status report before cancelling. This must be > "
    "--status_report_interval_ms. Effective only if --status_report_interval_ms > 0.");
DEFINE_int32(status_report_cancellation_padding, 20, "(Advanced) The coordinator will "
    "wait --status_report_max_retry_s * (1 + --status_report_cancellation_padding / 100) "
    "without receiving a status report before deciding that a backend is unresponsive "
    "and the query should be cancelled. This must be > 0.");

[[noreturn]] void ImpalaServer::UnresponsiveBackendThread() {
  int64_t max_lag_ms = FLAGS_status_report_max_retry_s * 1000
      * (1 + FLAGS_status_report_cancellation_padding / 100.0);
  DCHECK_GT(max_lag_ms, 0);
  VLOG(1) << "Queries will be cancelled if a backend has not reported its status in "
          << "more than " << max_lag_ms << "ms.";
  while (true) {
    vector<CancellationWork> to_cancel;
    query_driver_map_.DoFuncForAllEntries(
        [&](const std::shared_ptr<QueryDriver>& query_driver) {
          ClientRequestState* request_state = query_driver->GetActiveClientRequestState();
          Coordinator* coord = request_state->GetCoordinator();
          if (coord != nullptr) {
            NetworkAddressPB address;
            int64_t lag_time_ms = coord->GetMaxBackendStateLagMs(&address);
            if (lag_time_ms > max_lag_ms) {
              to_cancel.push_back(
                  CancellationWork::TerminatedByServer(request_state->query_id(),
                      Status(TErrorCode::UNRESPONSIVE_BACKEND,
                          PrintId(request_state->query_id()),
                          NetworkAddressPBToString(address), lag_time_ms, max_lag_ms),
                      false /* unregister */));
            }
          }
        });

    // We call Offer() outside of DoFuncForAllEntries() to ensure that if the
    // cancellation_thread_pool_ queue is full, we're not blocked while holding one of the
    // 'query_driver_map_' shard locks.
    for (auto cancellation_work : to_cancel) {
      cancellation_thread_pool_->Offer(cancellation_work);
    }
    SleepForMs(max_lag_ms * 0.1);
  }
}
```

따라서 max lag는 다음과 같이 계산할 수 있음

```
max_lag_ms = FLAGS_status_report_max_retry_s * 1000 * (1 + FLAGS_status_report_cancellation_padding / 100.0)
```
