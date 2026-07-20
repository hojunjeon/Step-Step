# HW/CONTROL 시험·인수 기준

## 1. 시험 순서

| Gate | 범위 | 통과 전 금지 |
|---|---|---|
| `T0 정적 검사` | 부품·배선·정격·기구·continuity | 전원 인가 |
| `T1 전원·I/O Bench` | rail, idle/peak/stall, GPIO/I²C, E-stop | 바닥 구동 |
| `T2 바퀴를 띄운 HIL` | manual, actuator feedback, watchdog, fault | 지면 주행 |
| `T3 센서·Replay` | calibration, TF, timestamp, bag 재현 | autonomous enable |
| `T4 저속 통제 코스` | 추종·갈림길·단절·장애물·복귀 | 속도 상향 |
| `T5 통합 부하` | AI·logging·DDS·outbox 동시 부하 | RC 승격 |

## 2. 필수 시험

| ID | 시험 | 합격 기준 |
|---|---|---|
| `TC-HC-POWER` | motor 부하 중 rail·reset·USB/I²C | 정격 내, reboot·disconnect 없음 |
| `TC-HC-ESTOP` | 모든 모드에서 E-stop | ROS와 무관하게 구동 차단 |
| `TC-HC-CMD` | expiry·sequence·NaN·range·slew | 잘못된 명령 100% 거부 |
| `TC-HC-LINK` | Ethernet 제거·packet loss | cutoff 내 정지·fault latch |
| `TC-HC-PROCESS` | Planner·Safety·container kill | 독립 watchdog 정지 |
| `TC-HC-TF` | TF 누락·timestamp skew | autonomous enable 금지/정지 |
| `TC-HC-ROUTE` | 지정 통제 코스 | 3회 연속, 충돌 0 |
| `TC-HC-BAG` | 10개 episode 기록·재생 | 필수 topic·metadata 품질 통과 |
| `TC-HC-SHADOW` | rule과 AI 동시 실행 | actuator 영향 0, 지표 기록 |
| `TC-HC-LOAD` | 모든 필수 process 동시 부하 | control·watchdog deadline 유지 |

## 3. 측정 지표

control/command/feedback p50·p95·p99, deadline miss, clock offset, CPU/GPU/RAM/temperature/power, topic rate/drop, 경로 오차, 개입·충돌·정지, 회피·복귀, inference latency와 outbox backlog를 기록한다.

## 4. 완료 기준

- [ ] `HC-M0~3`의 Exit Criteria가 증적으로 연결됐다.
- [ ] 필수 Safety 시험이 모두 통과했다.
- [ ] 목표·실장 버전과 HW Gate가 갱신됐다.
- [ ] Replay·Shadow가 actuator 권한 없이 재현된다.
- [ ] known issue와 시연 운용 한계가 승인됐다.
