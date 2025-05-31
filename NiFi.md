# NiFi (Cloudera Flow Management)

## 설치시 가장 먼저 해야할 일

* JDK 교체 (JDK 1.8 사용 금지, 최소 JDK 11 사용; 이유는 GC)
* Event Thread 개수 상향 (10 -> 80)
* Heap Size -> 4~8G

## NiFi의 응답이 느려지는 경우 점검

* NiFi Logging Level (절대로 DEBUG 사용하지 말 것)
* NiFi의 Heap Size 부족인지 확인
* NiFi에서 Flow File이 대량 생성되는 것이 있는지
  * Partition 생성시 과도한 파티션 생성 (데이터에 따라서 파티션 생성이 달라지므로 대량 생성시 Content Repository Full 발생)
  * FlowFile 대량 생성시 Content Repository에 과도한 Small File 생성

## Sensitive Key

* Sensitive Key는 민감한 설정값(예: 비밀번호, API 키 등)을 암호화하기 위한 키
* 이 키는 `flow.xml.gz` 파일 등에서 민감 정보를 암호화/복호화할 때 사용
* 민감한 값을 평문으로 저장하지 않고 암호화하여 flow.xml.gz 파일에 저장
* 이 sensitive key가 변경이 되면 flow.xml 파일을 로딩할 수 없게 됨
  * flow.xml 파일을 변경이 없으나 NiFi를 재설치 했을 때 (NiFi는 최초 설치시 sensitive key를 램던하게 재생성)
  * 임의로 sensitive key를 Cloudera Manager에서 변경한 경우 (=Master Password)

### 관련 설정

`nifi.properties` 파일에 다음의 설정이 관련 설정임

```
nifi.sensitive.props.key=0123456789abcdef
nifi.sensitive.props.key.protected=      # (선택적, 암호화된 키)
nifi.sensitive.props.algorithm=NIFI_PBKDF2_AES_GCM_256
nifi.sensitive.props.provider=BC
nifi.sensitive.props.additional.keys=
```

| 설정 항목                                | 설명                                      |
| ------------------------------------ | --------------------------------------- |
| `nifi.sensitive.props.key`           | 암호화에 사용할 **키 문자열(16\~32자)**             |
| `nifi.sensitive.props.key.protected` | 위 키의 암호화 버전 (NiFi Toolkit에서 생성)         |
| `nifi.sensitive.props.algorithm`     | 암호화 알고리즘 (`NIFI_PBKDF2_AES_GCM_256` 권장) |
| `nifi.sensitive.props.provider`      | 암호화 라이브러리 제공자 (`BC` = BouncyCastle)     |

### 암호화된 키로 설정하는 방법 (NiFi Toolkit 사용)

```
# 키 생성 예
./encrypt-config.sh \
  --key 0123456789abcdef \
  --algorithm NIFI_PBKDF2_AES_GCM_256 \
  --encrypt \
  --outputKey

# 출력 결과:
# nifi.sensitive.props.key.protected=enc{AES_GCM:...}
```

### 주의사항

* Sensitive Key가 바뀌면 기존 `flow.xml.gz` 내 민감 정보는 복호화되지 않음
* 따라서 Key를 변경하면 `flow.xml.gz`도 함께 재암호화해야 함
* Key가 유출되면 민감정보가 노출될 수 있음 → 보안적으로는 .key 파일로 별도 보관하거나 시스템 환경 변수 등으로 관리하기도 함
* Cloudera CFM 같은 통합 배포판에서는 별도 key 관리 매커니즘이 있음 (예: `nifi.bootstrap.sensitive.key` 등).

### 암호화된 sensitive value 예시

`flow.xml` 파일에 다음과 같이 sensitive value가 보관

```
<controllerService>
  <id>ab2c93e0-0190-1000-0000-0000025132ac</id>
  <versionedComponentId>f8e83adf-971e-3d8a-9ec4-016e31caf05a</versionedComponentId>
  <name>PostgreSQL</name>
  <comment/>
  <bulletinLevel>WARN</bulletinLevel>
  <class>org.apache.nifi.dbcp.HikariCPConnectionPool</class>
  <bundle>
    <group>org.apache.nifi</group>
    <artifact>nifi-dbcp-service-nar</artifact>
    <version>1.26.0.2.1.7.0-435</version>
  </bundle>
  <enabled>true</enabled>
  <property>
    <name>hikaricp-connection-url</name>
    <value>jdbc:postgresql://db.datalake.net/test</value>
  </property>
  <property>
    <name>hikaricp-driver-classname</name>
    <value>org.postgresql.Driver</value>
  </property>
  <property>
    <name>hikaricp-driver-locations</name>
    <value>/usr/share/java/postgresql-42.7.3.jar</value>
  </property>
  <property>
    <name>hikaricp-kerberos-user-service</name>
  </property>
  <property>
    <name>hikaricp-username</name>
    <value>test</value>
  </property>
  <property>
    <name>hikaricp-password</name>
    <value>enc{601ab598ee5397a3a0fb291da6c1e5e2d7bc02111ee6fc97946d3efb9245597aa0a1aa9a9943e3f1d92aec6f}</value>
  </property>
  <property>
    <name>hikaricp-max-wait-time</name>
    <value>500 millis</value>
  </property>
  <property>
    <name>hikaricp-max-total-conns</name>
    <value>10</value>
  </property>
  <property>
    <name>hikaricp-validation-query</name>
  </property>
  <property>
    <name>hikaricp-min-idle-conns</name>
    <value>10</value>
  </property>
  <property>
    <name>hikaricp-max-conn-lifetime</name>
    <value>-1</value>
  </property>
</controllerService>
```

## 관련 에러 메시지

### `Decryption Failed with Algorithm [AES/GCM/NoPadding]`

`flow.xml` 파일의 인코딩 되어 있는 sensitive value를 decryption하지 못해서 생기는 현상으로 다음의 이유가 있음

* 임의로 sensitive key를 변경
* Sensitive가 설정되어 있는 NiFi를 제거하고 NiFi 재설치 및 기존 `flow.xml` 파일 그대로 사용

에러 메시지는 다음과 같음

```
Failed to start web server... shutting down.
org.apache.nifi.encrypt.EncryptionException: Decryption Failed with Algorithm [AES/GCM/NoPadding]
	at org.apache.nifi.encrypt.CipherPropertyEncryptor.decrypt(CipherPropertyEncryptor.java:78)
	at org.apache.nifi.registry.flow.diff.StandardFlowComparator.decrypt(StandardFlowComparator.java:317)
	at org.apache.nifi.registry.flow.diff.StandardFlowComparator.lambda$compareProperties$2(StandardFlowComparator.java:342)
	at java.base/java.util.LinkedHashMap.forEach(LinkedHashMap.java:684)
	at org.apache.nifi.registry.flow.diff.StandardFlowComparator.compareProperties(StandardFlowComparator.java:340)
	at org.apache.nifi.registry.flow.diff.StandardFlowComparator.compare(StandardFlowComparator.java:302)
	at org.apache.nifi.registry.flow.diff.StandardFlowComparator.lambda$compareComponents$0(StandardFlowComparator.java:117)
	at java.base/java.util.HashMap.forEach(HashMap.java:1337)
	at org.apache.nifi.registry.flow.diff.StandardFlowComparator.compareComponents(StandardFlowComparator.java:115)
	at org.apache.nifi.registry.flow.diff.StandardFlowComparator.compare(StandardFlowComparator.java:545)
	at org.apache.nifi.registry.flow.diff.StandardFlowComparator.compare(StandardFlowComparator.java:105)
	at org.apache.nifi.registry.flow.diff.StandardFlowComparator.compare(StandardFlowComparator.java:89)
	at org.apache.nifi.controller.serialization.VersionedFlowSynchronizer.compareFlows(VersionedFlowSynchronizer.java:467)
	at org.apache.nifi.controller.serialization.VersionedFlowSynchronizer.sync(VersionedFlowSynchronizer.java:176)
	at org.apache.nifi.controller.serialization.StandardFlowSynchronizer.sync(StandardFlowSynchronizer.java:42)
	at org.apache.nifi.controller.FlowController.synchronize(FlowController.java:1530)
	at org.apache.nifi.persistence.StandardFlowConfigurationDAO.load(StandardFlowConfigurationDAO.java:104)
	at org.apache.nifi.controller.StandardFlowService.loadFromBytes(StandardFlowService.java:817)
	at org.apache.nifi.controller.StandardFlowService.load(StandardFlowService.java:457)
	at org.apache.nifi.web.server.JettyServer.start(JettyServer.java:940)
	at org.apache.nifi.NiFi.<init>(NiFi.java:172)
	at org.apache.nifi.NiFi.<init>(NiFi.java:83)
	at org.apache.nifi.NiFi.main(NiFi.java:332)
Caused by: javax.crypto.AEADBadTagException: Tag mismatch!
	at java.base/com.sun.crypto.provider.GaloisCounterMode.decryptFinal(GaloisCounterMode.java:623)
	at java.base/com.sun.crypto.provider.CipherCore.finalNoPadding(CipherCore.java:1116)
	at java.base/com.sun.crypto.provider.CipherCore.fillOutputBuffer(CipherCore.java:1053)
	at java.base/com.sun.crypto.provider.CipherCore.doFinal(CipherCore.java:853)
	at java.base/com.sun.crypto.provider.AESCipher.engineDoFinal(AESCipher.java:446)
	at java.base/javax.crypto.Cipher.doFinal(Cipher.java:2200)
	at org.apache.nifi.encrypt.CipherPropertyEncryptor.decrypt(CipherPropertyEncryptor.java:74)
	... 22 common frames omitted
```
