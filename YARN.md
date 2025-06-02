# YARN

## YARN의 3대 핵심 컴포넌트

YARN을 구성하는 핵심 컴포넌트는 다음의 3개임

| 구성 요소               | 역할                                      | 위치                |
| ---------------------- | --------------------------------------- | ----------------- |
| ResourceManager (RM)   | 클러스터 전체 리소스 관리, 스케줄링, Application 관리    | 클러스터 중앙           |
| NodeManager (NM)       | 노드 단위의 리소스 모니터링 및 Container 실행 담당       | 각 Worker 노드       |
| ApplicationMaster (AM) | 하나의 Application(Job)의 Task 실행 계획, 상태 관리 | Application마다 하나씩 |

항목별로 비교하면 다음과 같음

| 항목  | ResourceManager  | NodeManager             | ApplicationMaster          |
| --- | ---------------- | ----------------------- | -------------------------- |
| 역할  | 리소스 스케줄링 및 AM 관리 | 로컬 리소스 관리, Container 실행 | 단일 애플리케이션 실행 제어            |
| 배치  | 클러스터 중앙          | 각 노드                    | 각 애플리케이션당 1개               |
| 수명  | 장기 실행            | 장기 실행                   | 애플리케이션 수명 주기와 동일           |
| 재시작 | HA 가능            | 자동 재시작                  | 재시도 설정 가능 (`max-attempts`) |

## Map/Reduce Task와 Container 개념

* YARN Container
  * YARN에서 Container는 애플리케이션의 Task를 실행하는 데 필요한 리소스(메모리, vcore) 단위.
  * 각 Container는 ResourceManager로부터 NodeManager에 의해 할당됨.
  * MapReduce, Spark, Hive 등 YARN 위에서 실행되는 앱은 각각 Container를 요청하여 작업을 실행함.
* MapReduce Task
  * Hadoop MapReduce Job은 다음과 같은 Task로 구성
    * Map Task: 입력 데이터를 나누고 key/value 쌍 생성
    * Reduce Task: Map 출력 데이터를 집계/정리
  * 각 Task는 Container 하나에서 실행됨

## YARN의 주요 파라미터

| 구성 요소                  | 설정 항목                                       | 설명                        | 기본값 예시     |
| ---------------------- | ------------------------------------------- | --------------------------------- | ---------- |
| YARN NodeManager       | `yarn.nodemanager.resource.memory-mb`       | 해당 노드에서 YARN이 사용할 수 있는 총 메모리 (MB) | 8192       |
|                        | `yarn.nodemanager.resource.cpu-vcores`      | 해당 노드에서 YARN이 사용할 수 있는 CPU core 수    | 4          |
| ApplicationMaster (AM) | `yarn.app.mapreduce.am.resource.memory-mb`  | AM이 사용할 메모리                               | 1024       |
|                        | `yarn.app.mapreduce.am.resource.cpu-vcores` | AM이 사용할 core 수                              | 1          |
| Map Task               | `mapreduce.map.memory.mb`                   | Map Task Container에 할당되는 메모리             | 1024       |
|                        | `mapreduce.map.cpu.vcores`                  | Map Task에 할당되는 core 수                      | 1          |
| Reduce Task            | `mapreduce.reduce.memory.mb`                | Reduce Task Container 메모리                    | 1024~2048 |
|                        | `mapreduce.reduce.cpu.vcores`               | Reduce Task에 할당되는 core 수                   | 1          |

## Container Memory와 MapReduce Task Memory의 관계

| 용어                                                                           | 의미                                                          | 관계                         |
| ------------------------------------------------------------------------------ | ------------------------------------------------------------ | ---------------------------- |
| Container Memory (`mapreduce.map.memory.mb`, `mapreduce.reduce.memory.mb`) | YARN이 Task 실행을 위해 할당하는 전체 메모리 공간                   | 최대 크기                      |
| Task JVM Heap Size (`mapreduce.{map,reduce}.java.opts` 의 `-Xmx`)           | 실제 Java 코드가 사용할 수 있는 Heap 공간                         | 보통 Container memory의 70~90% |
| Container Overhead                                                         | JVM Heap 외에 필요로 하는 native memory, GC, thread stack, buffer 등 | Container memory - JVM Heap  |

## vCore와 관계 및 개념

* vCore (virtual core): YARN에서 논리적으로 정의한 CPU 자원의 단위.
* Container가 Task를 실행할 때 몇 개의 vCore를 사용할지 지정할 수 있음.
* 이는 실제 물리 CPU와 1:1 매칭되지는 않지만, 리소스 스케줄링 단위로 사용함.

Container Core 관련 주요 설정은 다음과 같음

| 구성 요소             | 설정 항목                                       | 설명                             | 기본값 예시             |
| ----------------- | ------------------------------------------- | ------------------------------ | ------------------ |
| NodeManager       | `yarn.nodemanager.resource.cpu-vcores`      | 해당 노드가 YARN에게 제공 가능한 총 vCore 수 | `8` (물리 core 수 기준) |
| MapReduce Task    | `mapreduce.map.cpu.vcores`                  | Map Task 하나가 요구하는 vCore 수      | `1`                |
|                   | `mapreduce.reduce.cpu.vcores`               | Reduce Task 하나가 요구하는 vCore 수   | `1`                |
| ApplicationMaster | `yarn.app.mapreduce.am.resource.cpu-vcores` | AM이 요구하는 core 수                | `1`                |

Core의 역할은 다음과 같음

| 역할            | 설명                                           |
| ------------- | -------------------------------------------- |
| 스케줄링 제한   | YARN은 사용 가능한 vCore 수를 기준으로 Container를 배치     |
| 병렬 실행 조절  | 많은 vCore를 요청하면 병렬성이 낮아지고, 적게 요청하면 작업 간 경합 증가 |
| 리소스 분배 통제 | 하나의 Job이 과도하게 cluster CPU를 독점하는 것을 방지        |

하지만 다음의 개념도 명확하게 이해하고 있어야 함

| 항목                               | 설명                                                                               |
| -------------------------------- | ----------------------------------------------------------------------------------- |
| YARN은 CPU를 강제 격리하지 않음 (기본)   | 즉, core 수를 1로 설정해도 실제로는 더 많은 CPU를 사용할 수도 있음                 |
| CPU 사용량 조절을 위해 cgroups 사용 가능 | OS-level에서 CPU 격리를 원한다면 cgroups 설정 필요 (`yarn.nodemanager.linux-container-executor`) |
| vCore ≠ 물리 코어                | YARN은 논리 단위일 뿐, 실제 CPU thread와 다를 수 있음. 일반적으로 hyper thread를 vCore라고 이해해도 무방   |

## 파라미터 설정시 이해해야 하는 개념

* File 1개당 1개의 Task(Container)가 배정됨. 단, 파일의 크기가 HDFS Block Size보다 크면 Block Size 별로 Task가 배정됨
* Small File이 많은 경우 많은 Task가 배정되므로 불필요하게 많은 vCore와 Memory를 소비하게 됨
* 작업의 유형에 따라서 Memory를 많이 설정할 수 있고, vCore를 많이 설정할 수도 있음
* 작업의 유형, 파일의 개수 및 파일의 크기에 따라서 Container의 vCore, Memory를 조절해야 함 (Job의 동작 특성을 잘 이해해야 함)

## Small File의 대처 방안

* Small File은 일반적으로 많은 Task를 동반하므로 compaction해서 큰 단위로 가지고 있는 것이 좋으나 그렇지 못한 경우 여러 옵션을 이용해서 리소스 사용을 변경할 수 있음

| 방법                              | 효과                      | 적용 방식            |
| ------------------------------- | ----------------------- | ---------------- |
| `mapreduce.job.jvm.numtasks=-1` | JVM 재사용 = Container 재사용 | 모든 Task에 적용 가능   |
| CombineFileInputFormat          | Task 수 자체를 줄임           | Small File 병합 처리 |
| split.maxsize 조절                | Task 수 줄임               | 파일 묶기 기준 변경      |

따라서 다음과 같이 적용할 수 있음

| 목적                  | 설정                                              | 값              |
| ------------------- | ----------------------------------------------- | -------------- |
| Container (JVM) 재사용 | `mapreduce.job.jvm.numtasks`                    | `-1`           |
| Small File 묶기       | `CombineFileInputFormat` + `split.maxsize`      | 64MB~256MB 권장 |
| 병렬 Task 줄이기         | `mapreduce.input.fileinputformat.split.maxsize` | 적절히 설정         |

## Hive에서 Small File 대처

### CombineHiveInputFormat 사용

* 여러 Small File을 하나의 InputSplit으로 병합하여 처리
* Hive 0.14+에서는 기본으로 활성화
* Merge 기준은 HDFS Block Size와 `mapreduce.input.fileinputformat.split.maxsize`에 의해 결정

```
SET hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
```

### INSERT OVERWRITE

Small File이 많은 HDFS Directory를 Map Task로 읽고 쓰게 되면 큰 단위로 묶이게 되는데 이를 이용한 방법

```
INSERT OVERWRITE TABLE target_table
SELECT * FROM source_table;
```

### File Merge (Compaction) for ORC, ACID

ACID 테이블에서 지원되는 파일 병합 기능을 사용할 수 있으며 다음과 같이 Hive에서 쿼리로처리

```
-- 실행 (major compaction = full merge)
ALTER TABLE mytable COMPACT 'MAJOR';

-- 실행된 compaction 적용 (만약 auto compaction이 꺼져 있다면)
ALTER TABLE mytable COMPACT 'MINOR';
```

ACID 테이블의 경우 다음의 설정을 통해서 자동화 가능

```
SET hive.compactor.initiator.on=true;
SET hive.compactor.worker.threads=2;
SET hive.compaction.check.interval=300;  -- 5분마다 확인
```

### MapReduce 병렬성 제어

MapReduce의 InputSplit 크기를 조절하여 작은 파일을 하나의 Map Task에서 처리할 수 있으므로 Container 수를 감소시켜 리소스 사용을 줄여주는 효과 발새 

```
SET mapreduce.input.fileinputformat.split.minsize=134217728;  -- 128MB
SET mapreduce.input.fileinputformat.split.maxsize=268435456;  -- 256MB
```

### Partition 병합

파티션도 많고, 파티션 별로 Small File이 많은 경우 Merge를 수행

```
SET hive.exec.dynamic.partition=true;
SET hive.exec.dynamic.partition.mode=nonstrict;

INSERT OVERWRITE TABLE target PARTITION (dt)
SELECT ... FROM source;
```

### 테이블 저장 포맷

Impala와 Iceberg를 사용한다면 Parquet가 기본 포맷이 될 수 있음

| 포맷     | 효과                                         |
| ---------| ------------------------------------------ |
| ORC      | columnar + 압축 + 파일 크기 조정 유리                |
| Parquet  | Spark/Hive 겸용 가능, splitable, block size 기반. Impala, Iceberg의 기본 파일 포맷 |
| TEXTFILE | splitable 하지만 small file 병합 어려움            |

## YARN CLI

### 주요 CLI 정리

| 명령어                                                        | 설명                                           |              |
| ---------------------------------------------------------- | -------------------------------------------- | ------------ |
| `yarn application -list`                                   | 실행 중인 애플리케이션 목록 표시                           |              |
| `yarn application -status <app_id>`                        | 특정 애플리케이션의 상세 정보 확인                          |              |
| `yarn application -kill <app_id>`                          | 실행 중인 애플리케이션 강제 종료                           |              |
| `yarn application -list -appStates <STATE>`                | 지정된 상태의 애플리케이션 조회 (예: `RUNNING`, `FINISHED`) |              |
| `yarn logs -applicationId <app_id>`                        | 애플리케이션 로그 확인 (stdout, stderr 포함)             |              |
| `yarn logs -applicationId <app_id> -appOwner <user>`       | 다른 사용자의 애플리케이션 로그 조회                         |              |
| `yarn node -list`                 | 클러스터의 모든 NodeManager 리스트 출력                   |
| `yarn node -status <node_id>`     | 특정 노드의 상태, 리소스, 컨테이너 정보 확인                    |
| `yarn node -all`                  | 전체 노드 상태 포함 (비활성 등)                           |
| `yarn node -list -states <state>` | 특정 상태 노드 필터링 (e.g., RUNNING, LOST, UNHEALTHY) |
| `yarn cluster`         | 클러스터 요약 정보 (Node 수, Memory, Containers 등) |
| `yarn cluster -status` | ResourceManager의 상태 (active/standby 등)    |
| `yarn top`             | 애플리케이션별 리소스 사용량 Top 목록 실시간 확인 (버전 3.x 이상) |
| `yarn queue -status <queue_name>`         | 특정 YARN 큐의 리소스, 상태, 접근 제어 정보 확인                          |
| `yarn schedulerconf -list`                | Scheduler에 등록된 queue 설정 리스트 출력 (Capacity Scheduler 사용 시) |
| `yarn schedulerconf -status <queue_name>` | 큐별 상세 설정 정보 확인                                           |
| `yarn timelineService v.2.0` | 애플리케이션 시간 정보(Timeline) 활용 (v2 TS가 활성화되어야 함) |


### Application 관련 CLI

| 목적                         | 명령어                                                 | 설명                              |
| -------------------------- | --------------------------------------------------- | ------------------------------- |
| 전체 실행 중 애플리케이션 목록          | `yarn application -list`                            | 현재 클러스터에서 실행 중인 애플리케이션 표시       |
| 특정 상태 필터링                  | `yarn application -list -appStates RUNNING`         | 실행 중인 애플리케이션만 표시                |
| 여러 상태 필터링                  | `yarn application -list -appStates FINISHED,KILLED` | 지정한 상태의 애플리케이션 조회               |
| 특정 사용자 필터링                 | `yarn application -list -user <username>`           | 해당 사용자의 애플리케이션만 표시              |
| 애플리케이션 상세 정보               | `yarn application -status <application_id>`         | Application ID에 해당하는 상세 정보 출력   |
| 애플리케이션 강제 종료               | `yarn application -kill <application_id>`           | 실행 중인 애플리케이션 종료                 |
| 애플리케이션 로그 조회               | `yarn logs -applicationId <application_id>`         | 실행된 애플리케이션의 stdout/stderr 로그 출력 |
| 다른 사용자 로그 조회               | `yarn logs -applicationId <id> -appOwner <user>`    | 다른 사용자의 애플리케이션 로그 보기            |
| 로그 파일 타입 지정                | `yarn logs -applicationId <id> -log_files stdout`   | 특정 로그 파일(stdout/stderr)만 출력     |
| 모든 상태 포함해서 목록 조회           | `yarn application -list -appStates ALL`             | 모든 상태의 애플리케이션 조회                |
| ResourceManager 재시작 이후도 포함 | `yarn application -list -include-all`               | 완료된 애플리케이션까지 전체 목록 조회           |

