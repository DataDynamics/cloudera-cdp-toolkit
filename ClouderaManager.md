# Cloudera Manager

## 주요 구성 요소

| 구분 | 서비스 | 비고 |
|-----|--------|------|
| Cloudera Manager Server | Server | 최소 1개의 노드에 설치해야 하며 이중화가 가능하여 운영 안정화가 중요한 곳에서는 2개 설치 (L4 필요) | 
| Cloudera Manager Agent | Agent <br/> Supervisor | 모든 노드는 Agent와 Supervisor(데몬 프로세스를 모니터링하고 관리하는 역할)가 동작 | 
| Cloudera Management Service | Alert Publisher <br/> Event Server <br/> Host Monitor <br/> Reports Manager <br/> Service Monitor | Service Monitor 등의 메모리를 크게 사용하는 이유로 클러스터의 규모가 크거나, Kudu/Kafka 등 서비스를 많이 사용하는 경우 별도 노드로 분리 | 

### Server

Cloudera Manager의 핵심 구성 요소 중 하나인 Cloudera Manager Server는 클러스터 전체의 중앙 제어 시스템 역할을 수행합니다.
Cloudera Manager Server는 클러스터 전체의 서비스 설치, 구성, 모니터링, 제어, 보안 정책 관리 등을 중앙에서 수행하는 핵심 관리 서버입니다.

* 클러스터 중앙 제어
  * 모든 노드에 설치된 Cloudera Manager Agent로 명령을 전송하고,
  * 클러스터 전체의 상태와 동작을 중앙 집중적으로 관리합니다.
    * 예: HDFS, YARN, Hive, Impala 등의 서비스 시작/중지/설정 변경
* 서비스 설치 및 구성 관리
  * 각 서비스(HDFS, Impala, etc.)를 자동 설치하고,
  * 사용자 정의에 따라 구성 파일을 생성한 뒤 Agent를 통해 각 노드에 배포합니다.
  * 구성 변경 시, 버전 관리와 롤백 기능도 제공합니다.
* 모니터링 및 알림
  * 각 노드의 Agent로부터 주기적으로 받은 health check, 메트릭, 로그 정보를 수집하고 시각화합니다.
  * 이상 탐지 또는 임계값 초과 시 알람(Event/Alert) 을 생성해 이메일, SNMP 등으로 알립니다.
* 명령 스케줄링 및 실행
  * 롤링 재시작, 설정 변경 반영, 업그레이드 등의 작업을 스케줄링하고 순차적으로 실행합니다.
  * 사용자 명령은 내부적으로 command queue로 관리됩니다.
* 보안 관리
  * Kerberos 인증 통합, TLS 인증서 배포, 사용자/그룹 관리, 역할 기반 접근 제어(RBAC)를 제공합니다.
  * Ranger, Sentry 등과 연동하여 보안 정책을 통합적으로 관리할 수 있습니다.
* Parcel 및 패키지 관리
  * Cloudera Runtime, 서비스 패키지를 Parcel 형식으로 다운로드, 배포, 활성화합니다.
  * 다수 노드에 대한 일괄 배포를 자동화하며, 버전 관리 및 롤백도 지원합니다.
* 웹 UI 및 API 제공
  * 관리자는 Cloudera Manager의 웹 UI를 통해 클러스터 상태를 시각적으로 확인하고 제어할 수 있습니다.
  * RESTful API를 통해 자동화 스크립트, CI/CD 파이프라인, 외부 시스템 통합도 가능합니다.

요약하면 다음과 같습니다.

| 역할      | 설명                       |
| ------- | ------------------------ |
| 중앙 제어   | 클러스터 전체 상태 관리 및 명령 제어    |
| 구성 관리   | 서비스 설정 생성 및 배포           |
| 모니터링    | 각 노드의 상태 및 리소스 사용 정보 수집  |
| 알림 및 로그 | 이벤트 감지 및 로그 통합           |
| 보안      | Kerberos, TLS, 사용자 권한 관리 |
| 패키지 관리  | Parcel 기반 설치, 업그레이드, 롤백  |
| 인터페이스   | 웹 UI 및 REST API 제공       |

### Agent

Agent는 각 클러스터 노드(서버)에 설치되어 있으며, Cloudera Manager Server와의 통신 및 로컬 서비스 제어를 담당하는 핵심 구성요소입니다. 
Cloudera Manager Agent는 모든 클러스터 노드에 설치되어 있고, Cloudera Manager Server의 지시를 받아 노드 상의 서비스를 실행·관리·모니터링하는 책임을 집니다.

* 서비스 시작/중지/재시작
  * CM Server가 서비스(예: HDFS DataNode, Impala Daemon 등)를 시작/중지/재시작하라는 명령을 내리면,
  * Agent는 해당 명령을 받아 로컬 Supervisor를 통해 서비스를 제어합니다.
* 구성 파일 배포
  * CM Server에서 전달된 구성 템플릿(template)을 바탕으로,
  * Agent는 각 서비스의 conf 디렉토리에 필요한 설정 파일을 생성하고 배포합니다.
    * 예: hdfs-site.xml, core-site.xml, yarn-site.xml 등
* 서비스 상태 및 리소스 모니터링
  * Agent는 실행 중인 각 서비스의 상태(UP/DOWN), CPU/Memory 사용량, 디스크 상태 등을 수집합니다.
  * 이 정보를 CM Server에 주기적으로 heartbeat 메시지로 전송합니다.
* Command 실행
  * CM Server로부터 받은 명령(CMD, 실행 스크립트, 롤링 업그레이드 등)을 처리합니다.
  * 각 명령은 JSON 구조의 작업으로 전달되며, Agent는 이를 해석해 실행합니다.
* 패키지/바이너리 설치 및 업데이트
  * 서비스가 처음 배포되거나 업데이트될 때, CM Server는 Agent를 통해 필요한 패키지와 바이너리를 배포합니다.
    * 예: parcel 또는 package 방식으로 CDH 컴포넌트 설치
* 보안 설정 적용
  * Kerberos, TLS와 같은 보안 설정(Credentials, Keytab, SSL cert 등) 을 노드에 배포하고 서비스에 반영합니다.

요약하면 다음과 같습니다.

| 역할     | 설명                             |
| ------ | ------------------------------ |
| 서비스 제어 | CM Server 명령에 따라 서비스 실행/중지/재시작 |
| 구성 배포  | 서버로부터 설정 파일을 받아 배포             |
| 상태 보고  | 각 서비스 상태 및 리소스 사용 정보를 주기적으로 보고 |
| 명령 실행  | 설치, 업그레이드, 진단 명령 등을 실행         |
| 보안 구성  | Kerberos/TLS 관련 설정을 배포 및 적용    |
| 패키지 설치 | 서비스 바이너리 또는 parcel 설치 수행       |

### Supervisor

Cloudera Manager의 **supervisor**는 Cloudera Manager Agent 내부의 서브 컴포넌트이며, 아래와 같은 기능을 수행합니다

* 서비스 프로세스의 실행 및 감시
  * Supervisor는 Cloudera Manager Agent에 의해 배포된 데몬(service) 들을 시작(Start), 중지(Stop), 재시작(Restart) 합니다.
  * 예를 들어 DataNode, NodeManager, Impala Daemon 등의 프로세스를 실행하고 상태를 주기적으로 체크합니다.
  * 만약 프로세스가 비정상적으로 종료되면 Supervisor가 이를 감지하고 자동 재시작하거나 상태를 Report합니다.
* 프로세스 생명주기 관리 (Lifecycle management)
  * Supervisor는 PID 파일, 로그 파일, 환경 변수 등을 관리하면서 각 서비스의 생명주기를 제어합니다.
  * 각 서비스는 고유한 run directory에 배포되며, Supervisor는 해당 디렉토리를 기반으로 서비스 인스턴스를 관리합니다.
* Cloudera Manager Server와 통신
  * Supervisor는 실행 중인 서비스들의 상태와 리소스 사용량(메모리, CPU) 을 수집해서 Cloudera Manager Server에 전송합니다.
  * 이를 통해 Cloudera Manager UI에서 서비스 상태를 실시간 모니터링할 수 있게 됩니다.
* 설정된 Command 수행
  * Cloudera Manager Server로부터 받은 명령 (예: 구성 변경, 로그 수집, 롤링 재시작)을 로컬에서 실행하는 책임도 가집니다.
  * Supervisor는 이러한 명령을 받아 적절한 프로세스를 제어합니다.
* Failover 방지 및 안정성 확보
  * Supervisor는 비정상 종료된 프로세스 재시작, 로그 롤링, 보안 환경 설정 적용(SASL, Kerberos 등) 등을 통해 시스템 안정성을 확보하는 역할도 수행합니다.

요약하면 다음과 같습니다.

| 기능         | 설명                                 |
| ---------- | ---------------------------------- |
| 서비스 실행     | DataNode, YARN NodeManager 등 데몬 실행 |
| 프로세스 모니터링  | 비정상 종료 시 재시작                       |
| 상태 보고      | 메모리, CPU, 상태 정보를 CM Server에 전달     |
| 명령 실행      | 구성 변경, 롤링 재시작 등의 명령 수행             |
| 시스템 안정성 유지 | 프로세스 감시 및 보안 설정 적용                 |

### 기능 비교표

| 구분   | 역할        | 설명                       |
|--------|----------- | ------------------------ |
| Server | 중앙 제어   | 클러스터 전체 상태 관리 및 명령 제어    |
|  | 구성 관리   | 서비스 설정 생성 및 배포           |
|  | 모니터링    | 각 노드의 상태 및 리소스 사용 정보 수집  |
|  | 알림 및 로그 | 이벤트 감지 및 로그 통합           |
|  | 보안        | Kerberos, TLS, 사용자 권한 관리 |
|  | 패키지 관리  | Parcel 기반 설치, 업그레이드, 롤백  |
|  | 인터페이스   | 웹 UI 및 REST API 제공       |
| Agent | 서비스 제어 | CM Server 명령에 따라 서비스 실행/중지/재시작 |
|  | 구성 배포  | 서버로부터 설정 파일을 받아 배포             |
|  | 상태 보고  | 각 서비스 상태 및 리소스 사용 정보를 주기적으로 보고 |
|  | 명령 실행  | 설치, 업그레이드, 진단 명령 등을 실행         |
|  | 보안 구성  | Kerberos/TLS 관련 설정을 배포 및 적용    |
|  | 패키지 설치 | 서비스 바이너리 또는 parcel 설치 수행       |
| Supervisor | 서비스 실행     | DataNode, YARN NodeManager 등 데몬 실행 |
|  | 프로세스 모니터링  | 비정상 종료 시 재시작                   |
|  | 상태 보고      | 메모리, CPU, 상태 정보를 CM Server에 전달   |
|  | 명령 실행      | 구성 변경, 롤링 재시작 등의 명령 수행        |
|  | 시스템 안정성 유지 | 프로세스 감시 및 보안 설정 적용          |

### 서비스 관리 커맨드

| 구분   | 재시작 커맨드  |                          |
|--------|---------------| ------------------------ |
| Server |  `systemctl restart cloudera-scm-server`  |  |
| Agent |  `systemctl restart cloudera-scm-agent`  |  |
| Supervisor |  `systemctl restart cloudera-scm-supervisord`  |  |

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

`/etc/cloudera-scm-agent/config.ini` 파일은 직접 수정할 수 있으며 그중 가장 중요한 정보는 다음가 같습니다.

```
[General]
# Hostname of the CM server.
server_host=10.0.1.60

# Port that the CM server is listening on.
server_port=7182
... 생략
```

 `server_host`는 Cloudera Manager Server의 IP 주소로서 Cloudera Manager Server가 이중화 되어 있는 경우 L4 Swich의 IP 주소를 기입합니다. `server_port` 또한 Cloudera Manager Server가 이중화 되어 있는 경우 L4 Switch의 포트를 기입합니다.

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

## TLS 활성화

Auto TLS를 적용하는 방법은 "Cloudera Manager > Administration > Enable Auto-TLS"를 통해서 가능하며 다음과 설정하도록 합니다.

| 설정명 | 설정값 | 기본값 | 비고 | 
| Trusted CA Certificated Location | | | 빈칸으로 놔두면 자동 생성. PEM Format의 CA 인증서를 제공할 수 있음 |
| Enable TLS for | All existing and future clusters <br/>  | Future clusters only | 일반적으로 All existing and future clusters을 사용 |
| SSH Username | | `root` | SSH Username |
| Authentication Method | All hosts accept same password <br/> All hosts accept same private key | All hosts accept same password | |
| Password | | | `root`의 패스워드 |
| Confirm Password | | | `root`의 패스워드 |
| SSH Port | | `22` | SSH Port |

Cloudera Manager Server의 웹 관리 콘솔에서 TLS를 활성화 하면 CM Server가 CA가 되며, CM Server와 CM Agent는 TLS 통신을 하게 됩니다.

TLS를 활성화 하면 모든 서비스를 재시작해야 합니다.

