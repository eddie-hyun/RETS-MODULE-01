# LLM 릴레이 서버 활용 가이드

> **릴레이 서버:** AISELab Gemini Relay API v1.0.0  
> **목적:** 단일 Gemini API Key를 여러 실습 팀이 공유하기 위한 중계 서버.  
> **대상:** 본 프로그램의 각 모듈을 개발하는 개발자  
> **환경:** 단일 HTML 파일에 통합되는 브라우저용 순수 JavaScript

---

## 1. 개요

- 각 모듈은 Gemini LLM을 직접 호출하지 않고, **공용 LLM 릴레이 서버**를 통해 LLM 기능을 사용합니다.
- 릴레이 서버는 인증, 사용량 추적, 팀별 세션 관리를 중앙에서 처리하며, 개발자는 아래의 2단계 흐름을 따르면 됩니다.
- 채팅 요청은 비동기로 처리하여 부드럽고 자연스러운 UX를 제공할 수 있도록 합니다.

```
session_id = POST /chat/new              ← 세션 초기화 (team_id, system_prompt)
response   = POST /chat/{session_id}     ← 반복 호출  (team_id, message)
```

단발성 처리가 필요한 경우에는 세션 없이 바로 응답을 받을 수 있는 **Oneshot** 엔드포인트도 제공됩니다.

---

## 2. API 문서

| 항목 | 값 |
|---|---|
| **OpenAPI 명세** | `https://ai-relay.mooo.com:11443/openapi.json` |
| **Swagger UI** | `https://ai-relay.mooo.com:11443/docs` |
| **기반 LLM** | Google Gemini |
| **프로토콜** | HTTPS |

---

## 3. 엔드포인트 레퍼런스

### 3.1 세션 생성 — `POST /chat/new`

새 챗 세션을 생성하고 `session_id`를 발급받습니다.

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 제약 | 설명 |
|---|---|---|---|---|
| `team_id` | string | ✅ | 1–64자 | 팀 식별자 |
| `system_prompt` | string \| null | ❌ | 최대 8,000자 | LLM에게 부여할 역할·지시 |

**Response** `200 OK`

| 필드 | 타입 | 설명 |
|---|---|---|
| `session_id` | uuid | 이후 모든 호출에 사용 |
| `team_id` | string | 요청한 팀 ID |
| `created_at` | datetime | 세션 생성 시각 |
| `expires_at` | datetime | 세션 만료 시각 |

---

### 3.2 메시지 전송 — `POST /chat/{session_id}`

세션에 메시지를 보내고 LLM 응답을 받습니다. 세션 내에서 이전 대화 이력이 유지되기 때문에 여러 단계에 걸친 작업을 자연스럽게 이어서 진행할 수 있습니다.

**Path Parameter**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `session_id` | uuid | `POST /chat/new`에서 받은 값 |

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 제약 | 설명 |
|---|---|---|---|---|
| `team_id` | string | ✅ | 1–64자 | 팀 식별자 (세션 생성 시와 동일) |
| `message` | string | ✅ | 1–32,000자 | 사용자 메시지 |

**Response** `200 OK`

| 필드 | 타입 | 설명 |
|---|---|---|
| `session_id` | uuid | 현재 세션 ID |
| `response` | string | LLM 응답 텍스트 |
| `input_tokens` | integer | 이번 호출의 입력 토큰 수 |
| `output_tokens` | integer | 이번 호출의 출력 토큰 수 |
| `expires_at` | datetime | 세션 만료 시각 |

---

### 3.3 단발성 호출 — `POST /chat/oneshot`

세션 생성 없이 단 한 번의 LLM 호출로 응답을 받습니다. 이전 대화 이력이 불필요한 독립적인 작업에 사용합니다.

**Request Body** (`application/json`)

| 필드 | 타입 | 필수 | 제약 | 설명 |
|---|---|---|---|---|
| `team_id` | string | ✅ | 1–64자 | 팀 식별자 |
| `message` | string | ✅ | 1–32,000자 | 사용자 메시지 |
| `system_prompt` | string \| null | ❌ | 최대 8,000자 | LLM에게 부여할 역할·지시 |

**Response** `200 OK`

| 필드 | 타입 | 설명 |
|---|---|---|
| `response` | string | LLM 응답 텍스트 |
| `input_tokens` | integer | 입력 토큰 수 |
| `output_tokens` | integer | 출력 토큰 수 |

---

### 3.4 보조 엔드포인트

| 메서드 | 경로 | 설명 |
|---|---|---|
| `GET` | `/chat/{session_id}` | 세션의 대화 이력 조회 (`role`, `content`, `timestamp`) |
| `GET` | `/chat/team/{team_id}` | 팀의 활성 세션 목록 조회 |

---

## 4. 코드 예시

아래 코드 예시는 구현 방법을 설명하기 위한 참고용입니다.  
실제 구현 시 모듈의 구조와 요구사항에 맞게 자유롭게 변형하여 사용하십시오.

---

### 4.0 파일 구성

`system_prompt.md`는 프로젝트 워크스페이스의 00_rfp_requirements 디렉토리에 위치합니다. 개발자는 이 파일의 내용을 시스템 프롬프트로 하여 LLM 채팅 세션을 초기화하고 LLM 기반의 지능화 기능을 구현하도록 합니다.

```
your-module/                               ← workspace
├── index.html                             ← JS가 통합된 단일 HTML 파일을 만들어야 함
└── 00_rfp_requirements/system_prompt.md   ← 고정된 system prompt
```

---

### 4.1 user prompt — JSON 출력 형식 지정 ⚠️ 필수

LLM이 자연어로 자유롭게 응답하면 코드에서 결과를 안정적으로 파싱할 수 없습니다.  
**LLM의 출력은 반드시 JSON 형태로만 반환되어야 하며**, 출력 형식 지정은 **각 LLM 호출의 user prompt에 직접 포함**합니다.

개발자는 각 호출별 user prompt 끝부분에 JSON 출력을 지시하는 구문과 응답 스키마를 명시합니다. 스키마는 데이터 설계서(DDD) 산출물(rets_module_0x_x_ddd.md) 내 X.X.X. LLM Response JSON 설계를 정확히 따릅니다.

#### user prompt에 출력 형식을 포함하는 예시

```js
const items = [/* 분석할 항목 배열 */];

const message = `
다음 항목들을 검토하고 문제점을 분석하라:
${items.map((it, i) => `${i + 1}. ${it}`).join('\n')}

## Output Format (STRICT JSON ONLY)
반드시 아래 JSON 스키마만 출력하라. 설명·마크다운·코드 펜스는 포함하지 말 것.
{
  "results": [
    {
      "index": <number>,        // 요청한 항목 번호 (1부터 시작)
      "status": <string>,       // "ok" | "warning" | "error"
      "issues": [<string>],     // 발견된 문제 목록 (없으면 빈 배열)
      "suggestion": <string>    // 개선 제안 (없으면 빈 문자열)
    }
  ]
}
항목을 분석할 수 없는 경우에도 "error"로 표시하고 issues에 설명을 기재하여 결과에 포함하라.
`.trim();

const raw = await sendMessage(message);
const parsed = parseLlmJson(raw);
```

#### JSON 응답 파싱 헬퍼

LLM의 `response` 필드는 문자열이므로 `JSON.parse()`로 파싱하면 JS 오브젝트를 얻을 수 있습니다.  
LLM이 간혹 응답을 ` ```json ... ``` ` 마크다운 펜스로 감싸는 경우에 대비해 이를 제거하는 처리를 포함하는 것이 안전합니다.

```js
// LLM 응답 문자열을 JSON으로 파싱
function parseLlmJson(responseText) {
  // 마크다운 코드 펜스(```json ... ```)가 포함된 경우 제거
  const cleaned = responseText.replace(/^```(?:json)?\s*/i, '').replace(/\s*```$/, '').trim();
  return JSON.parse(cleaned);
}

// 사용 예
const raw = await sendMessage(message);
const parsed = parseLlmJson(raw);

parsed.results.forEach(item => {
  console.log(`항목 ${item.index}: ${item.status}`);
  item.issues.forEach(issue => console.log('  -', issue));
});
```

---

### 4.2 세션 생성 (시스템 프롬프트를 포함하는 채팅 세션 초기화)

개발자는 주어진 `/＠rfp_01_requirements/rets_02a_llm_system_prompt.md`를 기반으로 채팅 세션을 생성합니다. system_prompt에는 LLM의 역할과 도메인 컨텍스트만 기술하며, JSON 출력 형식은 포함하지 않습니다.

아래 채팅 세션 생성 예시 코드의 `"<SYSTEM PROMPT GOES HERE>"` 자리에 `rets_02a_llm_system_prompt.md`의 내용을 넣어, LLM이 해당 세션에서 원하는 방식으로 동작하도록 지시할 수 있습니다. (코드상의 quoted string으로 들어가기 때문에 escape, new line 처리 등이 필요할 수 있음)

```html
<script>
const RELAY_BASE_URL = 'https://ai-relay.mooo.com:11443';
const TEAM_ID = 'team-alpha'; // 팀별 고유 ID로 변경

let sessionId = null;

async function initSession() {
  // system_prompt.md의 내용을 아래에 붙여넣습니다.
  const systemPrompt = "<SYSTEM PROMPT GOES HERE>";

  const res = await fetch(`${RELAY_BASE_URL}/chat/new`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      team_id: TEAM_ID,
      system_prompt: systemPrompt,
    }),
  });

  if (!res.ok) throw new Error(`세션 생성 실패: ${res.status}`);
  const data = await res.json();

  sessionId = data.session_id;
  console.log(`세션 생성 완료: ${sessionId} (만료: ${data.expires_at})`);
}

// 페이지 로드 시 세션을 1회 초기화합니다.
initSession().catch(console.error);
</script>
```

---

### 4.3 메시지 전송

```js
// LLM에 메시지를 전송하고 응답 텍스트를 반환합니다.
async function sendMessage(message) {
  const res = await fetch(`${RELAY_BASE_URL}/chat/${sessionId}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      team_id: TEAM_ID,
      message: message,
    }),
  });

  if (!res.ok) throw new Error(`메시지 전송 실패: ${res.status}`);
  const data = await res.json();
  return data.response; // 문자열 — parseLlmJson() 으로 파싱 필요
}

// 사용 예 — user prompt에 작업 내용과 출력 JSON 스키마를 함께 포함
const message = `
다음 함수의 버그를 찾아 분석하라:
function add(a, b) { return a - b; }

## Output Format (STRICT JSON ONLY)
반드시 아래 JSON 스키마만 출력하라. 설명·마크다운·코드 펜스는 포함하지 말 것.
{
  "bug_found": <boolean>,         // 버그 존재 여부
  "description": <string>,        // 버그 설명
  "suggestion": <string>          // 수정 제안
}
`.trim();

const raw = await sendMessage(message);
const parsed = parseLlmJson(raw);
console.log(parsed.bug_found, parsed.description);
```

---

### 4.4 단발성 호출 (Oneshot)

세션 생성 없이 독립적인 작업 하나를 처리할 때 사용합니다.  
팀 세션 한도를 소모하지 않으므로 분류, 요약, 변환 등 이력이 필요 없는 1회성 작업에 적합합니다.

```js
async function sendOneshot(message, systemPrompt = null) {
  const res = await fetch(`${RELAY_BASE_URL}/chat/oneshot`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      team_id: TEAM_ID,
      message: message,
      system_prompt: systemPrompt,
    }),
  });

  if (!res.ok) throw new Error(`Oneshot 호출 실패: ${res.status}`);
  const data = await res.json();
  return data.response; // 문자열 — JSON.parse() 필요
}
```

---

## 5. 활용상 제약사항

### 5.1 팀당 활성 세션은 1개로 제한

각 팀(`team_id`)이 동시에 유지할 수 있는 챗 세션은 **1개**입니다.

| 상황 | 올바른 처리 |
|---|---|
| 모듈 실행 시 | `POST /chat/new`를 1회 호출하여 `session_id` 획득 후 변수에 보관 |

```js
// ❌ 잘못된 예 — 매 요청마다 새 세션을 생성
for (const item of items) {
  const res = await fetch(`${RELAY_BASE_URL}/chat/new`, { /* ... */ });
  const { session_id } = await res.json(); // 세션 한도 초과!
  await fetch(`${RELAY_BASE_URL}/chat/${session_id}`, { /* ... */ });
}

// ✅ 올바른 예 — 세션 1회 생성 후 변수에 보관하여 재사용
const newRes = await fetch(`${RELAY_BASE_URL}/chat/new`, { /* ... */ });
const { session_id } = await newRes.json();

for (const item of items) {
  await fetch(`${RELAY_BASE_URL}/chat/${session_id}`, { /* ... */ });
}
```

> 💡 `GET /chat/team/{team_id}` 로 팀의 활성 세션 목록과 만료 시각을 조회할 수 있습니다. 기본적으로, 각 세션은 마지막 채팅 메시지 이후 5시간 유지됩니다.

---

### 5.2 제약사항 요약

| 항목 | 규칙 |
|---|---|
| 팀당 동시 세션 수 | **1개** |
| 루프 내 세션 생성 | ❌ 금지 |
| 단발성 처리 | `POST /chat/oneshot` 사용 (세션 없음) |
| LLM 출력 형식 | **JSON 정의대로** (중요!) |
| 메시지 최대 길이 | 32,000자 |
| system_prompt 최대 길이 | 8,000자 |

---

## 6. 체크리스트

- [ ] `team_id`를 확인하고 코드 상수로 정의했는가?
- [ ] 각 LLM 호출의 user prompt 끝에 호출 용도에 적합한 JSON 출력 지시와 응답 스키마를 포함했는가?
- [ ] LLM 호출에 대해 요구했던 응답을 적절하게 파싱하고 있는가?
- [ ] 세션 생성(`POST /chat/new`)은 페이지 로드 시 **1회만** 호출하는가?
- [ ] `session_id`를 변수에 보관하고, 이후 모든 메시지 전송에 재사용하는가?
