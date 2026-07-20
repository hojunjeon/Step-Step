# Safety·고장대응 요구사항

## 1. 방어 계층

| 계층 | 책임 | 다른 계층으로 대체 가능 여부 |
|---|---|---|
| 물리 E-stop·전원 | motor·servo energy 차단 | 불가 |
| RPi watchdog | command·Jetson heartbeat·E-stop 감시, EN 차단 | 불가 |
| Jetson Safety | 센서·TF·경로·명령 상위 검증 | RPi watchdog 대체 불가 |
| 운영자 | 감독·수동 정지·fault reset | 자동 안전장치 대체 불가 |

## 2. 안전 상태 요구

- 부팅·오류·재시작·무명령의 기본값은 drive disabled다.
- fault latch는 원인이 사라졌다는 이유만으로 자동 해제하지 않는다.
- reset은 정지 상태, E-stop 해제, health 회복, 운영자 승인 후에만 가능하다.
- stop reason과 최초 발생 시각, stale input, release ID를 로그로 남긴다.

## 3. Fault 반응

| Fault | Jetson 기대 | RPi 기대 | 재개 조건 |
|---|---|---|---|
| command 만료 | target 0·fault | EN 차단 | 새 sequence+reset |
| Jetson heartbeat timeout | 해당 process 진단 | EN 차단·latch | heartbeat 안정+reset |
| E-stop | 즉시 STOP/FAULT | 물리 차단·상태 보고 | 물리 해제+점검+reset |
| scan/odom/TF stale | 감속 또는 정지 | heartbeat 단절 시 차단 | 입력 안정·사전 점검 |
| NaN·범위·slew 위반 | 승인 거부 | 적용 거부 | 원인 제거+정상 명령 |
| actuator feedback 불일치 | 정지·fault | 출력 차단 | bench 점검+reset |
| AI timeout·invalid | AI 후보 폐기, 규칙 baseline 유지 | 영향 없음 | AI health 회복 |
| server/network 장애 | outbox 저장 | 주행 영향 없음 | 비실시간 재전송 |

## 4. 수치 초기값

command·Jetson heartbeat cutoff 200ms, IMU·odom 100ms, scan 200ms를 초기 시험값으로 사용한다. 이 수치는 실차 동시 부하 p99와 정지거리 시험 후 확정하며 안전 여유 없이 완화하지 않는다.

## 5. Safety 상세설계 산출물

Hazard 분석, 상태 전이, stop reason enum, reset 권한, 동적 정지거리, 외부 watchdog 회로와 fault injection 절차는 `HC-M1~2`에서 작성한다.
