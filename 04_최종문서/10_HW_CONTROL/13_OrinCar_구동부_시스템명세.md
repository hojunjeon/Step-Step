# OrinCar 구동부 시스템명세

## 1. 물리 구성

| 기능 | HW 경로 | 초기값 |
|---|---|---|
| Throttle | Jetson I2C → Motor HAT PCA9685 `0x40` → TB6612FNG B channel → GA25-370 | polarity·channel `[실물 확인 필요]` |
| Steering | Jetson I2C → external PCA9685 `0x60` → MG996R | assembly center 100도, 실제 center `[보정 필요]` |
| Power | battery shield 5V 표기 출력 → Motor HAT·PCA9685 | 부하 전압·전류 `[검증 필요]` |

## 2. 제어 Interface

상위 Module은 throttle과 steering을 각각 `[-1.0, 1.0]` 정규화 값으로 요청한다. `actuator_controller_node` Module이 다음 구현을 숨긴다.

- Motor HAT channel·방향·PWM tick 변환
- PCA9685 servo pulse·center·min·max 변환
- I2C write 순서와 재시도
- command expiry와 neutral 적용
- 마지막 적용값과 I2C 결과 보고

I2C address, PWM tick, servo pulse는 `VehicleCommand` Interface에 노출하지 않는다.

## 3. Parameter 계약

| Parameter | 초기값 | 상태 |
|---|---:|---|
| `i2c_bus` | `[실물 확인 필요]` | `i2cdetect`로 확정 |
| `motor_hat_address` | `0x40` | 매뉴얼 기준, scan 필요 |
| `steering_pca_address` | `0x60` | 매뉴얼 기준, scan 필요 |
| `motor_channel` | `B` | 매뉴얼 기준, 방향 확인 필요 |
| `steering_channel` | `0` | `[실물 확인 필요]` |
| `steering_center_deg` | `100` | 조립 초기값, 기구 보정 필요 |
| `steering_min/max` | `[보정 필요]` | 기구 간섭 전 여유 포함 |
| `bench_throttle_limit` | `0.15` | 바퀴를 띄운 초기 상한 |
| `command_timeout_ms` | `200` | 실시간 측정 후 강화 가능 |

## 4. 출력 순서

1. 시작 시 drive enable은 `false`다.
2. I2C address 두 개가 모두 확인되지 않으면 활성화를 거부한다.
3. 새 command의 sequence·timestamp·범위·만료를 검증한다.
4. steering 제한값을 계산한 뒤 throttle 제한값을 계산한다.
5. 정상 command만 I2C에 기록한다.
6. command 만료·I2C 오류·정상 종료 시 throttle 0을 먼저 기록한다.
7. 마지막 요청값, 계산값, raw PWM, I2C 성공 여부를 `ActuatorState`로 발행한다.

## 5. 알려진 제한

- encoder가 없어 `ActuatorState`는 실제 속도 feedback이 아니다.
- steering angle sensor가 없어 보고 각도는 command에서 계산한 값이다.
- Motor HAT/PCA9685는 process가 죽으면 마지막 PWM을 유지할 수 있으므로 kill 안전성이 보장되지 않는다.
- 위 제한은 단계가 올라가도 유지된다. 지면 사용은 `HC-M3` 이후 운영자 감독·통제구역 Gate에서만 조건부로 승인한다.
