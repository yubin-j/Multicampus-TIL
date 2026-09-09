# Day 33 (0909)

전일에 풀었던 주제 관련 설명하는 랭그래프 코드를 클래스화 하여 llm을 실행환경에 맞춰서 변경하여 실행할 수 있도록 수정

## ReAct agent

Reasoning + Acting의 줄임말

llm이 응답으로 텍스트 대신 tool calls를 반환하면 시스템이 이에 해당하는 도구(함수)를 실행하고 다시 llm에게 전달.

ReAct는 이 function calling을 루프 안에서 반복하는 패턴이다.

이전에 직접 llm응답 중 tool call에 대한 내용이 왔을 때 직접 실행하던 것을 langgraph에서는 자동으로 수행해준다.

함수 작성은 동일하다. docstring을 자세하게 써주는 것이 중요!

구성 순서
 - 툴 함수를 정의하고
 - llm 에 해당 툴들을 연결한 뒤
 - langgraph.prebuilt의 ToolNode를 임포트 하여 사용
 - 해당 ToolNode에 llm에 바인딩한 툴들을 인자로 넣어 인스턴스 생성하여 노드로 추가하고
 - 엣지에 연결하는 것은 tools_condition라는 langgraph.prebuilt에서 임포트한 것으로 연결한다.

```python
def chatbot(state: MessagesState):
    """LLM을 호출하는 노드. Tool 호출이 필요하면 tool_calls가 포함된 메시지를 반환한다."""
    return {"messages": [llm_with_tools.invoke(state["messages"])]}


# 그래프 구성
graph_builder = StateGraph(MessagesState)

# 노드 등록
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", ToolNode(tools))

# 엣지 연결
graph_builder.add_edge(START, "chatbot")
graph_builder.add_conditional_edges("chatbot", tools_condition)
graph_builder.add_edge("tools", "chatbot")

graph = graph_builder.compile()

print("그래프 컴파일 완료")
```

그런데 langchain의 create_agent를 이용하면 이 과정도 한번에 다 구성해준다..

```python
agent = create_agent(model=llm, tools=tools)
```

## 메모리 - checkpointer

메모리를 넣어야 한다. 현재까지 langgraph의 메모리는 인스턴스 생성 시의 스테이트 안에 한정되어 있다.

이를 이제 외부에서도 기록을 하기 위한 작업을 할 것이다.

```python
from langgraph.checkpoint.memory import MemorySaver
```

위 memory saver를 이용하면 되고 이 저장소는 여러가지 중 하나

여기서 메모리는 thread_id로 구분..

그래프를 빌드 시 checkpointer에다가 MemorySaver의 인스턴스를 주입한다.

checkpointer는 그래프의 스테이트를 저장하고 복원하는 기능. 같은 thread_id로 호출 시 기존 내용을 가지고 진행

저장 위치에 따라 메모리, SQLite, PostgreSQL 등의 Saver를 선택할 수 있으며, 어떤 Saver를 사용하더라도 thread_id로 상태 이력을 구분한다.

이 saver를 저장해두는 그래프의 변수가 checkpointer이다.

그래프 인스턴스의
 - .get_state 함수를 통해 현재 저장된 내역을 확인할 수 있고
 - .get_state_history 를 통해 기록 단위별로 내역을 확인할 수 있고 최신순으로 정렬된채로 나온다.

## 장기 기억 - store

store 개념 이용

checkpointer는 thread_id단위로 기억을 하지만 store는 이 상위에서 사용자의 정보를 저장하는 longterm memory이다.

checkpointer의 thread_id가 store에서는 namespace, key이다.
 - namespace는 사용자별 데이터를 묶는 저장 경로
 - key는 namespace에서 프로필을 식별하는 값

저장소는 inMemoryStore와 같은 것을 쓰면 된다.

store의 메서드
 - put - 기존 값 덮어쓰기. 그래서 추가하는 경우에는 값을 조회해서 다 같이 넣어줘야 한다
 - get - 조회
 - search - namespace에 저장된 목록 조회. 그런데 prefix 형태 검색이어서 ("users",) 로 검색시 하위 네임스페이스 모두 검색됨
 - delete, list_namespaces도 있음

checkpoint는 랭그래프가 알아서 저장 불러오기 관리를 한다. 그러나 store는 개발자가 직접 구성을 해야 한다.

그렇게 만들어진 그래프 인스턴스 호출 시 config값을 꼭 설정해서 호출해야 한다.

store의 경우 내부에서 개발자가 정한 키값에 값을 주입

## 대화 요약

이제 이러한 저장 기능을 이용해 state가 임계값을 넘어가면 요약을 하고 다시 정리하는 기능을 만들어 볼 것이다.

토큰의 수를 받아와서 넘어가면 요약하는 노드로 분기처리

요약을 언제 어떻게 할지가 관건이다..

예시에서는 토큰 중 최신 토큰은 어느정도 남겨두고 이전 것을 요약 처리하였음.

요약을 실행하는 토큰 기준과 대화내용을 어떻게 요약 할지 를 도메인에 따라서 다르게 적용해야 할듯 하다.

토큰의 수를 세는 것도 llm을 통해서 한다. 이거 아마 질문에 쓰는 llm과 동일한 걸로 해야 한다.

이렇게 요약을 진행을 했으면 요약 대상이었던 메시지들은 제거를 해줘야 한다. MessagesState를 이용한 환경에서는 RemoveMessage를 해야 한다.

그리고 시스템 메시지도 압축을 하면 바로 유저 메시지가 되어 버린다.. 시스템 메시지를 다시 작성을 해야 한다!..

그런데 원본 대화를 요약하더라도 원본 메시지들은 남겨놔야 한다. 왜냐하면 대화내역을 불러올때 필요하기 때문에.. 이러한 데이터는 저장소에 저장을 해둬야 할 것이다.

## 보충

ReAct의 원래 정의 - 원 논문은 Thought/Action/Observation을 텍스트로 생성하게 하는 프롬프팅 기법이었고, 지금은 tool calling으로 구현하는 게 표준이 됐다. "ReAct = function calling 루프"는 결과적으로 같아진 것.

create_agent import 경로 - LangChain 1.0의 langchain.agents.create_agent. 이전에는 langgraph.prebuilt.create_react_agent였고 지금도 남아 있음. create_agent에는 bind_tools() 안 한 원본 모델을 넘긴다.

InjectedStore - 실습에서 PydanticInvalidForJsonSchema 에러로 직접 겪은 부분. 도구에서 store를 받으려면 Annotated[BaseStore, InjectedStore()]로 표시해야 스키마에서 제외된다. 반면 config: RunnableConfig는 타입 힌트만으로 자동 제외된다. 이 비대칭이 헷갈리기 쉬움.

tools -> chatbot 루프 - add_edge("tools", END)로 하면 LLM이 도구 결과를 보고 답할 기회가 없어진다. 실습에서 걸렸던 부분.

요약 시 tool 메시지 쌍 - AIMessage의 tool_calls와 대응하는 ToolMessage는 쌍으로 남거나 쌍으로 지워져야 하고, 하나만 남으면 API가 거부한다. 고정 개수로 자르면 이 쌍이 깨질 수 있어서 trim_messages(start_on="human") 같은 처리가 필요.

원본 메시지 보관 - RemoveMessage로 지우는 건 State에서만 지우는 것이고 체크포인트 이력(get_state_history)에는 이전 스냅샷이 남아 있다. "LLM에 보내는 컨텍스트"와 "사용자에게 보여줄 전체 이력"을 분리하는 것이 일반적인 패턴.
 - LLM 컨텍스트용: State (요약 + 최근 메시지)
 - 표시용 전체 이력: 별도 store나 DB에 append

체크포인트 이력에 의존하는 건 가능하지만 조회가 불편해서 실무에서는 잘 안 쓴다.
