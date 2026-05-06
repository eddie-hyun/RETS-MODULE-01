# RETS 요구사항 스키마 설명서

> 파일: `req_schema_v3.json` (v3.0.0)
> 본 문서는 RETS(Requirements Engineering Tool Suite) 프로젝트에서 사용하는 요구사항 데이터 모델의 전체 구조와 각 필드의 의미를 정의한다.

---

## 1. 개요

RETS의 요구사항 데이터는 **두 가지 문서 유형**으로 구분된다.

| 구분 | 설명 | `req_type` 값 | ID 예시 |
|------|------|--------------|---------|
| **RFP 요구사항** | 발주처 RFP 문서에서 도출된 원시 요구사항. 수용 여부 검토의 대상이 됨. | `"RFP"` | `RFP_001`, `RFP_001_01_03` |
| **개발 요구사항** | RFP 요구사항을 분해·구체화하여 개발 단위로 정제한 요구사항. 기능(FR)/비기능(NFR)을 ID에 명시. | `"DEV"` | `DEV_FR_317`, `DEV_NFR_023_11_04` |

두 유형은 **공통 필드**를 공유하며, 각 유형에만 적용되는 **전용 필드**를 추가로 가진다.

---

## 2. 필드 구분 요약

### 2-1. 공통 필드 (RFP + DEV)

| 필드명 | 타입 | 필수 | 설명 |
|--------|------|------|------|
| `req_type` | enum | ✅ | RFP/DEV 구분 |
| `req_id` | string | ✅ | 요구사항 ID (채번 규칙 → 3절 참고) |
| `req_version` | string | ✅ | 요구사항 버전 (예: `1.0`, `1.2`) |
| `req_level` | integer (1~5) | ✅ | 요구사항 수준 — req_id 계층 깊이와 일치 |
| `req_name` | string | ✅ | 요구사항명 |
| `functional_class` | enum | ✅ | 기능/비기능 구분 |
| `req_category` | enum | ✅ | 요구사항 유형 |
| `domain_area` | enum | — | 도메인 영역 (참조·분류 용도) |
| `description` | string | ✅ | 요구사항 설명 (상세) |
| `acceptance_criteria` | string[] | — | 완료 기준 목록 |
| `verification_method` | enum | — | 검증 방법 |
| `metrics` | string[] | — | 측정 지표 목록 |
| `priority` | enum | — | 우선순위 (MoSCoW) |
| `importance` | enum | — | 중요도 |
| `stakeholders` | string[] | — | 연관 이해관계자 |
| `created_by` | string | ✅ | 발의자 |
| `last_modified_date` | string (date) | ✅ | 최종수정일 |
| `req_status` | enum | ✅ | 요구사항 상태 |
| `manager` | string | — | 관리 담당자 |
| `reviewer` | string | — | 검토자 |
| `change_history` | ChangeHistoryEntry[] | — | 변경이력 |

### 2-2. RFP 전용 필드

| 필드명 | 타입 | 필수 | 설명 |
|--------|------|------|------|
| `acceptance_status` | enum | ✅ | 수용여부 — RFP 기준 수용 검토 결과 |

### 2-3. DEV 전용 필드

| 필드명 | 타입 | 필수 | 설명 |
|--------|------|------|------|
| `io_spec` | IOSpec | — | 입력/출력 명세 |
| `story_points` | integer | — | 스토리포인트 |
| `estimated_effort` | number (MD) | — | 예상공수 (Man-Day) |
| `source_rfp_ids` | string[] | — | 근거 RFP 요구사항 ID 목록 |
| `predecessor_dev_ids` | string[] | — | 선행 개발요구사항 ID 목록 |
| `dev_status` | enum | — | 개발 진행 상태 |
| `developer` | string | — | 개발 담당자 |

---

## 3. 요구사항 ID 채번 규칙

### 3-1. 형식

```
{PREFIX}_{SEQ}[_{L2}[_{L3}[_{L4}[_{L5}]]]]
```

| 구성요소 | 설명 | 형식 |
|----------|------|------|
| `PREFIX` | 요구사항 유형 식별자 | `RFP`, `DEV_FR`, `DEV_NFR` |
| `SEQ` | 3자리 최상위 일련번호 | `001` ~ `999` |
| `L2`~`L5` | 각 하위 계층의 2자리 번호 | `01` ~ `99` |
| 구분자 | 모든 구성요소 사이 | 언더스코어(`_`) 고정 |

### 3-2. ★ 계층 깊이(Depth) = req_level 일치 규칙

`req_id`의 계층 깊이는 `req_level` 값과 **반드시 일치**해야 한다.

| req_level | ID 구조 | RFP 예시 | DEV FR 예시 | DEV NFR 예시 |
|-----------|---------|----------|------------|-------------|
| 1 | `{PREFIX}_{SEQ}` | `RFP_001` | `DEV_FR_317` | `DEV_NFR_023` |
| 2 | `{PREFIX}_{SEQ}_{L2}` | `RFP_001_01` | `DEV_FR_317_01` | `DEV_NFR_023_11` |
| 3 | `{PREFIX}_{SEQ}_{L2}_{L3}` | `RFP_001_01_03` | `DEV_FR_317_01_03` | `DEV_NFR_023_11_04` |
| 4 | `{PREFIX}_{SEQ}_{L2}_{L3}_{L4}` | `RFP_001_01_03_11` | `DEV_FR_317_01_03_11` | `DEV_NFR_023_11_04_02` |
| 5 | `{PREFIX}_{SEQ}_{L2}_{L3}_{L4}_{L5}` | `RFP_001_01_03_11_07` | `DEV_FR_317_01_03_11_07` | `DEV_NFR_023_11_04_02_01` |

> 계층 깊이 = SEQ 이후 등장하는 `_XX` 구성요소의 수 = `req_level - 1`

### 3-3. 정규식 패턴

| 대상 | 패턴 |
|------|------|
| 전체 통합 | `^(RFP\|DEV_(FR\|NFR))_\d{3}(_\d{2}){0,4}$` |
| RFP 전용 | `^RFP_\d{3}(_\d{2}){0,4}$` |
| DEV 전용 | `^DEV_(FR\|NFR)_\d{3}(_\d{2}){0,4}$` |

> `{0,4}` = `_XX` 구성요소가 0개(level 1)에서 4개(level 5)까지 허용됨을 의미한다.
> depth↔level 일치 규칙은 정규식으로 표현이 불가하므로 **애플리케이션 레벨에서 별도 검증**해야 한다.

### 3-4. 부모-자식 관계

하위 ID는 상위 ID를 완전히 포함한다. 부모 ID는 자식 ID에서 마지막 `_XX` 구성요소를 제거하면 도출된다.

```
DEV_FR_317                  (level 1 — 최상위)
└── DEV_FR_317_01           (level 2)
    └── DEV_FR_317_01_03    (level 3)
        └── DEV_FR_317_01_03_11     (level 4)
            └── DEV_FR_317_01_03_11_07  (level 5 — 최하위)
```

### 3-5. 채번 원칙

- `SEQ`는 유형(`RFP`, `DEV_FR`, `DEV_NFR`) 내에서 각각 **독립 채번**한다.
- 각 계층의 번호(`L2`~`L5`)는 **동일 부모 아래에서 독립 채번**한다.
- 결번이 발생하더라도 기존 번호를 **재사용하지 않는다**.
- DEV 요구사항의 `FR`/`NFR` 구분은 `functional_class` 필드(`기능`/`비기능`)와 **반드시 일치**해야 한다.
- RFP 요구사항 ID에는 FR/NFR 구분자를 사용하지 않는다.

---

## 4. Enum 값 상세 정의

### 4-1. `req_level` — 요구사항 수준

일반 프로젝트(전통적 개발 방법론)와 애자일 프로젝트에서의 수준 명칭을 함께 정의한다.

| 값 | 일반 프로젝트 수준명 | 애자일 수준명 | ID 계층 깊이 | 설명 |
|----|-------------------|------------|------------|------|
| `1` | 업무영역 | Epic | 0 (`SEQ`만 존재) | 최상위 업무 도메인 또는 목표 영역. 여러 기능집합을 포괄한다. |
| `2` | 기능집합 | Feature | 1 (`SEQ_L2`) | 관련 기능들의 그룹. 하나의 업무영역에서 파생된 기능 묶음. |
| `3` | 기능 | User Story | 2 (`SEQ_L2_L3`) | 구체적인 사용자 또는 시스템 요구사항의 단위. 핵심 작성 단위. |
| `4` | 세부기능 | Task | 3 (`SEQ_L2_L3_L4`) | 구현 단위로 분해된 세부 요구사항. 개발 작업 항목과 직접 연결. |
| `5` | 명세항목 | Sub-task | 4 (`SEQ_L2_L3_L4_L5`) | 최하위 세부 요구사항. 특정 조건·제약·세부 동작을 명세. |

### 4-2. `req_category` — 요구사항 유형

| 값 | 설명 |
|----|------|
| `기능` | 시스템이 수행해야 하는 동작·처리 |
| `인터페이스` | 외부 시스템·API·화면 간 연동 규격 |
| `데이터` | 데이터 구조, 저장, 마이그레이션 |
| `배치` | 스케줄 기반 자동화 처리 |
| `성능` | 응답시간, 처리량, 동시접속 등 성능 기준 |
| `보안` | 인증, 인가, 암호화, 보안 정책 |
| `가용성` | 장애 허용, 복구, 업타임 기준 |
| `유지보수성` | 로깅, 모니터링, 코드 품질 기준 |
| `운영` | 배포, 설정 관리, 운영 절차 |
| `제약사항` | 기술·환경·법적 제약 |
| `기타` | 위 유형에 해당하지 않는 요구사항 |

### 4-3. `verification_method` — 검증 방법

| 값 | 설명 | 주로 사용하는 경우 |
|----|------|------------------|
| `테스트` | 테스트케이스 실행으로 결과 확인 | 기능, 인터페이스 요구사항 |
| `검사` | 산출물·코드·설정을 직접 확인 | 보안, 데이터, 코드 품질 |
| `분석` | 수치·데이터·로그 분석으로 확인 | 성능, 가용성 요구사항 |
| `시연` | 기능을 직접 시연하여 확인 | UI/UX, 사용성 요구사항 |
| `검토` | 전문가 검토·리뷰로 확인 | 아키텍처, 제약사항, 문서 |

### 4-4. `priority` — 우선순위 (MoSCoW)

| 값 | 설명 |
|----|------|
| `MUST` | 반드시 구현해야 하는 핵심 요구사항 |
| `SHOULD` | 가능하면 구현해야 하나, 불가 시 협의 가능 |
| `COULD` | 여유가 있을 때 구현하면 좋은 요구사항 |
| `WONT` | 이번 범위에서는 제외 (향후 재검토 가능) |

### 4-5. `req_status` — 요구사항 상태

| 값 | 설명 |
|----|------|
| `Draft` | 초안 — 검토 전 작성 단계 |
| `InReview` | 검토 중 — 이해관계자 리뷰 진행 중 |
| `Approved` | 승인됨 — 확정된 요구사항 |
| `Rejected` | 반려됨 — 재작성 또는 삭제 필요 |
| `Deferred` | 보류 — 판단 유보 (후속 스프린트로 이월 등) |
| `Deprecated` | 폐기 — 더 이상 유효하지 않은 요구사항 |

### 4-6. `acceptance_status` — 수용여부 (RFP 전용)

| 값 | 설명 |
|----|------|
| `수용` | RFP 요구사항을 그대로 수용 |
| `조건부수용` | 조건·협의·범위 조정 후 수용 |
| `불수용` | 기술적·일정·비용 등의 이유로 수용 불가 |
| `검토중` | 수용 여부 미결, 추가 검토 필요 |
| `해당없음` | 수용 여부 판단이 필요 없는 항목 |

### 4-7. `dev_status` — 개발 상태 (DEV 전용)

| 값 | 설명 |
|----|------|
| `Todo` | 미착수 — 개발 예정 |
| `InProgress` | 진행 중 |
| `Done` | 완료 — 구현 및 검증 완료 |
| `Blocked` | 차단됨 — 선행 의존성 또는 외부 이슈로 진행 불가 |
| `Skipped` | 건너뜀 — 의도적으로 이번 배포에서 제외 |

---

## 5. 중첩 타입 정의

### 5-1. `ChangeHistoryEntry` — 변경이력 항목

| 필드명 | 타입 | 필수 | 설명 |
|--------|------|------|------|
| `version` | string | ✅ | 변경 후 버전 (예: `1.1`) |
| `date` | string (date) | ✅ | 변경일 (예: `2026-05-04`) |
| `author` | string | ✅ | 변경 작성자 |
| `description` | string | ✅ | 변경 내용 요약 |
| `changed_fields` | string[] | — | 변경된 필드명 목록 |

### 5-2. `IOSpec` — 입력/출력 명세 (DEV 전용)

| 필드명 | 타입 | 설명 |
|--------|------|------|
| `inputs` | string[] | 이 요구사항이 처리하는 입력 항목 |
| `outputs` | string[] | 이 요구사항이 생성·반환하는 출력 항목 |

---

## 6. DB 전환 시 타입 매핑 참고

| JSON 타입 | DB 전환 시 권장 타입 | 비고 |
|-----------|---------------------|------|
| `string` | `VARCHAR(200)` / `TEXT` | `req_name`은 VARCHAR, `description`은 TEXT |
| `string (date)` | `DATE` | `last_modified_date`, `change_history[].date` |
| `string (enum)` | `VARCHAR(30)` + CHECK 또는 별도 코드 테이블 | enum 값 변경 가능성 고려 시 코드 테이블 권장 |
| `integer` | `SMALLINT` / `INT` | `req_level`: SMALLINT, `story_points`: INT |
| `number` | `DECIMAL(6,2)` | `estimated_effort`: MD 단위 소수점 2자리 |
| `string[]` | 별도 연결 테이블 | `acceptance_criteria`, `metrics`, `stakeholders`, `source_rfp_ids`, `predecessor_dev_ids` |
| `object (IOSpec)` | 별도 테이블 또는 JSONB | PostgreSQL 사용 시 JSONB 권장 |
| `ChangeHistoryEntry[]` | 별도 이력 테이블 | `requirement_id` + `version`을 복합키로 |

> `req_id`는 계층 경로를 인코딩하므로, DB에서 특정 요구사항의 모든 하위 항목을 조회할 때 `LIKE 'DEV_FR_317%'` 패턴 검색을 활용할 수 있다.

---

## 7. 스키마 파일 구조 요약

```
req_schema_v3.json
├── definitions
│   ├── enums
│   │   ├── ReqType          — "RFP" | "DEV"
│   │   ├── ReqLevel         — integer 1~5
│   │   │                      (업무영역/Epic ~ 명세항목/Sub-task)
│   │   │                      ★ req_id 계층 깊이와 반드시 일치
│   │   ├── FunctionalClass  — "기능"(FR) | "비기능"(NFR)
│   │   ├── ReqCategory      — 11가지 유형
│   │   ├── DomainArea       — 11가지 도메인 (참조용, ID 미포함)
│   │   ├── VerificationMethod — 5가지 검증 방법
│   │   ├── Priority         — MoSCoW 4단계
│   │   ├── Importance       — Critical/High/Medium/Low
│   │   ├── ReqStatus        — 6가지 요구사항 상태
│   │   ├── AcceptanceStatus — 5가지 수용여부 (RFP 전용)
│   │   └── DevStatus        — 5가지 개발 상태 (DEV 전용)
│   └── types
│       ├── ChangeHistoryEntry — 변경이력 항목 객체
│       └── IOSpec             — 입력/출력 명세 객체 (DEV 전용)
└── schemas
    ├── CommonFields         — 공통 필드 정의 (14개 필드)
    │   req_id 통합 패턴:
    │   ^(RFP|DEV_(FR|NFR))_\d{3}(_\d{2}){0,4}$
    ├── RFPRequirement       — CommonFields + acceptance_status
    │   req_id 전용 패턴:
    │   ^RFP_\d{3}(_\d{2}){0,4}$
    ├── DevRequirement       — CommonFields + 7개 DEV 전용 필드
    │   req_id 전용 패턴:
    │   ^DEV_(FR|NFR)_\d{3}(_\d{2}){0,4}$
    └── Requirement          — oneOf 통합 스키마 (req_type으로 분기)
```

---

## 8. 변경이력

| 버전 | 일자 | 작성자 | 내용 |
|------|------|--------|------|
| 1.0.0 | 2026-05-04 | Anthony | 최초 스키마 설계 — 공통/RFP/DEV 필드 구분, enum 11종 정의, DOMAIN_CODE 포함 ID 체계 |
| 2.0.0 | 2026-05-04 | Anthony | ID 채번 규칙 전면 개정: DOMAIN_CODE 제거, DEV에 FR/NFR 구분 추가, 구분자 `-`→`_`, 계층 구조 1단계 지원. `DomainCode` enum 및 `domain_code` 필드 제거. `req_level` 수준명에 일반/애자일 이중 명칭 추가. |
| 3.0.0 | 2026-05-04 | Anthony | 계층 구조를 최대 5단계로 확장. `req_id` 정규식 패턴을 `(_\d{2}){0,4}`로 변경. **`req_level`과 `req_id` 계층 깊이의 일치 규칙** 명시. 5단계 계층 예시(RFP/DEV_FR/DEV_NFR) 및 부모-자식 관계 트리 추가. DB 계층 조회 힌트(`LIKE` 패턴) 추가. |
