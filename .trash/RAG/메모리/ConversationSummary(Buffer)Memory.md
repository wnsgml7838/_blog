조금 더 복잡한 메모리 유형.
시간 경과에 따른 **대화 요약**을 생성함. 장시간 대화의 정보를 압축하는 데 유용
대화 요약 메모리는 진행 중인 대화를 요약하고 **현재 요약을 메모리에 저장**함
이후 이 메모리를 사용해 지금까지의 대화 요약을 프롬프트/체인에 삽입할 수 있음
과거 메시지 기록을 그대로 프롬프트에 넣기 어려운 긴 대화에서 특히 유용
>[!note]
>테디노트 개인적 소감: 요약 메모리를 선호함
>엔티티/지식 그래프에 원하는 디테일이 없으면 답변이 어려울 수 있음
>요약에도 정보 손실은 있으나 → 버퍼 메모리로 보완 가능 

```python
from langchain.memory import ConversationSummaryMemory
from langchain_openai import ChatOpenAI

memory = ConversationSummaryMemory(
    llm=ChatOpenAI(model_name="gpt-4o", temperature=0),
    return_messages=True,
)

memory.save_context(
	inputs={"human": "유럽 여행 패키지의 가격은 얼마인가요?"},
	outputs={
	"ai": "유럽 14박 15일 패키지의 기본 가격은 3,500유로입니다. 이 가격에는 항공료, 호텔 숙박비, 지정된 관광지 입장료가 포함되어 있습니다. 추가 비용은 선택하신 옵션 투어나 개인 경비에 따라 달라집니다."},
)

memory.save_context(
	inputs={"human": "여행 중에 방문할 주요 관광지는 어디인가요?"},
	outputs={"ai": "이 여행에서는 파리의 에펠탑, 로마의 콜로세움, 베를린의 브란덴부르크 문, 취리히의 라이네폴 등 유럽의 유명한 관광지들을 방문합니다. 각 도시의 대표적인 명소들을 포괄적으로 경험하실 수 있습니다."},
)

```

저장된 메모리의 history를 확인
이전 대화를 압축적으로 요약한 내용을 확인할 수 있음
```python
# 저장된 메모리 확인
print(memory.load_memory_variables({})["history"])
```

```
[SystemMessage(content="The human asks about the price of a European travel package. The AI responds that the basic price for a 14-night, 15-day European package is 3,500 euros, which includes airfare, hotel accommodations, and entrance fees to designated tourist sites. Additional costs may vary based on optional tours or personal expenses. The human then inquires about the major tourist attractions included in the trip. The AI explains that the package includes visits to famous European landmarks such as the Eiffel Tower in Paris, the Colosseum in Rome, the Brandenburg Gate in Berlin, and the Rhine Falls in Zurich, offering a comprehensive experience of each city's iconic sites. The human asks if travel insurance is included, and the AI confirms that basic travel insurance is provided for all travelers, covering medical expenses and emergency support, with options for additional coverage available. The human asks if it is possible to upgrade to business class seats on the flight and what the cost would be. The AI confirms that an upgrade to business class is possible, with an additional cost of approximately 1,200 euros for a round trip, offering benefits such as wider seats, superior in-flight meals, and additional baggage allowance. The human inquires about the hotel ratings included in the package, and the AI states that the package includes accommodations in 4-star hotels, which offer comfort and convenience, are centrally located for easy access to tourist sites, and provide excellent service and amenities. The human asks for more details about meal options, and the AI explains that the package includes daily breakfast at the hotel, while lunch and dinner are not included, allowing travelers to explore local cuisine. The AI also provides a list of recommended restaurants in each city to enhance the local dining experience. The human asks about the deposit and cancellation policy for booking the package. The AI states that a deposit of 500 euros is required at the time of booking. The cancellation policy allows for a full refund if canceled 30 days before the booking date, but the deposit is non-refundable if canceled after that. If canceled 14 days before the start of the trip, 50% of the cost is charged, and after that, the full cost is charged.", additional_kwargs={}, response_metadata={})]
```

# ConversationSummaryBufferMemory
- 최근 대화 내용의 버퍼를 메모리에 유지하되, 이전 대화는 완전히 플러시하지 않고 요약으로 컴파일(두 가지를 병행)
- 대화 내용을 플러시할 시기는 상호작용 개수가 아니라 토큰 길이를 기준으로 결정

```python
from langchain_openai import ChatOpenAI
from langchain.memory import ConversationSummaryBufferMemory

llm = ChatOpenAI()
memory = ConversationSummaryBufferMemory(
	llm=llm,
	max_token_limit=200, # 요약의 기준이 되는 토큰 길이를 설정합니다.
	return_messages=True,
)
```
200 토큰까지는 원본을 그대로 유지하고, 200 토큰을 넘기면 요약함


메모리에 저장된 대화를 확인
아직 대화 요약은 수행되지 않음(200 토큰 기준에 도달하지 않았기 때문)
```python
# 메모리에 저장된 대화내용 확인
memory.load_memory_variables({})["history"]
```

```
[HumanMessage(content='유럽 여행 패키지의 가격은 얼마인가요?', additional_kwargs={}, response_metadata={}), AIMessage(content='유럽 14박 15일 패키지의 기본 가격은 3,500유로입니다. 이 가격에는 항공료, 호텔 숙박비, 지정된 관광지 입장료가 포함되어 있습니다. 추가 비용은 선택하신 옵션 투어나 개인 경비에 따라 달라집니다.', additional_kwargs={}, response_metadata={})]
```

대화를 추가로 저장하여 200 토큰 제한을 넘기도록 해 보겠습니다.
```python
memory.save_context(
	inputs={"human": "여행 중에 방문할 주요 관광지는 어디인가요?"},
	outputs={"ai": "이 여행에서는 파리의 에펠탑, 로마의 콜로세움, 베를린의 브란덴부르크 문, 취리히의 라이네폴 등 유럽의 유명한 관광지들을 방문합니다. 각 도시의 대표적인 명소들을 포괄적으로 경험하실 수 있습니다."},
)

memory.save_context(
	inputs={"human": "여행자 보험은 포함되어 있나요?"},
	outputs={"ai": "네, 모든 여행자에게 기본 여행자 보험을 제공합니다. 이 보험은 의료비 지원, 긴급 상황 발생 시 지원 등을 포함합니다. 추가적인 보험 보장을 원하시면 상향 조정이 가능합니다."},
)

memory.save_context(
	inputs={"human": "항공편 좌석을 비즈니스 클래스로 업그레이드할 수 있나요? 비용은 어떻게 되나요?"},
	outputs={"ai": "항공편 좌석을 비즈니스 클래스로 업그레이드하는 것이 가능합니다. 업그레이드 비용은 왕복 기준으로 약 1,200유로 추가됩니다. 비즈니스 클래스에서는 더 넓은 좌석, 우수한 기내식, 그리고 추가 수하물 허용량 등의 혜택을 제공합니다."},
)

memory.save_context(
	inputs={"human": "패키지에 포함된 호텔의 등급은 어떻게 되나요?"},
	outputs={"ai": "이 패키지에는 4성급 호텔 숙박이 포함되어 있습니다. 각 호텔은 편안함과 편의성을 제공하며, 중심지에 위치해 관광지와의 접근성이 좋습니다. 모든 호텔은 우수한 서비스와 편의 시설을 갖추고 있습니다."},
)
```

저장된 대화 내용을 확인합니다. 가장 최근 1개 대화는 요약되지 않았고, 그 이전 대화는 요약본으로 저장되어 있음
```python
# 메모리에 저장된 대화내용 확인
memory.load_memory_variables({})["history"]
```

```
[SystemMessage(content='The human asks about the price of a European travel package and the AI responds that it is 3,500 euros for a 14-night 15-day package, including airfare and hotel accommodation. Optional tours or personal expenses may incur additional costs. When asked about tourist attractions, the AI mentions landmarks like the Eiffel Tower, Colosseum, Brandenburg Gate, and Rhine Falls. The AI also includes basic travel insurance for all travelers, covering medical expenses and emergency support, with the option for additional coverage. The human inquires about upgrading the flight seats to business class and the AI confirms it is possible, with an additional cost of approximately 1,200 euros for the round trip. Business class offers benefits such as wider seats, premium meals, and increased luggage allowance.', additional_kwargs={}, response_metadata={}), HumanMessage(content='패키지에 포함된 호텔의 등급은 어떻게 되나요?', additional_kwargs={}, response_metadata={}), AIMessage(content='이 패키지에는 4성급 호텔 숙박이 포함되어 있습니다. 각 호텔은 편안함과 편의성을 제공하며, 중심지에 위치해 관광지와의 접근성이 좋습니다. 모든 호텔은 우수한 서비스와 편의 시설을 갖추고 있습니다.', additional_kwargs={}, response_metadata={})]
```

> [!note]
> 강의 추천: 이 방식을 가장 추천
> 최근 일부 토큰까지는 대화를 그대로 보존하고,
> 이전 대화는 요약본을 활용하는 편이 효율적
> 엔티티/지식 그래프 메모리는 구조화 저장이라 원문 기반의 요약·번역이 어려울 수 있음
> 반면, 이 방식은 최근 n개는 원본으로 남으므로 “이전 대화를 번역/요약” 요청에 유리함