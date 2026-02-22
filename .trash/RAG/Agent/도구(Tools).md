>[!info]
>도구는 에이전트, 체인 또는 LLM이 외부 세계와 상호작용하기 위한 인터페이스
>Langchain에서 시본 제공하는 도구를 사용하여 쉽게 도구를 활용할 수 있으며, 사용자 정의 도구(Custom Tool)를 쉽게 구축하는 것도 가능하다.
>LangChain에 통합된 도구 리스트는 아래 링크 확인
>https://python.langchain.com/v0.1/docs/integrations/tools/

# 빌트인 도구(built-in tools)
랭체인에서 제공하는 사전에 정의된 도구와 툴킷을 사용할 수 있다
tool은 단일 도구를 의미하며, toolkit은 여러 도구를 묶어서 하나의 도구로 사용할 수 있음
관련 도구는 아래 링크에서 참고
https://python.langchain.com/docs/integrations/tools/

**사용 방법만 정리하는게 좋을듯 ?**
invoke로 수행할 수 있음 

# 사용자 정의 도구(Custom Tool)

LangChain 에서 제공하는 빌트인 도구 외에도 사용자가 직접 도구를 정의하여 사용할 수 있습니다.
이를 위해서는 `langchain.tools` 모듈에서 제공하는 `tool` 데코레이터를 사용하여 함수를 도구로 변환합니다.

## @tool 데코레이터
이 데코레이터는 함수를 도구로 변환하는 기능을 제공합니다. 다양한 옵션을 통해 도구의 동작을 커스터마이즈할 수 있습니다.

**사용 방법**
1. 함수 위에 `@tool` 데코레이터 적용
2. 필요에 따라 데코레이터 매개변수 설정

이 데코레이터를 사용하면 일반 Python 함수를 강력한 도구로 쉽게 변환할 수 있으며, 자동화된 문서화와 유연한 인터페이스 생성이 가능합니다.

```python
from langchain.tools import tool

# 데코레이터를 사용하여 함수를 도구로 변환합니다.
@tool
def add_numbers(a: int, b: int) -> int:
"""Add two numbers"""
return a + b

@tool
def multiply_numbers(a: int, b: int) -> int:
"""Multiply two numbers"""
return a * b

# 도구 실행

add_numbers.invoke({"a": 3, "b": 4})
```

>[!note]
>이 함수는 우리가 정의했기 때문에 LLM이 이 tool을 언제 호출해야하는지 모른다
>여기서 중요한게 doc string 

>[!info]
>doc string이란?
>"""Add two numbers"""
>함수를 호출 할 때 어떤 상황에서 호출해야하는지 """ """ 안에 넣어서 알려주는 것
>나중에 LLM한테 tool을 줬을 때 이 doc string을 보고 상황판단을 하게됨 
>반드시 써주는게 중요!!, 한글보다는 영어로 작성

