# Atlas

## 의존 서비스

Atlas가 동작하기 위해서 필요한 서비스이며 Ranger의 TagSync를 적용하려면 Atlas가 필요함 (예; Tag 기반 Column Masking)

* HDFS
* HBase
* Solr
* Kafka

## 트러블 슈팅

### Admin 패스워드 변경

* CM 로그인 ＞ Atlas ＞ Configuration ＞ Admin Password ＞ 패스워드 변경
* CM 로그인 ＞ Atlas ＞ Configuration ＞ Enable File Authentication > ON
