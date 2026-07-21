# Jetson 단일보드 단계별 시험·인수기준

## 1. 순차 Gate

| Gate | 목표 | 주요 시험 | 통과 전 금지 |
|---|---|---|---|
| `HC-M0 ROS Setup` | ROS 2 graph·contract 확정 | package build, Node·Topic·QoS, diagnostics | device 접근 |
| `HC-M1 I/O Bench` | camera·I2C 식별 | stream, calibration, `0x40`·`0x60` scan | actuator 출력 |
| `HC-M2 Actuator Bench` | wheels-up steering·throttle | center·방향·limit·timeout·kill | 지면 주행 |
| `HC-M3 Simple Drive` | 저속 open-loop 주행 | deadman·start/stop·직진·조향 | Rule actuator 권한 |
| `HC-M4 Rule Drive` | camera 규칙 주행 | Replay·wheels-up·통제코스·lost 정지 | Agent Assist |
| `HC-M5 Agent Shadow` | Rule 대비 Agent 평가 | Replay·resource·disagreement·fallback | Agent actuator 권한 |
| `HC-M6 Limited Assist` | 제한 Agent 적용 | confidence·limit·fault injection·Rule fallback | 범위 확대 |
| `HC-M7 Teacher Loop` | failure data 재학습 | label provenance·split·held-out·재승격 | 새 model 자동 배포 |

문서상 다음 Gate가 정의되어 있어도 이전 Gate 증거가 없으면 실행하지 않는다.

## 2. 공통 필수 시험

| ID | 시험 | 합격 기준 |
|---|---|---|
| `TC-HC-ROS` | Node·Topic·QoS contract | stage별 graph가 명세와 일치 |
| `TC-HC-INVENTORY` | 매뉴얼과 실물 대조 | 기준선 HW 식별·미확인 항목 표시 |
| `TC-HC-I2C` | 반복 I2C scan | `0x40`, `0x60` 안정 검출 |
| `TC-HC-CAMERA` | 10분 camera stream | reset 없음, rate·drop 기록 |
| `TC-HC-STEER` | center/min/max | 기구 간섭·stall 없음 |
| `TC-HC-MOTOR` | `0.05→0.10→0.15` wheels-up | 방향·발열·전압강하 기록 |
| `TC-HC-CMD` | stale·sequence·NaN·range·stage | 잘못된 command 100% 거부 |
| `TC-HC-I2C-FAULT` | bus/device 오류 | fault latch·enable false |
| `TC-HC-SHUTDOWN` | 정상 종료 | throttle 0 write·정지 관찰 |
| `TC-HC-KILL` | actuator `SIGKILL` | PWM 유지 여부·운영 제한 기록 |
| `TC-HC-AUTHORITY` | stage별 source 주입 | 미허용 source 100% 거부 |

## 3. Simple Drive 시험

| ID | 시험 | 합격 기준 |
|---|---|---|
| `TC-DRIVE-DEADMAN` | deadman 해제 | 새 command 중지·neutral 시도 |
| `TC-DRIVE-STARTSTOP` | 저출력 출발·정지 5회 | 의도하지 않은 자동 재출발 0회 |
| `TC-DRIVE-STRAIGHT` | 짧은 직진 command | 방향 일치·기구 이상 없음 |
| `TC-DRIVE-STEER` | 좌·중앙·우 저속 command | linkage 간섭 없음 |
| `TC-DRIVE-TIMEOUT` | command 중단 | timeout·fault·운영자 차단 기록 |

실제 속도·거리는 측정값으로 판정하지 않는다. 시험 시간, command 값, 외부 영상, 바닥 표식은 별도 관찰 증거로만 사용한다.

## 4. Rule 시험

| ID | 시험 | 합격 기준 |
|---|---|---|
| `TC-RULE-UNIT` | center·heading·confidence 경계값 | 예상 steering·throttle·neutral |
| `TC-RULE-REPLAY` | 동일 bag 3회 | 같은 candidate sequence·값 |
| `TC-RULE-LOST` | path ambiguous/lost/stale | throttle 0·자동 재출발 없음 |
| `TC-RULE-WHEELS-UP` | camera→rule→actuator | sign·limit·source 일치 |
| `TC-RULE-DRIVE` | 빈 통제코스 반복 | 지정 구간 추종·lost 정지 |

## 5. Agent·Teacher 시험

| ID | 시험 | 합격 기준 |
|---|---|---|
| `TC-AGENT-REPLAY` | Jetson 동일 bag 3회 | artifact·output 재현, latency 기록 |
| `TC-AGENT-SHADOW` | Rule Drive 병행 | Agent가 target에 선택된 횟수 0 |
| `TC-AGENT-INVALID` | timeout·NaN·low confidence | Rule fallback 또는 neutral |
| `TC-AGENT-ASSIST` | 승인 구간 제한 권한 | limit 위반 0, fallback 성공 |
| `TC-TEACHER-LABEL` | disagreement·failure 평가 | 원본 frame·reason·evaluator 추적 |
| `TC-TEACHER-SPLIT` | episode split | 인접 frame leakage 0 |
| `TC-TEACHER-REGRESSION` | 새 Agent held-out Replay | Rule·이전 Agent 대비 회귀 보고 |
| `TC-TEACHER-REENTRY` | 새 artifact 재승격 | Replay·Shadow 재통과 전 Assist 0 |

## 6. 기록 지표

- camera rate·drop·latency, calibration ID
- candidate/proposed/target/actuator state rate·age
- I2C latency·error count, requested/applied normalized 값과 PWM tick
- Jetson CPU/GPU/RAM/temperature
- battery shield 전압·전류·발열 관찰
- Rule/Agent disagreement, Agent confidence·latency·artifact hash
- Teacher verdict·reason·dataset lineage
- stop reason, fallback, fault latch, 운영자 개입

## 7. 최종 제한

모든 Gate를 통과해도 현재 HW만으로 실제 speed·distance·odometry·장애물 정지거리·공공 보행로 안전성을 주장하지 않는다. 단계 완료는 해당 통제시험의 통과를 뜻하며 무인 운행 승인을 뜻하지 않는다.
