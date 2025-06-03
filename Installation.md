# Installation

## 사전 준비 사항

| 구분               | 예시                       |
|--------------------|---------------------------|
| OS 버전            | RHEL 9.4                  |
| NTP 서버 주소      | 10.0.1.10                  |
| DNS 서버 주소      | 10.0.1.1                   |
| AD 서버 주소       | 10.0.1.2                   |
| OS YUM Repository | /etc/yum.repos.d/rhel.repo |

https://supportmatrix.cloudera.com 에서 설치할 Cloudera CDP의 버전에 따라서 OS 및 Database 등의 설치 요건을 확인하도록 함

| 구분               | 예시                       | 확인 위치                               |
|--------------------|---------------------------|-----------------------------------------|
| Python 버전       | Python 3.9                  | Cloudera Documentation (Installation)  |
| Database 버전     | PostgreSQL 16               | Cloudera Support Matrix                |
| Cloudera Manager  | 7.11.0                      | Cloudera Support Matrix                |
| Cloudera CDP      | 7.1.9.SP1                   | Cloudera Support Matrix                |
| Cloudera CDS 3    | 3.1                         | Cloudera Support Matrix                |
| Cloudera CFM      | 2.1                         | Cloudera Support Matrix                |
| JDK               | JDK 17                      | Cloudera Support Matrix                |

* PostgreSQL : https://www.postgresql.org/download/
* MariaDB : https://mariadb.org/download/
* JDK : https://adoptium.net/temurin/releases/?os=any&arch=any&version=21

RHEL 7/8/9의 내장 Python과 JDK 버전

| RHEL 버전    | 기본 Python 버전                         | 기본 JDK 버전           | 비고                                       |
| ---------- | ------------------------------------ | ------------------- | ---------------------------------------- |
| **RHEL 7** | Python 2.7                           | OpenJDK 8 (`1.8.0`) | Python 3는 기본 미포함, 수동 설치 필요               |
| **RHEL 8** | Python 3.6 (기본),<br>Python 2.7 (호환용) | OpenJDK 11          | Python 3.8 및 JDK 8/17도 AppStream으로 설치 가능 |
| **RHEL 9** | Python 3.9                           | OpenJDK 17          | Python 3.11 및 JDK 11/21 선택 설치 가능         |


### Active Directory

| 구분           |         |
|---------------|----------|
| LDAP URL       | ldaps://10.0.1.2 |
| Bind DN        | |
| Bind Password | |
| Realm         | DATALAKE.NET |
| User OU       | |
| Group OU      | |

### `/etc/hosts` 파일

FQDN이 먼저, 호스트명이 뒤에 오도록 작성하고 모든 노드에 동기화

```
10.0.1.10 ntp.datalake.net ntp
10.0.1.1 ns1.datalake.net ns1
1.0.1.2 ad1.datalake.net ad1
```

## 기타

* TLS와 Kerberos는 둘 중에 하나만 적용할 수 있음
* TLS를 OFF하는경우 CM Server, CM Agent는 별도로 조치가 필요함(CM Agent의 config.ini 파일을 수정)
* LDAP 인증과 Kerberos 인증을 모두 적용할 수 있음
