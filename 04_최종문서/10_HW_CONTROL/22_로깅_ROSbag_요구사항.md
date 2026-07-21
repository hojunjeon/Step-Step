# 단계별 로깅·ROS bag 요구사항

## 1. Episode metadata

| 범주 | 필수 값 |
|---|---|
| 식별 | episode ID, stage ID, profile, 시작·종료 시각 |
| Release | Jetson image digest, Git SHA, interface/config/model hash |
| HW | camera ID, I2C bus, `0x40`·`0x60`, motor·servo ID |
| Calibration | camera ID, steering center/min/max, motor polarity·limit |
| Environment | 조명, 바닥, 바퀴 접촉, 통제구역, 운영자 |
| Authority | allowed source, selected source, fallback policy |
| Outcome | stop reason, fault, 운영자 개입, known limitation |
| Learning | dataset split, Agent artifact, Teacher evaluator ID `[해당 시]` |

## 2. 단계별 Topic

| 단계 | 반드시 기록할 Topic |
|---|---|
| ROS Setup | `/diagnostics`, `/safety/state` |
| Camera Bench | image raw/rect, camera info, diagnostics |
| Actuator Bench | bench/proposed/target command, actuator state, safety |
| Simple Drive | drive-test/proposed/target command, actuator state, safety |
| Rule Replay·Drive | image, tactile path, rule/proposed/target command, actuator state `[Drive만]` |
| Agent Replay·Shadow·Assist | image, tactile path, rule command, agent command/state, selected target |
| Teacher Replay | rule·Agent 후보, Agent state, Teacher evaluation, episode metadata |

현재 존재하지 않는 scan, IMU, wheel odom, filtered odom Topic을 필수 목록에 넣지 않는다.

## 3. 추적성

- image와 tactile path는 같은 frame stamp와 calibration ID로 연결한다.
- 후보 command는 source·frame·sequence를 기록한다.
- mux sequence는 proposed→target→actuator state로 추적되어야 한다.
- Shadow에서는 Rule과 Agent 후보를 같은 frame에서 비교할 수 있어야 한다.
- Teacher evaluation은 episode ID와 frame sequence로 원본을 가리켜야 한다.
- model·Teacher·dataset artifact는 immutable hash로 연결한다.

## 4. 품질 Gate

- 필수 Topic의 start/end timestamp와 drop count가 있다.
- Replay profile에는 I2C device와 enabled actuator state가 없다.
- command authority 변경 시점과 이유가 기록된다.
- Agent timeout·fallback·Teacher disagreement를 재현할 수 있다.
- 원본 bag, 파생 clip, label, dataset, model prediction을 분리한다.

## 5. 데이터 사용 제한

encoder가 없으므로 실제 speed·distance·odometry를 bag에서 만들어내지 않는다. 외부 영상이나 수동 표식으로 평가한 값은 ROS 측정값과 구분해 annotation provenance를 남긴다.

Teacher가 선택한 failure frame도 자동 정답이 아니다. 운영자 검토 상태와 evaluator identity를 함께 저장한다.
