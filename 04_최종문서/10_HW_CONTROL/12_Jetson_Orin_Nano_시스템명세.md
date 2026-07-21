# Jetson Orin Nano 단일보드 시스템명세

## 1. 역할

Jetson은 OrinCar의 유일한 컴퓨팅 보드다. ROS 2 graph, camera 수집, 인지, drive-test command, 규칙 기반 주행, Agent 추론, 명령 선택, software Safety, I2C actuator 제어, 진단과 rosbag 기록을 실행한다.

Teacher 학습 자체는 개발 PC 또는 학습 환경에서 수행할 수 있지만, 차량 runtime artifact 검증과 Replay·Shadow는 Jetson에서 수행한다. 외부 학습 환경은 차량 actuator 권한을 갖지 않는다.

## 2. 목표 실행환경

| 항목 | 목표 | 실장 상태 |
|---|---|---|
| Board | Jetson Orin Nano Developer Kit | board revision `[실물 확인 필요]` |
| OS | Ubuntu 22.04 / Jetson Linux 계열 | exact version `[검증 필요]` |
| ROS | ROS 2 Humble | package version `[검증 필요]` |
| Container | Jetson application image 1개 | digest `[검증 필요]` |
| GPU | JetPack/L4T 호환 NVIDIA runtime | exact version `[검증 필요]` |
| Camera | USB/V4L2 또는 CSI | 모델·interface `[실물 확인 필요]` |
| I2C | Motor HAT `0x40`, steering PCA9685 `0x60` | bus·address `[실물 확인 필요]` |

목표값과 설치 상태를 분리한다. `uname`, `/etc/os-release`, `dpkg`, `ros2`, `i2cdetect`, camera device 결과를 release manifest에 기록한다.

## 3. 장치 소유권

| Resource | 유일한 소유 Node | 다른 Node의 접근 |
|---|---|---|
| Camera device | `front_camera_node` | image Topic만 구독 |
| I2C bus·`0x40`·`0x60` | `actuator_controller_node` | `VehicleCommand` 후보 발행만 허용 |
| Motor HAT B channel | `actuator_controller_node` | 직접 접근 금지 |
| Steering PCA channel | `actuator_controller_node` | 직접 접근 금지 |
| Model artifact | `agent_policy_node` | immutable hash로만 선택 |
| ROS bag output | rosbag2 recorder | actuator device 접근 금지 |

## 4. Stage profile

| Profile | 주요 실행 Node | Camera | I2C | actuator 권한 |
|---|---|---:|---:|---:|
| `ros-setup` | monitor·contract test | 없음 | 없음 | 없음 |
| `camera-bench` | camera·image_proc·perception | 필요 | 없음 | 없음 |
| `actuator-bench` | drive-test·mux·Safety·actuator | 없음 | 필요 | 바퀴를 띄운 시험 |
| `drive-test` | drive-test·mux·Safety·actuator | 선택 | 필요 | 저속 open-loop |
| `rule-replay` | image source·perception·rule | 파일 | 없음 | 없음 |
| `rule-drive` | camera·perception·rule·control chain | 필요 | 필요 | Gate 통과 후 rule만 |
| `agent-replay` | image source·Agent·평가 | 파일 | 없음 | 없음 |
| `agent-shadow` | rule-drive + Agent 기록 | 필요 | 필요 | rule만, Agent 0 |
| `agent-assist` | rule·Agent·control chain | 필요 | 필요 | 승인된 제한 범위 |
| `teacher-replay` | bag source·Teacher evaluator | 파일 | 없음 | 없음 |

profile은 같은 image의 launch와 device 권한만 바꾼다. 낮은 단계 profile이 높은 단계 권한을 우회해서 열 수 없어야 한다.

## 5. 자원 격리

- Replay·Teacher profile에는 camera와 I2C device를 전달하지 않는다.
- camera-bench에는 I2C device를 전달하지 않는다.
- actuator-bench에는 camera device를 전달하지 않는다.
- rule-drive·agent-shadow·agent-assist만 camera와 I2C가 동시에 필요하다.
- `--privileged`를 금지한다.
- container restart는 drive enable을 자동 복구하지 않는다.
- camera·Agent 부하 중 I2C loop와 Safety가 starvation되지 않는지 측정한다.

## 6. 알려진 한계

현재 HW에는 encoder·IMU·거리 sensor·독립 watchdog이 없다. 따라서 단계별 주행은 camera 기반 open-loop steering·throttle이며 실제 속도·거리·자세·장애물 정지 성능을 주장하지 않는다. 운영자 감독과 통제구역 제한은 software Safety의 대체물이 아니라 추가 운영 조건이다.
