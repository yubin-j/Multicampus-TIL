# Day 23 (0826)

어제 진행한 제미나이 api 및 라이브러리 복습

## structured 출력

## stream 출력

model의 stream출력에 대한 설명

전체 내용을 한번에 응답할 때까지 기다리는 것이 아닌 청크 단위로 출력하여 답변이 진행되는 것을 확인할 수 있는 방식

sse(server-sent events)
 - http 연결을 유지하면서 서버에서 클라이언트로 데이터 전송하는 방식
 - websocket과는 다름
 - stream출력은 sse를 이용

## gemini function calling

호출할 수 있는 함수들의 리스트를 미리 알려주고

필요한 경우 이를 호출하도록 응답하게 한 후

호출 결과를 다시 입력으로 알려주고

이 정보를 활용해 응답을 하게 하는 것

mcp 이전 개념? 정도

제미나이 자체 빌트인 기능도 있음 - code 호출, search 호출 등등
