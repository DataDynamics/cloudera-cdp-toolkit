# Ranger

## 주요 컴포넌트

* Ranger Admin
* Ranger UserSync
* Ranger TagSync

### TagSync

#### 주요 역할

* 태그 메타데이터 수집
  * Apache Atlas 같은 데이터 분류 도구에서 **엔티티(예: 테이블, 컬럼)**에 연결된 태그 또는 분류 정보를 주기적으로 가져옵니다.
* Ranger에 태그 전송
  * 수집한 태그 정보를 Ranger Admin 서비스로 전송하여, 해당 리소스와 연결된 태그를 Ranger가 인지할 수 있도록 합니다.
* 정책과 연결
  * Ranger는 이 태그 정보를 기반으로 태그 기반 접근제어(Tag-Based Access Control) 정책을 실행할 수 있습니다.
  * 예: PII 태그가 붙은 컬럼은 특정 그룹만 접근 허용.
* 동기화 유지
리소스에 태그가 변경되거나 삭제되면, tagsync는 이를 탐지하여 Ranger에 갱신 정보를 반영합니다.

#### 필요한 이유

| 기능            | 설명                                        |
| ------------- | ----------------------------------------- |
| 분류 기반 접근 제어   | 민감 정보(PHI, PII 등)에 태그를 붙이고 그 태그에 따라 접근 제어 |
| 데이터 분류 시스템 통합 | Apache Atlas 등과 Ranger를 연결                |
| 자동화된 정책 유지    | 리소스 변경 시 수동 정책 수정 없이 자동 반영                |

#### Atlas에서 Tag 등록 절차

* Atlas UI 접속
  * http://<atlas-host>:21000
* Left Menu > Classifications > +Add Classification
  * Classification 이름 입력: 예) PII
  * Attribute를 추가하고 싶다면 Key/Value 속성도 설정 가능
* 저장 후 생성된 Classification을 엔티티에 부여
  * Entities > 원하는 테이블/컬럼 선택
  * 우측 상단 "Add Classification" 클릭
  * PII 또는 원하는 태그 선택 후 적용

#### Ranger Column Masking 적용

##### Apache Atlas에서 해야 할 일

| 단계                      | 설명                                                                              |
| ----------------------- | ------------------------------------------------------------------------------- |
| ① 태그(Classification) 정의 | 예: `PII`, `SENSITIVE`, `MASKED_SSN` 같은 이름으로 Classification 생성                   |
| ② 태그 대상 지정              | Hive 테이블의 **컬럼**에 태그 부여 (`users.ssn` 컬럼에 `MASKED_SSN` 태그 등)                     |
| ③ 대상 엔티티 생성 확인          | Hive Hook이 Atlas에 테이블과 컬럼 엔티티를 자동 등록하는지 확인<br>(Atlas에서 Hive Hook이 정상 동작 중이어야 함) |
| 단계                           | 설명                                                                                                                                                                         |

##### Ranger에서 해야 할 일

| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ① TagSync 설정 확인              | tagsync 서비스가 Atlas에서 태그를 Ranger로 동기화하도록 설정되어 있어야 함 (`tagsync-site.xml`에서 Atlas source 활성화)                                                                                 |
| ② 서비스 정의 확인                  | Ranger Admin에 Hive/Impala 등 태그 기반 정책을 설정할 수 있는 서비스가 등록되어 있어야 함 (`tag` 기반 policy service 존재 여부 확인)                                                                          |
| ③ 태그 기반 Column Masking 정책 생성 | - Resource type: `tag`<br>- Tag: `MASKED_SSN`<br>- Access type: `select`<br>- Masking Type: `Partial Mask`, `Custom Mask`, `Nullify`, etc.<br>- Users/Groups: 마스킹 적용 대상 지정 |
| ④ 정책 저장 및 배포                 | 저장 후 플러그인에 적용됨. 해당 컬럼에 접근 시 자동으로 마스킹됨                                                                                                                                      |
##### 예시 시나리오

* Atlas에서 MASKED_SSN 태그 생성
* Hive 테이블 users의 ssn 컬럼에 MASKED_SSN 태그 부여
* tagsync가 이 태그 정보를 Ranger로 전송
* Ranger Admin에서 MASKED_SSN 태그에 대해 Column Masking 정책 생성
* data_analyst 그룹 사용자가 SELECT ssn FROM users 실행 시 마스킹된 값 반환
