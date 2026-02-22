엔티티 메모리는 대화에서 특정 엔티티에 대한 주어진 사실을 기억함
엔티티 메모리는 엔티티에 대한 정보를 추출하고(LLM 사용) 시간이 지남에 따라 해당 엔티티에 대한 지식을 축적함(역시 LLM 사용)
- 이전에 사용하던 메모리들과는 동작 방식이 다름
    - 이전에는 토큰/턴을 기준으로 대화 내용의 절대적인 양을 조정함
    - 대화가 길어지면 주제가 이어지다가 끊기는 부분이 분명히 발생할 수 있음 
    - 이를 보완하기 위해 과거 대화에서 엔티티(핵심 정보)를 추출함
    - 대화에는 부가적인 내용도 많으므로, 향후 유용할 정보만 압축하여 보관하도록 설계된 메모리임
 엔티티를 추출하기 위한 템플릿도 별도로 존재함
```python
from langchain_openai import ChatOpenAI
from langchain.chains import ConversationChain
from langchain.memory import ConversationEntityMemory
from langchain.memory.prompt import ENTITY_MEMORY_CONVERSATION_TEMPLATE
```

Entity 메모리를 효과적으로 사용하기 위해, 제공되는 프롬프트를 사용함
```python
# Entity Memory를 사용하는 프롬프트 내용을 출력합니다.
print(ENTITY_MEMORY_CONVERSATION_TEMPLATE.template)
```
- LLM을 사용해 과거 대화에서 중요한 내용을 추출하는 것 
```python
# LLM 을 생성합니다.
llm = ChatOpenAI(model_name="gpt-4o", temperature=0)

# ConversationChain 을 생성합니다.
conversation = ConversationChain(
	llm=llm,
	prompt=ENTITY_MEMORY_CONVERSATION_TEMPLATE,
	memory=ConversationEntityMemory(llm=llm),
)
```
대화를 시작합니다.

입력한 대화를 바탕으로 `ConversationEntityMemory` 는 주요 Entity 정보를 별도로 저장합니다.
- LLM이 꼭 같을 필요는 없음. 엔티티를 추출하는 모델은 더 저비용 모델을 사용해도 됨

```python
conversation.predict(
	input="테디와 셜리는 한 회사에서 일하는 동료입니다."
	"테디는 개발자이고 셜리는 디자이너입니다. "
	"그들은 최근 회사에서 일하는 것을 그만두고 자신들의 회사를 차릴 계획을 세우고 있습니다."
)
```

엔티티는 memory.entity_store.store에서 확인 가능함 
```python
conversation.memory.entity_store.store
```
- 대화가 길어지더라도 과거의 핵심 내용만 기억하므로 효율적으로 관리 가능
- 중간에 LLM을 사용하므로 토큰 비용 산정을 고려해야 함 
- 지식 그래프 형식의 메모리와 비슷함 [[ConversationKGMemory]]