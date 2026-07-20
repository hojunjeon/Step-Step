# Raspberry Pi 5 시스템명세

## 1. 역할

Raspberry Pi 5는 encoder·IMU 수집, 속도 PID, steering PWM, actuator feedback, E-stop 보고와 Jetson-independent watchdog을 실행한다. 경로·AI 판단은 수행하지 않는다.

## 2. 목표 환경과 실장 확인

| 항목 | 설계 기준 | 실장 확인 | 상태 |
|---|---|---|---|
| 보드 | Raspberry Pi 5, RAM·revision 기록 | 미확인 | `[검증 필요]` |
| Host OS | Ubuntu 24.04 64-bit | 미확인 | Desktop/Server 기록 |
| Kernel | 설치 OS 지원 kernel | 미확인 | gpiochip 변화 검증 필요 |
| Docker Engine | 공식 Ubuntu arm64 package, exact version 고정 | 미확인 | `[검증 필요]` |
| ROS runtime | Jammy ARM64 기반 ROS 2 Humble container | 미확인 | image digest 필요 |
| Python | container ROS binary와 호환되는 exact version | 미확인 | `[검증 필요]` |
| RMW | Jetson과 동일한 `rmw_fastrtps_cpp` 초기 기준 | 미확인 | `[검증 필요]` |
| Network | host network | 미확인 | DDS 검증 필요 |

ROS 공식 문서는 64-bit Raspberry Pi OS에서 Docker로 ROS 2를 실행하는 경로와 `ros:humble-ros-core` 이미지를 안내한다. Docker 공식 문서는 Ubuntu 22.04·24.04 및 arm64를 지원한다. 실제 운영 이미지는 tag가 아니라 digest로 고정한다.

## 3. Container 요구사항

- control과 watchdog은 장애 격리를 위해 별도 실행 단위로 둔다.
- `--privileged`를 금지하고 검증된 `/dev/i2c-*`, `/dev/gpiochip*`, serial과 group만 전달한다.
- 같은 GPIO line을 둘 이상의 process가 소유하지 못해야 한다.
- source bind mount와 개발 도구를 운영 image에 포함하지 않는다.
- container restart는 drive enable을 자동 복구하지 않는다.
- config는 read-only로 배포하고 release manifest에 hash를 남긴다.

## 4. GPIO·I/O 요구사항

- GPIO는 3.3V logic이며 motor를 직접 연결하지 않는다.
- Raspberry Pi 5의 실제 gpiochip 번호와 line offset을 `gpioinfo`로 확인한다.
- I²C address, bus, pull-up, logic level과 장치 충돌을 기록한다.
- actuator GPIO와 watchdog EN GPIO의 소유권을 분리한다.
- boot·shutdown·container stop 때 EN은 물리적으로 OFF여야 한다.

## 5. 필수 성능·상태

| 항목 | 요구 |
|---|---|
| control loop | 목표 50~100Hz, 실제 p50/p95/p99 기록 |
| watchdog | monotonic clock 기반, Jetson heartbeat·command expiry 감시 |
| feedback | applied sequence, speed, steering, enable, fault, loop age |
| health | CPU·temperature·I²C/GPIO error·loop overrun 보고 |

## 6. 소유 ROS 계약 요약

| 범주 | 소유 노드 | 대표 외부 계약 |
|---|---|---|
| Sensor | `wheel_encoder_node`, `imu_node`, `estop_node` | `/wheel/odom`, `/imu/data`, `/safety/estop` |
| Actuation | `actuator_controller_node` | `/vehicle/target_cmd`, `/vehicle/feedback` |
| Safety | `safety_watchdog_node` | Jetson heartbeat, RPi safety state, EN |
| Operation | `rpi_monitor_node` | RPi heartbeat·diagnostics |

노드 내부 구현은 후속 상세설계에서 작성한다. 정확한 외부 계약은 [16_ROS2_노드_요구사항.md](./16_ROS2_노드_요구사항.md)와 [17_ROS2_인터페이스_명세.md](./17_ROS2_인터페이스_명세.md)를 따른다.

## 7. 외부 근거

- [ROS 2 Humble on Raspberry Pi](https://docs.ros.org/en/humble/How-To-Guides/Installing-on-Raspberry-Pi.html)
- [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Raspberry Pi hardware documentation](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html)
- [Raspberry Pi GPIO best practices](https://pip-assets.raspberrypi.com/categories/685-whitepapers-app-notes-compliance-guides/documents/RP-006553-WP/A-history-of-GPIO-usage-on-Raspberry-Pi-devices-and-current-best-practices)
