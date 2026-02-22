## Stream 출력으로 단계별 결과 확인
- AgentExecutor의 `stream()` 메소드를 사용하여 에이전트의 중간 단계를 스트리밍할 것입니다.
- `stream()`의 출력은 (Action, Observation) 쌍 사이에서 번갈아 나타나며, 최종적으로 에이전트가 목표를 달성했다면 답변으로 마무리됩니다.

  다음과 같은 형태로 보일 것입니다.
	1. Action 출력
	2. Observation 출력
	3. Action 출력
	4. Observation 출력
	... (목표 달성까지 계속) ...
	그 다음, 최종 목표가 달성되면 에이전트는 최종 답변을 출력할 것입니다.
	이러한 출력의 내용은 다음과 같이 요약됩니다.

| 출력           | 내용                                                                                         |
| ------------ | ------------------------------------------------------------------------------------------ |
| Action       | `actions`: AgentAction 또는 그 하위 클래스<br>`messages`: 액션 호출에 해당하는 채팅 메시지                       |
| messages     | 액션 호출에 해당하는 채팅 메시지                                                                         |
| Observation  | `steps`: 현재 액션과 그 관찰을 포함한 에이전트가 지금까지 수행한 작업의 기록<br>`messages`: 함수 호출 결과(즉, 관찰)를 포함한 채팅 메시지 |
| Final Answer | `output`: AgentFinish<br>`messages`: 최종 출력을 포함한 채팅 메시지                                     |
### 중간 단계 출력을 사용자 정의 함수로 출력
다음의 3개 함수를 정의하고 이를 통해 중간 단계 출력을 사용자 정의합니다.
- `tool_callback`: 도구 호출 출력을 처리하는 함수
- `observation_callback`: 관찰(Observation) 출력을 처리하는 함수
- `result_callback`: 최종 답변 출력을 처리하는 함수