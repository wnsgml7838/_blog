# 도구 호출 에이전트(Tool Calling Agnet)

- 도구 호출을 사용하면 모델이 하나 이상의 도구가 **호출되어야 하는 시기를 감지하고 해당 도구에 전달해야 하는 입력**으로 전달할 수 있다
- API 호출에서 도구를 설명하고 모델이 이러한 도구를 호출하기 위한 인수가 포함된 JOSN과 같은 구조화된 객체를 출력하도록 지능적으로 선택할 수 있다.
- 도구 API의 목표는 일반 텍스트 완성이나 채칭 API를 사용하여 수행할 수 있는 것보다 더 안정적으로 유효하고 유용한 도구 호출을 반환하는 것 
- 이러한 구조화된 출력을 도구 호출 채팅 모델에 여러 도구를 바인딩하고 모델이 호출할 도구를 선택할 수 있다는 사실과 결합하여 쿼리가 해결될 때까지 반복적으로 도구를 호출하고 결과를 수신하는 에이전트를 만들 수 있다
- 이것은 OpenAI의 특정 도구 호출 스타일에 맞게 설계된 OpenAI 도구 에이전트의 보다 일반화된 버전
	- [LangChain 공식 도큐먼트](https://python.langchain.com/v0.1/docs/modules/agents/agent_types/tool_calling/)

# Agent 프롬프트 생성
- chat_history : 이전 대화 내용을 저장하는 변수(멀티턴을 지원하지 않는다면 생략 가능)
- agent_scratchpad : 에이전트가 임시로 저장하는 변수
	- 끄적이는 "메모장" : 에이전트가 생각하는 과정들을 이런 메모장 같은곳에 저장할 수 있게 공간을 만들어주는 것
- input : 사용자의 입력

```python
from langchain_core.prompts import ChatPromptTemplate

# 프롬프트 생성
# 프롬프트는 에이전트에게 모델이 수행할 작업을 설명하는 텍스트를 제공함

prompt = ChatPromptTemplate.from_messages(
	[
		(
			"system",
			"You are a helpful assistant. "
			"Make sure to use the `search_news` tool for searching keyword related news.",
		),
		("placeholder", "{chat_history}"),
		("human", "{input}"),
		("placeholder", "{agent_scratchpad}"),
	]
)
```
>[!INFO]
>- 추가적으로 도구에 대해서 알려주면 도구를 잘 씀
>- 나중의 도구의 개수가 늘어나고, 도구 끼리의 내용이 겹치면 LLM이 적절한 도구를 판단하기 힘들때 디테일하게 작성

# Agent 생성
```python
from langchain_openai import ChatOpenAI

from langchain.agents import create_tool_calling_agent

# LLM 정의
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Agent 생성
agent = create_tool_calling_agent(llm, tools, prompt)
```

# AgentExecutor 
AgentExecutor는 도구를 사용하는 에이전트를 실행하는 클래스
- Agent를 컨트롤 
### **주요 속성**
- `agent`: 실행 루프의 각 단계에서 계획을 생성하고 행동을 결정하는 에이전트
- `tools`: 에이전트가 사용할 수 있는 유효한 도구 목록
- `return_intermediate_steps`: 최종 출력과 함께 에이전트의 중간 단계 경로를 반환할지 여부
- `max_iterations`: 실행 루프를 종료하기 전 최대 단계 수
- `max_execution_time`: 실행 루프에 소요될 수 있는 최대 시간
- `early_stopping_method`: 에이전트가 `AgentFinish`를 반환하지 않을 때 사용할 조기 종료 방법. ("force" or "generate")
	- `"force"` 는 시간 또는 반복 제한에 도달하여 중지되었다는 문자열을 반환합니다.
	- `"generate"` 는 에이전트의 LLM 체인을 마지막으로 한 번 호출하여 이전 단계에 따라 최종 답변을 생성합니다.
- `handle_parsing_errors`: 에이전트의 출력 파서에서 발생한 오류 처리 방법. (True, False, 또는 오류 처리 함수)
	- True : 중간 단계 오류를 대처하고 넘어감 
	- 거의 True로 두고 사용
- `trim_intermediate_steps`: 중간 단계를 트리밍하는 방법. (-1 trim 하지 않음, 또는 트리밍 함수)

### **주요 메서드**
1. `invoke`: 에이전트 실행
2. `stream`: 최종 출력에 도달하는 데 필요한 단계를 스트리밍

### **주요 기능**
1. **도구 검증**: 에이전트와 호환되는 도구인지 확인
2. **실행 제어**: 최대 반복 횟수 및 실행 시간 제한 설정 가능
3. **오류 처리**: 출력 파싱 오류에 대한 다양한 처리 옵션 제공
4. **중간 단계 관리**: 중간 단계 트리밍 및 반환 옵션
5. **비동기 지원**: 비동기 실행 및 스트리밍 지원

### **최적화 팁**
- `max_iterations`와 `max_execution_time`을 적절히 설정하여 실행 시간 관리
- `trim_intermediate_steps`를 활용하여 메모리 사용량 최적화
- 복잡한 작업의 경우 `stream` 메서드를 사용하여 단계별 결과 모니터링
