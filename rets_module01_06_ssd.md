# 시스템 화면 설계서 (SSD)
## RETS-MODULE-01: RFP 요구사항 상세화 및 품질 평가 도구

---

| 항목 | 내용 |
|------|------|
| 문서 ID | MD06 |
| 문서명 | 시스템 화면 설계서 (System Screen Design Document) |
| 모듈 ID | RETS-MODULE-01 |
| 모듈명 | RFP 요구사항 상세화 및 품질 평가 도구 |
| 작성 기준일 | 2026-05-07 |
| 버전 | v1.0 |
| 상태 | 초안 (Draft) |
| 참조 문서 | rets_02b_design_requirements.md, rets_02b_design_system_ref.html, MD02(PRD), MD03(SRS), MD05(DDD) |

---

## 목차

1. [화면 설계 개요](#1-화면-설계-개요)
2. [글로벌 레이아웃 및 공통 컴포넌트](#2-글로벌-레이아웃-및-공통-컴포넌트)
3. [화면 목록 및 내비게이션 맵](#3-화면-목록-및-내비게이션-맵)
4. [화면별 상세 설계](#4-화면별-상세-설계)
   - SCR-01: 대시보드
   - SCR-02: 요구사항 목록
   - SCR-03: 요구사항 상세/편집
   - SCR-04: LLM 자동 상세화
   - SCR-05: 품질 진단
   - SCR-06: 추적성 매트릭스
   - SCR-07: 품질 보고서
   - SCR-08: 시스템 설정
5. [사용자 흐름 시나리오](#5-사용자-흐름-시나리오)
6. [디자인 시스템 준수 명세](#6-디자인-시스템-준수-명세)
7. [요구사항 추적 매트릭스](#7-요구사항-추적-매트릭스)

---

## 1. 화면 설계 개요

### 1.1 설계 원칙

| 원칙 | 설명 |
|------|------|
| **단일 HTML 파일** | 모든 화면은 `rets_module01_07_app.html` 단일 파일 내 SPA(Single Page Application) 방식으로 구현. 화면 전환은 DOM 가시성 제어(display 토글)로 처리 |
| **탭 기반 내비게이션** | 최상위 내비게이션은 수평 탭 형태. 현재 활성 화면을 `data-screen` 속성으로 구분 |
| **반응형 레이아웃** | 최소 지원 해상도 1280×768. Flexbox 기반 레이아웃으로 패널 크기 조절 지원 |
| **디자인 시스템 준수** | `rets_02b_design_requirements.md` 및 `rets_02b_design_system_ref.html`의 CSS 변수, 컬러 토큰, 타이포그래피 체계 완전 준수 |
| **접근성** | ARIA 레이블, 키보드 내비게이션, 포커스 링 표시 적용. WCAG 2.1 AA 수준 목표 |
| **상태 반응성** | 모든 비동기 작업(파일 로드, LLM 호출, 진단 실행)에 로딩 인디케이터 및 진행률 표시 적용 |

### 1.2 기술 구현 기준

| 항목 | 기준 |
|------|------|
| 프레임워크 | Vanilla JavaScript (ES2020+). 외부 라이브러리 없음 |
| 스타일 | CSS Custom Properties (CSS 변수) 기반. Inline style 사용 금지 |
| 상태 관리 | 전역 `AppState` 객체 (메모리) + localStorage 동기화 (30초 주기) |
| 이벤트 | 이벤트 위임(Event Delegation) 패턴. 동적 DOM에 addEventListener 남용 금지 |
| 컴포넌트 | 화면별 `render{ScreenName}()` 함수 + 공통 컴포넌트 `renderXxx()` 함수 |
| 아이콘 | SVG 인라인 또는 Unicode 기호. 외부 아이콘 폰트 불허 |

---

## 2. 글로벌 레이아웃 및 공통 컴포넌트

### 2.1 전체 레이아웃 구조

```
┌─────────────────────────────────────────────────────────────────┐
│  #app-header (height: 56px, position: sticky)                    │
│  ┌────────────────┐  ┌───────────────────────────────────────┐  │
│  │  로고/앱명      │  │  탭 내비게이션 (SCR-01~SCR-08)         │  │
│  │  RETS-MODULE-01│  │  [대시보드][요구사항][상세화][품질진단] │  │
│  └────────────────┘  │  [추적성][보고서][설정]                │  │
│                      └───────────────────────────────────────┘  │
│                                           ┌──────────────────┐  │
│                                           │ 저장상태 | 설정   │  │
│                                           └──────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│  #toast-container (position: fixed, top-right)                   │
├─────────────────────────────────────────────────────────────────┤
│  #main-content (flex: 1, overflow: auto)                         │
│                                                                   │
│  ┌ SCR-01 ─────────────────────────────────────────────────────┐ │
│  │  (현재 활성 화면)                                            │ │
│  └──────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  #app-footer (height: 32px)                                      │
│  버전: v1.0.0 | 스키마: v1.0.0 | 마지막 저장: 11:30 | © RETS   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 헤더 컴포넌트 (ID: `#app-header`)

| 요소 ID / 클래스 | 타입 | 레이블 | 동작 | 비고 |
|-----------------|------|--------|------|------|
| `#app-logo` | div | RETS-M01 | 클릭 → SCR-01(대시보드)로 이동 | SVG 로고 + 텍스트 |
| `.nav-tab[data-screen]` | button | 각 화면명 | 클릭 → 해당 화면 활성화. `aria-selected=true` | active 상태: `--color-primary` 언더라인 |
| `#save-status` | span | "저장됨 11:30" / "변경사항 있음" | (표시 전용) | 자동 저장 상태 표시 |
| `#btn-manual-save` | button | 💾 저장 | 클릭 → 즉시 localStorage 저장 실행 | `Ctrl+S` 단축키 지원 |
| `#btn-header-import` | button | 📂 불러오기 | 클릭 → 파일 선택 다이얼로그 오픈 | hidden `<input type="file">` 연결 |

### 2.3 탭 내비게이션

| 탭 순서 | 탭 ID | 레이블 | data-screen 값 | 단축키 |
|---------|-------|--------|----------------|--------|
| 1 | `tab-dashboard` | 📊 대시보드 | `dashboard` | Alt+1 |
| 2 | `tab-requirements` | 📋 요구사항 | `requirements` | Alt+2 |
| 3 | `tab-detailing` | 🤖 자동 상세화 | `detailing` | Alt+3 |
| 4 | `tab-quality` | 🔍 품질 진단 | `quality` | Alt+4 |
| 5 | `tab-traceability` | 🔗 추적성 | `traceability` | Alt+5 |
| 6 | `tab-report` | 📄 보고서 | `report` | Alt+6 |
| 7 | `tab-settings` | ⚙️ 설정 | `settings` | Alt+7 |

### 2.4 공통 컴포넌트 명세

#### 2.4.1 모달 다이얼로그 (`#modal-overlay`)

```
┌──────────────────────────────────────────┐
│  ✕                          #modal-close │
│                                          │
│  [title]                  #modal-title   │
│  ─────────────────────────────────────── │
│                                          │
│  [content area]           #modal-body    │
│                                          │
│  ─────────────────────────────────────── │
│         [취소]  [확인/저장]  #modal-footer │
└──────────────────────────────────────────┘
```

| 속성 | 값 |
|------|---|
| max-width | 720px |
| 배경 오버레이 | rgba(0,0,0,0.5), `role="dialog"`, `aria-modal="true"` |
| ESC 키 | 모달 닫기 |
| 포커스 트랩 | 모달 내부 포커스 순환 (접근성) |

#### 2.4.2 토스트 알림 (`#toast-container`)

| 타입 | 배경색 CSS 변수 | 표시 시간 | 아이콘 |
|------|---------------|----------|--------|
| success | `--color-success` | 3초 | ✅ |
| error | `--color-error` | 5초 (닫기 버튼 포함) | ❌ |
| warning | `--color-warning` | 4초 | ⚠️ |
| info | `--color-info` | 3초 | ℹ️ |

#### 2.4.3 로딩 인디케이터

| 사용 상황 | 컴포넌트 | 구현 방법 |
|----------|---------|----------|
| LLM 호출 중 | 풀스크린 오버레이 + 스피너 + "LLM 응답 대기 중..." 텍스트 + 경과 시간 | `#loading-overlay` |
| 파일 파싱 중 | 인라인 스피너 (버튼 내부 치환) | CSS animation `spin` |
| 품질 진단 배치 처리 | 진행률 바 + "N/M 처리 중" | `#progress-bar` |

#### 2.4.4 공통 오류 메시지 코드

| 오류 코드 | 표시 메시지 | 발생 조건 |
|----------|-----------|----------|
| E-001 | "유효하지 않은 JSON 파일입니다. 파일 내용을 확인해 주세요." | JSON.parse 실패 |
| E-002 | "requirements 키가 없거나 배열 형식이 아닙니다." | 스키마 구조 오류 |
| E-003 | "N개 항목의 req_id 형식이 잘못되었습니다: [목록]" | req_id 패턴 불일치 |
| E-004 | "파일 크기가 10MB를 초과합니다. (현재 N.N MB)" | 파일 크기 초과 |
| E-005 | "LLM 서버 연결에 실패했습니다. 설정 > 릴레이 서버 URL을 확인해 주세요." | Fetch 오류 |
| E-006 | "LLM 응답 시간이 초과되었습니다. (N초)" | 타임아웃 |
| E-007 | "LLM 응답에서 JSON을 파싱할 수 없습니다. 원본 응답을 확인해 주세요." | JSON 파싱 3단계 모두 실패 |
| E-008 | "팀 ID가 설정되지 않았습니다. 설정 화면에서 팀 ID를 입력해 주세요." | team_id 미설정 |

---

## 3. 화면 목록 및 내비게이션 맵

### 3.1 화면 목록

| 화면 ID | 화면명 | 경로(data-screen) | FA | 주요 FUNC | 우선순위 |
|---------|--------|------------------|----|-----------|---------|
| SCR-01 | 대시보드 | `dashboard` | FA-06 | FUNC-050 | MUST |
| SCR-02 | 요구사항 목록 | `requirements` | FA-01, FA-02 | FUNC-001~004, FUNC-010~015 | MUST |
| SCR-03 | 요구사항 상세/편집 | `req-detail` (모달 또는 사이드 패널) | FA-02 | FUNC-011~014 | MUST |
| SCR-04 | LLM 자동 상세화 | `detailing` | FA-03 | FUNC-020~023 | MUST |
| SCR-05 | 품질 진단 | `quality` | FA-04 | FUNC-030~033 | MUST |
| SCR-06 | 추적성 매트릭스 | `traceability` | FA-05 | FUNC-040~041 | SHOULD |
| SCR-07 | 품질 보고서 | `report` | FA-04, FA-06 | FUNC-033, FUNC-050 | SHOULD |
| SCR-08 | 시스템 설정 | `settings` | FA-07 | FUNC-060~062 | MUST |

### 3.2 내비게이션 맵

```
[앱 진입]
    │
    ├──▶ SCR-01: 대시보드
    │         ├── KPI 카드 클릭 ──▶ SCR-02 (필터 적용)
    │         ├── "상세화 시작" 버튼 ──▶ SCR-04
    │         ├── "품질 진단" 버튼 ──▶ SCR-05
    │         └── "최신 보고서" 링크 ──▶ SCR-07
    │
    ├──▶ SCR-02: 요구사항 목록
    │         ├── 행 클릭 ──▶ SCR-03 (사이드 패널 슬라이드 인)
    │         ├── "신규" 버튼 ──▶ SCR-03 (신규 생성 모드)
    │         ├── "상세화" 버튼 ──▶ SCR-04 (선택 항목 전달)
    │         ├── "품질 진단" 버튼 ──▶ SCR-05 (선택 항목 전달)
    │         └── "파일 업로드" ──▶ (파일 선택 다이얼로그 → 결과 반영)
    │
    ├──▶ SCR-03: 요구사항 상세/편집 (사이드 패널)
    │         ├── 저장 ──▶ SCR-02 (목록 갱신)
    │         ├── "상세화 요청" ──▶ SCR-04 (현재 항목 전달)
    │         └── 닫기 ──▶ SCR-02 복귀
    │
    ├──▶ SCR-04: LLM 자동 상세화
    │         ├── 결과 Accept ──▶ SCR-02 (새 DEV 항목 추가)
    │         └── 결과 확인 ──▶ SCR-03 (생성된 DEV 항목 상세)
    │
    ├──▶ SCR-05: 품질 진단
    │         ├── 진단 결과 항목 클릭 ──▶ SCR-03 (해당 요구사항 상세)
    │         └── "보고서 생성" ──▶ SCR-07
    │
    ├──▶ SCR-06: 추적성 매트릭스
    │         └── 셀 클릭 ──▶ SCR-03 (해당 요구사항 상세)
    │
    ├──▶ SCR-07: 품질 보고서
    │         └── 요구사항 링크 ──▶ SCR-03 (해당 요구사항 상세)
    │
    └──▶ SCR-08: 시스템 설정
              └── 저장 ──▶ 현재 화면 유지 (설정 적용)
```

---

## 4. 화면별 상세 설계

---

### SCR-01: 대시보드

**화면 ID:** SCR-01  
**관련 FA:** FA-06  
**관련 FUNC:** FUNC-050  
**관련 FR:** FR-13 (대시보드 KPI), FR-14 (통계 시각화)  
**화면 목적:** 요구사항 현황, 품질 지표, 최근 활동을 한눈에 파악할 수 있는 요약 화면

#### 4.1.1 레이아웃 와이어프레임

```
┌─────────────────────────────────────────────────────────────────┐
│  대시보드          [+ 요구사항 불러오기]  [🤖 상세화 시작]  [🔍 진단] │
├──────────┬──────────┬──────────┬──────────────────────────────── │
│  전체    │  RFP     │  DEV     │  마지막 저장: 2026-05-07 11:30   │
│  47건    │  20건    │  27건    │  자동저장: 30초                  │
│ [KPI 카드]│[KPI 카드]│[KPI 카드]│                                 │
├──────────┴──────────┴──────────┼──────────────────────────────── │
│  품질 현황 (도넛 차트)           │  요구사항 우선순위 분포 (바 차트) │
│  ┌───────────────────────────┐ │  ┌──────────────────────────┐  │
│  │  A: 5   B: 18            │ │  │  MUST: 30 | SHOULD: 12  │  │
│  │  C: 17  D: 7             │ │  │  COULD: 4  | WONT: 1    │  │
│  │  평균: 72.3점             │ │  └──────────────────────────┘  │
│  └───────────────────────────┘ │                                 │
├────────────────────────────────┼──────────────────────────────── │
│  개발 상태 현황                  │  최근 활동 (Recent Activities)  │
│  Todo: 18  InProgress: 5       │  ▸ 2026-05-07 LLM 상세화 완료  │
│  Done: 3   Blocked: 1         │  ▸ 2026-05-07 품질 진단 실행   │
│                                │  ▸ 2026-05-07 RFP 파일 업로드  │
└────────────────────────────────┴──────────────────────────────── │
```

#### 4.1.2 UI 요소 명세

| 요소 ID | 타입 | 레이블 / 내용 | 동작 / 상태 변화 | 데이터 소스 | 비고 |
|---------|------|-------------|----------------|------------|------|
| `#btn-import-rfp` | button | 📂 요구사항 불러오기 | 클릭 → `#file-input-rfp` trigger → 파일 선택 다이얼로그 | - | FUNC-001 |
| `#btn-go-detailing` | button (primary) | 🤖 상세화 시작 | 클릭 → SCR-04 이동 | - | 요구사항 0건이면 disabled |
| `#btn-go-quality` | button (secondary) | 🔍 품질 진단 | 클릭 → SCR-05 이동 | - | 요구사항 0건이면 disabled |
| `#kpi-total` | div (클릭 가능) | 전체 N건 | 클릭 → SCR-02 이동 (필터 없음) | `AppState.meta.total_requirements` | 카드 호버 시 강조 |
| `#kpi-rfp` | div (클릭 가능) | RFP N건 | 클릭 → SCR-02 이동 (req_type=RFP 필터) | `AppState.meta.total_rfp` | |
| `#kpi-dev` | div (클릭 가능) | DEV N건 | 클릭 → SCR-02 이동 (req_type=DEV 필터) | `AppState.meta.total_dev` | |
| `#kpi-approved` | div | 승인됨 N건 | 클릭 → SCR-02 (req_status=Approved 필터) | requirements 집계 | |
| `#chart-quality-grade` | canvas | 등급 분포 도넛 차트 | 세그먼트 클릭 → SCR-05 (해당 등급 필터) | 최신 QualityDiagnosisResult 집계 | Pure Canvas API 사용 |
| `#chart-priority` | canvas | 우선순위 분포 가로 바 차트 | 바 클릭 → SCR-02 (priority 필터) | requirements 집계 | |
| `#chart-dev-status` | canvas | 개발 상태 분포 | 세그먼트 클릭 → SCR-02 (dev_status 필터) | DEV requirements 집계 | |
| `#recent-activities` | ul | 최근 활동 목록 (최대 10건) | 활동 항목 클릭 → 관련 화면 이동 | LLMSession, change_history 기반 | 타임스탬프 포함 |
| `#last-saved-info` | span | "마지막 저장: HH:MM" | (표시 전용) | AppState.meta.last_saved_at | |

#### 4.1.3 데이터 없음 상태 (Empty State)

요구사항이 0건일 때 차트 영역 대신:

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│     📂  시작하려면 RFP 요구사항 파일을 불러오세요.    │
│                                                     │
│         [RFP JSON 파일 불러오기]                    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### SCR-02: 요구사항 목록

**화면 ID:** SCR-02  
**관련 FA:** FA-01, FA-02  
**관련 FUNC:** FUNC-001~004, FUNC-010, FUNC-012, FUNC-014, FUNC-015  
**관련 FR:** FR-01~04, FR-05~08, FR-16~17  
**화면 목적:** 전체 요구사항의 목록 조회, 검색/필터, 파일 I/O, 일괄 작업 수행

#### 4.2.1 레이아웃 와이어프레임

```
┌─────────────────────────────────────────────────────────────────┐
│ 요구사항 목록 (47건)    [+ 신규]  [📂 RFP 업로드]  [💾 내보내기▼] │
├─────────────────────────────────────────────────────────────────┤
│ 🔍 [검색어 입력...]                                              │
│ 필터: [유형▼RFP/DEV/전체] [우선순위▼] [상태▼] [도메인▼]  [초기화] │
├──────────────────────────────────────────────────────────────── │
│ ☐  REQ ID          │ 요구사항명          │ 유형│우선│상태 │품질 │
│────────────────────┼─────────────────────┼─────┼────┼─────┼─────│
│ ☑  RFP_001         │ 파일 I/O 기능       │ RFP │MUST│Draft│  -  │
│ ☐  DEV_FR_001      │ RFP JSON 파일 업로드│ DEV │MUST│Draft│ B78 │
│ ☐  DEV_FR_002      │ DEV JSON 파일 업로드│ DEV │MUST│Done │ A91 │
│  ...               │  ...               │ ... │ ...│ ... │ ... │
├─────────────────────────────────────────────────────────────────┤
│ 1건 선택됨  [🤖 상세화] [🔍 진단] [🗑 삭제]   ◀ 1/5 ▶ (10개씩▼) │
└─────────────────────────────────────────────────────────────────┘
     │ 행 클릭 시
     ▼
     SCR-03 사이드 패널 슬라이드 인 (우측)
```

#### 4.2.2 UI 요소 명세

| 요소 ID | 타입 | 레이블 / 내용 | 동작 | 데이터 소스 | 비고 |
|---------|------|-------------|------|------------|------|
| `#btn-new-req` | button (primary) | + 신규 | 클릭 → SCR-03 패널 오픈 (빈 폼 생성 모드) | - | FUNC-012 |
| `#btn-import-rfp` | button | 📂 RFP 업로드 | 클릭 → 파일 선택. JSON 파싱 후 목록 갱신 | - | FUNC-001 |
| `#btn-import-dev` | button | 📂 DEV 업로드 | 클릭 → 파일 선택. JSON 파싱 후 목록 갱신 | - | FUNC-002 |
| `#btn-export` | button (dropdown) | 💾 내보내기 ▼ | 클릭 → 드롭다운: [JSON 전체], [JSON 선택], [CSV 전체], [CSV 선택] | - | FUNC-003, FUNC-004 |
| `#search-input` | text input | 🔍 검색어 입력... | 입력 300ms debounce → 목록 실시간 필터 | requirements | `placeholder="req_id, 요구사항명, 설명 검색"` |
| `#filter-req-type` | select | 유형 | 변경 → 필터 적용 | - | 옵션: 전체 / RFP / DEV |
| `#filter-priority` | select | 우선순위 | 변경 → 필터 적용 | - | 옵션: 전체 / MUST / SHOULD / COULD / WONT |
| `#filter-status` | select | 상태 | 변경 → 필터 적용 | - | 옵션: 전체 / Draft / InReview / Approved / Rejected / Deferred |
| `#filter-domain` | select | 도메인 | 변경 → 필터 적용 | - | 옵션: 전체 + domain_area enum 값 |
| `#btn-clear-filters` | button (text) | 초기화 | 클릭 → 모든 필터/검색어 초기화 | - | 필터 적용 중일 때만 표시 |
| `#req-table` | table | 요구사항 목록 | 행 클릭 → SCR-03 패널 오픈 (해당 req_id) | filtered requirements | |
| `#chk-select-all` | checkbox | ☐ 전체 선택 | 변경 → 현재 페이지 전체 선택/해제 | - | 헤더 행 |
| `.req-row` | tr | 요구사항 행 | 클릭 → SCR-03 패널. 체크박스 클릭 → 선택 상태 토글 | RequirementItem | `data-req-id` 속성 |
| `.req-priority-badge` | span | MUST / SHOULD / COULD | (표시) | priority | 색상: MUST=green, SHOULD=yellow, COULD=blue |
| `.req-status-badge` | span | Draft / Approved 등 | (표시) | req_status | 색상: Approved=green, Draft=gray |
| `.req-quality-badge` | span | A91 / B78 등 | 클릭 → SCR-05 해당 항목 포커스 | `_quality_score`, `quality_grade` | 미진단 시 `-` 표시 |
| `#bulk-action-bar` | div | N건 선택됨 + 일괄 작업 버튼 | 선택 항목 있을 때만 표시 | - | |
| `#btn-bulk-detailing` | button | 🤖 상세화 | 클릭 → 선택된 RFP 항목을 SCR-04로 전달 | - | RFP 항목 미선택 시 disabled |
| `#btn-bulk-quality` | button | 🔍 진단 | 클릭 → 선택된 항목을 SCR-05로 전달 | - | FUNC-030 |
| `#btn-bulk-delete` | button (danger) | 🗑 삭제 | 클릭 → 확인 모달 → 선택 항목 일괄 삭제 | - | FUNC-014 |
| `#pagination` | div | ◀ N/M ▶ | 이전/다음 페이지 이동 | - | 페이지당 건수: 10 / 25 / 50 |

#### 4.2.3 테이블 컬럼 정의

| 컬럼 순서 | 컬럼명 | 데이터 필드 | 정렬 | 최소 너비 | 설명 |
|----------|--------|-----------|------|-----------|------|
| 1 | ☐ | (체크박스) | - | 40px | 행 선택 |
| 2 | REQ ID | `req_id` | 클릭 정렬 ↑↓ | 180px | |
| 3 | 요구사항명 | `req_name` | 클릭 정렬 | flex(1) | 넘치면 말줄임 |
| 4 | 유형 | `req_type` | 클릭 정렬 | 60px | RFP / DEV 배지 |
| 5 | 우선순위 | `priority` | 클릭 정렬 | 80px | 배지 스타일 |
| 6 | 상태 | `req_status` | 클릭 정렬 | 90px | 배지 스타일 |
| 7 | 품질 | `_quality_score` + `quality_grade` | 클릭 정렬 | 70px | 배지 스타일 |
| 8 | 수정일 | `last_modified_date` | 클릭 정렬 | 100px | |
| 9 | 작업 | (버튼) | - | 60px | 수정 / 삭제 아이콘 버튼 |

---

### SCR-03: 요구사항 상세/편집 (사이드 패널)

**화면 ID:** SCR-03  
**구현 방식:** 화면 우측에서 슬라이드 인하는 사이드 패널 (`#req-detail-panel`). 너비 580px. 배경 오버레이 없음(목록과 동시 표시).  
**관련 FA:** FA-02  
**관련 FUNC:** FUNC-011, FUNC-012, FUNC-013, FUNC-014  
**관련 FR:** FR-05~08, FR-09

#### 4.3.1 레이아웃 와이어프레임

```
                    ┌───────────────────────────────────────────┐
                    │ ✕ [닫기]  REQ ID: DEV_FR_001      [수정]  │
                    ├───────────────────────────────────────────┤
                    │ 탭: [기본정보] [승인기준] [이력] [연관정보]  │
                    ├───────────────────────────────────────────┤
                    │ [기본정보 탭 활성 시]                       │
                    │                                           │
                    │ req_type  ○ RFP  ● DEV                   │
                    │ req_id    DEV_FR_001  [자동채번]           │
                    │ req_level [1▼]                            │
                    │ req_name  [RFP JSON 파일 업로드 처리    ]  │
                    │ priority  [MUST      ▼]                   │
                    │ req_status[Draft     ▼]                   │
                    │ domain    [데이터관리  ▼]                   │
                    │                                           │
                    │ description                               │
                    │ ┌─────────────────────────────────────┐  │
                    │ │ FileReader API를 사용하여...          │  │
                    │ └─────────────────────────────────────┘  │
                    │                                           │
                    │ source_rfp_ids: [RFP_001] [+추가]        │
                    │ dev_status: [Todo ▼]                      │
                    │ story_points: [3]   effort: [0.5 MD]      │
                    │                                           │
                    │ 품질 점수: ████████░░ B (78점)            │
                    ├───────────────────────────────────────────┤
                    │ [🤖 LLM 상세화 요청] [🔍 품질 진단]        │
                    │                    [취소] [저장]          │
                    └───────────────────────────────────────────┘
```

#### 4.3.2 UI 요소 명세 — 기본정보 탭

| 요소 ID | 타입 | 레이블 | 필드 | 편집 가능 | 유효성 |
|---------|------|--------|------|----------|--------|
| `#detail-close` | button | ✕ 닫기 | - | - | ESC 키도 닫기 |
| `#detail-req-id-display` | span | req_id 값 | `req_id` | 읽기 전용 | |
| `#btn-detail-edit` | button | 수정 | - | - | 클릭 시 모든 필드 편집 모드 활성화 |
| `#detail-tab-basic` | button | 기본정보 | - | - | 활성 탭 표시 |
| `#detail-tab-criteria` | button | 승인기준 | - | - | |
| `#detail-tab-history` | button | 이력 | - | - | |
| `#detail-tab-relations` | button | 연관정보 | - | - | |
| `#field-req-type` | radio group | RFP / DEV | `req_type` | 신규 시만 가능 | 수정 시 변경 불가 |
| `#field-req-id` | text input | REQ ID | `req_id` | 신규 시만 가능 | 패턴: `^(RFP\|DEV_(FR\|NFR))_\d{3}(_\d{2}){0,4}$` |
| `#btn-auto-id` | button | 자동채번 | - | - | 클릭 → 다음 사용 가능 ID 자동 입력 |
| `#field-req-level` | select | 수준 | `req_level` | ✅ | 옵션: 1~5. req_id 깊이와 불일치 시 경고 |
| `#field-req-name` | text input | 요구사항명 | `req_name` | ✅ | maxlength=200, 필수 |
| `#field-priority` | select | 우선순위 | `priority` | ✅ | 옵션: MUST / SHOULD / COULD / WONT |
| `#field-req-status` | select | 상태 | `req_status` | ✅ | 옵션: Draft / InReview / Approved 등 |
| `#field-domain-area` | select | 도메인 | `domain_area` | ✅ | domain_area enum 옵션 |
| `#field-req-category` | select | 카테고리 | `req_category` | ✅ | req_category enum 옵션 |
| `#field-description` | textarea | 설명 | `description` | ✅ | rows=6, maxlength=2000, 글자수 표시 |
| `#field-acceptance-status` | select | 수용여부 | `acceptance_status` | ✅ | RFP 타입일 때만 표시 |
| `#field-source-rfp-ids` | tag input | 근거 RFP ID | `source_rfp_ids` | ✅ | DEV 타입일 때만 표시. 태그 방식 입력 |
| `#field-dev-status` | select | 개발 상태 | `dev_status` | ✅ | DEV 타입일 때만 표시 |
| `#field-story-points` | number input | 스토리포인트 | `story_points` | ✅ | DEV 타입일 때만 표시. 피보나치 권장 표시 |
| `#field-effort` | number input | 공수(MD) | `estimated_effort` | ✅ | min=0, step=0.5 |
| `#quality-score-bar` | div (진행률 바) | 품질 점수 시각화 | `_quality_score` | 읽기 전용 | 색상: A=green, B=blue, C=yellow, D=red |
| `#btn-request-detailing` | button | 🤖 LLM 상세화 요청 | - | - | RFP 타입일 때만 표시. SCR-04로 이동 |
| `#btn-request-quality` | button | 🔍 품질 진단 | - | - | FUNC-030 |
| `#btn-detail-cancel` | button | 취소 | - | - | 변경사항 있으면 확인 다이얼로그 |
| `#btn-detail-save` | button (primary) | 저장 | - | - | 유효성 검사 통과 시만 활성화 |

#### 4.3.3 UI 요소 명세 — 승인기준 탭

| 요소 ID | 타입 | 레이블 | 동작 | 비고 |
|---------|------|--------|------|------|
| `#acceptance-criteria-list` | ul | 완료 기준 목록 | 항목 클릭 → 인라인 편집 | `acceptance_criteria[]` |
| `#btn-add-criteria` | button | + 기준 추가 | 클릭 → 빈 입력 행 추가 | max 10건 |
| `#btn-remove-criteria` | button (icon) | × | 클릭 → 해당 항목 삭제 | 각 항목 우측 |
| `#field-verification-method` | select | 검증 방법 | 변경 → 저장 대기 | 옵션: 테스트 / 검사 / 분석 / 시연 / 검토 |
| `#metrics-list` | ul | 측정 지표 목록 | 항목 클릭 → 인라인 편집 | `metrics[]` |
| `#btn-add-metric` | button | + 지표 추가 | 클릭 → 빈 입력 행 추가 | max 5건 |

#### 4.3.4 UI 요소 명세 — 이력 탭

| 요소 ID | 타입 | 레이블 | 동작 | 비고 |
|---------|------|--------|------|------|
| `#change-history-list` | table | 변경 이력 목록 | (표시 전용) | 버전 / 날짜 / 작성자 / 내용 / 변경필드 컬럼 |
| `#current-version` | span | 현재 버전 표시 | (표시 전용) | `req_version` |

#### 4.3.5 UI 요소 명세 — 연관정보 탭

| 요소 ID | 타입 | 레이블 | 동작 | 비고 |
|---------|------|--------|------|------|
| `#related-rfp-list` | ul | 연관 RFP 요구사항 | 항목 클릭 → SCR-03 해당 항목 | TraceabilityLink rfp_to_dev |
| `#related-dev-list` | ul | 연관 DEV 요구사항 | 항목 클릭 → SCR-03 해당 항목 | |
| `#related-tc-list` | ul | 연관 테스트케이스 | (표시 전용) | TraceabilityLink dev_to_test |
| `#related-screen-list` | ul | 연관 화면 | (표시 전용) | TraceabilityLink dev_to_screen |
| `#btn-add-trace-link` | button | + 추적 링크 추가 | 클릭 → 링크 추가 미니 모달 | FUNC-041 |

---

### SCR-04: LLM 자동 상세화

**화면 ID:** SCR-04  
**관련 FA:** FA-03  
**관련 FUNC:** FUNC-020~023  
**관련 FR:** FR-09~11  
**화면 목적:** 선택된 RFP 요구사항을 LLM을 통해 DEV 요구사항으로 자동 상세화하고, Human-in-the-Loop 방식으로 결과를 검토·수용

#### 4.4.1 레이아웃 와이어프레임

```
┌────────────────────────────────┬──────────────────────────────────┐
│  [좌측 패널] 입력 선택         │  [우측 패널] 상세화 결과          │
│                                │                                  │
│  상세화 대상 RFP 요구사항:     │  (대기 상태)                     │
│  ┌────────────────────────┐   │                                  │
│  │ ☑ RFP_001 파일 I/O 기능│   │  ℹ️ 좌측에서 RFP 요구사항을      │
│  │ ☑ RFP_002 요구사항 관리│   │     선택하고 [상세화 실행] 버튼을  │
│  │ ☐ RFP_003 LLM 자동상세 │   │     클릭하세요.                  │
│  │ ...                    │   │                                  │
│  └────────────────────────┘   │                                  │
│  2건 선택됨                   │  (실행 후 결과 표시 영역)         │
│                                │                                  │
│  프롬프트 전략:                │  [DEV_FR_001] RFP JSON 파일 업로드│
│  [표준 상세화 ▼]               │  ┌─────────────────────────────┐ │
│                                │  │ req_name: ...               │ │
│  배치 크기: [10 ▼]             │  │ description: ...            │ │
│                                │  │ acceptance_criteria: [...]  │ │
│  [🤖 상세화 실행]              │  └─────────────────────────────┘ │
│  (선택 없으면 disabled)        │  [✅ 수용]  [✏️ 편집]  [❌ 거부]  │
│                                │                                  │
│  ─────────────────────────── │  [DEV_FR_002] ...               │
│  세션 정보:                    │                                  │
│  상태: 대기 중                 │  ─────────────────────────────── │
│  세션 ID: -                   │  [전체 수용] [전체 거부] [일부 수용] │
└────────────────────────────────┴──────────────────────────────────┘
```

#### 4.4.2 UI 요소 명세

| 요소 ID | 타입 | 레이블 | 동작 | 데이터 소스 | 비고 |
|---------|------|--------|------|------------|------|
| `#detailing-rfp-list` | ul (체크리스트) | RFP 요구사항 선택 목록 | 체크 → 선택 상태 토글 | requirements (req_type=RFP) | FUNC-020 |
| `#detailing-selected-count` | span | N건 선택됨 | (표시 전용) | 선택 항목 수 | |
| `#detailing-prompt-strategy` | select | 프롬프트 전략 | 변경 → 프롬프트 템플릿 전환 | - | 옵션: 표준 상세화 / 심층 분해 / 간소화 |
| `#detailing-batch-size` | select | 배치 크기 | 변경 → UserSettings 업데이트 | `UserSettings.max_requirements_per_batch` | 옵션: 1 / 5 / 10 / 20 |
| `#btn-run-detailing` | button (primary, large) | 🤖 상세화 실행 | 클릭 → LLM 세션 초기화 → 메시지 전송 | - | FUNC-020~021. 선택 없으면 disabled |
| `#session-status` | div | 세션 상태 표시 | (표시 전용) | LLMSession.status | 상태별 색상: active=blue, completed=green, failed=red |
| `#session-id-display` | span | 세션 ID 표시 | (표시 전용) | LLMSession.session_id | |
| `#detailing-loading-overlay` | div | LLM 응답 대기 중... + 경과시간 | 실행 중 표시 | - | 타임아웃 카운트다운 포함 |
| `#detailing-results` | div | 상세화 결과 카드 목록 | - | 생성된 DEV RequirementItem[] | FUNC-022 |
| `.result-card` | div | 개별 DEV 요구사항 카드 | - | RequirementItem | `data-req-id` 속성. 배경: `--color-surface-elevated` |
| `.btn-accept-single` | button (success) | ✅ 수용 | 클릭 → 해당 DEV 항목 req_status 유지, TraceabilityLink 생성, 카드에 "수용됨" 표시 | - | FUNC-022 |
| `.btn-edit-single` | button | ✏️ 편집 | 클릭 → 해당 결과 카드를 인라인 편집 모드로 전환 | - | 편집 후 저장 가능 |
| `.btn-reject-single` | button (danger) | ❌ 거부 | 클릭 → 해당 DEV 항목 AppState에서 제거, 카드에 "거부됨" 표시 | - | |
| `#btn-accept-all` | button (success) | ✅ 전체 수용 | 클릭 → 모든 대기 중 결과 수용 | - | |
| `#btn-reject-all` | button (danger) | ❌ 전체 거부 | 클릭 → 확인 다이얼로그 → 모든 대기 중 결과 거부 | - | |
| `#btn-accept-selected` | button | 선택 항목 수용 | 클릭 → 체크된 결과만 수용 | - | |
| `#raw-response-toggle` | button | 🔍 LLM 원본 응답 보기 | 클릭 → 원본 텍스트 토글 표시 | - | `UserSettings.show_llm_raw_response=true` 시 기본 표시 |

#### 4.4.3 결과 카드 내부 레이아웃

```
┌───────────────────────────────────────────────────────────┐
│  DEV_FR_001  [🤖 LLM 생성]  [수용 대기 중]               │
│  ─────────────────────────────────────────────────────── │
│  req_name: RFP JSON 파일 업로드 처리                      │
│  priority: MUST  |  functional_class: 기능  |  level: 2  │
│  ─────────────────────────────────────────────────────── │
│  description:                                            │
│  FileReader API를 사용하여 JSON 파일을 비동기로 읽고...    │
│  ─────────────────────────────────────────────────────── │
│  acceptance_criteria:                                    │
│  • 파일 선택 즉시 FileReader.readAsText()가 호출된다      │
│  • JSON.parse() 실패 시 '유효하지 않은 JSON 파일' 오류...  │
│  ─────────────────────────────────────────────────────── │
│  source_rfp_ids: RFP_001                                 │
│  story_points: 3  |  estimated_effort: 0.5 MD            │
│  ─────────────────────────────────────────────────────── │
│                   [✅ 수용]  [✏️ 편집]  [❌ 거부]          │
└───────────────────────────────────────────────────────────┘
```

---

### SCR-05: 품질 진단

**화면 ID:** SCR-05  
**관련 FA:** FA-04  
**관련 FUNC:** FUNC-030~033  
**관련 FR:** FR-12, FR-14~15  
**화면 목적:** 요구사항의 3차원 품질 진단 실행, 이슈 하이라이팅, 중복/충돌/누락 탐지 결과 확인

#### 4.5.1 레이아웃 와이어프레임

```
┌─────────────────────────────────────────────────────────────────┐
│ 품질 진단     [🔍 전체 진단 실행]  [선택 항목 진단]  [📄 보고서 생성]│
├─────────────────────────────────────────────────────────────────┤
│ 필터: [등급▼A/B/C/D/전체] [이슈유형▼] [심각도▼]  [미진단만 보기] │
├──────────────────────────────────────────────────────────────── │
│ REQ ID       │ 요구사항명     │ 종합점수│ 명확성│ 일관성│ 검증성 │
│──────────────┼───────────────┼────────┼───────┼───────┼───────  │
│ DEV_FR_001   │ RFP JSON 업로드│ B  78  │  75% │  82% │  79%  │
│              │ ▸ [모호 M] desc의 '비동기'가 불명확              │
│              │ ▸ [미검 Mi] metrics 항목 없음                    │
│ DEV_FR_002   │ DEV JSON 업로드│ A  91  │  90% │  93% │  90%  │
│ RFP_001      │ 파일 I/O 기능  │ 미진단 │  -   │  -   │  -    │
│  ...         │               │       │      │      │       │
├─────────────────────────────────────────────────────────────────┤
│  진단 요약:  평균 72.3점  A:5  B:18  C:17  D:7  이슈: 31건     │
└─────────────────────────────────────────────────────────────────┘
                  [행 클릭 → 이슈 드롭다운 확장 / SCR-03 이동]
```

#### 4.5.2 UI 요소 명세

| 요소 ID | 타입 | 레이블 | 동작 | 데이터 소스 | 비고 |
|---------|------|--------|------|------------|------|
| `#btn-run-all-quality` | button (primary) | 🔍 전체 진단 실행 | 클릭 → 전체 요구사항 배치 품질 진단. 진행률 표시 | requirements | FUNC-030 |
| `#btn-run-selected-quality` | button | 선택 항목 진단 | 클릭 → SCR-02 선택 항목만 진단 | - | 선택 없으면 disabled |
| `#btn-generate-report` | button | 📄 보고서 생성 | 클릭 → QualityReport 생성 → SCR-07 이동 | qualityResults | FUNC-033 |
| `#quality-filter-grade` | select | 등급 필터 | 변경 → 목록 필터 | - | 옵션: 전체 / A / B / C / D |
| `#quality-filter-issue` | select | 이슈유형 필터 | 변경 → 목록 필터 | - | ambiguity / inconsistency 등 |
| `#quality-filter-severity` | select | 심각도 필터 | 변경 → 목록 필터 | - | Critical / Major / Minor / Trivial |
| `#quality-filter-undiagnosed` | checkbox | 미진단만 보기 | 변경 → 진단 결과 없는 항목만 표시 | - | |
| `#quality-table` | table | 품질 진단 결과 테이블 | 행 클릭 → 이슈 드롭다운 토글 | QualityDiagnosisResult[] | |
| `.quality-row` | tr | 진단 결과 행 | 클릭 → 이슈 상세 하위 행 확장/축소 | QualityDiagnosisResult | |
| `.quality-score-cell` | td | 등급 + 점수 표시 | (표시 전용) | `quality_grade`, `overall_score` | 배경색: A=green, B=blue, C=yellow, D=red |
| `.quality-dimension-bar` | div | 명확성/일관성/검증성 점수 바 | (표시 전용) | `clarity_score` 등 | 미니 진행률 바 |
| `.issue-row` | tr | 이슈 상세 행 (확장 시) | 클릭 → SCR-03 해당 req 포커스 | QualityIssue[] | 심각도별 배경색: Critical=red |
| `.issue-severity-badge` | span | Critical / Major 등 | (표시 전용) | `severity` | 색상 코딩 |
| `.issue-suggestion-text` | span | 개선 제안 텍스트 | (표시 전용) | `suggestion` | |
| `#quality-summary-bar` | div | 진단 요약 정보 | (표시 전용) | qualityResults 집계 | 평균점수 + 등급분포 + 이슈 총건수 |
| `#quality-progress-bar` | div | 배치 진단 진행률 | 진단 실행 중 표시 | - | "N/M 처리 중 (N%)" |

#### 4.5.3 이슈 하이라이팅 규칙

| 이슈 타입 | 하이라이팅 방식 | CSS 클래스 |
|----------|--------------|-----------|
| ambiguity (모호성) | 해당 텍스트 노란색 배경 밑줄 | `.highlight-ambiguity` |
| inconsistency (비일관성) | 해당 텍스트 주황색 배경 밑줄 | `.highlight-inconsistency` |
| unverifiable (검증불가) | 해당 텍스트 분홍색 배경 밑줄 | `.highlight-unverifiable` |
| missing (누락) | 빈 필드 빨간색 테두리 | `.highlight-missing` |
| duplicate (중복) | 해당 행 회색 배경 + 중복 뱃지 | `.highlight-duplicate` |
| conflict (충돌) | 해당 행 빨간색 좌측 보더 + 충돌 뱃지 | `.highlight-conflict` |

---

### SCR-06: 추적성 매트릭스

**화면 ID:** SCR-06  
**관련 FA:** FA-05  
**관련 FUNC:** FUNC-040~041  
**관련 FR:** FR-13  
**화면 목적:** RFP ↔ DEV 요구사항 간 추적 관계를 매트릭스(2D 그리드)로 시각화하고 수동 편집

#### 4.6.1 레이아웃 와이어프레임

```
┌─────────────────────────────────────────────────────────────────┐
│ 추적성 매트릭스   [보기: 매트릭스▼/목록]  [미승인 링크 강조]  [+ 링크 추가]│
├─────────────────────────────────────────────────────────────────┤
│ 행: [RFP 요구사항 ▼]  열: [DEV 요구사항 ▼]  필터: [승인됨만 ▼] │
├─────────────┬────────────┬────────────┬────────────┬───────────  │
│ (RFP \ DEV) │ DEV_FR_001 │ DEV_FR_002 │ DEV_FR_003 │ 커버리지 │
├─────────────┼────────────┼────────────┼────────────┼───────────  │
│ RFP_001     │  ●(95%)    │    ●(80%)  │     -      │  2/3    │
│ RFP_002     │     -      │     -      │  ○(미승인) │  0/3    │
│ RFP_003     │     -      │     -      │    ●(90%)  │  1/3    │
├─────────────┴────────────┴────────────┴────────────┴───────────  │
│ ● 승인됨  ○ 미승인  - 링크없음     커버리지: 3/9 (33%)          │
└─────────────────────────────────────────────────────────────────┘
         [셀 클릭 → 링크 상세 / 링크 추가 팝오버]
```

#### 4.6.2 UI 요소 명세

| 요소 ID | 타입 | 레이블 | 동작 | 데이터 소스 | 비고 |
|---------|------|--------|------|------------|------|
| `#trace-view-toggle` | select | 보기 방식 | 변경 → 매트릭스 / 목록 뷰 전환 | - | 매트릭스 / 목록 |
| `#trace-row-type` | select | 행 기준 | 변경 → 행/열 기준 전환 | - | RFP 요구사항 / DEV 요구사항 |
| `#trace-col-type` | select | 열 기준 | 변경 → 연동 갱신 | - | |
| `#trace-filter-approved` | select | 승인 필터 | 변경 → 표시 범위 필터 | - | 전체 / 승인됨만 / 미승인만 |
| `#btn-highlight-unnapproved` | button | 미승인 링크 강조 | 클릭 → 미승인 셀 노란색 배경 강조 토글 | - | |
| `#btn-add-trace-link` | button | + 링크 추가 | 클릭 → 링크 추가 모달 오픈 | - | FUNC-041 |
| `#trace-matrix` | table | 추적성 매트릭스 | - | TraceabilityLink[] | 가로 스크롤 지원 |
| `.trace-cell-linked` | td | ●(신뢰도%) | 클릭 → 링크 상세 팝오버 | TraceabilityLink | `data-link-id` 속성 |
| `.trace-cell-unapproved` | td | ○(미승인) | 클릭 → 승인/거부 팝오버 | TraceabilityLink | 노란색 배경 |
| `.trace-cell-empty` | td | - | 클릭 → 링크 추가 팝오버 | - | |
| `.trace-row-coverage` | td | N/M | (표시 전용) | TraceabilityLink 집계 | 빨간색: 0/M |
| `#trace-coverage-summary` | div | 전체 커버리지 요약 | (표시 전용) | TraceabilityLink 집계 | "커버리지: N/M (N%)" |

#### 4.6.3 링크 상세 팝오버 (셀 클릭 시)

```
┌─────────────────────────────────────────┐
│  링크 상세                    ✕         │
│  link_id: TL-20260507-0001             │
│  관계: derives (RFP→DEV)               │
│  신뢰도: 95% [LLM 자동 생성]           │
│  메모: "LLM 자동 생성 — 검토 필요"     │
│  ─────────────────────────────────────  │
│  [✅ 승인]  [✏️ 메모 수정]  [🗑 삭제]   │
└─────────────────────────────────────────┘
```

---

### SCR-07: 품질 보고서

**화면 ID:** SCR-07  
**관련 FA:** FA-04, FA-06  
**관련 FUNC:** FUNC-033, FUNC-050  
**관련 FR:** FR-15  
**화면 목적:** 집계된 품질 보고서 조회 및 Markdown/CSV 내보내기

#### 4.7.1 레이아웃 와이어프레임

```
┌─────────────────────────────────────────────────────────────────┐
│ 품질 보고서   [📄 Markdown 내보내기]  [📊 CSV 내보내기]  [+ 새 보고서]│
├──────────────────────────────────┬──────────────────────────────  │
│ 보고서 목록                      │  보고서 상세                   │
│                                  │                               │
│ RPT-20260507-0001                │  제목: RETS-MODULE-01 품질    │
│ 2026-05-07 11:00                 │  보고서 v1.0                  │
│ 전체 / 47건                      │  생성일: 2026-05-07 11:00     │
│ 평균: 72.3점                     │  범위: 전체 (47건)            │
│                                  │                               │
│ RPT-20260506-0003                │  종합 점수 분포:              │
│ 2026-05-06 16:30                 │  ████ A:5 B:18 C:17 D:7     │
│ DEV only / 27건                  │  평균 72.3점                  │
│ 평균: 74.1점                     │                               │
│                                  │  차원별 평균:                 │
│                                  │  명확성   68.5 ████░░        │
│                                  │  일관성   79.1 ████████░     │
│                                  │  검증가능 70.8 ███████░      │
│                                  │                               │
│                                  │  주요 이슈:                   │
│                                  │  • Critical 2건              │
│                                  │  • 모호성 12건               │
│                                  │  • 검증불가 8건              │
│                                  │                               │
│                                  │  권고사항:                    │
│                                  │  1. D등급 7건 우선 재검토... │
└──────────────────────────────────┴──────────────────────────────  │
```

#### 4.7.2 UI 요소 명세

| 요소 ID | 타입 | 레이블 | 동작 | 데이터 소스 | 비고 |
|---------|------|--------|------|------------|------|
| `#btn-export-md` | button | 📄 Markdown 내보내기 | 클릭 → 현재 보고서를 MD 파일로 다운로드 | 선택된 QualityReport | |
| `#btn-export-csv` | button | 📊 CSV 내보내기 | 클릭 → CSV 다운로드 | 선택된 QualityReport | |
| `#btn-new-report` | button (primary) | + 새 보고서 | 클릭 → 보고서 생성 모달 (범위 선택) | - | FUNC-033 |
| `#report-list` | ul | 보고서 목록 | 항목 클릭 → 우측 상세 패널 갱신 | QualityReport[] | |
| `.report-list-item` | li | 보고서 요약 카드 | 클릭 → 선택 상태 및 상세 표시 | QualityReport | |
| `#report-detail` | div | 보고서 상세 패널 | (표시 전용) | 선택된 QualityReport | |
| `#report-grade-chart` | canvas | 등급 분포 차트 | (표시 전용) | `grade_distribution` | 가로 바 차트 |
| `#report-dimension-bars` | div | 차원별 점수 바 | (표시 전용) | `avg_clarity` 등 | |
| `#report-critical-issues` | ul | Critical 이슈 목록 | 항목 클릭 → SCR-03 해당 요구사항 | `critical_issues` | |
| `#report-recommendations` | ol | 권고사항 목록 | (표시 전용) | `recommendations` | |

---

### SCR-08: 시스템 설정

**화면 ID:** SCR-08  
**관련 FA:** FA-07  
**관련 FUNC:** FUNC-060~062  
**관련 FR:** FR-16~17  
**화면 목적:** LLM 연결 설정, 품질 진단 가중치, UI 환경설정, 데이터 초기화 관리

#### 4.8.1 레이아웃 와이어프레임

```
┌─────────────────────────────────────────────────────────────────┐
│ 시스템 설정                                          [저장]      │
├─────────────────────────────────────────────────────────────────┤
│ ▼ LLM 연결 설정                                                  │
│   팀 ID      [team_rets_01                    ]  ❓            │
│   릴레이 URL [https://ai-relay.mooo.com:11443 ]  [연결 테스트]  │
│   타임아웃   [30    ] 초  (5~120)                               │
│   배치 크기  [10    ] 건  (1~50)                                │
│                                                                  │
│ ▼ 품질 진단 가중치                                               │
│   명확성      [0.4 ] (40%)  ────────────────████░░            │
│   일관성      [0.3 ] (30%)  ────────────────███░░░            │
│   검증가능성  [0.3 ] (30%)  ────────────────███░░░            │
│   합계: 1.0 ✅   (합계가 1.0이 아니면 저장 불가)               │
│                                                                  │
│ ▼ 내보내기 / 표시 설정                                           │
│   기본 내보내기 형식  ○ JSON  ● CSV  ○ Markdown                │
│   UI 테마             ○ 라이트  ○ 다크  ● 시스템               │
│   인터페이스 언어     ● 한국어  ○ English                       │
│   자동 저장 주기      [30    ] 초  (0 = 비활성화)               │
│   기본 발의자         [김담당                ]                  │
│   LLM 원본 응답 표시  ☐                                         │
│                                                                  │
│ ▼ 데이터 관리                                                    │
│   [📥 전체 데이터 내보내기 (JSON)]   [📂 데이터 가져오기 (JSON)] │
│   ─────────────────────────────────────────────────────────     │
│   ⚠️  [🗑 모든 데이터 초기화]   (실행 취소 불가)                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 4.8.2 UI 요소 명세

| 요소 ID | 타입 | 레이블 | 동작 | 데이터 소스 | 유효성 |
|---------|------|--------|------|------------|--------|
| `#settings-team-id` | text input | 팀 ID | 입력 → UserSettings 변경 대기 | `UserSettings.team_id` | 필수 |
| `#settings-relay-url` | url input | 릴레이 서버 URL | 입력 → UserSettings 변경 대기 | `UserSettings.relay_server_url` | URL 형식 |
| `#btn-test-connection` | button | 연결 테스트 | 클릭 → POST /chat/session/create 시도 → 성공/실패 토스트 | - | |
| `#settings-timeout` | number input | 타임아웃(초) | 입력 | `UserSettings.llm_timeout_sec` | 5~120 |
| `#settings-batch-size` | number input | 배치 크기 | 입력 | `UserSettings.max_requirements_per_batch` | 1~50 |
| `#settings-weight-clarity` | number input | 명확성 가중치 | 입력 → 합계 자동 재계산 | `UserSettings.quality_weights.clarity` | 0.1~0.8 |
| `#settings-weight-consistency` | number input | 일관성 가중치 | 입력 → 합계 자동 재계산 | `UserSettings.quality_weights.consistency` | 0.1~0.8 |
| `#settings-weight-verifiability` | number input | 검증가능성 가중치 | 입력 → 합계 자동 재계산 | `UserSettings.quality_weights.verifiability` | 0.1~0.8 |
| `#settings-weight-total` | span | 합계: N.N | (표시 전용) | 세 가중치 합 | 1.0이면 ✅, 아니면 ❌ 빨간색 |
| `#settings-export-format` | radio group | JSON / CSV / Markdown | 변경 | `UserSettings.export_format` | |
| `#settings-theme` | radio group | 라이트 / 다크 / 시스템 | 변경 → 즉시 테마 적용 미리보기 | `UserSettings.ui_theme` | |
| `#settings-language` | radio group | 한국어 / English | 변경 | `UserSettings.language` | |
| `#settings-auto-save` | number input | 자동 저장 주기(초) | 입력 | `UserSettings.auto_save_interval_sec` | 0~300 |
| `#settings-default-author` | text input | 기본 발의자 | 입력 | `UserSettings.default_created_by` | |
| `#settings-show-raw` | checkbox | LLM 원본 응답 표시 | 변경 | `UserSettings.show_llm_raw_response` | |
| `#btn-export-all-data` | button | 📥 전체 데이터 내보내기 | 클릭 → AppState 전체 JSON 다운로드 | AppState | |
| `#btn-import-all-data` | button | 📂 데이터 가져오기 | 클릭 → 파일 선택 → 확인 모달 → AppState 교체 | - | 덮어쓰기 주의 확인 필요 |
| `#btn-reset-all` | button (danger) | 🗑 모든 데이터 초기화 | 클릭 → 2단계 확인 모달 → localStorage 전체 삭제 후 초기화 | - | FUNC-062. 2단계: "RESET" 텍스트 직접 입력 확인 |
| `#btn-save-settings` | button (primary) | 저장 | 클릭 → UserSettings 저장. 가중치 합계 오류 시 저장 불가 | - | FUNC-061 |

---

## 5. 사용자 흐름 시나리오

### 5.1 시나리오 1: RFP 파일 업로드 → DEV 요구사항 생성 (핵심 플로우)

```
[사용자 진입]
     │
     ▼
SCR-01 대시보드 → [📂 요구사항 불러오기] 클릭
     │
     ▼
파일 선택 다이얼로그 → rfp_requirements.json 선택
     │
     ├── [성공] → 파싱 완료 토스트 → SCR-02 이동 (RFP 20건 표시)
     └── [실패] → E-001/E-002/E-003/E-004 오류 토스트
     │
     ▼
SCR-02 목록 → RFP 항목 2건 체크박스 선택 → [🤖 상세화] 클릭
     │
     ▼
SCR-04 자동 상세화 → (선택된 RFP 2건이 좌측 목록에 체크 상태로 표시됨)
     │
[🤖 상세화 실행] 클릭
     │
     ├── LLM 로딩 오버레이 표시 ("LLM 응답 대기 중... 00:07")
     ├── POST /chat/session/create
     ├── POST /chat/session/{id}/message (RFP 요구사항 포함 프롬프트)
     └── 응답 수신 → JSON 파싱 → DEV 결과 카드 6건 표시
     │
사용자 검토:
     ├── [✅ 수용] × 5건 → 수용됨 표시
     ├── [✏️ 편집] × 1건 → 인라인 편집 → 수정 후 수용
     └── [❌ 거부] × 0건
     │
     ▼
[전체 수용] 클릭 → TraceabilityLink 자동 생성 → SCR-02 이동 → DEV 6건 추가 확인
```

### 5.2 시나리오 2: 품질 진단 → 보고서 생성

```
SCR-05 품질 진단 → [🔍 전체 진단 실행]
     │
진행률 바: "5/27 처리 중 (18%)"
     │
     ▼
진단 완료 → 결과 테이블 갱신
     │
D 등급 7건 이슈 확인 → 행 클릭 → 이슈 하위 행 확장
     │
Critical 이슈 확인 → SCR-03 링크 클릭 → 해당 요구사항 수정
     │
     ▼
[📄 보고서 생성] 클릭 → SCR-07 이동 → 신규 보고서 생성 확인
     │
[📄 Markdown 내보내기] → 보고서 .md 파일 다운로드
```

### 5.3 시나리오 3: 신규 요구사항 수동 등록

```
SCR-02 → [+ 신규] 클릭
     │
SCR-03 사이드 패널 (빈 폼)
     │
req_type: DEV 선택 → [자동채번] → req_id: DEV_FR_028 자동 입력
     │
기본정보 입력: req_name, description, priority, acceptance_criteria
     │
source_rfp_ids: RFP_005 태그 추가
     │
[저장] → 유효성 검사 통과 → RequirementItem 추가 → SCR-02 목록 갱신 → 패널 유지
```

---

## 6. 디자인 시스템 준수 명세

### 6.1 CSS 변수 (rets_02b 준수)

| CSS 변수명 | 용도 | 라이트 모드 | 다크 모드 |
|-----------|------|-----------|---------|
| `--color-primary` | 주 강조색 (버튼, 탭 활성, 링크) | `#1A73E8` | `#4A9EFF` |
| `--color-secondary` | 보조 강조색 | `#5F6368` | `#9AA0A6` |
| `--color-success` | 성공/수용 표시 | `#1E8E3E` | `#34A853` |
| `--color-error` | 오류/거부 표시 | `#D93025` | `#F28B82` |
| `--color-warning` | 경고 표시 | `#E37400` | `#FDD663` |
| `--color-info` | 정보 표시 | `#1A73E8` | `#8AB4F8` |
| `--color-bg` | 페이지 배경 | `#FFFFFF` | `#202124` |
| `--color-surface` | 카드/패널 배경 | `#F8F9FA` | `#2D2E30` |
| `--color-surface-elevated` | 모달/드롭다운 배경 | `#FFFFFF` | `#3C4043` |
| `--color-border` | 구분선/테두리 | `#DADCE0` | `#5F6368` |
| `--color-text-primary` | 주 텍스트 | `#202124` | `#E8EAED` |
| `--color-text-secondary` | 보조 텍스트 | `#5F6368` | `#9AA0A6` |
| `--color-text-disabled` | 비활성 텍스트 | `#AAAAAA` | `#666666` |

### 6.2 타이포그래피

| 용도 | CSS 클래스 | font-size | font-weight | line-height |
|------|-----------|-----------|------------|------------|
| 화면 제목 (H1) | `.screen-title` | 20px | 500 | 1.4 |
| 섹션 제목 (H2) | `.section-title` | 16px | 500 | 1.4 |
| 본문 (기본) | `.body-text` | 14px | 400 | 1.5 |
| 보조 텍스트 | `.secondary-text` | 12px | 400 | 1.4 |
| 배지/태그 | `.badge-text` | 11px | 500 | 1.0 |
| 코드/ID | `.mono-text` | 13px | 400 | 1.4 |

### 6.3 컴포넌트 스타일 규칙

| 컴포넌트 | border-radius | padding | shadow |
|---------|--------------|---------|--------|
| 기본 버튼 | 4px | 8px 16px | none |
| 카드 | 8px | 16px | `0 1px 3px rgba(0,0,0,0.12)` |
| 모달 | 12px | 24px | `0 4px 12px rgba(0,0,0,0.2)` |
| 입력 필드 | 4px | 8px 12px | none, border 1px |
| 토스트 | 6px | 12px 16px | `0 2px 8px rgba(0,0,0,0.15)` |

### 6.4 우선순위 배지 색상 (MoSCoW)

| 값 | 배경색 | 글자색 |
|----|--------|--------|
| MUST | `#E6F4EA` | `#1E8E3E` |
| SHOULD | `#FEF9E7` | `#E37400` |
| COULD | `#E8F0FE` | `#1A73E8` |
| WONT | `#F1F3F4` | `#5F6368` |

### 6.5 품질 등급 배지 색상

| 등급 | 배경색 | 글자색 |
|------|--------|--------|
| A (≥90) | `#E6F4EA` | `#1E8E3E` |
| B (≥75) | `#E8F0FE` | `#1A73E8` |
| C (≥60) | `#FEF9E7` | `#E37400` |
| D (<60)  | `#FCE8E6` | `#D93025` |

---

## 7. 요구사항 추적 매트릭스

### 7.1 화면 ↔ FUNC 매트릭스

| 화면 ID | 화면명 | FUNC | 기능명 |
|---------|--------|------|--------|
| SCR-01 | 대시보드 | FUNC-050 | 대시보드 KPI 집계 |
| SCR-02 | 요구사항 목록 | FUNC-001 | RFP JSON 파일 업로드 |
| | | FUNC-002 | DEV JSON 파일 업로드 |
| | | FUNC-003 | JSON 내보내기 |
| | | FUNC-004 | CSV 내보내기 |
| | | FUNC-010 | 요구사항 목록 조회 |
| | | FUNC-012 | 요구사항 신규 등록 |
| | | FUNC-014 | 요구사항 삭제 |
| | | FUNC-015 | 요구사항 검색/필터 |
| SCR-03 | 요구사항 상세/편집 | FUNC-011 | 요구사항 상세 조회 |
| | | FUNC-013 | 요구사항 수정 |
| | | FUNC-041 | 추적성 링크 수동 관리 |
| SCR-04 | LLM 자동 상세화 | FUNC-020 | LLM 세션 초기화 |
| | | FUNC-021 | 요구사항 자동 상세화 |
| | | FUNC-022 | 상세화 결과 Human-in-the-Loop 처리 |
| | | FUNC-023 | LLM 세션 종료 |
| SCR-05 | 품질 진단 | FUNC-030 | 품질 진단 실행 |
| | | FUNC-031 | 품질 이슈 하이라이팅 |
| | | FUNC-032 | 중복/충돌/누락 탐지 |
| | | FUNC-033 | 품질 보고서 생성 |
| SCR-06 | 추적성 매트릭스 | FUNC-040 | 추적성 매트릭스 조회 |
| | | FUNC-041 | 추적성 링크 수동 관리 |
| SCR-07 | 품질 보고서 | FUNC-033 | 품질 보고서 생성 |
| | | FUNC-050 | 대시보드 KPI 집계 |
| SCR-08 | 시스템 설정 | FUNC-060 | 사용자 설정 조회 |
| | | FUNC-061 | 사용자 설정 수정 |
| | | FUNC-062 | 데이터 초기화 |

### 7.2 화면 ↔ 데이터 엔티티 매트릭스

| 화면 ID | 읽기(R) | 쓰기(W) | 삭제(D) |
|---------|---------|---------|---------|
| SCR-01 | RequirementItem, QualityDiagnosisResult, QualityReport, LLMSession | - | - |
| SCR-02 | RequirementItem, QualityDiagnosisResult | RequirementItem | RequirementItem, TraceabilityLink |
| SCR-03 | RequirementItem, QualityDiagnosisResult, TraceabilityLink, ChangeHistoryEntry | RequirementItem, ChangeHistoryEntry, TraceabilityLink | TraceabilityLink |
| SCR-04 | RequirementItem(RFP), LLMSession, UserSettings | RequirementItem(DEV), LLMSession, LLMMessage, TraceabilityLink | RequirementItem(DEV, 거부 시) |
| SCR-05 | RequirementItem, QualityDiagnosisResult | QualityDiagnosisResult, QualityReport | - |
| SCR-06 | TraceabilityLink, RequirementItem | TraceabilityLink | TraceabilityLink |
| SCR-07 | QualityReport, QualityDiagnosisResult | QualityReport | - |
| SCR-08 | UserSettings, AppState | UserSettings | AppState (초기화 시) |

### 7.3 화면 ↔ NFR 연결

| 화면 ID | 관련 NFR | 준수 내용 |
|---------|---------|---------|
| 전체 | NFR-P01 | 화면 전환(탭 클릭) 100ms 이내 |
| SCR-01 | NFR-P02 | 대시보드 집계 계산 500ms 이내 |
| SCR-02 | NFR-P03 | 100건 목록 렌더링 1초 이내 |
| SCR-04 | NFR-P04 | LLM 응답 대기 중 UI 비차단(Non-blocking) |
| SCR-05 | NFR-P02 | 배치 진단 진행률 실시간 표시 |
| 전체 | NFR-U01 | WCAG 2.1 AA — ARIA 레이블, 키보드 내비게이션 |
| 전체 | NFR-U02 | 모든 오류 상황에 구체적 사용자 피드백 메시지 제공 |
| SCR-08 | NFR-S01 | 팀 ID는 UI상 마스킹 처리 (비밀번호 입력 방식 옵션) |
| 전체 | NFR-C01 | Chrome 최신 버전 기준 동작. IE 미지원 명시 |

---

*본 문서는 `rets_02b_design_requirements.md`, MD02(PRD), MD03(SRS), MD05(DDD)와 일관성을 유지하며 작성되었다. 구현 시 본 명세를 화면 개발의 준거 기준으로 삼는다.*
