---
title: "with_structured_output()에서 텍스트가 잘리는 문제 해결하기"
---

# LangChain with_structured_output()에서 텍스트가 잘리는 문제 해결하기

최근 미팅 내용을 자동으로 요약해주는 에이전트를 만들다가 겪었던 문제를 공유하고자 합니다.

## 프로젝트 배경

음성 녹음된 회의 내용을 텍스트로 전사하는 STT를 거쳐서 AI 요약, Q&A 정리, 액션 아이템 추출 등을 하는 에이전트를 만들고 있었는데요
Pydantic 스키마를 정의하고 `with_structured_output()`을 사용해서 구조화된 출력을 받기로 했습니다. (`with_structured_output()`은 LLM이 특정 구조(Schema)에 맞춰 결과물을 생성하도록 하는 강력한 기능)

```python
# 예시 코드
class MeetingAnalysis(BaseModel):
    ai_summary: str
    qa_summary: str
    action_items: List[str]

chain = prompt | meeting_llm.with_structured_output(MeetingAnalysis)
```

## 문제 상황

그런데 간헐적으로 `qa_summary`와 `action_items`은 완벽하게 생성되는데, 유독 `ai_summary`만 마지막에 1~2 문장이 뚝 잘려서 나오는 현상이 발생했습니다.

## 디버깅 과정

### 첫 번째 의심: max_token 값이 너무 적은가?

가장 먼저 의심한 건 토큰 제한이었습니다. 혹시라도 모델의 최대 토큰 수에 도달해서 잘리는 건 아닐까?
랭스미스를 확인해본 결과:
- 실제 출력 토큰: 4000~5000 사이
- 설정한 max_token: 13,000

토큰은 충분했습니다. 
혹시 몰라서 max_token 값을 20,000으로도 늘려서 테스트했지만 같은 문제가 발생해서 max_token 값 때문은 아니라고 생각했어요

### 두 번째 의심: 프롬프트 엔지니어링

프롬프트에 다음과 같은 지침을 추가해보았습니다:
- "반드시 완전한 문장으로 끝내세요"
- "문장 중간에 자르지 마세요"
결과는? 여전히 간헐적으로 `ai_summary`만 잘려서 나왔습니다.

### 패턴 분석

왜 다른 필드는 정상적으로 나오는데 `ai_summary`만 잘릴까?
`ai_summary`의 특징:
1. 출력 형식이 다른 필드보다 **훨~~씬 복잡한 프롬프팅**
2. **분량이 가장 많음**


### 마지막 의심: 파서에 문제가 있을까?

`with_structured_output()`의 기본 동작 방식인 **Tool Calling**이 문제의 원인이었던 것 같아요
method 파라미터만 추가해서 문제가 해결됐거든요

## 문제의 원인: Tool Calling vs JSON Mode

### Tool Calling이란?

Tool Calling 방식에서 모델은 우리가 정의한 스키마를 하나의 '도구(Tool)'로 인식합니다. 마치 함수를 호출하듯이 각 필드를 채워나가죠.

```json
{
  "tool_calls": [{
    "function": {
      "name": "MeetingAnalysis",
      "arguments": {
        "ai_summary": "...",
        "qa_summary": "...",
        "action_items": [...]
      }
    }
  }]
}
```

문제는 이 방식이 **엄격한 구조 준수**에 너무 집중한다는 점입니다. 모델은 `ai_summary`를 채우다가 "아, 다음 필드도 채워야 하는구나"라고 판단하면, 현재 작성 중인 텍스트를 서둘러 마무리(사실상 포기)하고 다음으로 넘어가버립니다.

특히 `ai_summary`처럼 **복잡하고 긴 텍스트 필드**일수록 이런 현상이 더 자주 발생했습니다.

## 해결책: JSON Mode로의 전환

해결책은 의외로 간단했습니다. `method` 파라미터 하나만 추가하면 됐습니다.

```python
# 변경 전 (Tool Calling - 기본값)
chain = prompt | meeting_llm.with_structured_output(MeetingAnalysis)

# 변경 후 (JSON Mode)
chain = prompt | meeting_llm.with_structured_output(
    MeetingAnalysis, 
    method="json_mode"
)
```

JSON Mode에서는 모델이 "이 JSON 형식에 맞춰서 텍스트를 작성해줘"라는 지시를 받습니다. Tool을 사용하는 게 아니라, 주어진 형식 안에서 자유롭게 텍스트를 생성하는 거죠.

## 두 방식의 상세 비교

### 1. Tool Calling 방식 (PydanticToolsParser)

**작동 원리:**
1. Pydantic 스키마가 '호출 가능한 함수(function call)' 형태로 변환
2. `bind_tools`를 통해 모델에 전달
3. 모델은 일반 텍스트가 아닌 '함수 호출' 형식으로 응답
4. `PydanticToolsParser`가 `tool_calls` 결과에서 `arguments`를 추출

**모델이 받는 프롬프트 예시:**
```
사용자: 회의 내용을 분석해주세요: 오늘 마케팅 팀 회의에서...
시스템: [사용 가능한 도구: '회의분석' / 매개변수: ai_summary(string), qa_summary(string), ... ]
```

**모델의 응답 형태:**
```json
{
  "tool_calls": [{
    "type": "function",
    "function": {
      "name": "회의분석",
      "arguments": "{\"ai_summary\": \"요약...\", ...}"
    }
  }]
}
```

### 2. JSON Mode 방식 (PydanticOutputParser)

**작동 원리:**
1. `PydanticOutputParser`가 스키마 기반으로 '포맷 지침' 생성
2. 포맷 지침이 원본 프롬프트에 자동 주입
3. 모델은 JSON 형태의 텍스트 생성
4. Parser가 JSON 문자열을 Pydantic 객체로 변환

**모델이 받는 프롬프트 예시:**
```
회의 내용을 분석해주세요.
출력은 아래 JSON 스키마에 맞는 인스턴스로 형식화되어야 합니다...
(여기에 JSON 스키마 전체 내용이 들어감)

회의 내용: 오늘 마케팅 팀 회의에서 신제품 출시 계획을...
```

**모델의 응답 형태:**
```json
{
  "ai_summary": "오늘 마케팅 팀 회의에서는 신제품 출시 전략에 대해 논의했습니다...",
  "qa_summary": "Q: 출시 일정은? A: 3월 중순...",
  "action_items": ["마케팅 자료 준비", "프레스 릴리스 작성"]
}
```

## 각 방식의 장단점

### Tool Calling 방식
**장점:**
- 구조 준수 보장
- 타입 안정성
- 명확한 에러 처리

**단점:**
- 긴 텍스트 생성 시 중단 가능성
- 유연성 부족
- 창의적 텍스트 생성 제한

### JSON Mode 방식
**장점:**
- 자연스러운 텍스트 생성
- 긴 텍스트도 안정적으로 출력
- 유연한 응답 생성

**단점:**
- 파싱 실패 가능성
- 구조 일탈 위험
- 추가적인 검증 필요

## 결론

`with_structured_output()`을 사용할 때 텍스트가 잘리는 문제를 겪고 있다면:

1. **긴 텍스트 필드가 있는 경우**: `method="json_mode"` 파라미터를 한번 추가해보세요
2. **짧고 구조화된 데이터**: 기본 Tool Calling 방식 유지
3. **혼합된 경우**: JSON Mode를 사용하되 추가 검증 로직 구현

