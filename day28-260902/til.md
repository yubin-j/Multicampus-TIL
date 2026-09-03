# Day 28 (0902)

## 아웃풋 파서

PromptTemplate.from_template 사용 시 프롬프트에 partial을 통해 파서의 인스트럭트 구문을 집어넣고

마지막에 파싱 러너블 연결하여 파싱 수행

 - StrOutputParser
 - CommaSeparatedListOutputParser
 - JsonOutputParser
 - PydanticOutputParser
 - llm Structured Output -> llm이 지원하면 이걸 우선으로 하는 방향
   - -> llm의 자체 기능을 이용 llm.with_structured_output(Pydantic class) 를 호출해서 처리
   - 포맷 인스트럭터를 프롬프트에 넣어서 보낼 필요가 없음

## 예외 처리

llm.with_structured_output가 실패하는 경우에 대한 대책을 해야 한다.

### FakeListChatModel

매번 진짜 llm을 호출해서 오류 데이터 출력을 만들기 힘들기에 이 모델을 이용

mock 데이터를 구성해서 테스트를 수행

이 mock데이터를 통해서 테스트를 수행하면 자연스러운 예외상황을 테스트 할 수 있다.

### fallback chain

그리고 하나의 chain이 예외상황을 발생하게 되면 그대로 서비스를 종료시킬 수 없으니 대비용으로 다른 chain을 마련해 방어할 수도 있다.

chain 인스턴스의 with_fallbacks에 대비용 chain을 넣는 식으로 작성한다.

예로 llm자체에서 제공하는 with_structured_output이 실패하는 경우 PydanticOutputParser를 이용해 자체적으로 프롬프트에 출력 형식을 지정하여 요청하는 chain을 미리 작성해두고 대비하는 방식으로 이용하는 패턴이다.

이 때 입력이 완전 동일하게 대비용 chain으로 들어가니 입력 형식을 동일하게 맞춰서 작성해둬야 함. 배열로 chain을 여러개 지정도 가능

exceptions_to_handle를 이용해 특정 예외만 캐치해서 수행할 수도 있고 이를 이용하는게 중요.

## Tool 데코레이터

python 함수를 langchain tool로 사용하는 방식

docstring으로 함수의 기능과 입력, 출력에 대한 상세한 내용을 작성을 해야 llm이 판단해서 사용을 할 수 있다.

### Tool 설계 원칙

| 원칙 | 설명 |
| --- | --- |
| 명확한 이름 | `search_weather` > `func1` |
| 구체적인 설명 | "주어진 도시의 현재 날씨 정보를 검색한다" > "데이터를 가져온다" |
| 타입 힌트 필수 | `city: str` — LLM이 어떤 값을 넣어야 하는지 알 수 있다 |
| 예시 포함 | docstring에 입력 예시를 넣으면 정확도가 높아진다 |
| 에러 메시지 | Tool 실행 실패 시 LLM이 이해할 수 있는 메시지를 반환한다 |

### 예시코드

```python
@tool(parse_docstring=True)
def search_weather(city: str) -> str:
    """주어진 도시의 현재 날씨를 검색한다.

    Args:
        city: 날씨를 검색할 도시 이름. 예: '서울', '부산'
    """
    weather_data = {
        "서울": "맑음, 22도, 습도 45%",
        "부산": "흐림, 19도, 습도 72%",
        "제주": "비, 17도, 습도 88%",
    }
    return weather_data.get(city, f"{city}의 날씨 정보를 찾을 수 없습니다.")

# Tool 정보 확인
print(f"이름: {search_weather.name}")
print(f"설명: {search_weather.description}")
print(f"스키마: {search_weather.args_schema.model_json_schema()}")

# Tool 직접 호출
print(f"\n서울 날씨: {search_weather.invoke({'city': '서울'})}")
```

저번에 했던 수동으로 tool에 대한 스키마를 수동으로 특정형식으로 작성하는 것을 할 필요 없이 docstring을 작성하는 것이다.

tool데코레이터를 붙였기에 함수를 invoke로 호출할 수 있는 것이다.

그런데 결국 tool call 요청이 오면 수행하는 코드를 작성해놔야 한다..

그래도 수동으로 하던 것에 비해 langchain이 많은 맵핑을 해놨기에 아주 간단하게 코드를 작성할 수 있다.

관련 내용 실습 진행 - 인메모리와 tool 데코레이터를 활용한 기능 구현

## RAG

문서 -> 로드 -> 청킹(분할) -> 임베딩 -> 벡터 DB

사용자 질문 -> 임베딩 -> 벡터 DB 검색( 코사인 유사도과 같은 알고리즘) -> 관련 문서 추출 -> LLM에 같이 줌

### 임베딩

텍스트를 숫자로 변환

의미가 비슷한 텍스트는 비슷한 벡터로 변환됨.

벡터간의 거리를 측정하여 텍스트의 의미적 유사도를 계산

임베딩 알고리즘에 따라 차원과 숫자값들은 다르다

즉 데이터 저장 시 사용한 임베딩과 질문에 사용하는 임베딩은 동일해야 한다.
