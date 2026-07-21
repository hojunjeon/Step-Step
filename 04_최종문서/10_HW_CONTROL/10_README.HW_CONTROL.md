# HW/CONTROL Specification Guide

> 기준선: Jetson Orin Nano 단일보드 + OrinCar 매뉴얼의 camera·Motor HAT·PCA9685·GA25-370·MG996R·battery shield

## 1. 문서 목록

| 번호 | 문서 | 역할 |
|---:|---|---|
| 11 | [OrinCar 보유 HW 시스템 요구사항](./11_HW_시스템_요구사항.md) | 실제 장치·전원·현재 한계 |
| 12 | [Jetson Orin Nano 단일보드 시스템명세](./12_Jetson_Orin_Nano_시스템명세.md) | 단계별 runtime과 device 소유권 |
| 13 | [OrinCar 구동부 시스템명세](./13_OrinCar_구동부_시스템명세.md) | Motor HAT·PCA·motor·servo |
| 14 | [Jetson-OrinCar I2C 연동명세](./14_Jetson_OrinCar_I2C_연동명세.md) | ROS→I2C seam과 출력 규칙 |
| 15 | [실행환경·의존성·배포명세](./15_실행환경_의존성_배포명세.md) | 단일 Jetson image·stage profile |
| 16 | [ROS 2 Node 요구사항](./16_ROS2_노드_요구사항.md) | ROS 설정부터 Teacher까지 Node graph |
| 17 | [ROS 2 Topic·Message Interface](./17_ROS2_인터페이스_명세.md) | Topic, QoS, `.msg`·`.srv` 계약 |
| 18 | [센서·actuator 보정·동기화](./18_센서_보정_동기화_요구사항.md) | camera·steering·throttle 보정 |
| 19 | [규칙 기반 기능명세](./19_규칙기반_자율주행_기능명세.md) | camera 기반 규칙 주행 |
| 20 | [Agent·Teacher 단계요구사항](./20_자율주행_AI_단계요구사항.md) | Agent 승격과 Teacher 학습 loop |
| 21 | [Safety·고장대응](./21_Safety_고장대응_요구사항.md) | 단계별 권한·운영 한계 |
| 22 | [로깅·ROS bag](./22_로깅_ROSbag_요구사항.md) | episode·후보·Teacher 증거 |
| 23 | [시험·인수기준](./23_HW_CONTROL_시험_인수기준.md) | `HC-M0~M7` 순차 Gate |

## 2. 단계별 개발 흐름

```text
ROS 2 설정
  → camera/I2C·actuator bench
  → 간단한 open-loop 주행시험
  → camera 규칙 기반 주행
  → Agent Replay·Shadow·제한 Assist
  → Teacher 평가·failure data·재학습
```

| 단계 | 결과 | 다음 단계로 넘어가는 조건 |
|---|---|---|
| `HC-M0 ROS Setup` | ROS 2·package·launch·진단 기준 확정 | Node·Topic·QoS contract test 통과 |
| `HC-M1 I/O Bench` | camera와 I2C `0x40`·`0x60` 확인 | camera stream·address scan 안정 |
| `HC-M2 Actuator Bench` | 바퀴를 띄운 steering·throttle | invalid·timeout·shutdown 시험 통과 |
| `HC-M3 Simple Drive` | 빈 통제구역의 저속 open-loop 주행 | 운영자 감독·수동 전원 차단·정지 반복 통과 |
| `HC-M4 Rule Drive` | camera tactile path 기반 규칙 주행 | 지정 코스 반복·lost path 정지 통과 |
| `HC-M5 Agent Shadow` | Agent 후보를 rule 결과와 비교 | 지연·불일치·실패 episode 기준 통과 |
| `HC-M6 Limited Assist` | 승인된 Agent 후보의 제한적 actuator 권한 | rule baseline 대비 회귀 없음·즉시 fallback |
| `HC-M7 Teacher Loop` | 실패 구간 평가·재학습·새 artifact 승격 | held-out Replay와 Shadow 재검증 |

모든 단계는 앞 단계를 통과해야 열리는 순차 Gate다. 문서에 단계가 존재한다는 사실은 현재 통과를 의미하지 않는다.

## 3. 공통 명령 경로

```text
drive-test / rule / agent candidate
  → command_mux_node
  → safety_supervisor_node
  → /vehicle/target_cmd
  → actuator_controller_node
  → I2C Motor HAT·PCA9685
```

- `actuator_controller_node`만 `/dev/i2c-*`, `0x40`, `0x60`을 소유한다.
- Rule·Agent·Teacher Module은 I2C와 `/vehicle/target_cmd`를 직접 소유하지 않는다.
- Teacher는 episode를 평가하고 학습용 결과를 만들 뿐 주행 명령을 발행하지 않는다.
- encoder가 없으므로 속도·거리·odometry 폐루프는 성립하지 않는다.
- LiDAR·거리 sensor가 없으므로 장애물 회피·정지거리 성능은 단계 목표에 포함하지 않는다.

## 4. 단계 승격의 공통 제한

- camera model·device path·calibration이 확인되어야 한다.
- Jetson 40-pin header와 실제 I2C bus를 실물 검증해야 한다.
- GA25-370 전압·전류와 battery shield 출력 호환성을 확인해야 한다.
- process kill·Jetson hang에서 자동 정지가 보장되지 않는 한 모든 지면 시험은 운영자 감독의 통제구역으로 제한한다.
- drive enable은 부팅·재시작·fault reset 뒤 자동 복구하지 않는다.
- 없는 sensor의 값을 추정해 측정값으로 기록하지 않는다.
