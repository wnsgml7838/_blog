# ConversationKGMemory
지식 그래프를 활용해 정보를 저장하고 불러옴
이를 통해 모델이 서로 다른 개체 간의 관계를 이해하는 데 도움을 주고, 복잡한 연결망과 맥락 기반 대응 능력을 향상시킴 

```python
from langchain_openai import ChatOpenAI
from langchain.memory import ConversationKGMemory

llm = ChatOpenAI(temperature=0)
memory = ConversationKGMemory(llm=llm, return_messages=True)

memory.save_context(
	{"input": "이쪽은 Pangyo 에 거주중인 김셜리씨 입니다."},
	{"output": "김셜리씨는 누구시죠?"},
)

memory.save_context(
	{"input": "김셜리씨는 우리 회사의 신입 디자이너입니다."},
	{"output": "만나서 반갑습니다."},
)

memory.load_memory_variables({"input": "김셜리씨는 누구입니까?"})
```

```
{'history': [SystemMessage(content='On Pangyo: Pangyo has resident 김셜리씨.', additional_kwargs={}, response_metadata={}), SystemMessage(content='On 김셜리씨: 김셜리씨 is a 신입 디자이너. 김셜리씨 is in 우리 회사.', additional_kwargs={}, response_metadata={})]}
```
- 엔티티 메모리와 유사한 성격 [[ConversationEntityMemory]]
- 유용한 경우: 대화 내 개체 간 연결고리가 존재하고 이를 파악·정리해야 할 때 

## Chain에 메모리 활용하기
`ConversationChain`에 `ConversationKGMemory`를 메모리로 지정하여 대화를 나눈 후 memory를 확인해 보겠습니다.
```python
from langchain.prompts.prompt import PromptTemplate
from langchain.chains import ConversationChain

llm = ChatOpenAI(temperature=0)

template = """The following is a friendly conversation between a human and an AI.
The AI is talkative and provides lots of specific details from its context.
If the AI does not know the answer to a question, it truthfully says it does not know.
The AI ONLY uses information contained in the "Relevant Information" section and does not hallucinate.

Relevant Information:
{history}

Conversation:
Human: {input}

AI:"""

prompt = PromptTemplate(input_variables=["history", "input"], template=template)

conversation_with_kg = ConversationChain(
    llm=llm,
    prompt=prompt,
    memory=ConversationKGMemory(llm=llm),
)
```
- 프롬프트를 함께 설정함으로써 지식 그래프 형식으로 정리된 메모리를 확인할 수 있음


첫 번째 대화를 시작, 간단한 인물에 대한 정보 제공 
```python
conversation_with_kg.predict(

input="My name is Teddy. Shirley is a coworker of mine, and she's a new designer at our company."

)
```

```
"Hello Teddy! It's nice to meet you. Shirley must be excited to be starting a new job as a designer at your company. I hope she's settling in well and getting to know everyone. If you need any tips on how to make her feel welcome or help her adjust to the new role, feel free to ask me!"
```


Shirley에 대한 질문을 진행합니다.
```python
conversation_with_kg.memory.load_memory_variables({"input": "who is Shirley?"})
```

```
{'history': 'On Shirley: Shirley is a coworker. Shirley is a new designer. Shirley is at company.'}
```
- 메모리/지식 그래프를 기반으로 답변함
- 비슷한 맥락의 핵심 정보를 지식 그래프로 저장할지, 엔티티로 추출해 저장할지의 차이 