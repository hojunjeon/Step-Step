# Jetson–Raspberry Pi 실행환경·컨테이너 설계 조사

> 조사일: 2026-07-21  
> 대상: Jetson Orin Nano + Raspberry Pi 5, ROS 2 Humble  
> 문서 상태: 설계 권고. 실제 보드의 버전·장치·지연 시험을 통과해야 확정한다.

## 1. 결론

중간 migration 비용을 피하기 위해 현재 프로젝트는 **처음부터 양쪽 ROS application을 보드별 container로 운영**한다. 단, host kernel·driver와 Raspberry Pi의 최종 EN 차단은 container 밖에 둔다.

| 보드 | Host | ROS/application | 안전 경계 |
|---|---|---|---|
| Jetson Orin Nano | JetPack 6.2.x의 Ubuntu 22.04 기반 Jetson Linux | NVIDIA L4T/TensorRT 기반 Jetson 전용 ROS 2 Humble image | NVIDIA driver는 host, 상위 Safety Supervisor는 container process |
| Raspberry Pi 5 | Ubuntu Server 24.04 arm64 | Jammy 기반 `ros:humble-ros-base-jammy` 파생 image | 최종 EN 차단 watchdog은 container 밖의 native `systemd` service와 물리 E-stop/외부 watchdog으로 유지 |

두 보드 모두 Docker를 사용하지만 최종 image는 다르다. 두 보드가 같은 ARM64 CPU를 사용해도 **보드가 소유한 kernel·driver·가속기·안전 책임이 다르기 때문**이다.

- JetPack 6.2.2는 Jetson Linux 36.5, kernel 5.15, Ubuntu 22.04 기반 rootfs와 CUDA 12.6·TensorRT 10.3을 한 묶음으로 제공한다. Jetson의 GPU·카메라·TensorRT는 이 host BSP와 맞춰야 한다.
- Ubuntu 공식 지원표에서 Raspberry Pi 5의 첫 지원 LTS는 Ubuntu 24.04다. Pi 5에 Ubuntu 22.04를 native로 맞추는 안은 공식 지원 조합이 아니다.
- ROS 2 Humble의 Tier 1 Linux 기준은 Ubuntu 22.04 Jammy arm64다. 따라서 Pi 5는 지원되는 24.04 host 위에 Jammy/Humble container를 올리는 것이 두 제약을 동시에 만족한다.
- ROS 공식 Pi 문서도 64-bit host에서 `ros:humble-ros-core` Docker image를 사용하는 경로를 안내하고, 공식 ROS image에는 arm64용 `humble-ros-base-jammy`가 제공된다.

즉, **동일하게 관리해야 할 것은 OS image 자체가 아니라 source SHA, ROS interface, RMW, 설정 schema, release manifest와 시험 절차**다.

> 수명 주의: ROS 2 Humble의 지원 종료 예정은 2027년 5월이다. 현재 마일스톤에서는 양쪽 Humble을 유지하되, 그 이후 운영을 계획한다면 별도의 양 보드 동시 migration gate를 잡는다. 한쪽만 먼저 다른 ROS distro로 바꾸지 않는다.

## 2. 대안 비교

점수는 이 프로젝트의 GPU 사용, Pi GPIO/I²C, 200 ms watchdog, 짧은 통합 일정을 기준으로 한 상대 평가다.

| 안 | 재현성 | HW 접근 | 안전 격리 | 운영 난이도 | 판단 |
|---|---:|---:|---:|---:|---|
| Jetson native + Pi container | 높음 | 높음 | 높음 | 중간 | 문제 발생 시 fallback |
| **양쪽 서로 다른 container image** | **매우 높음** | 중간 | 높음 | 초기 중간 | **최종 권고** |
| 양쪽 동일 image | 겉보기만 높음 | 낮음 | 낮음 | 중간 | 비권고 |
| 양쪽 native | 낮음 | 높음 | 중간 | 높음 | 비권고 |
| Jetson container + Pi native | 중간 | 중간 | 중간 | 매우 높음 | 비권고 |

### 2.1 Jetson native + Pi container — fallback

장점:

- Jetson은 NVIDIA가 검증한 JetPack·CUDA·TensorRT 조합과 ROS deb package를 직접 사용한다.
- Pi 5 host는 공식 지원되는 Ubuntu 24.04를 사용하면서 ROS user space만 Jammy/Humble로 고정한다.
- Pi application의 rollback은 image digest 교체로 단순해진다.
- Jetson의 GPU/V4L2와 Pi의 GPIO/I²C를 모두 container 안으로 옮길 때 생기는 장치 mapping 복잡성을 절반으로 줄인다.

최종안으로 고정하지 않는 이유:

- Docker daemon 또는 control container가 죽어도 정지해야 하므로 watchdog과 EN 차단을 container에만 두면 안 된다.
- native와 container의 배포·rollback 방식이 달라 장기적으로 구성 drift가 생긴다.
- 이후 Jetson을 container로 옮기면 GPU·camera·launch·권한 시험을 다시 해야 한다.

### 2.2 양쪽 서로 다른 container image — 권고

배포·rollback을 image digest로 통일할 수 있지만 두 image는 달라야 한다.

- Jetson image: 설치된 Jetson Linux/L4T와 일치하는 NVIDIA L4T/JetPack 계열 base + ROS 2 Humble
- Pi image: 공식 `ros:humble-ros-base-jammy` arm64 파생 image

NVIDIA Container Toolkit은 GPU 장치와 필요한 host 자원을 container에 주입한다. 이 때문에 Jetson image는 단순한 Ubuntu/ROS image가 아니며 host JetPack/L4T 호환성 시험이 필요하다. 처음부터 두 image를 같은 release로 만들되, 각 장치를 `sensor → GPU → ROS 통신 → 전체 부하` 순서로 container 안에서 단계 검증한다.

### 2.3 양쪽 동일 image — 비권고

공식 ROS image는 arm64이므로 CPU-only ROS node는 양쪽에서 실행할 수 있다. 그러나 하나의 운영 image로 통일하면 다음 차이가 사라지지 않는다.

- Jetson: L4T, CUDA, TensorRT, NVIDIA Container Runtime, V4L2/GPU device
- Pi: gpiochip, I²C, PWM, HAT library, watchdog EN line

결과적으로 공통 image가 모든 dependency와 권한을 포함하는 큰 image가 되거나, 실행 시 board 분기가 늘어난다. 이는 공격면과 회귀 범위를 넓힌다. **공통 base Dockerfile과 공통 package lock은 사용하되 최종 image는 `jetson-runtime`과 `rpi-control`로 분리**한다.

### 2.4 양쪽 native — 비권고

Jetson native는 적합하지만 Pi 5 native Humble이 문제다.

- Pi 5의 공식 Ubuntu LTS host는 24.04이고 Humble binary의 기준은 22.04다.
- Pi에서 Humble을 source build하거나 비공식 22.04 boot 조합을 유지해야 하므로 설치 시간, 보안 update, 재현성과 장애 분석 비용이 커진다.

향후 양쪽 ROS 배포판을 함께 바꿀 때 다시 평가할 수 있지만, 지금 Pi만 Jazzy로 올려 혼용하지 않는다.

### 2.5 Jetson container + Pi native — 비권고

가장 까다로운 Jetson GPU·camera 쪽을 먼저 container화하면서 Pi에는 공식 native Humble 조합이 없다는 문제가 남는다. 현재 목표에는 이점이 없다.

## 3. 권고 image 설계

### 3.1 Raspberry Pi 운영 image

초기 base는 다음으로 한다.

```dockerfile
FROM ros:humble-ros-base-jammy
```

선택 이유:

- OSRF가 유지하는 Docker Official Image다.
- Ubuntu Jammy와 ROS 2 Humble binary를 포함한다.
- `linux/arm64/v8` manifest가 제공된다.
- `ros-core`보다 현장 진단 도구가 충분하고 `perception`/`desktop`보다 작다.

운영 규칙:

1. 개발 중에는 tag를 사용해 build하되 RC부터는 `image@sha256:<manifest-digest>`로 고정한다.
2. multi-stage build로 compile 도구를 runtime layer에서 제거한다.
3. `step_step_interfaces`와 application source SHA를 label/release manifest에 기록한다.
4. source bind mount를 운영에서 금지하고 config만 read-only mount한다.
5. root가 아닌 고정 UID/GID로 실행하고, 필요한 device group만 부여한다.
6. `privileged: true`를 쓰지 않고 `/dev/gpiochip*`, `/dev/i2c-*`, 필요한 serial device만 명시한다.
7. image 재시작 후 drive는 항상 disabled이며 물리/운영자 reset 뒤에만 enable한다.

Docker Compose의 image 문법은 digest 참조를 지원한다. 실제 digest는 CI에서 build/push한 뒤 release manifest에 기록한다.

```yaml
services:
  rpi_ros:
    image: ghcr.io/<owner>/step-step-rpi-control@sha256:<release-digest>
    network_mode: host
    read_only: true
    restart: unless-stopped
    cap_drop: [ALL]
    environment:
      ROS_DOMAIN_ID: "42"
      ROS_LOCALHOST_ONLY: "0"
      RMW_IMPLEMENTATION: rmw_fastrtps_cpp
      FASTRTPS_DEFAULT_PROFILES_FILE: /etc/step-step/fastdds.xml
    devices:
      - /dev/gpiochip0:/dev/gpiochip0
      - /dev/i2c-1:/dev/i2c-1
    volumes:
      - ./config:/etc/step-step:ro
      - /var/log/step-step/ros:/var/log/step-step/ros
```

장치명은 예시다. Pi 5 실기에서 `gpioinfo`, `i2cdetect -l`, `udevadm info`로 확인한 stable mapping으로 바꾼다.

### 3.2 Raspberry Pi native safety service

다음 기능은 Docker Compose와 별도의 작은 `systemd` unit으로 둔다. 이 daemon만 EN GPIO line을 boot부터 shutdown까지 독점한다.

- E-stop input 읽기
- ROS control container가 보내는 짧은 **enable lease**의 monotonic timeout 감시
- 200 ms 초기 기준 초과 시 motor driver EN을 물리적으로 LOW/OFF
- boot, shutdown, daemon crash 시 fail-safe OFF
- fault latch와 수동 reset

Container는 Jetson heartbeat와 command freshness, E-stop, local control health가 모두 정상일 때만 Unix domain socket으로 짧은 lease를 갱신한다. Native daemon은 ROS message를 해석하지 않고 lease age와 E-stop만 확인한다. 따라서 ROS/DDS, container 또는 Docker daemon이 죽으면 lease가 만료되어 EN이 내려간다. Native daemon package와 unit도 release manifest에서 hash/version을 고정한다.

이 lease는 물리 safety를 대체하지 않는다. 가능하면 외부 monostable 또는 safety MCU가 EN을 최종 차단하게 하여 host kernel 정지까지 덮는다. Container healthcheck는 진단 수단이지 safety watchdog이 아니다.

### 3.3 Jetson 운영 image

Jetson도 첫 통합부터 container를 사용한다. 다만 설치된 JetPack minor를 추정하지 않고 다음 절차로 base를 고정한다.

1. `cat /etc/nv_tegra_release`, JetPack/L4T/CUDA/TensorRT version을 기록한다.
2. 그 release와 호환된다고 NVIDIA가 명시한 NGC L4T/JetPack base tag를 고른다.
3. NVIDIA Container Toolkit runtime으로 GPU를 전달한다.
4. TensorRT engine은 target Jetson에서 생성하고 engine·model·preprocessing hash를 기록한다.
5. host 진단 실행 결과와 container의 정확도, p50/p95/p99, RAM, thermal, camera reconnect를 비교한다.

Pi용 `ros:humble-ros-base-jammy` image를 그대로 Jetson GPU image로 사용하지 않는다. 공통 CPU-only ROS layer가 필요하면 두 최종 image가 같은 source/lock file을 소비하도록 하고, NVIDIA runtime layer는 Jetson 쪽에만 둔다.

## 4. 보드 간 통신 설계

### 4.1 물리망과 주소

기존 연동명세를 유지한다.

| 항목 | Jetson | Raspberry Pi |
|---|---|---|
| 전용 Ethernet | `192.168.50.1/24` | `192.168.50.2/24` |
| ROS distro | Humble | Humble in Jammy container |
| RMW | `rmw_fastrtps_cpp` | `rmw_fastrtps_cpp` |
| Domain | `ROS_DOMAIN_ID=42` | `ROS_DOMAIN_ID=42` |
| localhost 제한 | `ROS_LOCALHOST_ONLY=0` | `ROS_LOCALHOST_ONLY=0` |

Pi container는 Linux의 `network_mode: host`를 사용한다. Docker 공식 문서상 host mode는 container가 host network namespace를 공유하고 별도 container IP/NAT를 만들지 않는다. DDS multicast와 동적 UDP port를 bridge/NAT 뒤에서 별도 조정하는 것보다 단순하다. 대신 network namespace 격리가 사라지므로 DDS를 전용 Ethernet NIC로 제한해야 한다.

### 4.2 DDS를 전용 NIC에 고정

Fast DDS의 UDP transport `interfaceWhiteList`에는 각 보드의 전용 Ethernet IP 또는 interface 이름만 넣는다. Fast DDS 공식 문서는 whitelist가 participant가 사용할 interface를 제한하며, 일치하는 interface가 없으면 통신이 성립하지 않는다고 명시한다.

설정 원칙:

- Jetson profile: `192.168.50.1`만 허용
- Pi profile: `192.168.50.2`만 허용
- Wi-Fi, server uplink, Docker bridge는 DDS에서 제외
- 양쪽에서 같은 Fast DDS profile version/hash 사용
- whitelist 적용 뒤 `ros2 multicast`와 실제 topic으로 양방향 검증

보드별 IP 값만 다르므로 하나의 template에서 release 시 두 XML을 생성한다. XML을 손으로 따로 유지하지 않는다.

### 4.3 보드 경계를 넘기는 데이터

기존 계약대로 카메라 image와 raw LiDAR는 Jetson 내부에 둔다. Ethernet 경계는 아래의 작은 제어·상태 topic 중심으로 유지한다.

- Jetson → Pi: `/vehicle/target_cmd`, `/system/heartbeat/jetson`
- Pi → Jetson: `/vehicle/feedback`, `/wheel/odom`, `/imu/data`, `/safety/estop`, `/system/heartbeat/rpi`

양쪽은 같은 `step_step_interfaces` commit/type hash를 사용한다. 다른 ROS distro 또는 다른 RMW를 한쪽만 적용하지 않는다. ROS 공식 문서도 분산 시스템에서 같은 ROS version과 같은 RMW implementation을 쓰도록 권고한다.

### 4.4 한 보드 안의 container 수

초기 Pi ROS stack은 한 container로 묶어 DDS와 device ownership 변수를 줄인다. watchdog만 native service로 분리한다. 여러 ROS container로 나누는 것은 다음이 모두 통과한 뒤에 한다.

- GPIO/I²C line 단일 소유권
- container kill 때 native watchdog 정지
- host-network 환경의 participant discovery
- 동시 부하 p99 control deadline
- log와 config volume 권한

## 5. Release 단위

“같은 image” 대신 아래 하나의 release manifest로 두 보드를 묶는다.

```text
release_id
git_sha
step_step_interfaces_sha + type_hash
Jetson: board + JetPack + L4T + kernel + ROS/RMW + CUDA/TensorRT
RPi: board_revision + host_OS/kernel + Docker + image_digest + ROS/RMW
config/QoS/FastDDS/calibration hash
model + preprocessing hash
wiring_revision
```

배포 순서는 `drive disabled → artifact 배포 → 양쪽 self-test → 통신 contract test → 운영자 reset/enable`이다. 한쪽만 새 release로 올라간 상태에서는 drive enable을 거부한다.

## 6. 검증 Gate

### G0 — 실장 확인

- [ ] Jetson의 exact JetPack/L4T/CUDA/TensorRT를 기록했다.
- [ ] Pi 5의 board revision, Ubuntu 24.04, kernel과 Docker arm64를 기록했다.
- [ ] Pi container가 실제 arm64 manifest digest로 실행된다.
- [ ] GPIO/I²C/serial stable mapping을 기록했다.

### G1 — 통신

- [ ] 고정 IP와 전용 Ethernet link가 재부팅 후 유지된다.
- [ ] 양쪽의 ROS distro, `RMW_IMPLEMENTATION`, `ROS_DOMAIN_ID`, interface hash가 같다.
- [ ] Fast DDS가 전용 NIC만 사용한다.
- [ ] `ros2 multicast`, demo talker/listener, 실제 7개 경계 topic이 양방향 통과한다.

### G2 — 안전

- [ ] Pi ROS container kill 후 200 ms 초기 기준 안에 native watchdog이 EN을 차단한다.
- [ ] Docker daemon kill, Jetson process kill, cable 제거에서도 정지한다.
- [ ] container restart만으로 drive가 다시 enable되지 않는다.
- [ ] E-stop은 두 OS와 Docker 상태에 무관하게 출력 전원을 차단한다.

### G3 — 부하와 rollback

- [ ] Jetson AI·camera·LiDAR·logging과 Pi 50~100 Hz control 동시 부하를 시험했다.
- [ ] command/heartbeat age와 control loop p50/p95/p99를 기록했다.
- [ ] 이전 Pi image digest와 Jetson package manifest로 rollback했다.
- [ ] rollback 뒤에도 수동 reset 전에는 drive disabled다.

## 7. 채택안과 보류안

### 지금 채택

- Jetson: JetPack 6.2.x host + NVIDIA runtime + Jetson 전용 ROS 2 Humble container
- Pi 5: Ubuntu Server 24.04 arm64 host + Docker Engine + `ros:humble-ros-base-jammy` 파생 image
- Pi watchdog/EN: native `systemd` + 물리 E-stop/외부 watchdog
- 통신: host network + 전용 Ethernet + Fast DDS NIC whitelist
- 배포 동일성: 동일 source/interface/config release, 보드별 container image 두 개

### 실기 시험 뒤 확정

- JetPack/L4T exact minor와 NVIDIA container base tag
- Pi image digest와 필요한 device 목록
- 200 ms cutoff, topic QoS, socket buffer, CPU affinity
- Fast DDS multicast 유지 또는 정적 discovery 도입 여부

## 8. 공개 image 후보 평가

> 확인일: 2026-07-21. 크기와 최근 push 시점은 registry 표시값이며 tag는 바뀔 수 있다. 실제 release에서는 현재 digest를 CI가 다시 조회해 고정한다.

### 8.1 점수 기준

10점 만점의 Step-Step 운영 적합도다.

| 기준 | 배점 | 확인 내용 |
|---|---:|---|
| 보드·가속기 호환 | 30% | JetPack/L4T·TensorRT 또는 Pi arm64 적합성 |
| 공급자 신뢰성 | 25% | NVIDIA·OSRF 공식 여부, signed image |
| Humble 준비도 | 20% | ROS 2 Humble 포함 여부와 추가 작업량 |
| 크기·운영성 | 15% | runtime 크기, 불필요한 desktop/devel 구성 |
| 유지보수성 | 10% | 최근 갱신, 명확한 tag·source·digest |

점수는 사전 선별용이다. Jetson의 exact L4T, USB/CSI camera, TensorRT, Pi GPIO/I²C와 200 ms cutoff 실기 시험을 통과해야 최종 채택한다.

### 8.2 Jetson Orin Nano 후보

| 순위 | image | 공급자·크기 | 점수 | 판단 |
|---:|---|---|---:|---|
| 1 | `nvcr.io/nvidia/l4t-tensorrt:r10.3.0-runtime` | NVIDIA signed, arm64, 1.52 GB | **9.2** | **운영 base 권고.** JetPack 6.2.x의 TensorRT 10.3과 맞고 CUDA/TensorRT runtime을 포함한다. ROS Humble·V4L2/GStreamer package는 직접 추가한다. |
| 2 | `nvcr.io/nvidia/l4t-jetpack:r36.4.0` | NVIDIA signed, arm64, 5.22 GB | **8.5** | **통합 검증용 권고.** CUDA·cuDNN·TensorRT·VPI·Jetson Multimedia가 모두 있어 시작은 쉽지만 크다. 설치 host가 r36.4.3·r36.4.4·r36.5이면 patch 호환 시험이 필요하다. |
| 3 | `nvcr.io/nvidia/isaac/ros:<release-arm64-jetpack-tag>` | NVIDIA signed, current release-generated tag | **7.9** | Humble·Nav2·CUDA·TensorRT가 준비된 개발환경. Isaac ROS/NITROS를 실제 채택할 때만 유리하며 일반 Step-Step runtime에는 과하다. hash tag를 임의로 고르지 않고 Isaac ROS release script가 선택한 값을 고정한다. |
| 4 | `nvcr.io/nvidia/l4t-cuda:12.6.11-runtime` | NVIDIA signed, arm64, 1.21 GB | **7.7** | 작고 공식적이지만 TensorRT·VPI·Multimedia·ROS를 추가해야 한다. TensorRT를 쓰는 현재 프로젝트에서는 1위 image보다 이점이 작다. |
| 5 | `dustynv/ros:humble-desktop-l4t-r36.4.0` | community, arm64, 6.39 GB, 약 2년 전 push | **5.2** | Humble+Jetson을 바로 시험하기에는 편하지만 desktop이 크고 오래됐다. 참고·smoke test용으로만 사용한다. |
| 6 | `dustynv/ros:humble-ros-base-l4t-r36.3.0` | community, arm64, 6.46 GB, 1년 이상 전 push | **4.8** | ROS base지만 L4T가 더 오래됐고 매우 크다. JetPack 6.2 운영 image로 채택하지 않는다. |
| 7 | `ros:humble-ros-base-jammy` | OSRF official, arm64, 244.51 MB | **4.5** | CPU-only ROS에는 좋지만 CUDA·TensorRT·Jetson Multimedia가 없다. Jetson의 주 image로 쓰면 NVIDIA 계층을 다시 조립해야 한다. |

#### Jetson 최종 선택

Step-Step은 USB camera·LiDAR와 TensorRT 추론이 중심이므로 다음 multi-stage 구성을 우선한다.

```dockerfile
# build/debug stage
FROM nvcr.io/nvidia/l4t-tensorrt:r10.3.0-devel AS builder

# production stage
FROM nvcr.io/nvidia/l4t-tensorrt:r10.3.0-runtime AS runtime
```

여기에 공식 Jammy/Humble deb와 필요한 ROS package만 설치한다. CSI·VPI·Jetson Multimedia 의존성이 실제로 필요하고 위 base에서 해결되지 않을 때만 `l4t-jetpack:r36.4.0`으로 올린다.

중요한 제약:

- JetPack 6.2는 L4T 36.4.3, 6.2.1은 36.4.4, 6.2.2는 36.5일 수 있으므로 `JetPack 6.2.x`라는 이름만으로 base를 고르지 않는다.
- `cat /etc/nv_tegra_release`, `dpkg-query -W nvidia-jetpack`, TensorRT/CUDA version을 먼저 기록한다.
- NGC의 전체 `l4t-jetpack` 최신 공개 tag는 조사 시점에 `r36.4.0`이므로 newer host에서 camera·VPI·GStreamer까지 contract test한다.
- TensorRT engine은 target Jetson에서 생성하고 model·engine hash를 release에 기록한다.

### 8.3 Raspberry Pi 5 후보

| 순위 | image | 공급자·크기 | 점수 | 판단 |
|---:|---|---|---:|---|
| 1 | `ros:humble-ros-base-jammy` | Docker Official/OSRF, arm64 244.51 MB, 최근 갱신 | **9.7** | **개발·초기 운영 base 권고.** colcon·rosdep·vcstool 등 개발 도구가 있어 하드웨어 package 통합이 쉽다. |
| 2 | `ros:humble-ros-core-jammy` | Docker Official/OSRF, arm64 131.09 MB, 최근 갱신 | **9.1** | **최종 경량 runtime 권고.** build 도구가 없으므로 multi-stage builder의 install 결과만 복사한다. |
| 3 | `ros:humble-perception-jammy` | Docker Official/OSRF, arm64 874.06 MB | **7.2** | 공식이지만 perception은 Jetson 책임이므로 Pi에는 불필요한 package와 공격면이 늘어난다. |
| 4 | `rostooling/setup-ros-docker:ubuntu-jammy-ros-humble-ros-base-latest` | community, arm64 379.4 MB, 활발한 갱신 | **6.2** | CI 확인용으로는 유용하지만 mutable `latest`와 추가 추상화가 있다. 공식 ROS image보다 채택 이유가 약하다. |
| 5 | `polymathrobotics/ros:humble-ros-base-jammy` | community mirror, arm64 245.12 MB | **5.8** | 공식 image와 기능·크기가 거의 같지만 공급망 단계만 하나 늘어난다. |
| 6 | `kahless/rpi5-ros2` | community, 9개월 전 update | **1.0** | 게시자가 `Please dont pull! Not ready yet!`라고 명시했다. 사용 금지. |
| 7 | `ahmedsoliman9/ubuntu-22.04-ros2-humble:latest` | community, amd64 only, 1.44 GB, 1년 이상 전 push | **0.5** | Pi/Jetson arm64에서 실행할 수 없고 mutable `latest`뿐이다. 사용 금지. |

`arm64v8/ros:humble-ros-base-jammy`는 별도 제품이 아니라 공식 `ros` image의 architecture별 저장소다. Compose와 Dockerfile에서는 multi-platform tag인 `ros:humble-ros-base-jammy`를 사용한다.

#### Raspberry Pi 최종 선택

```dockerfile
# build/debug stage
FROM ros:humble-ros-base-jammy AS builder

# production stage
FROM ros:humble-ros-core-jammy AS runtime
```

초기에는 `ros-base` 단일 stage로 센서와 제어를 검증하고, release 최적화 때 같은 Dockerfile의 `ros-core` runtime stage로 전환한다. 이는 보드 운영 방식을 바꾸는 migration이 아니라 동일 container build의 경량화다.

### 8.4 채택 조합

| 역할 | 채택 base | 대체 base |
|---|---|---|
| Jetson builder | `nvcr.io/nvidia/l4t-tensorrt:r10.3.0-devel` | `nvcr.io/nvidia/l4t-jetpack:r36.4.0` |
| Jetson runtime | `nvcr.io/nvidia/l4t-tensorrt:r10.3.0-runtime` | 검증된 `l4t-jetpack:r36.4.0` 파생 image |
| Pi builder | `ros:humble-ros-base-jammy` | 없음 |
| Pi runtime | `ros:humble-ros-core-jammy` | 초기에는 `ros-base` 유지 |

결론적으로 **바로 실행 가능한 community 완제품을 가져오는 것보다 NVIDIA·OSRF 공식 base에서 `step-step-jetson`과 `step-step-rpi`를 직접 만드는 것이 가장 높은 점수**다.

## 9. 공식 근거

- [NVIDIA JetPack 6.2.2](https://developer.nvidia.com/embedded/jetpack-sdk-622)
- [NVIDIA L4T TensorRT image](https://catalog.ngc.nvidia.com/orgs/nvidia/-/containers/l4t-tensorrt/-/tags)
- [NVIDIA L4T JetPack image](https://catalog.ngc.nvidia.com/orgs/nvidia/-/containers/l4t-jetpack/-/tags)
- [NVIDIA L4T CUDA image](https://catalog.ngc.nvidia.com/orgs/nvidia/-/containers/l4t-cuda/-/tags)
- [NVIDIA Isaac ROS Dev Base](https://catalog.ngc.nvidia.com/orgs/nvidia/isaac/containers/ros/-/)
- [NVIDIA Container Toolkit 개요](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/)
- [Ubuntu의 Raspberry Pi 지원표](https://ubuntu.com/hardware/docs/boards/how-to/ubuntu_supported/raspberry-pi/)
- [Ubuntu 24.04 release notes — Pi 5 첫 LTS 지원](https://documentation.ubuntu.com/release-notes/24.04/)
- [ROS 2 Humble on Raspberry Pi](https://docs.ros.org/en/humble/How-To-Guides/Installing-on-Raspberry-Pi.html)
- [ROS 2 Humble release/platform support](https://docs.ros.org/en/humble/Releases/Release-Humble-Hawksbill.html)
- [ROS 2 distribution별 지원 기간](https://docs.ros.org/en/humble/Releases.html)
- [ROS Docker Official Image](https://hub.docker.com/_/ros/)
- [dustynv/ros community image](https://hub.docker.com/r/dustynv/ros/tags)
- [Raspberry Pi community image의 미완성 경고](https://hub.docker.com/r/kahless/rpi5-ros2)
- [Docker Engine on Ubuntu — 24.04와 arm64 지원](https://docs.docker.com/engine/install/ubuntu/)
- [Docker host network driver](https://docs.docker.com/engine/network/drivers/host/)
- [Docker Compose image digest 문법](https://docs.docker.com/reference/compose-file/services/)
- [ROS 2 RMW 선택 방법](https://docs.ros.org/en/humble/How-To-Guides/Working-with-multiple-RMW-implementations.html)
- [Fast DDS interface whitelist](https://fast-dds.docs.eprosima.com/en/2.x/fastdds/transport/whitelist.html)
