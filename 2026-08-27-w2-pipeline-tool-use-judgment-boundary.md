---
date: 2026-08-27
week: 2
session: S6
practice_type:
  - pipeline
topic:
  - tool-use
  - tool-schema
  - function-calling
  - llm-judgment-boundary
model: claude-sonnet
status: draft
result: tool-call-wiring-verified-in-real-code-live-judgment-boundary-unverified-no-api-key
---

# 핵심 개념 정리

세션 중 미니 실습과 별개로 질문·답변을 통해 정리된 이론들. 개념 자체(S6 제목의
Tool Schema/호출조건/인자생성/오류복구)뿐 아니라, 오가는 동안 반복적으로 헷갈렸던
지점들도 그대로 남긴다.

## 1. Tool Schema란 — 코드를 LLM이 읽을 수 있게 번역한 계약서
LLM은 텍스트만 주고받는 API라서, 서버 프로세스 안에 어떤 메서드가 있는지 스스로 알
방법이 없다. `name`/`description`/`input_schema`로 "이런 함수가 있고, 이렇게 부르면
된다"를 매 요청마다 알려줘야만 존재를 인지한다. Java로 치면 구현체는 안 보여주고
인터페이스 시그니처+Javadoc만 외부 caller에게 넘기는 것과 같다.

## 2. `tools`는 선택 필드다 — 모든 LLM 호출에 필요한 게 아니다
API 요청 형식(`model`/`max_tokens`/`messages`)은 모든 호출이 지켜야 하지만, `tools`
필드는 **"이 요청에서 LLM이 판단해서 내 코드의 함수를 실행시켜야 할 때만"** 추가하는
선택 파라미터다. 순수 텍스트 in/out이면 tools 자체가 필요 없다.

## 3. Tool은 두 종류다
- **Anthropic이 이미 만들어둔 tool** (web_search, bash, text_editor, code_execution 등)
  → [tool-reference](https://platform.claude.com/docs/ko/agents-and-tools/tool-use/tool-reference)에서
  `type` 문자열만 확인해서 그대로 쓰면 됨. Anthropic이 스키마를 이미 정의해둠.
- **내가 만든 함수** (`LearningMemoryClient.save()` 같은 것) → tool-reference에 없음.
  Anthropic은 내 도메인 로직을 모르므로 `name`/`description`/`input_schema`를 직접
  작성해야 함.

## 4. 언제 LLM 판단(tool)이 필요한가 — 판단 기준이 틀렸다가 정정된 과정
처음엔 "교정이 실제로 일어났는지"가 LLM 판단이 필요한 지점이라고 예시를 들었는데
**이건 틀린 예시였다.** 사용자가 바로 지적: `!original.equals(corrected)` 한 줄로
충분히 구분 가능한 걸 왜 LLM에 맡기냐는 반박.

**정정된 기준**: *"결정 기준을 if문/equals/정규식 같은 규칙으로 표현할 수 있으면 →
코드. 그 기준 자체가 자연어의 의미/의도를 이해해야만 판단 가능하면 → LLM(tool)."*

이 기준으로 다시 든 예시:
- `"She don't like coffee"` → `"She doesn't like coffee"`: 진짜 문법 오류 교정 (저장 O)
- `"I really enjoy playing basketball"` → `"I really love playing basketball"`: 원문이
  이미 문법적으로 완벽한데 코치가 스타일만 바꾼 것 (저장 X)

두 케이스 모두 `!original.equals(corrected)`는 동일하게 `true`라서 문자열 비교로는
전혀 구분이 안 된다 — "이게 문법적으로 틀렸던 표현인가"라는 판단 자체가 언어 이해를
요구하기 때문에 규칙으로는 근본적으로 못 잡는다. 이게 실제로 tool이 필요한 지점이다.

## 5. 왜 프롬프트에 "이 형식으로 답해줘"라고 텍스트로 요청하는 것과 다른가
판단(judgment) 자체는 tool을 쓰든 안 쓰든 LLM이 한다 — 그건 동일하다. 차이는
**"판단 결과를 얼마나 안정적으로 코드가 받아낼 수 있냐"**다.

프롬프트로 포맷을 지정하면: 앞에 설명이 붙어 파싱 위치가 안 변함/입력에 구분자
문자(`|`)가 섞이면 깨짐/"호출했는지"를 텍스트에서 문자열 매칭으로 추론해야 해서
오탐 가능성이 있음. `tools`는 **`tool_use`라는 별도의 구조화된 content 블록**으로
호출 의도를 명확히 분리해서 주고, `input_schema`(특히 `strict: true`)로 필드 누락/
타입 불일치를 API 레벨에서 원천 차단한다. 정리하면: "누가 판단하냐"가 아니라
**"판단 결과를 신뢰성 있게 뽑아내는 채널이 다르다"**가 핵심.

## 6. 아키텍처: Controller vs Service — LLM 호출은 어디서 일어나는가
처음에 "LLM이 저장 필요 없다고 판단하면 Service까지 안 부른다"고 잘못 이해하고
있었음. **정정**: 사용자 요청 → Controller → Service 흐름은 **항상 실행**된다
(LLM 판단과 무관). LLM 호출은 Service **내부**에서 일어나고, 판단이 좌우하는 건
Service 안의 딱 한 메서드 호출(`learningMemoryClient.save()`)뿐이다. Controller는
이 판단이 뭔지 알 필요도 없다. 실제로 코드를 연결하고 테스트를 돌려 스택트레이스로
이 호출 순서(`CoachingService.processMessage` → `AnthropicLlmClient.correct` →
`AnthropicLlmClient.apiKey`)를 직접 확인함 (실행 결과 참고).

## 7. 번외 — Claude 제품군 구분과 hooks의 위치
Claude.ai 채팅 / Cowork / Claude Code / 직접 API 호출(이번 실습)이 다 비슷해
보여서 헷갈렸던 부분을 정리.

**공통 기반은 Messages API 하나다** (`model` + `messages` + `tools`). 다른 건
"누가 그 위에 뭘 지었냐"뿐이다.

| | 누가 만듦 | 등록된 tools | 특이 기능 |
|---|---|---|---|
| Claude.ai 채팅 | Anthropic | web_search, memory 등 Anthropic이 미리 정의 | 일반 대화 UI |
| Cowork | Anthropic | 위 + Slack/Drive 같은 업무 도구 연결(MCP) | 업무 데이터 연결 특화 |
| Claude Code | Anthropic | Read/Edit/Bash/Grep 등 코딩 전용 tool 세트 | **hooks**, subagent, 슬래시 커맨드 |
| english-coach-ai-lab(이번 실습) | 나 | `save_learning_record` 등 직접 정의 | 원하는 아무 기능 |

**hooks는 Messages API 개념이 아니다.** `tools`는 Anthropic 서버가 이해하는
파라미터지만, hooks는 **Claude Code라는 제품이 자체적으로 만든 기능**
(`settings.json` 설정, 도구 사용 전후/세션 시작 등 특정 시점에 셸 명령을 자동
실행)이다. `tools` = "무엇을 호출할 수 있는가"(모든 제품 공통), `hooks` =
"Claude Code가 자기 행동 전후에 뭘 자동으로 끼워넣는가"(Claude Code 전용).

---

# 실험 목적
Tool Use 메커니즘(스키마 선언 → 요청에 첨부 → 응답의 tool_use 판별 → 조건부 실행)을
개념 설명으로만 남기지 않고, `english-coach-ai-lab`에 실제 Java 코드로 구현해서
호출 지점이 실행 경로에 진짜로 존재하는지 확인한다.

## 가설
1. tool schema를 실제 코드에 반영하면, LLM 호출 지점(`AnthropicLlmClient`)이
   `CoachingService`의 실행 경로 안에 실제로 포함된다.
2. (미검증) 진짜 문법 오류(케이스 A: "She don't like coffee")에는
   `save_learning_record`가 호출되고, 스타일 제안(케이스 B: "I really enjoy
   playing basketball")에는 호출되지 않는다.

## 조건
- **코드 구현**: `resources/tools/save_learning_record.json` 스키마 +
  `ai/AnthropicLlmClient`(`java.net.http.HttpClient` 기반) +
  `CoachingService`의 조건부 저장 로직
- **실행 조건**: `ANTHROPIC_API_KEY` 미설정 상태에서 기존
  `DuplicateRequestTest`(`@SpringBootTest`) 실행

## 고정한 변수
기존 프로젝트 계층 구조(Controller/Service/Tool), 테스트 시나리오
(`DuplicateRequestTest`), 모델명(`claude-sonnet-5`, 코드에 상수로 고정)

## 변경한 변수
`CoachingService`의 저장 로직 (무조건 저장 → LLM `tool_use` 조건부 저장),
교정 로직 (`text.replace()` naive 치환 → LLM 호출로 대체)

## 평가 기준
1. 코드가 실제로 컴파일되는가
2. LLM 호출 지점이 실행 경로에 실제로 포함되는가 (스택트레이스로 확인 가능한가)
3. (미검증) 케이스 A/B에서 실제로 `tool_use` 유무가 갈리는가
4. (미검증) `is_error`를 이용한 오류 복구가 실제로 재현되는가

## 실행 결과
- `./mvnw -o compile` → **BUILD SUCCESS** (신규 클래스 4개: `LlmClient`,
  `AnthropicLlmClient`, `ToolCallResult`, `CorrectionResult` 모두 컴파일됨)
- `./mvnw -o test -Dtest=DuplicateRequestTest` → **실패 (의도된 안전한 실패)**:
  ```
  java.lang.IllegalStateException: ANTHROPIC_API_KEY가 설정되지 않았습니다
      at AnthropicLlmClient.apiKey(AnthropicLlmClient.java:97)
      at AnthropicLlmClient.correct(AnthropicLlmClient.java:40)
      at CoachingService.processMessage(CoachingService.java:23)
  ```
  네트워크 요청(`httpClient.send()`)에 도달하기 **전에** 키 체크에서 막혀 비용은
  0원. 기존엔 무조건 저장이라 검증 대상도 아니었던 `learningMemoryClient
  .getRecords("user1")`가 이번엔 예외로 인해 0건이 됨 (기대값 1건과 다름 —
  기존 중복요청 버그와는 별개의, 이번 변경으로 인한 새로운 실패 원인).
- **가설 1 확인됨**: 스택트레이스가 정확히 `Controller(호출 안 보임, 항상 실행)
  → Service(23행) → AnthropicLlmClient.correct(40행) → apiKey(97행)` 순서를
  보여줌 — LLM 호출 코드가 실제 실행 경로 안에 존재함을 코드 레벨로 증명.

## 예상과 달랐던 점
- "LLM이 저장 필요 없다고 판단하면 Service까지 안 부른다"고 처음엔 생각했는데,
  실제로는 Controller→Service는 항상 실행되고 Service 내부의 한 메서드 호출만
  조건부가 됨을 스택트레이스로 확인.
- "tool 없이 프롬프트 텍스트로 형식만 요청하면 안 되냐"는 질문에 대해, 판단
  주체는 동일하지만 결과를 구조화된 채널로 안정적으로 뽑아내는 게 핵심 차이라는
  점을 스스로 재구성해야 했음 (처음 설명이 이 차이를 명확히 못 짚었음).

## 실패 사례
- 최초 예시("교정 여부는 equals()로 못 가리니 LLM이 필요하다")가 틀렸음 —
  실제로는 `!original.equals(corrected)`로 충분히 판별 가능한 결정론적 조건을
  LLM 판단이 필요한 사례로 잘못 제시함. 사용자 지적으로 "판단 기준 자체가 의미
  이해를 요구하는가"로 기준을 정정함 (섹션 4 참고).
- `AnthropicLlmClient`를 `CoachingService`에 실제로 연결하는 걸 처음엔 "테스트가
  네트워크/비용에 의존하게 된다"는 이유로 미룸 → 사용자가 "코드는 작성해서
  보여줘야지, 테스트를 안 돌리면 되잖아"라고 재지적 → 실제로는 API 키가 없으면
  네트워크 도달 전에 안전하게 예외가 나므로 비용 걱정 없이 연결 가능했음(과한
  조심이었음을 실제 실행으로 확인).

## 결론
Tool Use의 핵심 메커니즘(스키마 선언 → 요청 첨부 → 응답 판별 → 조건부 실행) 중
**"호출 조건"과 "인자 생성"** 부분은 실제 Java 코드로 구현하고 실행 경로에 실제로
존재함을 코드 레벨로 확인했다. 다만 이번 세션에서 API 키 발급(결제)을 하지 않기로
해서, S6 제목에 포함된 **"오류 복구"(`is_error` 왕복)와, 핵심 가설이었던 "LLM이
실제로 케이스 A/B를 구분해서 tool을 호출하는가"는 검증되지 않은 채로 남아있다.**
`status: draft`로 남기는 이유가 이것 — 코드 배선(wiring)은 검증됐지만, 이 세션이
원래 확인하려던 "판단 정확도" 자체는 아직 실측되지 않았다.

## 실제 프로젝트 적용 가능성
"판단 기준이 규칙으로 표현 가능한가, 의미 이해가 필요한가"를 구분하는 기준(섹션 4)은
"이 기능에 AI를 써야 하나 그냥 코드로 짜도 되나"를 판단할 때 다른 프로젝트에도 그대로
재사용 가능 — Week 4의 S17(AI 적용 판단 기준)과 직접 연결되는 지점.

## 재현 방법
1. [console.anthropic.com](https://console.anthropic.com)에서 `ANTHROPIC_API_KEY`
   발급 (최소 $5 충전 필요)
2. 케이스 A(`"She don't like coffee"`)와 케이스 B(`"I really enjoy playing
   basketball"`)로 각각 `save_learning_record` 스키마를 실은 실제 요청을 보내
   `stop_reason`이 `tool_use`인지 비교 (curl 명령어는 이 세션 대화 기록 기반으로
   재생성 가능)
3. 같은 키로 `./mvnw -o test -Dtest=DuplicateRequestTest` 재실행 — 이번엔 실제로
   LLM 판단까지 거쳐 저장이 되는지, 기존 S4/S5에서 발견된 중복요청 버그가 아직
   남아있어 다른 이유로 실패하는지 확인
4. `is_error: true`를 강제로 재현하려면 `save_learning_record` 호출 후 일부러
   잘못된 `tool_result`(예: 필수 필드 누락 메시지)를 돌려보내고 Claude가 재시도
   하는지 관찰
