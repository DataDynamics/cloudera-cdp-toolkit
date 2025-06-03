# Time Synchronization

## Cloudera CDP에서 Raft를 사용하는 Runtime Component

* Cloudera CDP의 Runtime Component 중 Raft Algorithm을 기반으로 동작하는 컴포넌트가 존재함
* Raft Algorithm을 사용하는 컴포넌트는 시간 동기화가 매우 중요하므로 이에 대해서 기술적 이해가 필요함

| 컴포넌트         | Raft 사용 여부   | 설명                                 |
| --------------- | --------------- | ------------------------------------ |
| **Apache Kudu** | ✅ Yes         | Raft 기반 Tablet 복제 및 리더 선출     |
| Apache HDFS     | ❌ No          | Raft 미사용, Zookeeper + NameNode     |
| Apache Hive     | ❌ No          | RDB 기반, consensus 없음              |
| Apache Impala   | ❌ No          | Catalog 서비스 내장, Raft 미사용       |
| Apache Kafka    | ✅ Yes         | ZooKeeper, Raft 선택 가능             |
| Apache Ozone    | ✅ (선택적)     | Apache Ratis 기반 Raft 사용          |

추가로 Kerberos 또한 시간이 중요함

## Raft Algorithm과 시간 동기화

* Raft 알고리즘에서 시간 동기화가 중요한 이유는 리더 선출 및 로그 복제 과정에서 "시간 기반 타이머"에 의존하기 때문임.
* 시간 동기화가 부정확하거나 불균형하면 잘못된 리더 선출, 불필요한 선거, 클러스터 불안정 등의 문제가 발생할 수 있음.

### Raft에서 "시간"이 사용되는 주요 지점

| 사용 목적                  | 설명                                                                 |
| ---------------------- | ------------------------------------------------------------------ |
| **Election Timeout**   | Follower가 일정 시간 동안 리더의 heartbeat를 못 받으면 후보(Candidate)로 전환하여 선거를 시작 |
| **Heartbeat Interval** | 리더가 주기적으로 Follower에게 heartbeat를 보내서 자신이 아직 리더임을 알림                 |
| **RPC Timeout**        | 리더나 후보가 다른 노드와 통신할 때 응답을 기다리는 시간                                   |

### 시간 동기화가 중요한 이유

* Election Storm (선거 폭풍) 방지
  * Election Timeout이 노드마다 너무 비슷하거나 동기화가 안 되면, 여러 노드가 동시에 후보가 되며 split vote가 발생
  * 지속적으로 election이 반복되어 리더가 안정적으로 선출되지 않음
  * 해결책: 각 노드는 Election Timeout을 무작위(Randomized) 간격으로 설정하지만, 전체 시스템의 시간 지연(jitter)이 크면 효과가 약화됨
* 리더 임기 안정성
  * 리더가 heartbeat를 보낼 때 follower들이 정확한 시간 내에 heartbeat를 받아야 "리더가 살아 있다"고 판단
  * 시간 불일치가 있으면 follower가 리더를 잘못 죽은 것으로 판단하고 선거를 시작함
  * 해결책: 리더와 follower 간 네트워크 지연 및 시스템 시간차가 작아야 함
* Log Replication Timeout 및 데이터 불일치
  * 리더가 로그를 보내고 follower가 응답하는 데 걸리는 시간 기준으로 timeout이 발생
  * 시스템 시간이 흐트러져 있으면 불필요한 재전송, 충돌, 심하면 데이터 손실 위험

### Raft는 절대적인 "시계 동기화"를 요구하지 않음

* Raft는 논리적인 타이머를 기반으로 동작하지, 노드 간에 UTC 동기화가 정확히 일치할 필요는 없음.
* 하지만 **상대적인 타이밍(응답 속도, 타이머 정확성)**이 중요.
  * 타이머가 너무 빨리 또는 느리게 작동하면 오작동 발생.

### 결론

* Raft는 정확한 절대 시계가 아닌, 일관된 상대 타이머에 의존함
* 하지만 각 노드의 타이머가 너무 느리거나 빨라지면 리더 선출 실패, 불필요한 선거, 로그 복제 오류 등을 유발하므로 시스템 시간은 정기적으로 동기화되어야 하고, 시간 지연(jitter)은 최소화되어야 함

| 설정 또는 환경 요소      | 권장 사항                          |
| ---------------- | ------------------------------ |
| 시스템 시계           | `chrony` 등으로 NTP 기반 정기 동기화 유지  |
| Election Timeout | 무작위 간격 (ex: 150ms \~ 300ms 범위) |
| Clock Drift      | 초당 <10ms 정도 오차 이하로 유지          |
| 시스템 부하           | GC나 디스크 IO 과부하로 타이머 지연 방지      |

## NTP & Chrony 비교

### 기능적 비교

| 항목    | `chrony`                       | `ntpd` (NTP)                   |
| ----- | ------------------------------ | ------------------------------ |
| 주요 데몬 | `chronyd`                      | `ntpd`                         |
| 프로토콜  | NTP (Network Time Protocol) 사용 | NTP (Network Time Protocol) 사용 |
| 등장 시기 | 2010년 이후 (경량, 정확도 향상 목적)       | 1980년대 후반 (오래된 구현체)            |
| 정확도   | 더 빠른 동기화, 더 높은 정확도             | 상대적으로 느림                       |
| 패키지명  | `chrony`                       | `ntp` 또는 `ntpdate`             |

### 기술적 비교

| 항목                | `chrony`                | `ntpd`             |
| ----------------- | ----------------------- | ------------------ |
| 동기화 속도            | 매우 빠르게 시간 동기화           | 느리게 점진적으로 동기화      |
| 배터리 없는 장비 (IoT 등) | 부팅 후 빠르게 시간 복원 가능       | 부팅 후 시간이 어긋날 수 있음  |
| 오프라인 동기화          | 로컬 RTC(하드웨어 시계)와의 연동 우수 | RTC와의 정확한 동기화가 어려움 |
| 네트워크 지연 보정        | 빠르게 반영                  | 보정이 느림             |
| 시간 서버로의 사용        | 가능                      | 가능                 |
| Leap Second 보정    | 자동 반영 가능                | 반영에는 설정이 필요        |

### 설정 파일 비교

| 항목       | chrony                                | ntpd                            |
| -------- | ------------------------------------- | ------------------------------- |
| 기본 설정 파일 | `/etc/chrony.conf`                    | `/etc/ntp.conf`                 |
| 서버 설정    | `server time.bora.net iburst` 등       | `server time.bora.net iburst` 등 |
| 상태 확인    | `chronyc tracking`, `chronyc sources` | `ntpq -p`, `ntpstat`            |

`iburst` 옵션은 시간 동기화를 빠르게 시작하기 위한 초기 가속화 옵션으로 NTP 시작시 1초 간격으로 천천히 요청하는 대신 짧은 시간 안에 여러번 요청하는 방식으로 동기화를 수행하며, 동기화 속도도 수초~10초 이내로 빠르게 동기화를 수행

### OS 버전별 비교

| OS            | 기본 시간 동기화 도구                              |
| ------------- | ----------------------------------------- |
| RHEL 7        | `chrony`                                  |
| RHEL 8/9      | `chrony`                                  |
| Ubuntu 18.04+ | `systemd-timesyncd` (기본), `chrony`도 자주 사용 |
| Debian        | `ntp` 또는 `chrony` 선택 가능                   |

### 요약

| 용도                        | 추천 도구                        |
| ------------------------- | ---------------------------- |
| 최신 시스템, 빠른 시간 보정 필요       | ✅ **chrony**                 |
| 기존 레거시 시스템, 오랜 운용 경험      | ❗ **ntpd** (단, 유지 중단 가능성 있음) |
| 서버가 자주 재부팅되거나 오프라인 주기적 발생 | ✅ **chrony**                 |

## Chrony 설치

### 패키지 설치

```
dnf install -y chrony
systemctl enable --now chronyd
systemctl status chronyd
```

### Chrony 설정

`/etc/chrony.conf` 설정 파일을 다음과 같이 수정하도록 함

```
# 시간 동기화할 NTP 서버 설정
server time1.kisa.re.kr iburst
server time2.kornet.net iburst

# 시스템 클럭이 너무 틀렸을 때 바로 동기화
makestep 1.0 3

# 로컬에 시간 제공 허용 (선택적)
allow 192.168.0.0/16

# 로컬 RTC (하드웨어 클럭) 기록
driftfile /var/lib/chrony/drift

# 로그 디렉토리
logdir /var/log/chrony
```

| 옵션          | 설명                                           |
| ----------- | -------------------------------------------- |
| `server`    | 동기화할 NTP 서버. `iburst` 옵션으로 빠른 초기 동기화         |
| `makestep`  | 클럭이 많이 틀렸을 경우 NTP time으로 "즉시 조정" (보통 부팅 초기용) |
| `allow`     | 다른 클라이언트가 이 서버를 NTP 서버로 사용하도록 허용             |
| `driftfile` | 로컬 하드웨어 클럭의 오차 저장                            |
| `logdir`    | 로그 저장 위치                                     |

### 방화벽 개방

```
sudo firewall-cmd --add-service=ntp --permanent
sudo firewall-cmd --reload
```

### 수동 동기화

```
chronyc makestep
```

특정 서버와 강제 동기화

```
chronyd -q 'server time.bora.net iburst'
```

### 동기화 상태 확인

```
chronyc tracking       # 현재 시간 오차, 지연 등 확인
chronyc sources -v     # 동기화 중인 서버와 응답 상태 확인

MS Name/IP address     Stratum Poll Reach LastRx Last sample
===============================================================================
^* time1.kisa.re.kr          2   6   377   15   -0.123 +0.004  0.002
```

### 하드웨어 클럭(HW clock)도 동기화

```
hwclock --systohc
```

## 동기화 에러 확인 방법

### `/var/log/messages`의 동기화 관련 에러 메시지

| 메시지                                                               | 의미 및 원인                                             |
| ----------------------------------------------------------------- | --------------------------------------------------- |
| `chronyd[PID]: Can't synchronise: no reachable sources`           | ❗ **NTP 서버에 도달할 수 없음** (네트워크 차단, DNS 오류, 서버 다운 등)   |
| `chronyd[PID]: Source X.X.X.X unreachable`                        | 해당 NTP 서버로부터 응답이 오지 않음 (IP 또는 방화벽 문제)               |
| `chronyd[PID]: Selected source X.X.X.X appears to be falseticker` | 서버가 **부정확한 시간**을 제공 중                               |
| `chronyd[PID]: Can't synchronise: clock stepped too far`          | 시스템 시간이 서버와 너무 차이 나서 **자동 동기화 포기** (makestep 설정 필요) |
| `chronyd[PID]: System clock wrong by N seconds`                   | 현재 시간 오차가 N초 (makestep 조건 초과)                       |
| `chronyd[PID]: NTP packet received from X.X.X.X is invalid`       | NTP 응답 패킷이 잘못되었거나 조작된 것                             |
| `chronyd[PID]: Name or service not known`                         | DNS 문제로 NTP 호스트 이름을 해석 못 함                          |
| `chronyd[PID]: Could not resolve hostname time.example.com`       | DNS 미설정, `resolv.conf` 오작동 등                        |
| `chronyd[PID]: System clock was stepped by N seconds`             | 시간 오차가 너무 커서 **강제 시간 보정 수행됨**                       |

### 로그로 확인이 안되는 경우

* rsyslog가 꺼져있거나, `/var/log/messages`가 비활성화시 로그 확인 불가
  * `systemctl status rsyslog` 로 rsyslog 동작 확인 --> `systemctl enable --now rsyslog`
  * `vi /etc/rsyslog.conf ` 또는 `vi /etc/rsyslog.d/50-default.conf`
    * `*.info;mail.none;authpriv.none;cron.none                /var/log/messages` 포함되어야 함
* journald로만 기록되는 경우 `journalctl -u chronyd` 커맨드 사용

