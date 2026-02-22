# LLM 에 도구 바인딩(Binding Tools)

LLM 모델이 도구(tool) 를 호출할 수 있으려면 chat 요청을 할 때 모델에 도구 스키마(tool schema) 를 전달해야 합니다. 

도구 호출(tool calling) 기능을 지원하는 LangChain Chat Model 은 `.bind_tools()` 메서드를 구현하여 LangChain 도구 객체, Pydantic 클래스 또는 JSON 스키마 목록을 수신하고 공급자별 예상 형식으로 채팅 모델에 바인딩(binding) 합니다.

바인딩된 Chat Model 의 후속 호출은 모델 API에 대한 모든 호출에 도구 스키마를 포함합니다.

주어진 도구들을 활용해서 스스로 판단해서 도구들을 활용해서 지식을 보강을 하거나
해야할 상황들에 따라 도구를 적절히 사용할 수 있다 
툴 콜링 = 펑션콜링 = 바인딩 툴스 = LLM에게 도구를 쥐어주는 행위

아쉽게돈 모든 모델이 지원하진 않음
도구를 사용할 줄 아는 LLM이 있음
근데 최신 모델들은 대부분 가능함 
# API KEY를 환경변수로 관리하기 위한 설정 파일
from dotenv import load_dotenv

# API KEY 정보로드
load_dotenv()
# LangSmith 추적을 설정합니다. https://smith.langchain.com
# !pip install -qU langchain-teddynote
from langchain_teddynote import logging

# 프로젝트 이름을 입력합니다.
logging.langsmith("CH15-Bind-Tools")
## LLM에 바인딩할 Tool 정의

실험을 위한 도구(tool) 를 정의합니다.

- `get_word_length` : 단어의 길이를 반환하는 함수
- `add_function` : 두 숫자를 더하는 함수
- `naver_news_crawl` : 네이버 뉴스 기사를 크롤링하여 본문 내용을 반환하는 함수

**참고**
- 도구를 정의할 때 `@tool` 데코레이터를 사용하여 도구를 정의합니다.
- docstring 은 가급적 영어로 작성하는 것을 권장합니다.
import re
import requests
from bs4 import BeautifulSoup
from langchain.agents import tool


# 도구를 정의합니다.
@tool
def get_word_length(word: str) -> int:
    """Returns the length of a word."""
    return len(word)


@tool
def add_function(a: float, b: float) -> float:
    """Adds two numbers together."""
    return a + b


@tool
def naver_news_crawl(news_url: str) -> str:
    """Crawls a 네이버 (naver.com) news article and returns the body content."""
    # HTTP GET 요청 보내기
    response = requests.get(news_url)

    # 요청이 성공했는지 확인
    if response.status_code == 200:
        # BeautifulSoup을 사용하여 HTML 파싱
        soup = BeautifulSoup(response.text, "html.parser")

        # 원하는 정보 추출
        title = soup.find("h2", id="title_area").get_text()
        content = soup.find("div", id="contents").get_text()
        cleaned_title = re.sub(r"\n{2,}", "\n", title)
        cleaned_content = re.sub(r"\n{2,}", "\n", content)
    else:
        print(f"HTTP 요청 실패. 응답 코드: {response.status_code}")

    return f"{cleaned_title}\n{cleaned_content}"


tools = [get_word_length, add_function, naver_news_crawl]
## bind_tools() 로 LLM 에 도구 바인딩

llm 모델에 `bind_tools()` 를 사용하여 도구를 바인딩합니다.
from langchain_openai import ChatOpenAI

# 모델 생성
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# 도구 바인딩
llm_with_tools = llm.bind_tools(tools)
실행결과를 확인합니다. 

결과는 `tool_calls` 에 저장됩니다. 따라서, `.tool_calls` 를 확인하여 도구 호출 결과를 확인할 수 있습니다.

**참고**
- `name` 은 도구의 이름을 의미합니다.
- `args` 는 도구에 전달되는 인자를 의미합니다.
- `type` 에 tool_call이 없으면 툴을 사용하지 않은 것 
# 실행 결과
llm_with_tools.invoke("What is the length of the word 'teddynote'?").tool_calls
# teddynote의 길이를 물어봤기 때문에 get_word_length를 사용하는게 이상적
다음으로는 `llm_with_tools` 와 `JsonOutputToolsParser` 를 연결하여 `tool_calls` 를 parsing 하여 결과를 확인합니다.
from langchain_core.output_parsers.openai_tools import JsonOutputToolsParser

# 도구 바인딩 + 도구 파서
chain = llm_with_tools | JsonOutputToolsParser(tools=tools)

# 실행 결과
tool_call_results = chain.invoke("What is the length of the word 'teddynote'?")
print(tool_call_results)
실행 결과는 다음과 같습니다.

**참고**
- `type`: 도구의 이름
- `args`: 도구에 전달되는 인자
print(tool_call_results, end="\n\n==========\n\n")
# 첫 번째 도구 호출 결과
single_result = tool_call_results[0]
# 도구 이름
print(single_result["type"])
# 도구 인자
print(single_result["args"])
도구 이름과 일치하는 도구를 찾아 실행합니다.
tool_call_results[0]["type"], tools[0].name
`execute_tool_calls` 함수는 도구를 찾아 args 를 전달하여 도구를 실행합니다.

즉, `type` 은 도구의 이름을 의미하고 `args` 는 도구에 전달되는 인자를 의미합니다.
def execute_tool_calls(tool_call_results):
    """
    도구 호출 결과를 실행하는 함수

    :param tool_call_results: 도구 호출 결과 리스트
    :param tools: 사용 가능한 도구 리스트
    """
    # 도구 호출 결과 리스트를 순회합니다.
    for tool_call_result in tool_call_results:
        # 도구의 이름과 인자를 추출합니다.
        tool_name = tool_call_result["type"]  # 도구의 이름(함수명)
        tool_args = tool_call_result["args"]  # 도구에 전달되는 인자

        # 도구 이름과 일치하는 도구를 찾아 실행합니다.
        # next() 함수를 사용하여 일치하는 첫 번째 도구를 찾습니다.
        matching_tool = next((tool for tool in tools if tool.name == tool_name), None)

        if matching_tool:
            # 일치하는 도구를 찾았다면 해당 도구를 실행합니다.
            result = matching_tool.invoke(tool_args)
            # 실행 결과를 출력합니다.
            print(f"[실행도구] {tool_name} [Argument] {tool_args}\n[실행결과] {result}")
        else:
            # 일치하는 도구를 찾지 못했다면 경고 메시지를 출력합니다.
            print(f"경고: {tool_name}에 해당하는 도구를 찾을 수 없습니다.")


# 도구 호출 실행
# 이전에 얻은 tool_call_results를 인자로 전달하여 함수를 실행합니다.
execute_tool_calls(tool_call_results)
## bind_tools + Parser + Execution

이번에는 일련의 과정을 한 번에 실행합니다.

- `llm_with_tools` : 도구를 바인딩한 모델
    도구를 써야 겠다는 판단까지만 하고 완성된 답변을 내뱉는건 아님
- `JsonOutputToolsParser` : 도구 호출 결과를 파싱하는 파서
- `execute_tool_calls` : 도구 호출 결과를 실행하는 함수

**흐름 정리**
1. 모델에 도구를 바인딩
2. 도구 호출 결과를 파싱
3. 도구 호출 결과를 실행
from langchain_core.output_parsers.openai_tools import JsonOutputToolsParser

# bind_tools + Parser + Execution
chain = llm_with_tools | JsonOutputToolsParser(tools=tools) | execute_tool_calls
# 실행 결과
chain.invoke("What is the length of the word 'teddynote'?")
# 실행 결과
chain.invoke("114.5 + 121.2")
print(114.5 + 121.2)
# 실행 결과
chain.invoke(
    "뉴스 기사 내용을 크롤링해줘: https://n.news.naver.com/mnews/hotissue/article/092/0002347672?type=series&cid=2000065"
)
## bind_tools > Agent & AgentExecutor 로 대체

`bind_tools()` 는 모델에 사용할 수 있는 스키마(도구)를 제공합니다. 

`AgentExecutor` 는 실제로 llm 호출, 올바른 도구로 라우팅, 실행, 모델 재호출 등을 위한 실행 루프를 생성합니다.

**참고**
- `Agent` 와 `AgentExecutor` 에 대해서는 다음 장에서 자세히 다룹니다.
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_openai import ChatOpenAI

# Agent 프롬프트 생성
prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "You are very powerful assistant, but don't know current events",
        ),
        ("user", "{input}"),
        MessagesPlaceholder(variable_name="agent_scratchpad"),
    ]
)

# 모델 생성
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
from langchain.agents import create_tool_calling_agent
from langchain.agents import AgentExecutor

# 이전에 정의한 도구 사용
tools = [get_word_length, add_function, naver_news_crawl]

# Agent 생성
agent = create_tool_calling_agent(llm, tools, prompt)

# AgentExecutor 생성
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    handle_parsing_errors=True,
)
# Agent 실행
result = agent_executor.invoke({"input": "How many letters in the word `teddynote`?"})

# 결과 확인
print(result["output"])
# Agent 실행
result = agent_executor.invoke({"input": "114.5 + 121.2 의 계산 결과는?"})

# 결과 확인
print(result["output"])
한 번의 실행으로 끝나는 것이 아닌, 모델이 자신의 결과를 확인하고 다시 자신을 호출하는 과정을 거칩니다.
# Agent 실행
result = agent_executor.invoke(
    {"input": "114.5 + 121.2 + 34.2 + 110.1 의 계산 결과는?"}
)

# 결과 확인
print(result["output"])
print("==========\n")
print(114.5 + 121.2 + 34.2 + 110.1)
이번에는 뉴스 결과를 크롤링 해서 요약 해달라는 요청을 수행합니다.

result = agent_executor.invoke(
    {
        "input": "뉴스 기사를 요약해 줘: https://n.news.naver.com/mnews/hotissue/article/092/0002347672?type=series&cid=2000065"
    }
)
print(result["output"])