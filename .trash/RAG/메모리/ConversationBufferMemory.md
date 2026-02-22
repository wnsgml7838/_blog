# ConversationBufferMemory

## 정의

메시지를 저장하고, 이후 변수로 추출할 수 있게 해 주는 메모리입니다.

> [!note] 핵심 개념
> - 메모리 중에 가장 기본이 되는 메모리
> - Conversation에 활용하는 메모리로 이해 (buffer라는 단어에 집중하지 않아도 됨)
> - 사람의 입력과 AI의 답변을 페어로 저장 → 다른 메모리들도 비슷한 구조

## 주요 특징

- Human이 질문을 했을 때, 질문만 단독으로 저장하지 않음
- 질문에 대한 AI의 답변을 항상 쌍으로 저장  

## 기본 사용법

```python
from langchain.memory import ConversationBufferMemory

memory = ConversationBufferMemory()

memory.save_context(
    inputs={
        "human": "안녕하세요"
    },
    outputs={
        "ai": "안녕하십니까?"
    }
)
```

> [!info] save_context 메서드
> - 대화 내용을 하나씩 저장 가능
> - `inputs`, `outputs` → 복수형(s)으로 사용
> - 메모리에 한 개의 대화 턴이 저장됨 (질문-답변 쌍)

### load_memory_variables() 함수

메시지 히스토리를 반환합니다.

```python
memory.load_memory_variables({})
```

- `history`라는 key에 대화 내역이 담겨 있음

### return_messages 옵션

ConversationBufferMemory를 생성할 때 `return_messages=True` 인자값을 주면, Human 메시지와 AI 메시지 객체를 반환합니다.
```python
from langchain.memory import ConversationBufferMemory

memory = ConversationBufferMemory(return_messages=True)

memory.save_context(
    inputs={
        "human": "안녕하세요"
    },
    outputs={
        "ai": "안녕하십니까?"
    }
)

memory.save_context(
    inputs={
        "human": "안녕하세요222"
    },
    outputs={
        "ai": "안녕하십니까?222"
    }
)

memory.load_memory_variables({})['history']
```

- 이전에는 문자열 형식으로 나왔다면, `HumanMessage()`, `AIMessage()`와 같은 객체 형식으로 제공됨
- 이렇게 객체로 만들면 이후 체인에 대화 내용을 가공 없이 바로 입력으로 전달할 수 있음
- `ChatPromptTemplate`의 `messages`도 객체 리스트를 받으므로 문자열 대신 메시지 객체를 그대로 사용할 수 있음

## Chain에 적용

```python
from langchain.chains import ConversationChain
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(temperature=0, model_name='gpt-4o')

conversation = ConversationChain(
    llm=llm,
    memory=ConversationBufferMemory(),
)
```

- `ConversationChain` → 메모리를 생성해 주입하면 자동으로 대화 내역을 기록해 줌
- `ConversationChain`은 향후 deprecated 예정. 실습에서는 사용하되 실제 구현은 `RunnableWithMessageHistory` 사용 권장

### ConversationChain을 사용해서 대화를 진행

```python
response = conversation.predict(
    input="안녕하세요, 비대면으로 은행 계좌를 개설하고 싶습니다"
)
print(response)
```

### 이전의 대화 기록을 기억하고 있는지 확인

```python
response = conversation.predict(
    input="이전 답변을 불렛포인트 형식으로 정리해서 알려줘"
)
print(response)
```

## 정리

- 가장 기본적인 메모리
- 모든 대화 내용을 저장하므로 무한정 보관할 수는 없음
- 버퍼 메모리는 가용 범위 내에서 계속 저장함
- 대화가 길어지면 입력 토큰 허용치를 초과할 수 있음
- 이 경우 오류가 발생할 수 있으므로 적절한 컷오프가 필요함
- 관련 내용은 [[ConversationBufferWindowMemory]]에서 다룸