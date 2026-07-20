# HW/CONTROL Spec 안내와 마일스톤

> 범위: `10~29` · 소유: 로봇 HW, ROS, 제어, 안전, 주행 인지와 자율주행 AI

## 1. 문서 목록

|  번호 | 문서                                                       | 상태                          |
| --: | -------------------------------------------------------- | --------------------------- |
|  11 | [HW 시스템 요구사항](./11_HW_시스템_요구사항.md)                       | 주요 원칙 `[확정]`, 실물값 `[검증 필요]` |
|  12 | [Jetson Orin Nano 시스템명세](./12_Jetson_Orin_Nano_시스템명세.md) | 목표 환경 확정, minor 버전 검증 필요    |
|  13 | [Raspberry Pi 5 시스템명세](./13_Raspberry_Pi5_시스템명세.md)      | 목표 환경 확정, 장치 mapping 검증 필요  |
|  14 | [Jetson–RPi 연동명세](./14_Jetson_RPi_연동명세.md)               | 계약 초안                       |
|  15 | [실행환경·의존성·배포명세](./15_실행환경_의존성_배포명세.md)                   | 잠금 정책 확정                    |
|  16 | [ROS 2 노드 요구사항](./16_ROS2_노드_요구사항.md)                    | 외부 책임 확정                    |
|  17 | [ROS 2 인터페이스 명세](./17_ROS2_인터페이스_명세.md)                  | 이름·주기 초기 기준                 |
|  18 | [센서·보정·동기화 요구사항](./18_센서_보정_동기화_요구사항.md)                 | 실물 검증 필요                    |
|  19 | [규칙 기반 자율주행 기능명세](./19_규칙기반_자율주행_기능명세.md)                | MVP 기능 경계 확정                |
|  20 | [자율주행 AI 단계요구사항](./20_자율주행_AI_단계요구사항.md)                 | Replay·Shadow 목표 확정         |
|  21 | [Safety·고장대응 요구사항](./21_Safety_고장대응_요구사항.md)             | 안전 원칙 확정                    |
|  22 | [로깅·ROS bag 요구사항](./22_로깅_ROSbag_요구사항.md)                | episode 계약 초안               |
|  23 | [HW/CONTROL 시험·인수기준](./23_HW_CONTROL_시험_인수기준.md)         | Gate 기준                     |

## 2. 담당하지 않는 기능

- 불량 점자블록 모델의 데이터셋·학습·정확도 개선
- BE 내부 저장소·API·대시보드 구현
- 실제 공공 보도의 무인 운행 승인

단, 결함 AI가 Jetson에서 실행될 때 필요한 자원·입출력·failure 계약은 HW/CONTROL이 함께 검증한다.

## 3. Gate 기반 마일스톤

| 단계 | 목표 | 필수 산출물 | Exit Criteria |
|---:|---|---|---|
| `HC-M0 실물 확정` | 부품·정격·기구값 확인 | BOM, 배선, pin·전원·치수 증적 | 미확정 HW Gate 모두 해소 |
| `HC-M1 기본 구동` | 수동 저속 제어와 독립 정지 | actuator feedback, E-stop·watchdog 로그 | 명령 이상·통신 단절 시 정지 |
| `HC-M2 규칙 기반` | 센서·TF·경로·제어 통합 | 반복 주행 baseline | 지정 코스 3회 연속, 충돌 0 |
| `HC-M3 데이터 기반` | 동기화된 episode 확보 | 재생 가능한 bag 10개 이상 | timestamp·TF·metadata Gate 통과 |
| `HC-M4 AI Replay` | 공개·자체 모델 offline 비교 | 지연·메모리·예측 비교표 | 같은 입력에서 재현 가능 |
| `HC-M5 AI Shadow` | 실차에서 예측만 기록 | 규칙 Planner와 disagreement·개입 지표 | actuator 권한 0, timeout 안전 |
| `HC-M6 Assist` | Safety 뒤 제한 후보 반영 | 폐루프 Go/No-Go 증거 | `[추후 추가 예정]` 실패 데이터 Gate 필요 |
| `HC-M7 최적화` | 검증 모델의 Jetson 실시간화 | FP16 기준선과 동시 부하 benchmark | `[추후 추가 예정]` 정확도·지연 동시 통과 |

평가용 MVP의 필수 완료선은 `HC-M3`, 목표 완료선은 `HC-M5`다. `HC-M6~7`은 앞 단계의 증거 없이 일정만으로 진입하지 않는다.

## 4. 인계물

| 받는 파트 | HW/CONTROL 인계물 |
|---|---|
| AI | timestamp가 있는 원본 이미지, camera calibration, 촬영 조건, Jetson 자원 예산 |
| BE | event schema, 파일 hash, 장치·release ID, outbox·재시도 조건 |
| 통합 | rosbag, fault 로그, release manifest, 시험 결과 |

## 5. 현재 차단 항목

- `[검증 필요]` 보드별 실제 OS·kernel·ROS·Python·Docker 버전
- `[검증 필요]` motor driver IC·논리·정격·stall current
- `[검증 필요]` wheelbase·wheel radius·조향 pulse·최대각
- `[검증 필요]` RPi gpiochip·line·I²C 주소와 장치 권한
- `[결정 필요]` JetPack 6 계열 exact minor와 release 동결일
