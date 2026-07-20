# 로깅·ROS bag 요구사항

## 1. 목적

운행 재현, 고장 원인 분석, 자율주행 AI 학습·Replay와 제품 KPI 산출에 같은 episode 증거를 사용한다.

## 2. Episode 필수 metadata

| 영역 | 필드 |
|---|---|
| 식별 | episode ID, patrol ID, route ID/version, 시작·종료 시각 |
| Release | Jetson/RPi release, interface/config/model/calibration hash |
| 환경 | 장소, 날씨·조명, 코스, 운영자, 데이터 사용 동의 `[결정 필요]` |
| 결과 | 완주, 개입, 충돌, stop/fault, 회피·복귀 결과 |
| 품질 | topic rate/drop, clock offset, bag integrity, 저장 용량 |

## 3. 필수 기록 범주

- camera image·camera info 또는 승인된 압축 표현
- scan, IMU, wheel odom, filtered odom과 TF
- route·segment·점자 경로와 confidence
- manual/autonomy/proposed/target command와 actuator feedback
- Safety state, E-stop, 양 보드 heartbeat·diagnostics
- 자율주행 AI output·latency·model ID와 rule baseline
- 결함·장애물 event와 outbox 상태

정확한 topic allowlist·압축·storage plugin은 상세설계에서 확정한다.

## 4. 데이터 품질 Gate

- bag을 별도 개발 PC에서 재생할 수 있다.
- 필수 topic과 TF가 누락되지 않는다.
- timestamp가 역행하지 않고 clock offset이 허용 범위다.
- command, feedback, intervention과 영상이 같은 timeline에서 비교된다.
- 손상 bag과 정상 bag을 자동 구분할 integrity 결과가 있다.

## 5. MVP 수집 목표

`HC-M3` 전에 2~5분짜리 bag 10개 이상을 수집하고, route·날짜·조명·실패 mode의 편향을 표로 확인한다. 데이터 품질 Gate를 통과하지 못한 episode는 학습·성능 주장에 사용하지 않는다.

## 6. 보존·개인정보

원본·대표 이벤트·중간 frame의 보존 우선순위를 분리한다. 얼굴·번호판 처리 시점, 암호화, 접근권한과 삭제기간은 `[결정 필요]`이며 BE 보존 정책과 함께 확정한다.
