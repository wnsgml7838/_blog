# 17장 LangGraph 핵심 정리

  

LangGraph 모듈 전체를 빠르게 훑고, 실무에 바로 쓸 핵심 패턴만 추려놓은 치트시트입니다. 각 항목은 “이 기능이 필요할 때 어떤 키와 함수를 떠올려야 하는가?”에 초점을 맞췄습니다.

  

아래 정리는 강의 노트북을 처음부터 다시 열어보지 않아도 흐름을 복기할 수 있도록, 설명 → 코드 → 체크포인트 순으로 구성했습니다. 먼저 어떤 상황에서 해당 패턴이 필요한지와 주의해야 할 개념을 서술한 뒤, 노트북에서 바로 가져온 대표 코드 조각을 붙였습니다. 설명만으로 감이 잘 안 잡힐 때는 이어진 코드 블록을 보고, 코드가 낯설 때는 그 위의 서사를 읽으며 목적과 배경을 다시 정리하는 방식으로 활용하면 좋습니다.

  

## 1. Core 빌딩 블록

- LangGraph에서 가장 먼저 챙겨야 할 네 가지 기본기(상태 정의, 노드/엣지 구성, 체크포인트, 실행 설정)를 묶어 정리했습니다. 노트북을 처음 보거나 오랜만에 복기할 때는 아래 항목들을 순서대로 따라가며 “상태에서 어떤 키를 노출했는가 → 그 키가 노드에서 어떻게 소비되는가 → 실행 시 어떤 설정으로 이어지는가”를 확인하세요. 각 요소는 서로 결합되므로 한 가지라도 빠뜨리면 나머지 기능이 정상 동작하지 않습니다.

- **State 설계**: `TypedDict` + `Annotated[..., add_messages]`를 기본값으로 쓰고, 분기용 플래그를 상태에 추가해 두면 다른 노드에서 바로 참조할 수 있습니다 (`08-LangGraph-State-Customization.ipynb`). 상태 정의를 할 때 “이 필드는 어느 노드가 생산하고 어느 노드가 소비하는가”를 적어 두면, 추후 수정을 할 때도 불필요한 리듀서 충돌을 막을 수 있습니다. 특히 `add_messages` 리듀서를 붙인 키는 기존 리스트에 append 되므로, 값을 덮어쓰고 싶은 경우 별도의 키를 마련하거나 `tool_call_id`를 맞춰 같은 메시지를 업데이트해야 합니다.

  

```python

from typing import Annotated

from typing_extensions import TypedDict

from langgraph.graph.message import add_messages

  

class State(TypedDict):

messages: Annotated[list, add_messages]

ask_human: bool # 조건 분기에 활용할 플래그

```

  

- **노드 & 엣지**: 노드는 상태 일부만 돌려주도록 작성하고, `add_edge`/`add_conditional_edges`로 다음 실행 경로를 고정합니다. LLM → Tool → LLM 루프를 만들 땐 도구 호출 여부만 판단하는 라우터 함수를 둡니다 (`02-LangGraph-ChatBot.ipynb`, `03-LangGraph-Agent.ipynb`). 조건부 라우터에는 반드시 모든 반환 값에 대한 매핑을 명시해 예외를 예방하고, 라우터에서 오류가 날 때 어떤 상태가 넘어오는지 로깅해두면 디버깅이 훨씬 수월합니다.

  

```python

from langgraph.graph import StateGraph, START, END

from langgraph.prebuilt import ToolNode

  

graph_builder = StateGraph(State)

graph_builder.add_node("chatbot", chatbot)

  

def route_tools(state: State):

ai_message = state["messages"][-1]

return "tools" if ai_message.tool_calls else END

  

graph_builder.add_node("tools", ToolNode([...]))

graph_builder.add_conditional_edges(

"chatbot",

route_tools,

{"tools": "tools", END: END},

)

graph_builder.add_edge("tools", "chatbot")

graph_builder.add_edge(START, "chatbot")

graph = graph_builder.compile()

```

  

- **체크포인트/쓰레드**: `MemorySaver`를 `compile(checkpointer=...)`에 넣고 `RunnableConfig(configurable={"thread_id": ...})`로 호출하면 스냅샷을 언제든 복구할 수 있습니다 (`04-LangGraph-Agent-With-Memory.ipynb`). 운영 환경에서는 `thread_id`를 사용자 ID나 세션 키와 매핑해 두고, 에러 발생 시 `graph.get_state`로 직전 상태를 덤프 떠두면 재시작이 간단해집니다. 인메모리 저장소이기 때문에 영속 저장이 필요하면 DB 기반 체크포인터로 교체해야 한다는 점도 잊지 마세요.

  

```python

from langgraph.checkpoint.memory import MemorySaver

from langchain_core.runnables import RunnableConfig

  

memory = MemorySaver()

graph = graph_builder.compile(checkpointer=memory)

  

config = RunnableConfig(configurable={"thread_id": "customer-001"})

snapshot = graph.get_state(config)

print(snapshot.values["messages"][-1].content)

```

  

- **실행 구성**: `RunnableConfig`에 `recursion_limit`, `tags`, `interrupt_before` 등을 미리 넣어두면 스트리밍/인터럽트 동작을 제어할 수 있습니다 (`05-LangGraph-Streaming-Outputs.ipynb`). `recursion_limit`은 기본값이 25라서 복잡한 그래프는 의도치 않게 중간에 끊길 수 있으니, 설계 단계에서 적절한 값을 산정해 두어야 합니다. 태그를 함께 붙여두면 LangSmith나 자체 로깅 시스템에서 해당 실행을 빠르게 필터링할 수 있습니다.

  

```python

from langchain_core.runnables import RunnableConfig

  

config = RunnableConfig(

recursion_limit=10,

configurable={"thread_id": "demo"},

tags=["my-rag"],

)

  

for event in graph.stream(

input={"messages": [("user", "최신 뉴스를 요약해줘")]},

config=config,

stream_mode="values",

):

print(event)

```

  

## 2. 그래프 운영 패턴

- 여기서는 설계한 그래프를 운영 단계에서 다루며 자주 맞닥뜨리는 패턴들을 정리했습니다. 스트리밍으로 중간 결과를 노출하거나, 사람 개입이 필요할 때 중단하고 다시 재개하는 흐름, 메시지 누적 관리, 도구 호출 제어, 병렬 분기 구성, 서브그래프 재사용까지 “서비스화” 단계에서 필요한 기능들이 모여 있습니다. 한 번만 실행하는 실험 코드가 아니라, 실제 사용자와 상호작용하는 시스템이라면 아래 항목들을 개별적으로 익혀 두는 것이 좋습니다.

- **스트리밍 전략**: `stream_mode`를 바꾸면 어떤 이벤트를 받을지 즉시 바뀝니다. UI용으로는 `"values"` 또는 `"updates"`를 쓰고, 특정 노드만 보고 싶을 땐 태그나 `interrupt_*` 옵션을 같이 넘겨주세요 (`05-LangGraph-Streaming-Outputs.ipynb`). 동일한 입력이라도 스트리밍 모드에 따라 출력 형태가 완전히 달라지므로, 프런트엔드와 연동하기 전에 Mock 데이터를 만들어 각 모드별 payload 구조를 확인하는 것이 안전합니다.

  

```python

question = "2024년 노벨 문학상 관련 뉴스를 알려줘"

  

for event in graph.stream(

input={"messages": [("user", question)]},

config=config,

stream_mode="updates",

interrupt_before=["tools"],

):

for node, payload in event.items():

print(f"[{node}] keys → {payload.keys()}")

```

  

- **Human-in-the-loop**: 인터럽트가 걸리면 스냅샷을 읽고 툴 메시지나 수동 답변을 삽입한 뒤 재개합니다. 메시지를 꾸밀 때는 기존 `ToolMessage`의 `tool_call_id`를 재사용해야 합니다 (`08-LangGraph-State-Customization.ipynb`). 인터럽트된 상태에서 너무 오래 머물면 세션 타임아웃이 발생할 수 있으니, 운영 환경에서는 작업 타임라인을 UI에 표시하거나 자동 재시도 로직을 두는 것도 고려하세요.

  

```python

from langchain_core.messages import AIMessage, ToolMessage

  

def create_response(response: str, ai_message: AIMessage) -> ToolMessage:

return ToolMessage(content=response, tool_call_id=ai_message.tool_calls[0]["id"])

  

snapshot = graph.get_state(config)

ai_message = snapshot.values["messages"][-1]

  

tool_message = create_response("전문가 답변입니다.", ai_message)

graph.update_state(config, {"messages": [tool_message]})

  

for event in graph.stream(None, config, stream_mode="values"):

print(event.get("messages"))

```

  

- **메시지 관리**: 오래된 메시지를 정리하려면 `RemoveMessage` 리듀서를 그대로 상태 업데이트에 넘기면 됩니다. 실행 종료 직전에 호출하면 대화 길이를 자동으로 제한할 수 있습니다 (`09-LangGraph-DeleteMessages.ipynb`). 상태에 따라 어떤 메시지를 삭제했는지 기록이 남지 않으므로, 규제 준수나 감사 로그가 필요하다면 삭제 전에 별도 저장소에 백업해 두는 것이 좋습니다.

  

```python

from langchain_core.messages import RemoveMessage

  

messages = app.get_state(config).values["messages"]

app.update_state(config, {"messages": RemoveMessage(id=messages[0].id)})

```

  

- **ToolNode 활용**: 도구를 실행하는 노드를 직접 만들 필요 없이 `ToolNode`에 도구 리스트를 넘기면 LLM의 `tool_calls`를 자동으로 처리해 줍니다 (`02-Structures/06-LangGraph-Agentic-RAG.ipynb`). 도구 실행 중 오류가 발생해도 `ToolNode`가 예외를 자동으로 포장해 메시지에 남기므로, 후속 노드에서 오류 메시지를 스캔해 재시도 여부를 판단할 수 있습니다.

  

```python

from langgraph.prebuilt import ToolNode

  

workflow.add_node("agent", agent)

workflow.add_node("retrieve", ToolNode([retriever_tool]))

  

workflow.add_conditional_edges(

"agent",

tools_condition,

{"tools": "retrieve", END: END},

)

```

  

- **병렬/조건 분기**: 조건 함수가 리스트를 반환하면 fan-out이 되고, 각 분기에서 처리한 뒤 공통 노드로 fan-in 시킬 수 있습니다. 분기 선택 로직은 상태 필드만 보면 됩니다 (`11-LangGraph-Branching.ipynb`). 병렬 노드가 많을수록 상태병합 시점에 메시지 순서가 달라질 수 있으니, 결과 정렬 기준(`sort_key`)을 추가하거나 fan-in 노드에서 직접 정렬하는 습관을 들이세요.

  

```python

def route_bc_or_cd(state: State) -> Sequence[str]:

return ["c", "d"] if state["which"] == "cd" else ["b", "c"]

  

builder.add_conditional_edges("a", route_bc_or_cd, ["b", "c", "d"])

for node in ["b", "c", "d"]:

builder.add_edge(node, "e")

```

  

- **Subgraph 재사용**: 자주 쓰는 파이프라인은 서브그래프로 컴파일해서 부모 그래프의 노드로 꽂아 사용하세요. 부모/자식이 공유하는 키만 맞추면 그대로 재사용됩니다 (`13-LangGraph-Subgraph.ipynb`). 서브그래프를 여러 번 호출하면 내부 상태도 반복해서 누적되므로, 필요하다면 서브그래프에서 자체적으로 상태 초기화를 수행하거나 입력 키를 매번 새롭게 주입해야 합니다.

  

```python

subgraph_builder = StateGraph(ChildState)

subgraph_builder.add_node(subgraph_node_1)

subgraph_builder.add_node(subgraph_node_2)

subgraph_builder.add_edge(START, "subgraph_node_1")

subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")

subgraph = subgraph_builder.compile()

  

builder.add_node("node_1", node_1)

builder.add_node("node_2", subgraph) # 서브그래프를 그대로 노드로 등록

builder.add_edge("node_1", "node_2")

```

  

## 3. RAG 및 구조 템플릿 (02-Structures)

- RAG 파이프라인은 작은 블록이 수직으로 쌓인 구조이기 때문에, 어디서든 새로운 노드를 끼워 넣기 쉽고 동시에 의도치 않은 재귀나 반복이 발생하기도 합니다. 아래 항목들은 노트북에서 반복적으로 등장하는 핵심 템플릿으로, “검색 → 평가 → 재검색”과 같은 흐름을 그대로 가져와도 되고, 개별 노드를 바꿔치기해 맞춤형 파이프라인을 구성할 수도 있습니다. 각 템플릿의 상태 정의와 조건 함수가 어떤 역할을 맡는지 서술했으니, 프로젝트에 맞춰 필드를 추가하거나 제한을 강화할 때 참고하세요.

- **기본 그래프**: 질문, 검색 결과, 답변을 분리한 `GraphState`를 정의하고 `retrieve → llm_answer` 흐름을 만든 뒤 `MemorySaver`로 상태를 보존합니다 (`02-Structures/02-LangGraph-Naive-RAG.ipynb`). 최소한의 구조지만, 메시지 히스토리를 함께 저장해 두면 후속 개선(평가, 재작성 등)을 붙이기 쉬운 토대가 됩니다.

  

```python

from typing import Annotated, TypedDict

from langgraph.graph.message import add_messages

  

class GraphState(TypedDict):

question: Annotated[str, "Question"]

context: Annotated[str, "Context"]

answer: Annotated[str, "Answer"]

messages: Annotated[list, add_messages]

  

workflow = StateGraph(GraphState)

workflow.add_node("retrieve", retrieve_document)

workflow.add_node("llm_answer", llm_answer)

workflow.add_edge("retrieve", "llm_answer")

workflow.add_edge("llm_answer", END)

app = workflow.compile(checkpointer=MemorySaver())

```

  

- **Groundedness 체크**: 관련성 점수를 상태에 저장하고 임계값 미달 시 재검색하도록 `add_conditional_edges`를 사용합니다. 동시에 `recursion_limit`과 `GraphRecursionError` 핸들링으로 무한 루프를 막습니다 (`02-Structures/03-LangGraph-Add-Groundedness-Check.ipynb`). 재검색이 잦은 시스템은 비용이 급증할 수 있으니, 상태에 재검색 횟수를 기록해 모니터링하는 것도 추천합니다.

  

```python

from typing import Annotated, TypedDict

from langgraph.graph.message import add_messages

from langgraph.errors import GraphRecursionError

from langchain_core.runnables import RunnableConfig

from langchain_teddynote.messages import stream_graph, random_uuid

  

class GraphState(TypedDict):

question: Annotated[str, "Question"]

context: Annotated[str, "Context"]

answer: Annotated[str, "Answer"]

messages: Annotated[list, add_messages]

relevance: Annotated[str, "Relevance"]

  

config = RunnableConfig(recursion_limit=20, configurable={"thread_id": random_uuid()})

  

try:

stream_graph(app, GraphState(question="..."), config, ["retrieve", "relevance_check"])

except GraphRecursionError as err:

print(f"GraphRecursionError: {err}")

```

  

- **웹 검색 결합**: 관련성 체크 후 `"not relevant"`일 때 웹 검색 노드로 보내고, 결과를 다시 `llm_answer`로 연결합니다. 추가 데이터 소스는 `context` 필드에서 바로 병합합니다 (`02-Structures/04-LangGraph-Add-Web-Search.ipynb`). 외부 검색 결과는 포맷이 제각각이므로, `web_search` 노드에서 바로 요약하거나 정규화해 두면 이후 노드에서의 처리 비용이 줄어듭니다.

  

```python

workflow.add_node("web_search", web_search)

workflow.add_conditional_edges(

"relevance_check",

is_relevant,

{"relevant": "llm_answer", "not relevant": "web_search"},

)

workflow.add_edge("web_search", "llm_answer")

```

  

- **쿼리 재작성**: 질문 목록을 누적해 두었다가 최신 질문을 LLM으로 재작성한 뒤 다시 검색하도록 루프를 구성합니다. 진입점을 `query_rewrite`로 설정해 항상 재작성부터 출발하게 합니다 (`02-Structures/05-LangGraph-Add-Query-Rewrite.ipynb`). 재작성 로그를 남기면 사용자가 어떤 표현에서 자주 막히는지 역으로 분석할 수 있으니, 상태에 `rewrite_history` 같은 키를 추가해 저장해 두는 것도 좋습니다.

  

```python

class GraphState(TypedDict):

question: Annotated[list, add_messages]

context: Annotated[str, "Context"]

answer: Annotated[str, "Answer"]

messages: Annotated[list, add_messages]

relevance: Annotated[str, "Relevance"]

  

def query_rewrite(state: GraphState) -> GraphState:

latest = state["question"][-1].content

rewritten = question_rewriter.invoke({"question": latest})

return {"question": rewritten}

  

workflow.add_node("query_rewrite", query_rewrite)

workflow.add_edge("query_rewrite", "retrieve")

workflow.set_entry_point("query_rewrite")

```

  

- **Agentic/Adaptive RAG**: LLM이 자체적으로 도구 호출과 재계획을 수행하도록 `ToolNode`와 평가 노드를 조합합니다. 평가 결과에 따라 `"generate"` 또는 `"rewrite"` 분기로 이동하게 해서 반복 학습을 구현합니다 (`02-Structures/06-LangGraph-Agentic-RAG.ipynb`). 구조가 복잡해지는 만큼, 각 노드에서 남기는 로그 메시지를 일관되게 관리해야 운영 시 혼선을 줄일 수 있습니다.

  

```python

from langgraph.prebuilt import ToolNode, tools_condition

  

workflow.add_node("agent", agent)

workflow.add_node("retrieve", ToolNode([retriever_tool]))

workflow.add_node("rewrite", rewrite)

workflow.add_node("generate", generate)

  

workflow.add_conditional_edges("agent", tools_condition, {"tools": "retrieve", END: END})

workflow.add_conditional_edges("retrieve", grade_documents)

workflow.add_edge("rewrite", "agent")

workflow.add_edge("generate", END)

graph = workflow.compile(checkpointer=MemorySaver())

```

  

## 4. 대표 Use Case 패턴 (03-Use-Cases)

- 실습 노트북 후반부는 실제 업무 시나리오를 모델링한 예제들로 구성돼 있습니다. 여기서는 “어떤 문제를 해결하려고 그래프를 이렇게 설계했는가”와 “그 문제에 맞춰 어떤 노드/상태가 추가되었는가”를 서술한 뒤, 핵심 로직을 담은 코드 블록을 함께 붙였습니다. 필요한 케이스만 골라 재사용해도 되지만, 여러 예제를 조합해 자신만의 복합 시나리오를 만들 때도 참고가 됩니다.

- **Agent Simulation & Prompt Generation**: 시뮬레이터/상담사 두 노드를 조건부 루프로 연결해 자동 대화를 돌립니다. 프롬프트 생성 플로우는 도구 메시지를 삽입하는 노드까지 포함돼 있어 그대로 재사용하면 됩니다 (`03-Use-Cases/01-LangGraph-Agent-Simulation.ipynb`, `02-LangGraph-Prompt-Generation.ipynb`). 사용자 행동을 사전에 정의해 테스트하고 싶을 때 유용하며, 조건 함수만 교체하면 다양한 종료 조건을 실험할 수 있습니다.

  

```python

graph_builder = StateGraph(State)

graph_builder.add_node("simulated_user", simulated_user_node)

graph_builder.add_node("ai_assistant", ai_assistant_node)

graph_builder.add_edge("ai_assistant", "simulated_user")

graph_builder.add_conditional_edges(

"simulated_user",

should_continue,

{"end": END, "continue": "ai_assistant"},

)

simulation = graph_builder.compile()

```

  

- **검증 포함 RAG**: CRAG/Self-RAG 템플릿은 `grader` 노드가 yes/no를 돌려주고 조건부 엣지에서 재작성 또는 웹 검색으로 분기합니다. 답변 평가도 동일 패턴으로 반복해 품질을 보장합니다 (`03-LangGraph-CRAG.ipynb`, `04-LangGraph-Self-RAG.ipynb`). 평가 기준을 명확히 하기 위해 프롬프트에 채점 Rubric을 넣어두고, 점수 결과는 상태에 저장해 추적하세요.

  

```python

workflow.add_node("grade_docs", grade_documents)

workflow.add_node("grade_answer", grade_answer)

workflow.add_conditional_edges("grade_docs", route_docs, {"good": "answer", "bad": "rewrite"})

workflow.add_conditional_edges("grade_answer", route_answer, {"good": END, "bad": "web_search"})

```

  

- **Plan-and-Execute**: Planner가 TODO 리스트를 만들고 Executor가 순차 처리한 뒤 실패하면 Re-plan으로 돌아가는 구조입니다. 상태에 계획과 진행률을 저장해 장기 작업을 추적합니다 (`03-Use-Cases/05-LangGraph-Plan-and-Execute.ipynb`). 긴 프로젝트일수록 계획 리스트를 외부 DB와 동기화해 두면 중간에 프로세스를 재시작해도 안정적입니다.

  

```python

plan_graph.add_node("planner", planner_node)

plan_graph.add_node("executor", executor_node)

plan_graph.add_edge("planner", "executor")

plan_graph.add_conditional_edges("executor", route_execution, {"continue": "executor", "replan": "planner", END: END})

```

  

- **멀티 에이전트 팀**: 노드 간을 라운드 형태로 연결해 각 에이전트가 번갈아가며 작업하도록 구성합니다. 조건 함수가 `END`를 반환하면 루프가 종료됩니다 (`03-Use-Cases/06-LangGraph-Multi-Agent-Collaboration.ipynb`). 복수 에이전트가 동시에 도구 요청을 할 수 있으므로, ToolNode나 기타 공유 자원에 대한 동시성 제어도 함께 고려해야 합니다.

  

```python

workflow = StateGraph(MessagesState) # MessagesState, router 는 앞에서 정의된 상태/라우터 함수

workflow.add_node("researcher", research_node)

workflow.add_node("chart_generator", chart_node)

  

workflow.add_conditional_edges(

"researcher",

router,

{"continue": "chart_generator", END: END},

)

workflow.add_conditional_edges(

"chart_generator",

router,

{"continue": "researcher", END: END},

)

workflow.add_edge(START, "researcher")

app = workflow.compile(checkpointer=MemorySaver())

```

  

- **SQL/Research Assistant**: SQL 워크플로우는 `list_tables → get_schema → query_gen → correct_query → execute_query` 순환 구조를 사용해 쿼리를 검증·실행합니다. 연구 보조 에이전트도 동일하게 노드를 나눠 STORM 단계를 구현합니다 (`03-Use-Cases/09-LangGraph-SQL-Agent.ipynb`, `10-LangGraph-Research-Assistant.ipynb`). DB에 쓰기 권한이 있는 경우 반드시 프롬프트에서 DML 금지 규칙을 명시하고, 도구 레벨에서도 차단 로직을 추가해야 합니다.

  

```python

from langchain_openai import ChatOpenAI

  

workflow = StateGraph(State) # State, create_tool_node_with_fallback, MODEL_NAME, should_continue 는 위에서 정의

model_get_schema = ChatOpenAI(model=MODEL_NAME, temperature=0).bind_tools([get_schema_tool])

workflow.add_node("first_tool_call", first_tool_call)

workflow.add_node("list_tables_tool", create_tool_node_with_fallback([list_tables_tool]))

workflow.add_node("model_get_schema", lambda state: {"messages": [model_get_schema.invoke(state["messages"])]})

workflow.add_node("get_schema_tool", create_tool_node_with_fallback([get_schema_tool]))

workflow.add_node("query_gen", query_gen_node)

workflow.add_node("correct_query", model_check_query)

workflow.add_node("execute_query", create_tool_node_with_fallback([db_query_tool]))

  

workflow.add_edge(START, "first_tool_call")

workflow.add_edge("first_tool_call", "list_tables_tool")

workflow.add_edge("list_tables_tool", "model_get_schema")

workflow.add_edge("model_get_schema", "get_schema_tool")

workflow.add_edge("get_schema_tool", "query_gen")

workflow.add_conditional_edges("query_gen", should_continue)

workflow.add_edge("correct_query", "execute_query")

workflow.add_edge("execute_query", "query_gen")

app = workflow.compile(checkpointer=MemorySaver())

```

  

## 5. 프로젝트 적용 체크리스트

- 위의 패턴을 실제 프로젝트에 이식할 때 빠뜨리기 쉬운 항목들을 다시 정리했습니다. 새 그래프를 만들 때마다 아래 항목을 순서대로 점검하면, 설계 누락이나 운영 중 발생하는 장애를 크게 줄일 수 있습니다.

1. **상태 정의**: 프로젝트에서 추적해야 할 모든 데이터(메시지, 플래그, 평가 점수 등)를 `TypedDict`로 선언하고 필요한 리듀서를 지정합니다.

2. **노드 모듈화**: LLM 호출, 도구 실행, 평가/재계획 단계를 각각 함수로 나누고 독립적으로 테스트합니다.

3. **분기 설계**: 성공/실패/재시도 기준을 명시적으로 함수로 작성해 `add_conditional_edges`에 연결합니다.

4. **체크포인트 & 인터럽트**: 운영 환경에서는 반드시 `MemorySaver` 또는 LangSmith 체크포인터를 사용하고, 위험 지점에는 `interrupt_before`로 인간 검수를 붙입니다.

5. **모니터링**: `graph.get_state(config)`로 중간 상태를 점검하고, 필요 시 `update_state`나 `RemoveMessage`로 데이터를 조정하는 경로를 마련합니다.

6. **스트리밍 UI 연동**: 사용자 경험이 중요한 경우 `stream_mode`와 태그 필터를 활용해 메시지를 실시간으로 표시하거나 특정 노드 결과만 UI에 노출합니다.

  

## 6. 빠른 시작 코드 조각

```python

from typing import Annotated, TypedDict

from langgraph.graph import StateGraph, START, END

from langgraph.graph.message import add_messages

from langgraph.checkpoint.memory import MemorySaver

  

class State(TypedDict):

messages: Annotated[list, add_messages]

ask_human: bool

  

def chatbot(state: State):

# TODO: LLM 호출 + 도구 바인딩

return {"messages": [...], "ask_human": False}

  

def decide_next(state: State):

return "human" if state["ask_human"] else END

  

graph = StateGraph(State)

graph.add_node("chatbot", chatbot)

graph.add_node("human", lambda state: {"messages": [...]})

graph.add_conditional_edges("chatbot", decide_next, {"human": "human", END: END})

graph.add_edge(START, "chatbot")

graph.add_edge("human", "chatbot")

  

app = graph.compile(checkpointer=MemorySaver(), interrupt_before=["human"])

config = {"configurable": {"thread_id": "42"}}

  

for event in app.stream({"messages": [("user", "안녕하세요")]}, config, stream_mode="values"):

...

```

  

위 패턴에 프로젝트 요구사항(도구, 평가, 멀티 에이전트 등)을 덧붙여 17장 전체 예제를 재조합하면 실전용 LangGraph 워크플로우를 빠르게 구축할 수 있습니다.