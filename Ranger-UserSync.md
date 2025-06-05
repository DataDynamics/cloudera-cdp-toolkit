# Ranger UserSync

## LDAP과 Sync 순서

Ranger UserSync는 다음과 같은 LDAP 기반 정보를 가져옴

* 사용자(user) 객체 → Ranger에 사용자로 등록
* 그룹(group) 객체 → Ranger에 그룹으로 등록
* 그룹 내 사용자(member) 관계 → Ranger에서 user ↔ group 매핑

기본적으로 Group에서 사용자(`member` 속성) 를 읽는 방식으로 작동하며, 필요 시 `memberOf` 방식도 일부 지원함

## UserSync의 Source

| 모드       | 설명                     | 권장 환경         |
| -------- | ---------------------- | ------------- |
| **LDAP** | OpenLDAP 등 표준 LDAP 서버  | OpenLDAP      |
| **AD**   | Active Directory 전용 설정 | Microsoft AD  |
| **Unix** | /etc/passwd 기반         | 로컬 테스트용       |
| **File** | 수동 파일로 동기화             | 테스트 또는 특수 환경용 |

## `ranger-ugsync-site.xml` 파일의 설정

```xml
<property>
  <name>ranger.usersync.source.impl.class</name>
  <value>org.apache.ranger.ldapusersync.process.LdapUserGroupBuilder</value>
</property>

<property>
  <name>ranger.usersync.ldap.url</name>
  <value>ldaps://ad.example.com:636</value>
</property>

<property>
  <name>ranger.usersync.ldap.binddn</name>
  <value>CN=ldap-sync,OU=ServiceAccounts,DC=example,DC=com</value>
</property>

<property>
  <name>ranger.usersync.ldap.credential.file</name>
  <value>/etc/ranger/usersync/conf/cred.jceks</value>
</property>

<property>
  <name>ranger.usersync.ldap.user.searchbase</name>
  <value>OU=Users,DC=example,DC=com</value>
</property>

<property>
  <name>ranger.usersync.ldap.user.objectclass</name>
  <value>user</value>
</property>

<property>
  <name>ranger.usersync.ldap.user.nameattribute</name>
  <value>sAMAccountName</value>
</property>

<property>
  <name>ranger.usersync.ldap.group.searchbase</name>
  <value>OU=Groups,DC=example,DC=com</value>
</property>

<property>
  <name>ranger.usersync.ldap.group.objectclass</name>
  <value>group</value>
</property>

<property>
  <name>ranger.usersync.ldap.group.nameattribute</name>
  <value>cn</value>
</property>

<property>
  <name>ranger.usersync.ldap.group.memberattributename</name>
  <value>member</value> <!-- group 객체에 포함된 사용자 DN -->
</property>

<property>
  <name>ranger.usersync.group.searchenabled</name>
  <value>true</value>
</property>
```
`memberOf`를 사용하는 경우 설정은 다음과 같음
`memberOf`는 AD에서는 자동 생성, OpenLDAP에서는 overlay 필요

```xml
<property>
  <name>ranger.usersync.ldap.group.memberattributename</name>
  <value>memberOf</value>
</property>
<property>
  <name>ranger.usersync.ldap.groupname.caseconversion</name>
  <value>lower</value>
</property>
```
