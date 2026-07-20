# Jetson–Raspberry Pi 연동명세

## 1. 물리·네트워크 계약

| 항목 | 기준 | 상태 |
|---|---|---|
| 물리망 | 전용 Gigabit Ethernet 직결 또는 전용 switch | `[확정]` |
| subnet | `192.168.50.0/24` | `[확정]` |
| Jetson | `192.168.50.1/24` 초기값 | `[검증 필요]` |
| RPi | `192.168.50.2/24` 초기값 | `[검증 필요]` |
| Wi-Fi | 원격 관리 보조망, safety command 금지 | `[확정]` |
| ROS distro | 양쪽 ROS 2 Humble | `[확정]` |
| RMW | 양쪽 동일, Fast DDS 초기 기준 | `[검증 필요]` |
| Domain ID | `42` 초기 기준 | 충돌 조사 후 `[확정]` |
| 시간 | Jetson 기준 로컬 동기화, RPi client | 방식 검증 필요 |

## 2. 호환성 계약

- 양 보드는 동일한 `step_step_interfaces` commit과 ROS type hash를 사용한다.
- RPi만 Jazzy로 올리거나 서로 다른 RMW를 혼용하지 않는다.
- DDS는 전용 Ethernet NIC로 한정하고 원격 인터넷에 직접 노출하지 않는다.
- image는 기본적으로 Jetson 내부에 유지하고 보드 사이는 command·feedback·IMU·odom·상태 중심으로 전송한다.

## 3. Safety command 계약

`/vehicle/target_cmd`만 actuator가 수락하는 최종 명령이다. 명령은 sequence, 생성 시각, 유효기간, 속도, 조향각, enable, mode를 포함해야 한다.

RPi는 다음 명령을 거부한다.

- 만료되거나 sequence가 역행·중복된 명령
- NaN·Inf 또는 설정 범위를 벗어난 값
- 허용 slew rate를 넘는 값
- E-stop·fault latch·heartbeat timeout 상태의 enable 요청

### 보드 경계 Topic

| 방향 | Topic | 목적 |
|---|---|---|
| Jetson → RPi | `/vehicle/target_cmd` | 승인된 유일한 actuator 명령 |
| Jetson → RPi | `/system/heartbeat/jetson` | 상위 제어 생존·release 상태 |
| RPi → Jetson | `/vehicle/feedback` | 적용 명령·속도·조향·fault |
| RPi → Jetson | `/wheel/odom` | wheel 기반 위치·속도 입력 |
| RPi → Jetson | `/imu/data` | 자세·각속도·가속도 입력 |
| RPi → Jetson | `/safety/estop` | 물리 E-stop 상태 |
| RPi → Jetson | `/system/heartbeat/rpi` | 하위 제어 생존·health |

타입·QoS·fresh 기준의 단일 원본은 [17_ROS2_인터페이스_명세.md](./17_ROS2_인터페이스_명세.md)다.

## 4. Heartbeat와 시간

- Jetson·RPi는 서로 독립 heartbeat를 발행한다.
- watchdog은 wall clock 보정과 무관한 monotonic elapsed time으로 정지한다.
- message timestamp는 센서 융합·진단용이며 clock offset·drift를 별도 기록한다.
- 초기 fresh 기준은 command·Jetson heartbeat 200ms, IMU·odom 100ms, scan 200ms다. 실제 부하 시험 후 확정한다.

## 5. 연결 검증 Gate

1. link·고정 IP·MTU 확인
2. 양방향 ping과 시간 동기화 상태 확인
3. `ros2 multicast` 양방향 확인
4. 같은 RMW·Domain·interface로 demo 통신
5. 실제 topic의 type·QoS·rate 확인
6. image·logging·AI 동시 부하에서 command·heartbeat 지연 측정
7. cable 제거·packet loss·RPi/Jetson restart 고장 주입

내부 DDS XML과 chrony 설정은 `[상세설계 단계에서 작성]`한다.
