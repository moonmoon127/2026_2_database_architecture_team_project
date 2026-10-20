#[시나리오2]

## 1. 프로젝트 개요
본 프로젝트는 100명 규모의 SI 업체(마케팅/경영 30명, 개발자 70명)가 월평균 10여 개의 프로젝트를 수행하는 환경을 지원하기 위한 **사내 업무용 웹 기반 시스템**입니다. 
단순한 데이터 저장을 넘어, E-R 모델링의 심화 개념(순환 관계의 Role, 관계의 속성, 참여 제약조건)을 엄격히 적용하였으며, **대용량 데이터 조회 시의 쿼리 처리(Query Processing) 비용 최소화 및 트랜잭션 무결성 보장**에 초점을 맞추어 설계되었습니다.

---

## 2. 심화 E-R 모델링 (Advanced E-R Modeling)
강의에서 다룬 개체-관계 모델의 핵심 이론을 다음과 같이 실제 비즈니스 룰에 적용했습니다.

### ① 개체 세트(Entity Set)와 관계 세트(Relationship Set)의 명확한 분리
*   **관계의 속성 (Attribute of Relationship):** 사원(Employee)과 프로젝트(Project) 사이의 다대다(N:M) 투입 관계를 해소하기 위해 `Project_Assignment` 릴레이션을 생성했습니다. 여기서 '투입 시작일', '종료일', '직무(PM, 테스터 등)'는 개체의 속성이 아니라 **두 개체가 만났을 때 비로소 발생하는 '관계의 속성(Attribute of Relationship)'**으로 올바르게 도식화했습니다.
*   **순환 관계와 역할 (Recursive Relationship & Roles):** 프로젝트 종료 후 진행되는 **'동료 평가' 및 'PM 평가'**는 동일한 `Employee` 엔티티 세트 내에서 발생합니다. 이를 위해 평가 관계선에 `evaluator_id(평가자)`와 `evaluatee_id(피평가자)`라는 **역할(Role)** 라벨을 부여하여 동일 엔티티 간의 멤버십(Membership) 관점을 명확히 했습니다.

### ② 카디널리티(Cardinality)와 참여 제약조건 (Participation Constraints)
*   **부분 참여 (Partial Participation - Default):** 사원(Employee)은 대기 발령 상태일 수 있으므로 프로젝트 투입 관계에 대해 '부분 참여'로 설계했습니다.
*   **전체 참여 (Total Participation):** 프로젝트 투입 이력(`Project_Assignment`)과 평가(`Evaluation`) 데이터는 반드시 부모 엔티티인 사원이 존재해야만 하므로, 사원 엔티티에 대해 '전체 참여' 제약을 갖도록 설계하여 고아(Orphan) 데이터가 발생하지 않도록 했습니다.

---

## 3. 정규화 및 무결성 제약조건 (Normalization & Integrity)

### ① 비식별자 관계(Non-Identifying Relationship)의 전략적 채택
*   `Project_Assignment` 테이블 생성 시, 사번(PK)과 프로젝트번호(PK)를 상속받아 복합키(Composite PK)로 사용할 수 있으나, 이는 자식 테이블로 갈수록 식별자가 무한 증식하는 문제를 낳습니다. 
*   이를 방지하기 위해 **비식별자 관계**를 채택하여 독립적인 `투입ID`를 기본키로 생성하고, 사번과 프로젝트번호는 일반 외래키(FK)로 두어 테이블 간의 결합도를 낮추고 쿼리의 복잡성을 줄였습니다.

### ② 무손실 조인 분해(Lossless-Join Decomposition)와 3NF/BCNF 타협
*   삽입/수정/삭제 이상(Anomaly)을 방지하기 위해 기본키에 직접 종속되지 않는 일반 속성들을 분리(비이행적 종속, Non-transitively dependent)하여 **제3정규형(3NF)**을 달성했습니다.
*   단, BCNF 정규화를 강제할 경우 기존의 **의존성 보존(Dependency Preservation)**이 깨져 잦은 조인(Join) 검사가 발생할 수 있는 구간에서는, 성능 오버헤드를 막기 위해 의도적으로 3NF 수준에서 타협하는 실무적 접근을 취했습니다.

---

## 4. 트랜잭션 관리와 쿼리 처리 최적화 (Transactions & Query Processing)

### ① 트랜잭션(Transaction) 원자성(Atomicity) 보장
*   PM이 고객사 평가와 프로젝트 팀원 평가를 동시에 입력할 때, 시스템 오류로 일부 데이터만 들어가는 것을 막기 위해 모든 평가 삽입(INSERT) 과정을 **하나의 트랜잭션**으로 묶어 처리합니다. 에러 발생 시 즉각 `ROLLBACK` 처리하여 데이터베이스의 일관성(Consistency)을 유지합니다. (※ DDL인 TRUNCATE, DROP 등은 Auto-commit 되어 롤백 대상이 아님을 시스템적으로 차단)

### ② UNIQUE 제약조건을 통한 인덱스 자동 생성 및 스캔 최적화
*   회원가입 시 ID 중복 체크 속도를 극대화하기 위해, 로그인 ID 컬럼에 **UNIQUE 제약조건**을 부여했습니다. 이로 인해 DBMS 내부적으로 **B-Tree 인덱스가 자동 생성**되어, 수만 건의 데이터가 쌓여도 Linear File Scan(전체 스캔)이 아닌 Index Scan을 통해 0.01초 내에 중복 여부를 판별합니다.

### ③ 경영진 대시보드 쿼리 최적화 (Join Algorithms & Pipelining)
경영진이 '특정 과거 시점의 직원 투입 현황 및 인센티브 추출 데이터'를 조회할 때 가장 무거운 쿼리가 발생합니다. (`Employee` ⋈ `Project_Assignment` ⋈ `Evaluation` 조인 연산)
*   **Indexed Nested-Loop Join 활용:** 가장 무식하고 비용이 비싼 Nested-Loop Join이나 Block Nested-Loop Join을 피하기 위해, 조인의 매개변수가 되는 FK(사번, 프로젝트번호)에 인덱스를 선제적으로 생성합니다. 이를 통해 Inner Relation의 전체 블록을 읽지 않고 목차(Index)만 찔러서 찾아오는 **Indexed Nested-Loop Join** 알고리즘이 동작하도록 최적화했습니다.
*   **중복 제거 최소화:** SQL에서 중복 제거(Duplicate elimination)는 내부적으로 무거운 External Sort-Merge를 유발하므로, 경영진 통계 추출 시 불필요한 `DISTINCT` 사용을 지양하고, 다중 행 서브쿼리(EXISTS, IN)를 적절히 튜닝하여 파이프라이닝(Pipelining) 방식으로 중간 결과가 바로바로 상위 연산에 전달되도록 설계했습니다.
*   **집계 함수 분석:** 프로젝트 참여 횟수 통계 시 중복 투입을 방지하기 위해 단순 `COUNT(project_id)`가 아닌 `COUNT(DISTINCT project_id)`를 명확히 구분하여 사용합니다.

---

## 5. 데모 시연 시나리오 (End-to-End Use Case Demo)

*   **Scene 1: 시스템 초기화 및 인덱스 동작 (System Admin)**
    *   가상의 사원 및 프로젝트 더미 데이터 10만 건을 CTAS(Create Table As Select) 방식으로 미리 Import.
    *   신규 직원 회원가입 시 UNIQUE 인덱스를 타서 중복 ID를 즉시 차단하는 모습 시연.
*   **Scene 2: 트랜잭션 기반 프로젝트 투입 및 평가 (PM View)**
    *   프로젝트 종료 후 평가 데이터를 입력하는 도중 의도적인 에러를 발생시켜 `ROLLBACK`이 정상 작동하는지 (일부 평가만 DB에 남는 이상 현상이 없는지) 시연.
*   **Scene 3: 경영진 대시보드 및 조인 성능 (Executive View)**
    *   복잡한 다중 조인(Join)과 집계 함수(Aggregation)가 포함된 경영진 통계 대시보드를 조회.
    *   실행 계획(Execution Plan)을 설명하며, 인덱스가 적용된 **Indexed Nested-Loop Join** 덕분에 디스크 접근(Disk Access) 비용이 획기적으로 줄어들어 대시보드가 빠르게 로딩됨을 증명.
