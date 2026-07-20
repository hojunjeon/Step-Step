# Jetson Orin Nano 시스템명세

## 1. 역할

Jetson은 센서 수집, 위치추정, 규칙 Planner, 주행 인지·자율주행 AI, 상위 Safety, 결함 AI 추론 연동과 서버 gateway를 실행한다. Motor Driver GPIO와 PCA9685를 직접 소유하지 않는다.

## 2. 목표 환경과 실장 확인

| 항목 | 설계 기준 | 실장 확인 | 상태 |
|---|---|---|---|
| 보드 | Jetson Orin Nano, 메모리 용량 기록 | 미확인 | `[검증 필요]` |
| JetPack | Ubuntu 22.04 기반 JetPack 6 계열 | 미확인 | exact minor `[결정 필요]` |
| Jetson Linux/L4T | 선택 JetPack과 일치 | 미확인 | `[검증 필요]` |
| ROS | ROS 2 Humble native | 미확인 | `[검증 필요]` |
| Python | 선택 OS·ROS binary 기본과 호환되는 버전 | 미확인 | exact version `[검증 필요]` |
| CUDA·TensorRT·cuDNN | 선택 JetPack bundle과 일치 | 미확인 | `[검증 필요]` |
| RMW | `rmw_fastrtps_cpp` 초기 기준 | 미확인 | 양 보드 동일 필요 |
| Container | 기본 runtime native | 미확인 | inference container는 후속 평가 |

NVIDIA 공식 자료상 JetPack 6.0은 Ubuntu 22.04 기반이며 Orin 모듈을 지원한다. 그러나 JetPack archive에는 여러 6.x release가 있으므로 “최신”을 자동 사용하지 않고 검증한 exact version을 release manifest에 고정한다.

## 3. 필수 확인 증적

- board·memory model, JetPack, L4T, kernel, Ubuntu release
- ROS distro, RMW, Python, CUDA, TensorRT, cuDNN version
- camera·LiDAR·GNSS의 stable device path와 권한
- power mode, clock mode, storage 여유, thermal·throttling
- package lock, interface SHA, config hash와 model artifact hash

확인 명령과 원본 출력은 `HC-M0` 운영 점검표에 보존한다.

## 4. 자원 요구사항

| 자원 | 요구 |
|---|---|
| GPU | 주행 인지와 결함 AI 동시 부하에서 측정 |
| RAM | OOM·swap thrashing 없이 필수 process 유지 |
| CPU | Safety·DDS·logging이 AI 부하에 굶지 않음 |
| Storage | bag·outbox 임계치와 보존 정책 설정 |
| Thermal | 평가 코스 연속 운행 중 throttle·shutdown 없음 |

정확한 FPS·RAM·전력 예산은 실제 model과 동시 부하 benchmark 후 `[검증 필요]`에서 `[확정]`으로 전환한다.

## 5. 소유 ROS 계약 요약

| 범주 | 소유 노드 | 대표 외부 계약 |
|---|---|---|
| Sensor | `front_camera_node`, `ydlidar_node` | image/camera info, `/scan` |
| State | `ekf_filter_node`, `route_manager_node` | filtered odom·TF, route·segment |
| Autonomy | `tactile_path_node`, `rule_planner_node`, `autonomy_ai_node` | tactile path, autonomy/AI 후보 |
| Control | `manual_control_node`, `command_mux_node`, `safety_supervisor_node` | proposed command, `/vehicle/target_cmd` |
| Operation | `system_monitor_node`, `server_gateway_node` | heartbeat·diagnostics, event/outbox |

정확한 책임·주기는 [16_ROS2_노드_요구사항.md](./16_ROS2_노드_요구사항.md), 타입·QoS·fresh 기준은 [17_ROS2_인터페이스_명세.md](./17_ROS2_인터페이스_명세.md)를 단일 원본으로 사용한다.

## 6. 외부 근거

- [NVIDIA JetPack Archive](https://developer.nvidia.com/embedded/jetpack-archive)
- [NVIDIA JetPack 6.0](https://developer.nvidia.com/embedded/jetpack-sdk-60)
- [NVIDIA Jetson Linux Archive](https://developer.nvidia.com/embedded/jetson-linux-archive)
