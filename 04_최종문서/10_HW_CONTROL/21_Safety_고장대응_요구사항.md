# Jetson 단일보드 Safety·고장대응 요구사항

## 1. 현재 Safety 능력

| 계층 | 가능한 대응 | 한계 |
|---|---|---|
| 운영자 | battery shield·Motor HAT 전원 수동 차단, deadman 해제 | 자동 차단이 아님 |
| `safety_supervisor_node` | command freshness·범위·stage·diagnostics 검증 | Jetson process에 의존 |
| `actuator_controller_node` | timeout·I2C fault 시 throttle 0 write 시도 | process와 I2C가 살아 있어야 함 |
| Motor HAT·PCA9685 | PWM 출력 | process death 시 마지막 출력 유지 가능 |

현재 HW에는 독립 비상정지 회로, 독립 watchdog, motor current sensor, encoder가 없다. 따라서 이 문서는 software risk reduction과 단계별 운영 조건을 정의하며 기능 안전·무인 운행을 승인하지 않는다.

## 2. 단계별 안전 상태

| Mode | motion_allowed | 허용 조건 |
|---|---:|---|
| `STOP` | false | 부팅·종료·무명령·fault 기본 상태 |
| `ROS_SETUP` | false | ROS contract·monitor만 실행 |
| `CAMERA_BENCH` | false | camera pipeline만 실행 |
| `ACTUATOR_BENCH` | 조건부 | 바퀴를 띄우고 `BENCH` source만 |
| `DRIVE_TEST` | 조건부 | `HC-M2` 통과, 운영자 deadman, 짧은 open-loop |
| `RULE_REPLAY` | false | file input, I2C 없음 |
| `RULE_DRIVE` | 조건부 | `HC-M3` 통과, 빈 통제코스 |
| `AGENT_REPLAY` | false | file input, I2C 없음 |
| `AGENT_SHADOW` | 조건부 | actuator에는 Rule만 적용 |
| `AGENT_ASSIST` | 제한적 | `HC-M5` 통과, threshold·fallback 강제 |
| `TEACHER_REPLAY` | false | 평가·label만 수행 |

fault latch는 원인이 사라져도 자동 해제하지 않는다. reset은 target neutral, I2C 정상, stage config 일치, 운영자 확인 뒤 software latch만 해제한다.

## 3. 명령 방어

다음 조건 중 하나라도 발생하면 target command를 neutral로 바꾸고 fault를 기록한다.

- command age 또는 `valid_for` 초과
- sequence 역행·중복
- NaN·Inf·정규화 범위 초과
- source와 stage profile 불일치
- throttle·steering absolute/slew limit 초과
- I2C address 미검출 또는 write 오류
- actuator·monitor diagnostics 오류
- Rule에서 camera/tactile path stale
- Agent artifact hash·state·confidence·latency invalid
- Shadow 단계에서 Agent source 선택 시도
- Teacher process의 command 발행 시도

## 4. Stage별 source 정책

| Profile | 허용 source | fallback |
|---|---|---|
| `actuator-bench` | `BENCH` | neutral |
| `drive-test` | `DRIVE_TEST` | neutral |
| `rule-drive` | `RULE` | neutral |
| `agent-shadow` | `RULE` | neutral |
| `agent-assist` | `AGENT`, 조건부 `RULE` | stage config의 Rule 또는 neutral |
| Replay·Teacher | 기록용 후보만 | actuator 자체가 없음 |

허용 source 표는 runtime config와 contract test의 단일 기준이다.

## 5. Fault 반응

| Fault | Software 반응 | 물리 한계 | 재개 조건 |
|---|---|---|---|
| command stale | throttle 0 write, latch | actuator process가 살아 있어야 함 | 새 sequence·neutral·reset |
| I2C error | 제한 retry 후 throttle 0 시도 | bus 단절이면 write 불가 | 전원 OFF 점검 후 재시작 |
| camera/path stale | Rule·Agent 후보 폐기 | drive-test에는 camera를 쓰지 않을 수 있음 | 입력 안정·calibration 확인 |
| Agent invalid | Agent 폐기, 정책에 따라 Rule/neutral | Rule도 stale이면 neutral | artifact·health 회복 후 새 stage 시작 |
| actuator 정상 종료 | throttle 0 write 후 종료 | write 결과 확인 필요 | 재시작·수동 reset |
| actuator `SIGKILL` | 대응 실행 불가 | 마지막 PWM 유지 가능 | 즉시 수동 전원 차단·원인 조사 |
| Jetson hang·power loss | 대응 실행 불가 | 독립 cutoff 없음 | 물리 전원 차단·점검 |
| battery voltage drop | 자동 진단 불가 | voltage sensor 없음 | multimeter 측정 |

## 6. 지면 시험 운영 조건

- `HC-M2 Actuator Bench`를 통과하기 전 바퀴를 지면에 놓지 않는다.
- 사람이 통제하는 평평한 짧은 구역만 사용한다.
- 장애물·보행자·차량이 없는 상태에서 시험한다.
- 운영자는 전원 스위치에 즉시 접근하고 deadman을 유지한다.
- 최초 throttle limit은 `0.05`이며 증거 기반으로만 올린다.
- stage별 최대 command 시간과 주행 시간을 config에 둔다.
- fault·reset·container restart 뒤 자동 출발하지 않는다.
- Rule·Agent 단계에서도 무인·원격 운행으로 전환하지 않는다.

## 7. Go/No-Go

| 항목 | 판정 |
|---|---|
| ROS·camera·I2C 확인 | 선행 Gate 통과 후 Go |
| 바퀴를 띄운 actuator 시험 | 운영자 감독 조건부 Go |
| 간단한 open-loop 주행 | `HC-M2` 통과 후 조건부 Go |
| Rule Drive | `HC-M3`·Rule Replay 통과 후 조건부 Go |
| Agent Shadow | Rule Drive baseline 통과 후 Go |
| 제한 Assist | Shadow·fault injection 통과 후 조건부 Go |
| Teacher Replay·재학습 | immutable episode·split 검증 후 Go |
| 공공 보행로·무인 운행·장애물 회피 주장 | No-Go |
| process kill·Jetson hang 자동 정지 보장 | No-Go |

단계가 뒤로 갈수록 software capability는 증가하지만 현재 HW의 물리 안전 한계는 그대로 유지된다.
