# YARN

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

| 구성 요소                  | 설정 항목                                       | 설명                                | 기본값 예시     |
| ---------------------- | ------------------------------------------- | --------------------------------- | ---------- |
| YARN NodeManager       | `yarn.nodemanager.resource.memory-mb`       | 해당 노드에서 YARN이 사용할 수 있는 총 메모리 (MB) | 8192       |
|                        | `yarn.nodemanager.resource.cpu-vcores`      | 해당 노드에서 YARN이 사용할 수 있는 CPU core 수 | 4          |
| ApplicationMaster (AM) | `yarn.app.mapreduce.am.resource.memory-mb`  | AM이 사용할 메모리                       | 1024       |
|                        | `yarn.app.mapreduce.am.resource.cpu-vcores` | AM이 사용할 core 수                    | 1          |
| Map Task               | `mapreduce.map.memory.mb`                   | Map Task Container에 할당되는 메모리      | 1024       |
|                        | `mapreduce.map.cpu.vcores`                  | Map Task에 할당되는 core 수             | 1          |
| Reduce Task            | `mapreduce.reduce.memory.mb`                | Reduce Task Container 메모리         | 1024~2048 |
|                        | `mapreduce.reduce.cpu.vcores`               | Reduce Task에 할당되는 core 수          | 1          |

## Container Memory와 MapReduce Task Memory의 관계

| 용어                                                                           | 의미                                                          | 관계                         |
| ------------------------------------------------------------------------------ | ------------------------------------------------------------ | ---------------------------- |
| Container Memory (`mapreduce.map.memory.mb`, `mapreduce.reduce.memory.mb`) | YARN이 Task 실행을 위해 할당하는 전체 메모리 공간                   | 최대 크기                      |
| Task JVM Heap Size (`mapreduce.{map,reduce}.java.opts` 의 `-Xmx`)           | 실제 Java 코드가 사용할 수 있는 Heap 공간                         | 보통 Container memory의 70~90% |
| Container Overhead                                                         | JVM Heap 외에 필요로 하는 native memory, GC, thread stack, buffer 등 | Container memory - JVM Heap  |
