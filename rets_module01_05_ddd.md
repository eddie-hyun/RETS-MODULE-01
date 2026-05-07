# 데이터 설계서 (DDD)
## RETS-MODULE-01: RFP 요구사항 상세화 및 품질 평가 도구

---

| 항목 | 내용 |
|------|------|
| 문서 ID | MD05 |
| 문서명 | 데이터 설계서 (Data Design Document) |
| 모듈 ID | RETS-MODULE-01 |
| 모듈명 | RFP 요구사항 상세화 및 품질 평가 도구 |
| 작성 기준일 | 2026-05-07 |
| 버전 | v1.0 |
| 상태 | 초안 (Draft) |
| 참조 문서 | rets_00_rfp.pdf, rets_02c_req_schema.json, MD02(PRD), MD03(SRS) |

---

## 목차

1. [데이터 아키텍처 개요](#1-데이터-아키텍처-개요)
2. [데이터 저장소 구조](#2-데이터-저장소-구조)
3. [엔티티 정의](#3-엔티티-정의)
   - 3.1 RequirementItem (요구사항 항목)
   - 3.2 QualityDiagnosisResult (품질 진단 결과)
   - 3.3 TraceabilityLink (추적성 연결)
   - 3.4 ChangeHistoryEntry (변경 이력)
   - 3.5 LLMSession (LLM 세션)
   - 3.6 LLMMessage (LLM 메시지)
   - 3.7 UserSettings (사용자 설정)
   - 3.8 QualityReport (품질 보고서)
   - 3.9 AppState (앱 루트 상태)
4. [localStorage 스키마 정의](#4-localstorage-스키마-정의)
5. [데이터 흐름 다이어그램](#5-데이터-흐름-다이어그램)
6. [입출력 데이터 명세](#6-입출력-데이터-명세)
7. [데이터 유효성 규칙](#7-데이터-유효성-규칙)
8. [데이터 마이그레이션 및 버전 관리](#8-데이터-마이그레이션-및-버전-관리)
9. [요구사항 추적 매트릭스](#9-요구사항-추적-매트릭스)

---

## 1. 데이터 아키텍처 개요

### 1.1 아키텍처 원칙

RETS-MODULE-01은 **백엔드 없는 순수 클라이언트 사이드 애플리케이션**으로, 모든 데이터 영속성은 브라우저의 `localStorage`를 통해 관리된다. 외부 API 호출은 오직 LLM 릴레이 서버(`https://ai-relay.mooo.com:11443`)와의 단방향 I/O에 한정된다.

| 원칙 | 설명 |
|------|------|
| 단일 소스 | 모든 애플리케이션 데이터는 `localStorage`의 단일 키(`rets_m01_state`)에 JSON 직렬화하여 저장 |
| 스키마 호환성 | `rets_02c_req_schema.json` v3.0.0 완전 준수 — REQ ID, 필드명, 열거값 동일 적용 |
| 불변 ID | `req_id`, `session_id`, `link_id`, `report_id` 는 생성 후 변경 불가 |
| 변경 이력 | 요구사항 수정 시 반드시 `change_history` 배열에 항목 추가 (소프트 감사 로그) |
| 크기 한계 | localStorage 최대 5MB. 대용량 데이터는 압축(LZ-String) 또는 분할 저장 적용 |
| 내보내기 | 모든 핵심 엔티티는 JSON 및 CSV 내보내기 가능한 구조로 설계 |

### 1.2 데이터 계층 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                         RETS-MODULE-01 App                       │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │  Input Layer │  │ Process Layer│  │    Output Layer       │   │
│  │              │  │              │  │                        │   │
│  │ RFP JSON     │─▶│ RequirementItem│─▶│ DEV JSON Export     │   │
│  │ (File Upload)│  │ (localStorage) │  │ Quality Report (MD) │   │
│  └──────────────┘  │              │  │ CSV Export            │   │
│                    │ QualityDiag- │  └──────────────────────┘   │
│  ┌──────────────┐  │ nosisResult  │                              │
│  │  LLM Layer   │  │              │  ┌──────────────────────┐   │
│  │              │  │ Traceability │  │   Persistence Layer   │   │
│  │ Gemini API   │◀─│ Link         │─▶│   localStorage        │   │
│  │ (Relay Srv)  │─▶│              │  │   (AppState JSON)     │   │
│  └──────────────┘  │ LLMSession   │  └──────────────────────┘   │
│                    │              │                              │
│                    │ UserSettings │                              │
│                    └──────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 엔티티 관계도 (ERD 개요)

```
RequirementItem (RFP) ──┐
                        ├──[TraceabilityLink]──── RequirementItem (DEV)
                        │         │
                        │    (link_type: rfp_to_dev)
                        │
RequirementItem (DEV) ──┼──[QualityDiagnosisResult] (1:N)
                        │
                        └──[ChangeHistoryEntry][] (embedded, N)

LLMSession ─────────────┬──[LLMMessage][] (embedded, N)
                        └── references RequirementItem[]

QualityReport ──────────┬── aggregates QualityDiagnosisResult[]
                        └── references RequirementItem[]

AppState ───────────────┬── requirements: RequirementItem[]
                        ├── qualityResults: QualityDiagnosisResult[]
                        ├── traceabilityLinks: TraceabilityLink[]
                        ├── llmSessions: LLMSession[]
                        ├── qualityReports: QualityReport[]
                        ├── userSettings: UserSettings
                        └── meta: AppMeta
```

---

## 2. 데이터 저장소 구조

### 2.1 localStorage 키 맵

| 키 이름 | 타입 | 용도 | 최대 크기 | 초기화 조건 |
|---------|------|------|-----------|------------|
| `rets_m01_state` | JSON string | 앱 전체 상태 (AppState) | 4.5 MB | 최초 실행 또는 설정 초기화 |
| `rets_m01_schema_version` | string | 저장된 스키마 버전 | 20 bytes | 앱 업데이트 시 자동 갱신 |
| `rets_m01_last_session` | string | 마지막 활성 LLM 세션 ID | 50 bytes | 세션 종료 시 |
| `rets_m01_draft_backup` | JSON string | 편집 중 임시 자동 저장 (30초 주기) | 500 KB | 저장 완료 또는 취소 시 |

### 2.2 AppState 최상위 구조

```json
{
  "meta": { ... },
  "requirements": [...],
  "qualityResults": [...],
  "traceabilityLinks": [...],
  "llmSessions": [...],
  "qualityReports": [...],
  "userSettings": { ... }
}
```

---

## 3. 엔티티 정의

### 3.1 RequirementItem (요구사항 항목)

**설명:** `rets_02c_req_schema.json` v3.0.0의 `RFPRequirement` 또는 `DevRequirement`를 완전히 준수하는 요구사항 레코드. 애플리케이션 내에서 RFP 요구사항과 DEV 요구사항을 동일 배열에 통합 관리한다.

**관련 기능:** FA-01, FA-02, FA-03 / FUNC-010~015, FUNC-020~023

#### 3.1.1 공통 필드 (CommonFields)

| # | 필드명 | 타입 | 필수 | 길이/범위 | 설명 | 예시 |
|---|--------|------|------|-----------|------|------|
| 1 | `req_type` | enum | ✅ | - | 요구사항 유형 — `"RFP"` 또는 `"DEV"` | `"RFP"` |
| 2 | `req_id` | string | ✅ | max 40 | 요구사항 ID — 스키마 패턴 준수 필수. 생성 후 불변 | `"RFP_001_01"` |
| 3 | `req_version` | string | ✅ | - | Semantic Version 간소화 (`\d+\.\d+(\.\d+)?`) | `"1.0"` |
| 4 | `req_level` | integer | ✅ | 1~5 | 계층 수준. `req_id`의 깊이와 반드시 일치 | `2` |
| 5 | `req_name` | string | ✅ | max 200 | 요구사항명 — 동사형 명사구 또는 명사 | `"RFP 파일 업로드 기능"` |
| 6 | `functional_class` | enum | ✅ | - | `"기능"` 또는 `"비기능"` | `"기능"` |
| 7 | `req_category` | enum | ✅ | - | 기능 / 인터페이스 / 데이터 / 배치 / 성능 / 보안 / 가용성 / 유지보수성 / 운영 / 제약사항 / 기타 | `"기능"` |
| 8 | `domain_area` | enum | - | - | UI/UX / 인증·인가 / 데이터관리 / 통합/인터페이스 / AI/LLM / 성능 / 보안 / 운영 / 인프라 / 공통 / 기타 | `"AI/LLM"` |
| 9 | `description` | string | ✅ | max 2000 | 요구사항 상세 설명 | `"사용자는 ..."` |
| 10 | `acceptance_criteria` | string[] | - | max 10 items, each max 500 | 완료 기준 목록 (Given-When-Then 형식 권장) | `["파일 크기 ≤ 10MB 시 업로드 성공"]` |
| 11 | `verification_method` | enum | - | - | 테스트 / 검사 / 분석 / 시연 / 검토 | `"테스트"` |
| 12 | `metrics` | string[] | - | max 5 items | 측정 지표 — 수치화된 기준 | `["처리시간 < 3초"]` |
| 13 | `priority` | enum | - | - | MUST / SHOULD / COULD / WONT | `"MUST"` |
| 14 | `importance` | enum | - | - | Critical / High / Medium / Low | `"High"` |
| 15 | `stakeholders` | string[] | - | max 10 items | 연관 이해관계자 역할명 | `["BA", "개발팀"]` |
| 16 | `created_by` | string | ✅ | max 100 | 발의자 | `"김담당"` |
| 17 | `last_modified_date` | string | ✅ | ISO 8601 date | 최종수정일 | `"2026-05-07"` |
| 18 | `req_status` | enum | ✅ | - | Draft / InReview / Approved / Rejected / Deferred / Deprecated | `"Draft"` |
| 19 | `manager` | string | - | max 100 | 관리 담당자 | `"이팀장"` |
| 20 | `reviewer` | string | - | max 100 | 검토자 | `"박시니어"` |
| 21 | `change_history` | ChangeHistoryEntry[] | - | max 100 items | 변경 이력 배열 (§3.4 참조) | `[{...}]` |

#### 3.1.2 RFP 전용 필드

| # | 필드명 | 타입 | 필수 | 설명 | 허용값 |
|---|--------|------|------|------|--------|
| 1 | `acceptance_status` | enum | ✅ | RFP 수용여부 | 수용 / 조건부수용 / 불수용 / 검토중 / 해당없음 |

#### 3.1.3 DEV 전용 필드

| # | 필드명 | 타입 | 필수 | 설명 | 허용값 / 범위 |
|---|--------|------|------|------|--------------|
| 1 | `io_spec` | IOSpec | - | 입출력 명세 `{inputs: string[], outputs: string[]}` | - |
| 2 | `story_points` | integer | - | 스토리포인트 (피보나치 권장) | 0, 1, 2, 3, 5, 8, 13, 21 |
| 3 | `estimated_effort` | number | - | 예상공수 (MD, Man-Day) | 0.5 ~ 100.0 |
| 4 | `source_rfp_ids` | string[] | - | 근거 RFP 요구사항 ID 목록 | `^RFP_\d{3}(_\d{2}){0,4}$` |
| 5 | `predecessor_dev_ids` | string[] | - | 선행 DEV 요구사항 ID 목록 | `^DEV_(FR\|NFR)_\d{3}(_\d{2}){0,4}$` |
| 6 | `dev_status` | enum | - | 개발 진행 상태 | Todo / InProgress / Done / Blocked / Skipped |
| 7 | `developer` | string | - | 개발 담당자 | max 100 |

#### 3.1.4 앱 전용 확장 필드 (rets_02c_req_schema 비표준, 앱 내부 전용)

| # | 필드명 | 타입 | 설명 | 비고 |
|---|--------|------|------|------|
| 1 | `_id` | string | 내부 UUID (브라우저 `crypto.randomUUID()`) | 내보내기 시 제외 |
| 2 | `_created_at` | string | ISO 8601 datetime. 레코드 최초 생성 타임스탬프 | 내보내기 시 제외 |
| 3 | `_updated_at` | string | ISO 8601 datetime. 레코드 최종 수정 타임스탬프 | 내보내기 시 제외 |
| 4 | `_llm_generated` | boolean | LLM이 자동 생성한 레코드 여부 (Human-in-the-Loop 표시) | 내보내기 시 제외 |
| 5 | `_quality_score` | number | 마지막 품질 진단 점수 (0~100). 캐시 목적 | 내보내기 시 제외 |
| 6 | `_selected` | boolean | UI 목록에서 현재 선택 상태 (런타임 전용) | 저장 불필요 |
| 7 | `_dirty` | boolean | 저장되지 않은 변경사항 존재 여부 (런타임 전용) | 저장 불필요 |

#### 3.1.5 REQ ID 채번 규칙

| 타입 | 패턴 | level 1 예시 | level 3 예시 |
|------|------|-------------|-------------|
| RFP 요구사항 | `^RFP_\d{3}(_\d{2}){0,4}$` | `RFP_001` | `RFP_001_01_03` |
| DEV 기능 요구사항 | `^DEV_FR_\d{3}(_\d{2}){0,4}$` | `DEV_FR_001` | `DEV_FR_001_01_03` |
| DEV 비기능 요구사항 | `^DEV_NFR_\d{3}(_\d{2}){0,4}$` | `DEV_NFR_001` | `DEV_NFR_001_01` |

> **채번 정책:** `req_type` + `functional_class` 별로 SEQ(3자리)를 독립 관리한다. 삭제 시 SEQ는 재사용하지 않는다 (공백 허용).

---

### 3.2 QualityDiagnosisResult (품질 진단 결과)

**설명:** 단일 RequirementItem에 대한 3차원(명확성·일관성·검증가능성) 품질 진단 결과를 담는 엔티티. 동일 req_id에 대해 복수의 진단 결과가 시계열로 누적된다.

**관련 기능:** FA-04 / FUNC-030~033

| # | 필드명 | 타입 | 필수 | 설명 | 예시 / 범위 |
|---|--------|------|------|------|------------|
| 1 | `diagnosis_id` | string | ✅ | 진단 결과 고유 ID. 형식: `QD-{YYYYMMDD}-{4자리순번}` | `"QD-20260507-0001"` |
| 2 | `req_id` | string | ✅ | 진단 대상 RequirementItem의 `req_id` | `"DEV_FR_001_01"` |
| 3 | `diagnosed_at` | string | ✅ | 진단 수행 일시 (ISO 8601 datetime) | `"2026-05-07T10:30:00.000Z"` |
| 4 | `diagnosed_by` | string | ✅ | 진단 수행 주체 — `"LLM"` 또는 사용자 이름 | `"LLM"` |
| 5 | `llm_session_id` | string | - | 진단에 사용된 LLM 세션 ID (§3.5 참조) | `"sess_abc123"` |
| 6 | `clarity_score` | number | ✅ | 명확성 점수 (0.0~1.0, 소수점 2자리) | `0.75` |
| 7 | `consistency_score` | number | ✅ | 일관성 점수 (0.0~1.0) | `0.80` |
| 8 | `verifiability_score` | number | ✅ | 검증가능성 점수 (0.0~1.0) | `0.60` |
| 9 | `overall_score` | number | ✅ | 종합 점수. 계산식: `(clarity × 0.4 + consistency × 0.3 + verifiability × 0.3) × 100` | `71.5` |
| 10 | `quality_grade` | enum | ✅ | 품질 등급 — A(≥90) / B(≥75) / C(≥60) / D(<60) | `"C"` |
| 11 | `issues` | QualityIssue[] | ✅ | 발견된 품질 이슈 목록 (하위 §3.2.1 참조) | `[...]` |
| 12 | `suggestions` | string[] | - | LLM이 제안하는 개선 방향 (최대 5개) | `["'명확히'를 구체적 수치로 대체할 것"]` |
| 13 | `raw_llm_response` | string | - | LLM 원본 응답 텍스트 (디버깅 목적) | `"..."` |
| 14 | `is_latest` | boolean | ✅ | 해당 req_id에 대한 최신 진단 여부 | `true` |

#### 3.2.1 QualityIssue (품질 이슈 항목, QualityDiagnosisResult 내 임베디드)

| # | 필드명 | 타입 | 필수 | 설명 | 허용값 |
|---|--------|------|------|------|--------|
| 1 | `issue_id` | string | ✅ | 이슈 ID. 형식: `{diagnosis_id}-I{2자리순번}` | `"QD-20260507-0001-I01"` |
| 2 | `issue_type` | enum | ✅ | 이슈 유형 | ambiguity / inconsistency / unverifiable / missing / duplicate / conflict |
| 3 | `dimension` | enum | ✅ | 해당 품질 차원 | clarity / consistency / verifiability |
| 4 | `severity` | enum | ✅ | 심각도 | Critical / Major / Minor / Trivial |
| 5 | `location` | string | - | 이슈 발생 위치 (필드명 또는 텍스트 위치) | `"description: '신속하게'"` |
| 6 | `description` | string | ✅ | 이슈 상세 설명 | `"'신속하게'는 측정 불가능한 모호한 표현"` |
| 7 | `suggestion` | string | - | 개선 제안 | `"'3초 이내'로 대체"` |
| 8 | `related_req_ids` | string[] | - | 연관된 다른 req_id (중복/충돌 이슈용) | `["DEV_FR_002_01"]` |

---

### 3.3 TraceabilityLink (추적성 연결)

**설명:** RequirementItem 간 또는 RequirementItem과 외부 산출물(TC, SCR, API 등) 사이의 추적 관계를 표현하는 엔티티.

**관련 기능:** FA-05 / FUNC-040~041

| # | 필드명 | 타입 | 필수 | 설명 | 예시 |
|---|--------|------|------|------|------|
| 1 | `link_id` | string | ✅ | 추적 링크 고유 ID. 형식: `TL-{YYYYMMDD}-{4자리순번}` | `"TL-20260507-0001"` |
| 2 | `link_type` | enum | ✅ | 연결 유형 (§3.3.1 참조) | `"rfp_to_dev"` |
| 3 | `source_id` | string | ✅ | 출발점 ID (req_id, TC-XXX, SCR-XX 등) | `"RFP_001_01"` |
| 4 | `source_type` | enum | ✅ | 출발점 타입 | rfp_req / dev_req / test_case / screen / api / data_object |
| 5 | `target_id` | string | ✅ | 도착점 ID | `"DEV_FR_001_01"` |
| 6 | `target_type` | enum | ✅ | 도착점 타입 | rfp_req / dev_req / test_case / screen / api / data_object |
| 7 | `relationship` | enum | - | 관계 성격 | derives / refines / tests / implements / conflicts / duplicates |
| 8 | `confidence` | number | - | LLM이 자동 생성한 링크의 신뢰도 (0.0~1.0) | `0.92` |
| 9 | `created_by` | string | ✅ | 생성 주체 (`"LLM"` 또는 사용자명) | `"LLM"` |
| 10 | `created_at` | string | ✅ | 생성 일시 (ISO 8601 datetime) | `"2026-05-07T10:00:00.000Z"` |
| 11 | `note` | string | - | 링크 부가 설명 (max 500자) | `"LLM 자동 생성 — 사용자 검토 필요"` |
| 12 | `is_approved` | boolean | ✅ | 사용자 승인 여부 (Human-in-the-Loop) | `false` |

#### 3.3.1 link_type 정의

| link_type | source_type | target_type | 설명 |
|-----------|-------------|-------------|------|
| `rfp_to_dev` | rfp_req | dev_req | RFP 요구사항에서 DEV 요구사항 도출 |
| `dev_to_test` | dev_req | test_case | DEV 요구사항을 검증하는 테스트케이스 |
| `dev_to_screen` | dev_req | screen | DEV 요구사항이 구현되는 화면 |
| `dev_to_api` | dev_req | api | DEV 요구사항이 사용하는 API |
| `dev_to_data` | dev_req | data_object | DEV 요구사항이 다루는 데이터 객체 |
| `rfp_conflict` | rfp_req | rfp_req | 상호 충돌하는 RFP 요구사항 쌍 |
| `dev_duplicate` | dev_req | dev_req | 중복된 DEV 요구사항 쌍 |
| `hierarchy` | rfp_req / dev_req | rfp_req / dev_req | 상위-하위 계층 관계 (부모-자식) |

---

### 3.4 ChangeHistoryEntry (변경 이력)

**설명:** `rets_02c_req_schema.json`의 `ChangeHistoryEntry` 타입과 100% 동일. RequirementItem의 `change_history` 배열에 임베딩되는 서브 엔티티.

| # | 필드명 | 타입 | 필수 | 설명 | 예시 |
|---|--------|------|------|------|------|
| 1 | `version` | string | ✅ | 변경 후 버전 (`^\d+\.\d+(\.\d+)?$`) | `"1.1"` |
| 2 | `date` | string | ✅ | 변경일 (ISO 8601 date) | `"2026-05-07"` |
| 3 | `author` | string | ✅ | 변경 작성자 | `"김담당"` |
| 4 | `description` | string | ✅ | 변경 내용 요약 (max 500자) | `"수용 여부를 '수용'으로 변경"` |
| 5 | `changed_fields` | string[] | - | 변경된 필드명 목록 | `["acceptance_status", "req_status"]` |

> **정책:** RequirementItem 수정 시, 시스템은 자동으로 `req_version`을 증가(`minor` +0.1)시키고 `change_history`에 새 항목을 append한다. 사용자가 직접 `description`을 입력하지 않으면 시스템이 변경 필드 목록을 기반으로 자동 생성한다.

---

### 3.5 LLMSession (LLM 세션)

**설명:** Gemini 릴레이 서버와의 대화 세션을 추적하는 엔티티. 하나의 세션은 단일 요구사항 상세화 또는 품질 진단 작업에 대응한다.

**관련 기능:** FA-03 / FUNC-020~023

| # | 필드명 | 타입 | 필수 | 설명 | 예시 |
|---|--------|------|------|------|------|
| 1 | `session_id` | string | ✅ | 릴레이 서버로부터 받은 세션 ID. 생성 후 불변 | `"sess_xk9mq2..."` |
| 2 | `purpose` | enum | ✅ | 세션 사용 목적 | detailing / quality_diagnosis / conflict_check / general |
| 3 | `target_req_ids` | string[] | - | 처리 대상 RequirementItem의 req_id 목록 | `["RFP_001_01"]` |
| 4 | `status` | enum | ✅ | 세션 상태 | initializing / active / completed / failed / expired |
| 5 | `created_at` | string | ✅ | 세션 시작 일시 (ISO 8601 datetime) | `"2026-05-07T10:00:00Z"` |
| 6 | `completed_at` | string | - | 세션 완료 일시 | `"2026-05-07T10:05:23Z"` |
| 7 | `team_id` | string | ✅ | 릴레이 서버 인증용 팀 ID (UserSettings에서 로드) | `"team_rets_01"` |
| 8 | `messages` | LLMMessage[] | ✅ | 대화 메시지 목록 (§3.6 참조) | `[...]` |
| 9 | `total_tokens_used` | integer | - | 세션 누적 토큰 사용량 (가용 시) | `3420` |
| 10 | `error_code` | string | - | 실패 시 오류 코드 | `"RELAY_TIMEOUT"` |
| 11 | `error_message` | string | - | 실패 시 오류 메시지 (max 500자) | `"Connection timed out after 30s"` |
| 12 | `result_req_ids` | string[] | - | 세션 결과로 생성된 DEV RequirementItem의 req_id 목록 | `["DEV_FR_001_01", "DEV_FR_001_02"]` |

#### 3.5.1 세션 생명주기

```
initializing ──▶ active ──▶ completed
     │              │
     └── failed ◀──┘
     
expired: 세션 생성 후 30분 경과 시 자동 만료 (서버 측 정책)
```

---

### 3.6 LLMMessage (LLM 메시지, LLMSession 내 임베디드)

**설명:** LLM 세션 내 개별 요청/응답 메시지 레코드.

| # | 필드명 | 타입 | 필수 | 설명 | 허용값 |
|---|--------|------|------|------|--------|
| 1 | `message_id` | string | ✅ | 메시지 순번 ID (형식: `{session_id}-M{3자리순번}`) | `"sess_abc-M001"` |
| 2 | `role` | enum | ✅ | 메시지 발신 주체 | user / assistant / system |
| 3 | `content` | string | ✅ | 메시지 본문 (max 10,000자) | `"다음 RFP 요구사항을 DEV 요구사항으로..."` |
| 4 | `timestamp` | string | ✅ | 메시지 전송/수신 일시 (ISO 8601 datetime) | `"2026-05-07T10:01:00Z"` |
| 5 | `parsed_json` | object | - | 응답 텍스트에서 파싱된 JSON 구조체 (성공 시) | `{requirements: [...]}` |
| 6 | `parse_success` | boolean | - | JSON 파싱 성공 여부 | `true` |
| 7 | `latency_ms` | integer | - | 응답 수신까지 소요 시간 (밀리초) | `2340` |

#### 3.6.1 LLM 응답 JSON 파싱 전략

릴레이 서버 응답 텍스트에서 JSON을 추출하는 우선순위 전략:

```
1순위: 정규식 코드블록 추출
       /```json\n([\s\S]*?)\n```/ 으로 첫 번째 매치 추출

2순위: 첫 번째 JSON 배열/객체 탐색
       response.indexOf('[') 또는 response.indexOf('{') 중 앞에 오는 위치부터 추출
       → JSON.parse() 시도

3순위: 전체 텍스트 JSON.parse() 시도

실패 시: parse_success = false, 사용자에게 오류 토스트 표시 후 raw_response 보관
```

---

### 3.7 UserSettings (사용자 설정)

**설명:** 사용자별 애플리케이션 환경설정을 저장하는 싱글톤 엔티티.

**관련 기능:** FA-07 / FUNC-060~062

| # | 필드명 | 타입 | 필수 | 기본값 | 설명 |
|---|--------|------|------|--------|------|
| 1 | `team_id` | string | ✅ | `""` | LLM 릴레이 서버 인증용 팀 ID |
| 2 | `relay_server_url` | string | ✅ | `"https://ai-relay.mooo.com:11443"` | 릴레이 서버 URL |
| 3 | `default_req_version` | string | - | `"1.0"` | 신규 요구사항 생성 시 기본 버전 |
| 4 | `default_created_by` | string | - | `""` | 신규 요구사항의 기본 발의자명 |
| 5 | `auto_save_interval_sec` | integer | - | `30` | 자동 저장 주기 (초). 0 = 비활성화 |
| 6 | `llm_timeout_sec` | integer | - | `30` | LLM 요청 타임아웃 (초). 범위: 5~120 |
| 7 | `quality_weights` | QualityWeights | - | `{clarity:0.4, consistency:0.3, verifiability:0.3}` | 품질 점수 차원별 가중치. 합계 반드시 1.0 |
| 8 | `export_format` | enum | - | `"json"` | 기본 내보내기 포맷 | json / csv / markdown |
| 9 | `ui_theme` | enum | - | `"light"` | UI 테마 | light / dark / system |
| 10 | `language` | enum | - | `"ko"` | 인터페이스 언어 | ko / en |
| 11 | `show_llm_raw_response` | boolean | - | `false` | LLM 원본 응답 노출 여부 (개발자 모드) |
| 12 | `max_requirements_per_batch` | integer | - | `10` | LLM 일괄 처리 시 배치 크기. 범위: 1~50 |

#### 3.7.1 QualityWeights (UserSettings 내 임베디드)

| 필드명 | 타입 | 필수 | 범위 | 설명 |
|--------|------|------|------|------|
| `clarity` | number | ✅ | 0.1~0.8 | 명확성 가중치 |
| `consistency` | number | ✅ | 0.1~0.8 | 일관성 가중치 |
| `verifiability` | number | ✅ | 0.1~0.8 | 검증가능성 가중치 |

> **유효성 규칙:** `clarity + consistency + verifiability = 1.0` (소수점 오차 허용 범위 ±0.001)

---

### 3.8 QualityReport (품질 보고서)

**설명:** 전체 또는 선택된 요구사항 집합에 대한 집계 품질 보고서. 내보내기 및 리뷰 프로세스에 사용된다.

**관련 기능:** FA-04, FA-06 / FUNC-033, FUNC-050

| # | 필드명 | 타입 | 필수 | 설명 | 예시 |
|---|--------|------|------|------|------|
| 1 | `report_id` | string | ✅ | 보고서 ID. 형식: `RPT-{YYYYMMDD}-{4자리순번}` | `"RPT-20260507-0001"` |
| 2 | `title` | string | ✅ | 보고서 제목 (max 200자) | `"RETS-MODULE-01 요구사항 품질 보고서 v1.0"` |
| 3 | `generated_at` | string | ✅ | 생성 일시 (ISO 8601 datetime) | `"2026-05-07T11:00:00Z"` |
| 4 | `generated_by` | string | ✅ | 생성 주체 | `"김담당"` |
| 5 | `scope` | enum | ✅ | 보고서 범위 | all / selected / rfp_only / dev_only |
| 6 | `target_req_ids` | string[] | - | scope=selected 일 때 대상 req_id 목록 | `["DEV_FR_001", ...]` |
| 7 | `total_count` | integer | ✅ | 진단 대상 요구사항 총 건수 | `47` |
| 8 | `avg_overall_score` | number | ✅ | 전체 평균 종합 점수 (0~100) | `72.3` |
| 9 | `avg_clarity` | number | ✅ | 명확성 평균 (0~100) | `68.5` |
| 10 | `avg_consistency` | number | ✅ | 일관성 평균 (0~100) | `79.1` |
| 11 | `avg_verifiability` | number | ✅ | 검증가능성 평균 (0~100) | `70.8` |
| 12 | `grade_distribution` | GradeDist | ✅ | 등급 분포 `{A, B, C, D}` (건수) | `{A:5, B:18, C:17, D:7}` |
| 13 | `issue_summary` | IssueSummary | ✅ | 이슈 유형별 건수 집계 | `{ambiguity:12, inconsistency:5, ...}` |
| 14 | `critical_issues` | QualityIssue[] | - | severity=Critical인 이슈 목록 (최대 20건) | `[...]` |
| 15 | `recommendations` | string[] | - | 전체 품질 개선 권고사항 (최대 10개) | `["모호한 부사 표현 일괄 검토 필요"]` |
| 16 | `export_format` | enum | ✅ | 내보내기 포맷 | json / markdown / csv |
| 17 | `diagnosis_ids` | string[] | ✅ | 집계에 사용된 QualityDiagnosisResult ID 목록 | `["QD-20260507-0001", ...]` |

---

### 3.9 AppState (앱 루트 상태)

**설명:** localStorage의 `rets_m01_state` 키에 직렬화되는 최상위 상태 객체. 앱의 모든 영속성 데이터를 단일 트리로 관리한다.

| # | 필드명 | 타입 | 필수 | 설명 |
|---|--------|------|------|------|
| 1 | `meta` | AppMeta | ✅ | 앱 메타 정보 (§3.9.1 참조) |
| 2 | `requirements` | RequirementItem[] | ✅ | 전체 요구사항 목록 (RFP + DEV 통합) |
| 3 | `qualityResults` | QualityDiagnosisResult[] | ✅ | 전체 품질 진단 결과 목록 |
| 4 | `traceabilityLinks` | TraceabilityLink[] | ✅ | 전체 추적성 연결 목록 |
| 5 | `llmSessions` | LLMSession[] | ✅ | LLM 세션 이력 목록 |
| 6 | `qualityReports` | QualityReport[] | ✅ | 생성된 품질 보고서 목록 |
| 7 | `userSettings` | UserSettings | ✅ | 사용자 환경설정 (싱글톤) |

#### 3.9.1 AppMeta (AppState 내 임베디드)

| # | 필드명 | 타입 | 필수 | 설명 | 예시 |
|---|--------|------|------|------|------|
| 1 | `app_version` | string | ✅ | 앱 버전 (Semantic Versioning) | `"1.0.0"` |
| 2 | `schema_version` | string | ✅ | 데이터 스키마 버전 (마이그레이션 판단 기준) | `"1.0.0"` |
| 3 | `created_at` | string | ✅ | AppState 최초 생성 일시 | `"2026-05-07T09:00:00Z"` |
| 4 | `last_saved_at` | string | ✅ | 마지막 저장 일시 | `"2026-05-07T11:30:00Z"` |
| 5 | `total_requirements` | integer | ✅ | 전체 요구사항 건수 (실시간 카운트 캐시) | `47` |
| 6 | `total_rfp` | integer | ✅ | RFP 요구사항 건수 | `20` |
| 7 | `total_dev` | integer | ✅ | DEV 요구사항 건수 | `27` |
| 8 | `checksum` | string | - | 데이터 무결성 검증용 SHA-256 해시 (16진수 앞 16자) | `"a3f9bc12e0d51234"` |

---

## 4. localStorage 스키마 정의

### 4.1 AppState 전체 JSON 구조 (타입 주석 포함)

```jsonc
{
  // === AppMeta ===
  "meta": {
    "app_version": "1.0.0",         // string, Semantic Version
    "schema_version": "1.0.0",      // string, 마이그레이션 판단 기준
    "created_at": "2026-05-07T09:00:00Z",
    "last_saved_at": "2026-05-07T11:30:00Z",
    "total_requirements": 47,
    "total_rfp": 20,
    "total_dev": 27,
    "checksum": "a3f9bc12e0d51234"  // 선택적, 무결성 검증
  },

  // === RequirementItem[] ===
  "requirements": [
    {
      // --- 공통 필드 (CommonFields) ---
      "req_type": "RFP",             // "RFP" | "DEV"
      "req_id": "RFP_001",           // 스키마 패턴 준수 필수
      "req_version": "1.0",
      "req_level": 1,                // 1~5, req_id 깊이와 일치
      "req_name": "파일 I/O 기능",
      "functional_class": "기능",   // "기능" | "비기능"
      "req_category": "기능",
      "domain_area": "UI/UX",
      "description": "사용자는 RFP JSON 파일을 업로드하여 시스템에 로드할 수 있어야 한다.",
      "acceptance_criteria": [
        "JSON 파일 선택 후 파싱 성공 시 요구사항 목록이 표시된다",
        "파일 크기 10MB 초과 시 오류 메시지가 표시된다"
      ],
      "verification_method": "테스트",
      "metrics": ["파일 로드 처리 시간 < 3초 (10MB 기준)"],
      "priority": "MUST",
      "importance": "Critical",
      "stakeholders": ["BA", "개발팀"],
      "acceptance_status": "수용",   // RFP 전용 필드
      "created_by": "김담당",
      "last_modified_date": "2026-05-07",
      "req_status": "Draft",
      "manager": "이팀장",
      "reviewer": "박시니어",
      "change_history": [
        {
          "version": "1.0",
          "date": "2026-05-07",
          "author": "김담당",
          "description": "초안 작성",
          "changed_fields": []
        }
      ],
      // --- 앱 전용 내부 필드 (내보내기 제외) ---
      "_id": "550e8400-e29b-41d4-a716-446655440000",
      "_created_at": "2026-05-07T09:05:00.000Z",
      "_updated_at": "2026-05-07T09:05:00.000Z",
      "_llm_generated": false,
      "_quality_score": null
    },
    {
      // --- DEV 요구사항 예시 ---
      "req_type": "DEV",
      "req_id": "DEV_FR_001",
      "req_version": "1.0",
      "req_level": 1,
      "req_name": "RFP JSON 파일 업로드 처리",
      "functional_class": "기능",
      "req_category": "기능",
      "domain_area": "데이터관리",
      "description": "FileReader API를 사용하여 JSON 파일을 비동기로 읽고, rets_02c_req_schema에 따라 유효성을 검증한 후 requirements 배열에 저장한다.",
      "acceptance_criteria": [
        "파일 선택 즉시 FileReader.readAsText()가 호출된다",
        "JSON.parse() 실패 시 '유효하지 않은 JSON 파일' 오류가 표시된다",
        "스키마 검증 실패 시 오류 필드 목록이 포함된 메시지가 표시된다"
      ],
      "verification_method": "테스트",
      "metrics": ["파싱 완료까지 3초 이내 (100개 요구사항 기준)"],
      "priority": "MUST",
      "importance": "Critical",
      "io_spec": {
        "inputs": ["File 객체 (JSON, max 10MB)"],
        "outputs": ["RequirementItem[] (parsed, localStorage 저장)"]
      },
      "story_points": 3,
      "estimated_effort": 0.5,
      "source_rfp_ids": ["RFP_001"],
      "predecessor_dev_ids": [],
      "dev_status": "Todo",
      "developer": "",
      "created_by": "시스템 (LLM)",
      "last_modified_date": "2026-05-07",
      "req_status": "Draft",
      "change_history": [
        {
          "version": "1.0",
          "date": "2026-05-07",
          "author": "LLM",
          "description": "LLM 자동 생성 — 사용자 검토 전"
        }
      ],
      "_id": "660f9500-f30c-52e5-b827-557766550111",
      "_created_at": "2026-05-07T10:00:00.000Z",
      "_updated_at": "2026-05-07T10:00:00.000Z",
      "_llm_generated": true,
      "_quality_score": 78.5
    }
  ],

  // === QualityDiagnosisResult[] ===
  "qualityResults": [
    {
      "diagnosis_id": "QD-20260507-0001",
      "req_id": "DEV_FR_001",
      "diagnosed_at": "2026-05-07T10:30:00.000Z",
      "diagnosed_by": "LLM",
      "llm_session_id": "sess_xk9mq2abc",
      "clarity_score": 0.75,
      "consistency_score": 0.82,
      "verifiability_score": 0.79,
      "overall_score": 78.5,
      "quality_grade": "B",
      "issues": [
        {
          "issue_id": "QD-20260507-0001-I01",
          "issue_type": "ambiguity",
          "dimension": "clarity",
          "severity": "Minor",
          "location": "description: '비동기로 읽고'",
          "description": "비동기 처리 방식이 구체적으로 명시되지 않음",
          "suggestion": "FileReader API의 onload 이벤트 콜백 방식 명시",
          "related_req_ids": []
        }
      ],
      "suggestions": [
        "비동기 처리 메커니즘을 구체적으로 명시할 것",
        "오류 처리 시나리오를 acceptance_criteria에 추가할 것"
      ],
      "is_latest": true
    }
  ],

  // === TraceabilityLink[] ===
  "traceabilityLinks": [
    {
      "link_id": "TL-20260507-0001",
      "link_type": "rfp_to_dev",
      "source_id": "RFP_001",
      "source_type": "rfp_req",
      "target_id": "DEV_FR_001",
      "target_type": "dev_req",
      "relationship": "derives",
      "confidence": 0.95,
      "created_by": "LLM",
      "created_at": "2026-05-07T10:00:00.000Z",
      "note": "LLM 자동 생성",
      "is_approved": false
    }
  ],

  // === LLMSession[] ===
  "llmSessions": [
    {
      "session_id": "sess_xk9mq2abc",
      "purpose": "detailing",
      "target_req_ids": ["RFP_001"],
      "status": "completed",
      "created_at": "2026-05-07T09:55:00.000Z",
      "completed_at": "2026-05-07T10:00:00.000Z",
      "team_id": "team_rets_01",
      "messages": [
        {
          "message_id": "sess_xk9mq2abc-M001",
          "role": "user",
          "content": "다음 RFP 요구사항을 DEV 요구사항으로 상세화해 주세요: ...",
          "timestamp": "2026-05-07T09:55:05.000Z",
          "latency_ms": null
        },
        {
          "message_id": "sess_xk9mq2abc-M002",
          "role": "assistant",
          "content": "```json\n[{...}]\n```",
          "timestamp": "2026-05-07T09:55:12.000Z",
          "parsed_json": [{"req_type": "DEV", "req_id": "DEV_FR_001", "...": "..."}],
          "parse_success": true,
          "latency_ms": 7000
        }
      ],
      "total_tokens_used": 3420,
      "result_req_ids": ["DEV_FR_001"]
    }
  ],

  // === QualityReport[] ===
  "qualityReports": [
    {
      "report_id": "RPT-20260507-0001",
      "title": "RETS-MODULE-01 요구사항 품질 보고서 v1.0",
      "generated_at": "2026-05-07T11:00:00.000Z",
      "generated_by": "김담당",
      "scope": "all",
      "total_count": 47,
      "avg_overall_score": 72.3,
      "avg_clarity": 68.5,
      "avg_consistency": 79.1,
      "avg_verifiability": 70.8,
      "grade_distribution": {"A": 5, "B": 18, "C": 17, "D": 7},
      "issue_summary": {
        "ambiguity": 12,
        "inconsistency": 5,
        "unverifiable": 8,
        "missing": 3,
        "duplicate": 2,
        "conflict": 1
      },
      "recommendations": [
        "D등급 7건 우선 재검토 — 명확성 지표 집중 개선",
        "모호한 부사(신속히, 적절히 등) 발견 시 수치 기준으로 대체"
      ],
      "export_format": "markdown",
      "diagnosis_ids": ["QD-20260507-0001", "QD-20260507-0002"]
    }
  ],

  // === UserSettings ===
  "userSettings": {
    "team_id": "team_rets_01",
    "relay_server_url": "https://ai-relay.mooo.com:11443",
    "default_req_version": "1.0",
    "default_created_by": "김담당",
    "auto_save_interval_sec": 30,
    "llm_timeout_sec": 30,
    "quality_weights": {
      "clarity": 0.4,
      "consistency": 0.3,
      "verifiability": 0.3
    },
    "export_format": "json",
    "ui_theme": "light",
    "language": "ko",
    "show_llm_raw_response": false,
    "max_requirements_per_batch": 10
  }
}
```

---

## 5. 데이터 흐름 다이어그램

### 5.1 RFP JSON 업로드 → DEV 요구사항 생성 흐름

```
┌──────────┐     ┌─────────────────────────────────────────────────────────────┐
│  사용자  │     │                       App (Browser)                          │
└────┬─────┘     └──────────────────────────────────────────────────────────────┘
     │
     │ [1] RFP JSON 파일 선택
     │────────────────────────────────▶ FileReader.readAsText()
     │                                          │
     │                                [2] JSON.parse()
     │                                          │
     │                                [3] Schema Validate
     │                                  (rets_02c_req_schema)
     │                                          │
     │                                [4] Store as RequirementItem[]
     │                                  → requirements[req_type=RFP]
     │                                  → localStorage.setItem()
     │
     │ [5] 상세화 요청 (선택된 RFP 항목)
     │────────────────────────────────▶
     │                                [6] LLMSession 생성 (initializing)
     │                                [7] POST /chat/session/create
     │                                  body: {team_id}
     │                                          │
     │                                          ▼
     │                              ┌──────────────────────┐
     │                              │  Relay Server         │
     │                              │  ai-relay.mooo.com   │
     │                              └──────────────────────┘
     │                                          │
     │                                [8] session_id 수신
     │                                [9] POST /chat/session/{id}/message
     │                                  body: {team_id, message: prompt+RFP JSON}
     │                                          │
     │                                [10] 응답 텍스트 수신
     │                                [11] JSON 파싱 (3단계 전략)
     │                                [12] DEV RequirementItem[] 생성
     │                                   → _llm_generated = true
     │                                   → req_status = "Draft"
     │                                          │
     │◀───────────────────────────── [13] UI 갱신: 상세화 제안 표시
     │
     │ [14] Accept / Edit / Reject
     │────────────────────────────────▶
     │                                [15] Human-in-the-Loop 처리
     │                                   Accept → req_status 유지
     │                                   Edit   → 사용자 수정 반영
     │                                   Reject → 해당 항목 삭제
     │                                [16] TraceabilityLink 자동 생성
     │                                   (RFP→DEV, link_type=rfp_to_dev)
     │                                [17] localStorage 저장 (전체 AppState)
     │
     │◀───────────────────────────── [18] 완료 알림
```

### 5.2 품질 진단 흐름

```
┌──────────┐
│  사용자  │──[1] 품질 진단 요청 (단건 또는 전체)──▶
└──────────┘                                    │
                                    [2] target RequirementItem[] 선택
                                    [3] 배치 분할 (max_requirements_per_batch)
                                                │
                              ┌─ 배치 반복 ────┘
                              │
                              [4] LLMSession 생성 (purpose=quality_diagnosis)
                              [5] 품질 진단 프롬프트 구성
                              [6] POST /chat/session/{id}/message
                                          │
                                          ▼
                                  Relay Server
                                          │
                              [7] 응답 수신 및 JSON 파싱
                              [8] QualityDiagnosisResult 생성
                                  overall_score 계산:
                                  (clarity×0.4 + consistency×0.3 + verifiability×0.3) × 100
                              [9] quality_grade 부여 (A/B/C/D)
                              [10] RequirementItem._quality_score 업데이트
                              └─ 배치 반복 ──▶

                              [11] QualityReport 집계 생성
                              [12] localStorage 저장
                              [13] UI 갱신 — 품질 하이라이팅 표시
```

### 5.3 데이터 내보내기 흐름

```
사용자 → [내보내기 클릭] → 포맷 선택 (JSON/CSV/Markdown)
    → [JSON] → AppState.requirements 필터링 → _id, _created_at 등 앱 전용 필드 제거
              → rets_02c_req_schema 호환 순수 JSON 생성 → Blob 다운로드
    → [CSV]  → 필드 목록 헤더 행 생성 → 배열 필드는 세미콜론(;) 구분 문자열 변환
              → UTF-8 BOM 포함 CSV → Blob 다운로드
    → [MD]   → QualityReport 기반 Markdown 보고서 생성 → Blob 다운로드
```

---

## 6. 입출력 데이터 명세

### 6.1 파일 입력 (RFP JSON 업로드)

**입력 포맷:** `application/json`, UTF-8 인코딩, 최대 10MB

```json
{
  "project": "string (optional)",
  "version": "string (optional)",
  "requirements": [
    // RequirementItem[] — rets_02c_req_schema 준수
    // req_type = "RFP" 항목만 유효
  ]
}
```

**유효성 검사 규칙:**

| 규칙 | 조건 | 오류 처리 |
|------|------|-----------|
| JSON 파싱 | `JSON.parse()` 성공 | 오류 시 → E-001 |
| 필수 키 존재 | `requirements` 키 존재 및 배열 | 오류 시 → E-002 |
| req_id 패턴 | 각 항목의 `req_id`가 스키마 패턴 일치 | 불일치 항목 목록 → E-003 |
| req_type 일치 | 업로드 시 `req_type = "RFP"` 권장 (DEV도 허용) | 경고 표시 |
| 파일 크기 | ≤ 10MB | 초과 시 → E-004 |
| 아이템 수 | ≤ 1,000건 | 초과 시 경고, 처리 계속 |

### 6.2 파일 출력 (DEV JSON 내보내기)

**출력 포맷:** `application/json`, UTF-8 인코딩

```json
{
  "export_metadata": {
    "exported_at": "2026-05-07T12:00:00Z",
    "exported_by": "김담당",
    "app_version": "1.0.0",
    "schema_version": "rets_02c_req_schema v3.0.0",
    "total_count": 27,
    "req_types": ["DEV"]
  },
  "requirements": [
    // RequirementItem[] — 앱 전용 필드(_로 시작) 제외
    // rets_02c_req_schema v3.0.0 완전 호환
  ]
}
```

### 6.3 LLM API 요청/응답

**세션 초기화 요청 (POST `/chat/session/create`):**

```json
{ "team_id": "team_rets_01" }
```

**세션 초기화 응답:**

```json
{ "session_id": "sess_xk9mq2abc..." }
```

**메시지 전송 요청 (POST `/chat/session/{session_id}/message`):**

```json
{
  "team_id": "team_rets_01",
  "message": "## 요구사항 상세화 지시\n\n다음 RFP 요구사항을 rets_02c_req_schema v3.0.0 기준으로 DEV 요구사항 JSON 배열로 변환해 주세요.\n\n...(프롬프트 본문)...\n\n```json\n[RFP 요구사항 JSON]\n```"
}
```

**메시지 응답 (텍스트):**

```
다음 DEV 요구사항을 생성했습니다.

```json
[
  {
    "req_type": "DEV",
    "req_id": "DEV_FR_001",
    ...
  }
]
```
```

---

## 7. 데이터 유효성 규칙

### 7.1 필드 수준 규칙

| ID | 대상 엔티티 | 필드 | 규칙 | 오류 처리 |
|----|------------|------|------|-----------|
| DV-001 | RequirementItem | `req_id` | `rets_02c_req_schema` 패턴 정규식 일치 필수 | 저장 거부 + 오류 메시지 |
| DV-002 | RequirementItem | `req_level` | `req_id`의 언더스코어 구분 깊이와 일치 필수 | 저장 거부 + 불일치 안내 |
| DV-003 | RequirementItem | `req_version` | `^\d+\.\d+(\.\d+)?$` 패턴 일치 | 기본값 `"1.0"` 자동 설정 |
| DV-004 | RequirementItem | `functional_class` | DEV 타입 시 `req_id`의 FR/NFR 프리픽스와 일치 필수 | 저장 거부 + 불일치 안내 |
| DV-005 | QualityDiagnosisResult | `overall_score` | `(clarity×0.4 + consistency×0.3 + verifiability×0.3) × 100`으로 재계산 후 일치 확인 | 자동 재계산 후 저장 |
| DV-006 | UserSettings | `quality_weights` | `clarity + consistency + verifiability = 1.0` (±0.001 허용) | 저장 거부 + 합계 표시 |
| DV-007 | TraceabilityLink | `source_id`, `target_id` | 참조하는 `req_id`가 `requirements` 배열에 실제로 존재해야 함 | 경고 표시, 저장은 허용 |
| DV-008 | LLMSession | `session_id` | 중복 불허 (`llmSessions` 배열 내 unique) | 신규 세션 생성 중단 |
| DV-009 | RequirementItem | `source_rfp_ids` (DEV) | 참조 `RFP_XXX` ID가 실제 `requirements`에 존재해야 함 | 경고 표시, 저장 허용 |
| DV-010 | AppState | 전체 | localStorage 직렬화 결과가 5MB 초과 시 경고 | LZ-String 압축 제안 |

### 7.2 비즈니스 규칙

| ID | 규칙명 | 설명 |
|----|--------|------|
| BR-001 | 불변 ID | `req_id`, `diagnosis_id`, `link_id`, `session_id`, `report_id` 생성 후 수정 불가 |
| BR-002 | 변경 이력 자동 추가 | RequirementItem 수정 시 `change_history`에 항목 자동 추가 (수동 입력 없어도) |
| BR-003 | 버전 자동 증가 | RequirementItem 수정 시 `req_version`의 minor 버전 자동 +0.1 증가 |
| BR-004 | 최신 진단 표시 | 동일 `req_id`에 대해 가장 최근 `QualityDiagnosisResult`만 `is_latest = true` |
| BR-005 | LLM 승인 표시 | `_llm_generated = true`인 RequirementItem은 사용자 명시적 Accept 전까지 `req_status = "Draft"` 유지 |
| BR-006 | 세션 만료 정책 | `created_at` 기준 30분 경과 시 `status = "expired"` 자동 처리 (서버 연동 기준) |
| BR-007 | 내보내기 정제 | 내보내기 JSON에서 앱 전용 필드(`_id`, `_created_at`, `_updated_at`, `_llm_generated`, `_quality_score`) 반드시 제거 |
| BR-008 | 가중치 합계 | `quality_weights.clarity + consistency + verifiability = 1.0` 항상 유지 |

---

## 8. 데이터 마이그레이션 및 버전 관리

### 8.1 스키마 버전 관리 전략

앱 로드 시 `rets_m01_schema_version` 키를 확인하고, 현재 앱 버전과 불일치 시 마이그레이션을 수행한다.

```javascript
// 앱 초기화 시 마이그레이션 검사 흐름
const storedVersion = localStorage.getItem('rets_m01_schema_version');
const currentVersion = APP_SCHEMA_VERSION; // e.g., "1.0.0"

if (!storedVersion) {
  // 최초 실행 → 초기 AppState 생성
  initializeAppState();
} else if (storedVersion !== currentVersion) {
  // 버전 불일치 → 마이그레이션 실행
  migrateAppState(storedVersion, currentVersion);
} else {
  // 동일 버전 → 기존 상태 로드
  loadAppState();
}
```

### 8.2 마이그레이션 원칙

| 원칙 | 설명 |
|------|------|
| 하위 호환 | 신규 필드 추가 시 기존 데이터에 기본값으로 자동 채움 |
| 마이그레이션 전 백업 | 마이그레이션 실행 전 기존 데이터를 `rets_m01_draft_backup`에 임시 저장 |
| 롤백 지원 | 마이그레이션 실패 시 백업에서 복원 |
| 사용자 알림 | 마이그레이션 완료 시 토스트 메시지 표시 |

### 8.3 마이그레이션 함수 목록 (예정)

| 버전 전환 | 함수명 | 변경 내용 |
|-----------|--------|-----------|
| v0.9 → v1.0 | `migrate_0_9_to_1_0()` | UserSettings에 `max_requirements_per_batch` 필드 추가 |

---

## 9. 요구사항 추적 매트릭스

### 9.1 FUNC ↔ 데이터 엔티티 매트릭스

| FUNC ID | 기능명 | 주요 엔티티 | 읽기(R) | 쓰기(W) | 삭제(D) |
|---------|--------|------------|---------|---------|---------|
| FUNC-001 | RFP JSON 파일 업로드 | RequirementItem (RFP) | - | W | - |
| FUNC-002 | DEV JSON 파일 업로드 | RequirementItem (DEV) | - | W | - |
| FUNC-003 | JSON 내보내기 | RequirementItem | R | - | - |
| FUNC-004 | CSV 내보내기 | RequirementItem, QualityReport | R | - | - |
| FUNC-010 | 요구사항 목록 조회 | RequirementItem | R | - | - |
| FUNC-011 | 요구사항 상세 조회 | RequirementItem, QualityDiagnosisResult | R | - | - |
| FUNC-012 | 요구사항 신규 등록 | RequirementItem | - | W | - |
| FUNC-013 | 요구사항 수정 | RequirementItem, ChangeHistoryEntry | R | W | - |
| FUNC-014 | 요구사항 삭제 | RequirementItem, TraceabilityLink | R | - | D |
| FUNC-015 | 요구사항 검색/필터 | RequirementItem | R | - | - |
| FUNC-020 | LLM 세션 초기화 | LLMSession, UserSettings | R | W | - |
| FUNC-021 | 요구사항 자동 상세화 | LLMSession, LLMMessage, RequirementItem (DEV) | R | W | - |
| FUNC-022 | 상세화 결과 Human-in-the-Loop 처리 | RequirementItem, TraceabilityLink | R | W | D |
| FUNC-023 | LLM 세션 종료 | LLMSession | R | W | - |
| FUNC-030 | 품질 진단 실행 | QualityDiagnosisResult, LLMSession, RequirementItem | R | W | - |
| FUNC-031 | 품질 이슈 하이라이팅 | QualityDiagnosisResult | R | - | - |
| FUNC-032 | 중복/충돌/누락 탐지 | QualityDiagnosisResult (QualityIssue), TraceabilityLink | R | W | - |
| FUNC-033 | 품질 보고서 생성 | QualityReport, QualityDiagnosisResult | R | W | - |
| FUNC-040 | 추적성 매트릭스 조회 | TraceabilityLink, RequirementItem | R | - | - |
| FUNC-041 | 추적성 링크 수동 관리 | TraceabilityLink | R | W | D |
| FUNC-050 | 대시보드 KPI 집계 | RequirementItem, QualityDiagnosisResult, QualityReport | R | - | - |
| FUNC-060 | 사용자 설정 조회 | UserSettings | R | - | - |
| FUNC-061 | 사용자 설정 수정 | UserSettings | R | W | - |
| FUNC-062 | 데이터 초기화 | AppState (전체) | - | - | D |

### 9.2 FA ↔ 엔티티 매트릭스

| FA | 기능 영역 | 핵심 엔티티 | 보조 엔티티 |
|----|----------|------------|------------|
| FA-01 | 파일 I/O | RequirementItem | AppState, AppMeta |
| FA-02 | 요구사항 관리 | RequirementItem, ChangeHistoryEntry | TraceabilityLink |
| FA-03 | LLM 자동 상세화 | LLMSession, LLMMessage | RequirementItem (DEV), TraceabilityLink |
| FA-04 | 품질 진단 | QualityDiagnosisResult, QualityIssue | LLMSession, QualityReport |
| FA-05 | 추적성 관리 | TraceabilityLink | RequirementItem |
| FA-06 | 대시보드 | QualityReport | RequirementItem, QualityDiagnosisResult |
| FA-07 | 시스템 설정 | UserSettings | AppState, AppMeta |

---

*본 문서는 `rets_02c_req_schema.json` v3.0.0 및 MD02(PRD), MD03(SRS)와 일관성을 유지하며 작성되었다. 구현 시 본 명세를 준거 기준으로 삼는다.*
