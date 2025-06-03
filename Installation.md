# Installation

## 사전 준비 사항

| 구분               | 예시                       |
|--------------------|---------------------------|
| OS 버전            | RHEL 9.4                  |
| NTP 서버 주소      | 10.0.1.10                  |
| DNS 서버 주소      | 10.0.1.1                   |
| AD 서버 주소       | 10.0.1.2                   |
| OS YUM Repository | /etc/yum.repos.d/rhel.repo |
| Python 버전       | Python 3.9                  |
| Database 버전     | PostgreSQL 16               |
| Cloudera Manager  | 7.11.0                      |
| Cloudera CDP      | 7.1.9.SP1                   |
| Cloudera CDS 3    | 3.1                         |
| Cloudera CFM      | 2.1                         |   

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
