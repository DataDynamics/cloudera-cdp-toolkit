# ZooKeeper

## 성능 튜닝

| 파라미터                      | 의미                                                    | 기본값             | 설정 시 유의사항                                                                             |
| ------------------------- | ----------------------------------------------------- | --------------- | ------------------------------------------------------------------------------------- |
| `tickTime`                | Zookeeper의 기본 시간 단위 (ms). heartbeat, timeout 계산에 사용됨. | `2000` (2초)     | 너무 짧게 설정하면 network latency에 민감하고, 너무 길면 장애 감지 및 failover가 느려짐. 일반적으로 2000~3000ms 권장. |
| `initLimit`               | Follower가 Leader로부터 초기 상태를 동기화할 수 있는 tick 수           | `10`            | Leader election 이후 Follower가 초기 상태 복제하는 데 필요한 시간. 느린 네트워크나 대용량 데이터 환경에선 더 크게 설정해야 함.  |
| `syncLimit`               | Follower가 Leader와의 상태 동기화를 유지하는 데 허용되는 최대 tick 수      | `5`             | Follower가 지정 시간 안에 응답하지 않으면 장애 노드로 간주. 너무 작으면 불필요한 재선거 발생 가능성 있음.                     |
| `maxClientCnxns`          | 하나의 IP 주소에서 허용되는 최대 클라이언트 연결 수                        | `60`            | 수천 개의 클라이언트가 하나의 프록시를 통해 연결되는 경우 늘려야 함. `0`이면 무제한.                                    |
| `minSessionTimeout`       | 클라이언트 세션의 최소 타임아웃 (ms)                                | `tickTime * 2`  | 너무 짧으면 세션이 자주 끊김. 클라이언트가 설정한 세션 타임아웃이 이보다 작으면 강제로 min 값으로 설정됨.                        |
| `maxSessionTimeout`       | 클라이언트 세션의 최대 타임아웃 (ms)                                | `tickTime * 20` | 너무 크면 죽은 클라이언트 감지가 지연됨. 필요에 따라 늘릴 수 있음.                                               |
| `autopurge.purgeInterval` | 자동으로 snapshot 및 로그를 주기적으로 삭제하는 주기 (단위: 시간)            | `0` (비활성화)      | `> 0`으로 설정 시 자동 청소 활성화됨. 보통 `24` (1일 1회) 권장. 디스크 가득 참 방지 목적.                          |

## 설정시 주의사항

* `tickTime` 관련
  * 모든 시간 단위가 `tickTime`을 기준으로 계산됨.
  * ZK 내부 heartbeat, timeout의 기준이 되므로 과도하게 짧게 설정하면 불안정해질 수 있음.
* `initLimit` / `syncLimit`
  * 복제 네트워크 품질, 데이터 크기 등을 고려해 조정 필요.
  * 너무 낮게 설정하면 느린 follower들이 leader와 동기화 실패.
* `maxClientCnxns`
  * 클라이언트 수가 많은 환경에서는 늘려야 함. 보통 장애를 자주 유발하는 설정으로 생각보다 ZooKeeper에 연결하는 서비스가 많다는 것에 주의.
  * 프록시를 통한 연결이라면 특히 주의 필요.
* Session Timeout
  * 클라이언트는 session timeout을 서버에 요청하지만, `minSessionTimeout` ~ `maxSessionTimeout` 사이로 강제 조정됨.
  * 애플리케이션이 짧은 세션을 원해도 제한될 수 있음.
* autopurge 설정
  * 로그 파일이 누적되면 디스크가 가득 차서 Zookeeper가 죽을 수 있음.
  * `autopurge.snapRetainCount`와 함께 설정해 오래된 로그만 지우도록 조정 권장.

## Cloudera Manager

* Cloudera Manager를 통해 하나의 클러스터에 N개의 ZooKeeper 구성 가능
* 단, ZooKeeper를 의존하는 서비스를 재시작하는 경우 재시작 불가능 현상이 발생할 수 있으므로 하나의 클러스터에 N개의 ZooKeeper를 구성할 때에는 조심해야 함

## `zoo.cfg` 설정 예제

```
tickTime=2000
initLimit=10
syncLimit=5
4lw.commands.whitelist=conf,cons,crst,dirs,dump,envi,gtmk,ruok,stmk,srst,srvr,stat,wchs,mntr,isro
dataDir=/var/lib/zookeeper
dataLogDir=/var/lib/zookeeper
clientPort=2181
maxClientCnxns=60
minSessionTimeout=4000
maxSessionTimeout=60000
autopurge.purgeInterval=24
autopurge.snapRetainCount=5
quorum.auth.enableSasl=false
quorum.cnxn.threads.size=20
admin.enableServer=false
admin.serverPort=5181
server.1=hdm2.datalake.net:3181:4181
server.2=hdm3.datalake.net:3181:4181
server.3=hdm1.datalake.net:3181:4181
leaderServes=yes
```

## ZooKeeper CLi

| 명령어                            | 설명                                         |
| ------------------------------ | ------------------------------------------ |
| `zkCli.sh`                     | ZooKeeper CLI 실행 (`$ZK_HOME/bin/zkCli.sh`) |
| `zkCli.sh -server <host:port>` | 특정 ZooKeeper 서버에 CLI로 접속                   |
| `quit`                         | CLI 종료                                     |
| `ls /path`   | 해당 경로 하위의 znode 목록 조회                           |
| `ls2 /path`  | `ls` + znode의 메타정보 (ctime, mtime, version 등) 포함 |
| `get /path`  | znode의 데이터 값 조회                                 |
| `stat /path` | znode의 상태 정보(버전, 자식 노드 수 등) 조회                  |
| `create /path data`    | znode 생성 및 값 설정                      |
| `create -e /path data` | **ephemeral** znode 생성 (세션 종료 시 삭제됨) |
| `create -s /path data` | **sequential** znode 생성 (번호 자동 부여)   |
| `set /path data`       | 기존 znode의 값을 수정                      |
| `delete /path`         | znode 삭제                             |
| `deleteall /path`      | 하위 znode 포함하여 재귀적으로 삭제               |
| `get /path true` | 해당 znode 변경에 대한 watch 설정    |
| `ls /path true`  | 하위 znode 목록 변경에 대한 watch 설정 |

다음과 같이 CLI를 실행할 수 있음

```
# ZooKeeper CLI 접속
$ zkCli.sh -server localhost:2181

# znode 생성
[zk: localhost:2181] create /config "mydata"

# 데이터 조회
[zk: localhost:2181] get /config

# 데이터 수정
[zk: localhost:2181] set /config "newdata"

# 삭제
[zk: localhost:2181] delete /config
```
