---
title: OpenMetadata를 활용한 WAP DQ에 대한 고민
date: 2026-07-04 00:00:00 +0900
categories: [Data Engineering, OpenMetadata]
tags:
  - OpenMetadata
  - WAP
  - Data Quality
  - Iceberg
---

## 내가 생각한 이상적인 DQ 흐름

내가 처음 생각한 흐름은 크게 두 가지였다.

**1. 데이터 파이프라인 안에서 수행되는 DQ**

```text
source
  -> transfer
  -> 검증 branch에 write
  -> OpenMetadata의 DQ test case 조회 및 실행
  -> DQ 통과 시 검증 branch를 main branch에 merge
```

**2. 스케줄 기반의 주기적인 DQ**

```text
source of truth table
  -> OpenMetadata TestSuite 스케줄 실행
  -> main branch 기준 품질 모니터링
```

파이프라인 안의 WAP DQ는 잘못된 데이터가 publish되는 것을 막기 위한 목적이고, 주기적인 DQ는 파이프라인 외부에서 발생한 수기 작업이나 예상하지 못한 변경을 감지하기 위한 목적이다. 두 흐름은 목적이 다르기 때문에 함께 필요하다고 봤다.

---

## WAP 적용을 고민한 이유

**1. DQ 실행 시점 문제**

테이블 A에 데이터가 이미 publish된 뒤 DQ가 실행되면, DQ가 실패하더라도 그 사이에 다운스트림 잡이나 분석가, 마케터 같은 데이터 소비자는 잘못된 데이터를 보게 된다. 문제가 발견되었을 때는 이미 잘못된 데이터가 대시보드나 리포트에 반영된 뒤일 수 있다.

WAP은 이 문제를 줄일 수 있다. 데이터를 바로 main branch에 쓰지 않고 검증 branch에 먼저 쓴 뒤, DQ를 통과한 경우에만 main으로 publish하기 때문이다.

**2. Iceberg branch 기능 활용 가능**

사내 OLAP 환경이 Iceberg 기반이라는 점도 WAP을 고민하게 된 이유다. Iceberg는 branch 기능을 제공하기 때문에 WAP을 위한 격리된 write 영역을 만들기 상대적으로 쉽다.

**3. 증분 데이터만으로는 부족한 DQ**

증분 데이터에 대해서만 파이프라인 내부에서 DQ를 수행하는 방법도 생각해볼 수 있다. 하지만 이 방식만으로는 전체 테이블 관점의 품질이나 dimension/reference 데이터와의 정합성 검증이 부족할 수 있다. 예를 들어 새로 들어온 증분 데이터 자체는 문제가 없어 보여도, 전체 테이블 기준으로 보면 중복이나 참조 무결성 문제가 생길 수 있다.

**4. DQ 로직과 쿼리의 파편화 방지**

또 하나의 문제는 DQ 로직의 파편화다. 각 파이프라인마다 DQ 쿼리를 직접 들고 있으면, 시간이 지날수록 어떤 테이블에 어떤 품질 기준이 적용되는지 추적하기 어려워진다. 가능하다면 DQ 정의는 OpenMetadata에 모으고, 파이프라인은 OpenMetadata에 정의된 test case를 가져와 실행하는 구조가 더 낫다고 생각했다.

---

## OpenMetadata에서 DQ를 관리하고 싶은 이유

**1. 데이터 관리 지점을 OpenMetadata로 모으고 싶다**

OpenMetadata는 이미 데이터 리니지, 데이터 사전, 데이터 탐색에 사용하고 있다. 여기에 DQ까지 같은 도구에서 관리하면 데이터 자산에 대한 설명, 흐름, 품질 상태를 한 곳에서 볼 수 있다.

**2. UI에서 DQ를 쉽게 추가할 수 있다**

또한 OpenMetadata는 UI에서 DQ test case를 비교적 쉽게 추가할 수 있다. 데이터 엔지니어뿐 아니라 데이터 분석가 같은 데이터 소비자도 테이블에 필요한 품질 기준을 추가할 수 있다는 점이 장점이다. 품질 기준이 코드 안에만 있으면 엔지니어가 아니면 접근하기 어렵지만, UI에서 관리되면 협업 범위가 넓어진다.

**3. Table suite로 test case를 묶어 재사용할 수 있다**

TestSuite로 test case를 묶을 수 있다는 점도 중요하다. 특정 테이블의 TestSuite에 test case를 추가하면, 해당 테이블을 사용하는 파이프라인의 WAP 과정에서도 같은 품질 기준을 재사용할 수 있다. 내가 원했던 구조는 "OpenMetadata에 정의된 표준 test case를 파이프라인 실행 시점에 가져와 검증 branch에 대해 수행하는 것"이었다.

---

## 실제로 부딪힌 문제

**1. OpenMetadata의 DQ 실행 대상은 기본적으로 등록된 테이블이다**

문제는 OpenMetadata의 DQ 실행 대상이 기본적으로 OpenMetadata에 등록된 테이블이라는 점이다. OpenMetadata에 Iceberg 테이블을 Trino나 Athena 같은 쿼리 엔진을 통해 등록하면 main branch 기준의 테이블은 DQ 대상으로 삼을 수 있다.

하지만 WAP에서 검증하고 싶은 대상은 main branch가 아니라 WAP branch다. 즉, 같은 논리 테이블이라도 실제로는 다음처럼 특정 branch/ref를 바라봐야 한다.

```sql
SELECT *
FROM some_table FOR VERSION AS OF '<wap_branch>'
```

**2. OpenMetadata 1.12.6에서는 런타임 branch/ref 전달이 지원되지 않는다**

OpenMetadata 1.12.6 기준으로는 기존 TestSuite를 실행할 때 런타임에 Iceberg branch나 ref 이름을 넘겨서, 동일한 test case를 WAP branch에 대해 직접 실행하는 방식이 지원되지 않는다.

![OpenMetadata Slack 문의 답변 캡처](/assets/img/openmetadata-wap-dq/openmetadata-slack-inquiry.png)

*OpenMetadata Slack에 문의한 내용. OpenMetadata 1.12.6에서는 TestSuite 실행 시 런타임 Iceberg branch/ref를 넘겨 WAP branch에 직접 실행하는 방식이 지원되지 않는다는 답변을 받았다.*

**3. pandas dataframe 기반 DQ 실행은 Iceberg + WAP에는 맞지 않는다**

OpenMetadata의 DQ는 pandas dataframe 기반 실행도 가능하지만, 내가 고민하는 Iceberg + WAP 구조에는 잘 맞지 않는다. 이 경우에는 branch를 조회할 수 있는 query engine을 통해 DQ를 실행해야 한다.

결국 핵심 문제는 이것이다.

```text
OpenMetadata에는 main table 기준 test case가 정의되어 있다.
하지만 파이프라인에서는 같은 test case를 WAP branch 기준으로 실행하고 싶다.
```

---

## 현재 생각하는 해결 방향

현실적인 방향은 파이프라인에서 OpenMetadata의 test case 정의를 가져오고, 실제 실행은 파이프라인 쪽에서 수행하는 것이다.

예를 들면 Airflow DAG에서 다음과 같은 흐름을 만들 수 있다.

1. Iceberg WAP branch 생성
2. Spark 또는 SQL job으로 WAP branch에 데이터 write
3. OpenMetadata API로 대상 테이블의 TestSuite / TestCase 조회
4. test case의 SQL 또는 조건을 WAP branch를 바라보도록 변환
5. Trino, Spark, Athena 등에서 DQ 실행
6. 성공 시 Iceberg branch를 main에 publish
7. 실패 시 publish 중단 및 알림
8. 실행 결과를 OpenMetadata에 다시 기록

이 구조에서는 OpenMetadata가 DQ 정의와 결과 저장소 역할을 하고, 실제 WAP branch 대상 실행은 파이프라인이 담당한다. 이상적인 형태는 아니지만, 현재 기능 범위 안에서는 가장 현실적인 방식으로 보인다.

중요한 점은 DQ 결과를 다시 OpenMetadata에 남기는 것이다. 실행만 외부에서 하고 결과를 OpenMetadata에 기록하지 않으면, OpenMetadata를 중심으로 품질 상태를 관리하려는 목적이 약해진다. 따라서 외부 실행을 하더라도 최종 결과는 OpenMetadata의 test result나 incident 흐름과 연결하는 것이 좋다.

---

## 추가로 시도해볼 만한 것

**1. OpenMetadata에 branch/ref 런타임 파라미터 지원을 요청하거나 직접 기여해본다.**

장기적으로는 OpenMetadata 자체에서 TestSuite 실행 시 런타임 파라미터로 branch/ref를 받을 수 있으면 좋다.

내가 원하는 형태는 대략 이렇다.

```text
TestSuite: orders_quality_suite
Runtime target: orders FOR VERSION AS OF '<wap_branch>'
Result: OpenMetadata에 동일한 TestCase 결과로 저장
```

이렇게 되면 canonical test case는 OpenMetadata에 유지하면서도, 파이프라인 실행 시점에는 WAP branch를 대상으로 DQ를 수행할 수 있다. 현재는 직접 지원되지 않으므로 feature request를 올리거나, OpenMetadata 확장 포인트를 검토해볼 만하다.

---

## 유의사항

**1. Iceberg 쓰기 충돌 가능성이 커질 수 있다**

WAP을 적용하면 일반 write보다 Iceberg 쓰기 충돌 가능성이 조금 더 커질 수 있다. 일반적인 쓰기는 DQ를 먼저 하고 최종 write 시점의 충돌만 신경 쓰면 된다. 반면 WAP은 branch에 먼저 쓰고, DQ가 끝난 뒤 main으로 publish하는 과정이 추가된다.

이 사이에 main branch가 변경되면 merge나 fast-forward 과정에서 충돌이 발생할 수 있다. 따라서 WAP을 도입할 때는 DQ 실행 시간이 너무 길어지지 않도록 관리해야 하고, publish 실패 시 재시도나 branch 정리 정책도 함께 정해야 한다.

**2. 모든 DQ를 WAP 단계에서 수행하면 파이프라인이 무거워질 수 있다**

또한 모든 DQ를 WAP 단계에서 수행하려고 하면 파이프라인이 무거워질 수 있다. publish 전에 반드시 막아야 하는 핵심 테스트와, publish 이후 주기적으로 모니터링해도 되는 테스트를 구분하는 것이 필요하다.

---

## 정리

내가 원하는 구조는 OpenMetadata를 DQ의 중심 저장소로 두고, Iceberg WAP branch를 활용해 publish 전에 데이터를 검증하는 것이다.

정리하면 다음과 같다.

```text
OpenMetadata
  - test case 정의
  - test suite 관리
  - DQ 결과 저장
  - 품질 이력 추적

Pipeline / Airflow / Spark
  - WAP branch 생성
  - branch 대상 데이터 write
  - OpenMetadata test case 조회
  - branch 기준 DQ 실행
  - 성공 시 main publish
```

현재 OpenMetadata 1.12.6에서는 TestSuite 실행 시 런타임 Iceberg branch/ref를 넘기는 기능이 직접 지원되지 않는다. 그래서 당장은 파이프라인에서 test case를 가져와 외부에서 실행하고, 결과를 다시 OpenMetadata에 기록하는 방식이 현실적인 대안으로 보인다.

WAP DQ의 핵심은 DQ를 사후 알림이 아니라 publish gate로 사용하는 것이다. 잘못된 데이터가 main에 올라간 뒤 발견하는 것보다, 애초에 main으로 올라가지 못하게 막는 것이 더 안정적인 데이터 플랫폼에 가깝다.


<추후에 예시 코드들도 같이 추가하기>
