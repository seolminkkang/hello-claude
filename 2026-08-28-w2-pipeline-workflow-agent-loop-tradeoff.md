---
date: 2026-08-28
week: 2
session: S7
practice_type:
  - pipeline
topic:
  - workflow
  - agent-loop
  - termination-condition
  - human-in-the-loop
  - cost-tradeoff
model: claude-sonnet
status: verified
result: workflow-vs-agent-judgment-correct-cost-tradeoff-and-eval-definition-clarified
---

# 핵심 개념 정리

세션 중 미니 확인과 별개로, 질문·답변을 통해 정리된 이론들.

## 1. Workflow vs Agent Loop — 정의
Workflow는 개발자가 미리 정한 코드 경로로 LLM과 tool을 orchestrate하는 것 — 분기가
코드에 고정됨. Agent Loop는 LLM이 매 반복(iteration)마다 환경 피드백(tool 결과)을 보고
**다음 행동 자체를 스스로 결정**하는 것 — 반복문은 있는데 그 본문이 뭘지를 LLM이
고른다는 점이 다름.

Java 비유: Workflow = `@Service` 메서드 체인처럼 호출 순서가 코드에 고정된 파이프라인.
Agent = 반복문은 있는데 "이번 턴에 뭘 할지"를 매번 LLM이 정하는 구조.

## 2. 종료조건 · 재시도 · 사람 승인은 코드가 강제해야 한다
- **종료조건**: LLM은 스스로 안 멈추므로, "최대 N번" 같은 정지 조건이 없으면 무한
  루프·비용 폭주로 이어짐.
- **재시도**: Workflow의 에러 처리는 고정된 catch 경로지만, Agent는 `is_error` 같은
  "환경에서 온 사실"을 보고 다음 행동을 다시 판단함 (S6에서 다룬 tool_result 왕복과
  같은 메커니즘).
- **사람 승인**: 자동으로 생기는 게 아니라 설계 선택. 위험한 행동(삭제·결제·전송·배포처럼
  되돌릴 수 없는 것) 직전에 멈추는 지점을 개발자가 명시적으로 넣어야 함.

## 3. Workflow와 Agent의 비용 트레이드오프
- Agent Loop는 반복마다 누적된 대화 전체를 다시 실어 보내므로 반복될수록 호출당 비용이
  커지고, 잘못된 경로로 갔다 오는 왕복 자체도 비용이다.
- 반면 Workflow의 진짜 비용은 런타임 토큰이 아니라 **분기를 다 하드코딩하는 개발·유지보수
  비용**이다 — 케이스가 늘어날수록 이 비용이 커진다.
- 결론: "토큰비용 vs 개발/유지보수 비용"의 트레이드오프지, 한쪽이 무조건 싼 게 아니다.

## 4. Workflow vs Agent 선택 기준 (Week4 S17 예고편)
1. 분기 개수가 유한하고 미리 다 나열 가능한가 → Workflow
2. 한 스텝 실수의 대가가 큰가(비가역적) → Workflow (+ 위험 지점만 사람 승인 게이트)
3. 그 유연성이 필요한 상황이 자주 오는가, 드문 예외인가 → 드물면 Workflow에 예외
   처리만 추가하는 게 Agent 전체 도입보다 쌈
4. Anthropic의 원칙: "평가를 통과하는 가장 단순한 패턴부터 써라"

## 5. "평가를 통과하는"의 의미 — 감(feeling)이 아니라 사전 정의된 채점 기준
"Workflow 먼저 써보고 느껴보라는 게 평가냐"는 질문에 대한 정정: eval은 주관적 느낌이
아니라 ① 성공 기준을 먼저 숫자/조건으로 정의 → ② 가장 단순한 패턴으로 구현 → ③ 그
결과를 1번 기준으로 채점 → ④ 통과하면 종료, 미달이면 다음 단계(Agent)로 올리는
절차다. S4/S5에서 이미 "실제 원인 파악 여부 / 코드 지어내지 않았는지 / 변경 범위
적절성"이라는 채점 기준으로 이 사이클을 해본 적이 있음 — 채점 대상이 "프롬프트 조건"에서
"Workflow vs Agent"로 바뀌는 것뿐. Week3 S12(Evals 기초)에서 이 절차가 Golden
Dataset·LLM-as-a-Judge로 도구화될 예정.

---

# 실험 목적
Workflow와 Agent Loop의 구분 기준(고정 경로 vs 동적 판단, 종료조건, 재시도, 사람 승인)을
개념으로만 남기지 않고, `english-coach-ai-lab`에 이미 존재하는 실제 코드
(`CoachingService.processMessage`)에 대입해서 "지금 짠 코드가 어느 쪽인지" 스스로
판별할 수 있는지 확인.

## 실행 결과
`CoachingService.processMessage` 코드를 보고 스스로 판정: **"완전 Workflow"**
- 근거 1: 직접 판단해서 작업을 수행하기보다는 LLM을 트리거로만 사용하고 있음
- 근거 2: `save()` 조건을 결정하기 위해서만 LLM을 쓰고, 호출 이후에는 코드가 순차적으로
  작업을 수행함

실제 코드로 확인:
```java
CorrectionResult result = llmClient.correct(userId, text);   // LLM 호출 1번, 고정
result.learningRecord().ifPresent(record -> ...save(...));   // 조건부지만 "무엇을 할지"는 코드가 정함
String explanation = "문법 교정 결과입니다.";                    // 하드코딩
```
LLM이 "저장할지 말지"는 판단하지만, 다음에 뭘 할지 자체를 고르는 반복 구조는 없음 —
정확한 판정.

이어서 "Agent Loop로 바꾸려면 뭐가 필요한지 모르겠다"는 질문에 실제 코드 기준으로
구체화:
1. `correct` 외에 `get_learning_records`(과거 실수 이력 조회), `save_learning_record`를
   동시에 노출해서 어떤 tool을 부를지 LLM이 매 턴 선택하게 함
2. `processMessage`를 `while` 루프로 바꿔, 응답의 `stop_reason`이 `tool_use`가 아니면
   자연 종료, 맞으면 tool 실행 후 결과를 대화에 반영해 재호출
3. 종료조건 두 가지가 새로 필요: (a) LLM이 스스로 끝내는 자연 종료, (b) `MAX_ITER` 같은
   강제 상한(무한루프 방지) — 지금 코드엔 반복 자체가 없어 둘 다 존재하지 않음

## 예상과 달랐던 점
"Agent로 가면 분기 코드를 안 짜도 되니 더 편할 수도 있겠다"는 직관이 들었는데, 실제로는
그만큼 런타임 비용(반복마다 누적 대화 재전송, 잘못된 경로 왕복)을 대가로 지불한다는 걸
대화로 재확인함. "Workflow가 무조건 싸다"도 아니고 "Agent가 무조건 편하다"도 아닌,
토큰비용 vs 개발/유지보수 비용의 트레이드오프라는 게 처음 예상보다 미묘했음.

## 결론
Week2에서 나온 두 판단 기준이 이번 세션으로 연결됨 — S6의 "규칙(if/equals) vs 의미
이해가 필요한가"(tool 필요 여부)와 S7의 "고정 경로 vs 동적 판단"(Workflow vs Agent).
둘 다 결국 "이 지점에 AI 판단을 얼마나 허용할 것인가"라는 하나의 축으로 수렴하며, 이
축은 Week4 S17(AI 적용 판단 기준)에서 공식 체크리스트로 정리될 예정.

## 실제 프로젝트 적용 가능성
"분기 개수가 유한한가 / 실수 대가가 큰가 / 예외가 자주 오는가"라는 3문항 체크리스트는
이 랩을 넘어 다른 프로젝트에서 Workflow vs Agent를 고를 때 바로 쓸 수 있음. 특히 "위험한
행동 앞에서는 자동화 트렌드와 무관하게 사람 승인 게이트를 남겨둔다"는 원칙은, 지금 이
학습 세션 자체의 기본 지침(파괴적 git 명령 실행 전 항상 확인받기)과 동일한 패턴이라는
점도 함께 확인함.

## 재현 방법
1. `english-coach-ai-lab`의 아무 Service 클래스나 열어서, 다음 두 질문에 답해보기:
   (a) LLM 호출이 몇 번 일어나는가, (b) "다음에 뭘 할지"를 LLM이 직접 고르는 지점이
   있는가. 없으면 Workflow, 있으면 Agent Loop.
2. S9 통합실습(결함 API 3방식 비교)에서 `CoachingService`를 실제로 Agent Loop 버전으로
   구현해, 이번 세션에서 예측한 비용 방향(토큰/시간 증가)이 실측으로도 확인되는지 검증.
