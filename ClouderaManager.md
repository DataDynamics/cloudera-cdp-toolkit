# Cloudera Manager Server & Agent

## 실행 프로세스

| 구분 | 서비스 | 비고 |
| Cloudera Manager Server | Server | 최소 1개의 노드에 설치해야 하며 이중화가 가능하여 운영 안정화가 중요한 곳에서는 2개 설치 (L4 필요) | 
| Cloudera Manager Agent | Agent <br/> Supervisor | 모든 노드는 Agent와 Supervisor가 동작 | 
| Cloudera Management Service | Alert Publisher <br/> Event Server <br/> Host Monitor <br/> Reports Manager <br/> Service Monitor | Service Monitor 등의 메모리를 크게 사용하는 이유로 클러스터의 규모가 크거나, Kudu/Kafka 등 서비스를 많이 사용하는 경우 별도 노드로 분리 | 

## Directory

### Configuration Directory

| Componnet | Logging Path |
|-----------|--------------|
| Cloudera Manager Server | `/etc/cloudera-scm-server` |
| Cloudera Manager Agent | `/etc/cloudera-scm-agent` |

### Logging Directory

| Componnet | Logging Path |
|-----------|--------------|
| Cloudera Manager Server | `/var/log/cloudera-scm-server` |
| Cloudera Manager Agent | `/var/log/cloudera-scm-agent` |

## Configuration

### Cloudera Manager Server

### Cloudera Manager Agent

`/etc/cloudera-scm-agent`

## Logging

### Cloudera Manager Server

CM 7.13.1의 경우 `/etc/cloudera-scm-server/log4j.properties` 파일에 다음을 추가합니다. DEBUG Level 로 남아야할 로그가 INFO Level로 남는 상황이어서 추가가 필요합니다.

```
# === Custom Appender with Filters ===
log4j.appender.filteredlog=org.apache.log4j.ConsoleAppender
log4j.appender.filteredlog.layout=org.apache.log4j.PatternLayout
log4j.appender.filteredlog.layout.ConversionPattern=%d{ISO8601} %p %c: %m%n

# === Filter #1: Drop warning ===
log4j.appender.filteredlog.filter.1=org.apache.log4j.varia.StringMatchFilter
log4j.appender.filteredlog.filter.1.StringToMatch=Received Process Heartbeat for unknown (or duplicate) process.
log4j.appender.filteredlog.filter.1.AcceptOnMatch=false

# === Filter #2: Drop telemetry config warning ===
log4j.appender.filteredlog.filter.2=org.apache.log4j.varia.StringMatchFilter
log4j.appender.filteredlog.filter.2.StringToMatch=TELEMETRY_ALTUS_ACCOUNT is not configured for Otelcol
log4j.appender.filteredlog.filter.2.AcceptOnMatch=false

# === Accept all other messages ===
log4j.appender.filteredlog.filter.3=org.apache.log4j.varia.AcceptAllFilter
# === Specific logger for AgentProtocolImpl ===
log4j.logger.com.cloudera.server.cmf.AgentProtocolImpl=WARN, filteredlog
log4j.additivity.com.cloudera.server.cmf.AgentProtocolImpl=false

# === Specific logger for BaseMonitorConfigsEvaluator === 
log4j.logger.com.cloudera.cmf.service.config.BaseMonitorConfigsEvaluator=WARN, filteredlog
log4j.additivity.com.cloudera.cmf.service.config.BaseMonitorConfigsEvaluator=false
```

## Cloudera Management Service (CMS)

* Alert Publisher
* Event Server
* Host Monitor
* Reports Manager
* Service Monitor

### Service Monitor

CMS에서 가장 메모리 및 디스크 공간을 많이 사용하는 서비스로 서비스에 대한 메트릭 정보를 수집하는 주체가 서비스 모니터이며, 서비스 모니터에 가장 큰 부담을 주는 Runtime Component는 다음 2개의 서비스입니다.

* Kudu
* Kafka

주요 설정은 다음과 같습니다.

| 설정      |        설정값 | 의미      |
|-----------|--------------|----------|
| Service Monitor Storage Directory | `/var/lib/cloudera-service-monitor` | 서비스 모니터의 저장소 경로 |
| Time-Series Storage | 10 GB (Default) | Health 및 시계열 데이터의 저장소 크기 |
| Reports Time-series Storage | 1 GB (Default) | 리포팅 데이터의 시계열 데이터에 대한 저장소의 크기 |
| Impala Storage | 1 GB (Default) | Impala Query Profile 저장소의 크기 |
| YARN Storage | 1 GB (Default) | YARN Application 저장소의 크기 |
| Enable Metric Collection | ON | 메트릭 수집 여부 |
| Heap Dump Directory | `/tmp` | 힙 덤프 생성 경로 |
| Kill When Out Of Memory | ON | OOM 발생시 Kill 여부 |
| Java Heap Size of Service Monitor in Bytes | 1 GB (Default) | Java Heap Size로 -Xmx 옵션 |
| Maximum Non-Java Memory of Service Monitor | 2 GB (Default) | JVM 외부의 native code나 라이브러리(C/C++ 등), memory-mapped files, threads, buffers, JIT 컴파일러, JNI 등이 사용하는 메모리 |












