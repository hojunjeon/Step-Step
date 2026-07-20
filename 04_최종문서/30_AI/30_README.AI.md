# AI Spec 안내와 마일스톤

> 범위: `30~49` · 소유: 불량 점자블록 탐지·분류·분석 AI
>
> 자율주행 AI는 [HW/CONTROL](../10_HW_CONTROL/10_README.HW_CONTROL.md)이 소유한다.

## 1. 문서 목록

| 번호 | 문서 | 상태 |
|---:|---|---|
| 31 | [AI 기능·시스템 요구사항](./31_AI_기능_시스템_요구사항.md) | 범위 확정, 세부 KPI 추후 |
| 32 | [데이터셋·라벨링 요구사항](./32_데이터셋_라벨링_요구사항.md) | 템플릿 |
| 33 | [모델 학습·평가 요구사항](./33_모델_학습_평가_요구사항.md) | 템플릿 |
| 34 | [AI 추론 인터페이스 명세](./34_AI_추론_인터페이스_명세.md) | 최소 계약 |
| 35 | [AI 배포·모니터링 요구사항](./35_AI_배포_모니터링_요구사항.md) | 템플릿 |
| 36~49 | 예약 | 상세 Spec 추가 |

## 2. 마일스톤

| Gate | 목표 | Exit Criteria |
|---:|---|---|
| `AI-M0 문제 동결` | 클래스·사용자 판단·오류비용 합의 | 기능 ID, class definition, 인수 지표 확정 |
| `AI-M1 데이터 기준선` | 수집·분할·라벨·검수 계약 | leakage 없는 dataset version 생성 |
| `AI-M2 Offline 기준선` | 재현 가능한 학습·평가 | baseline과 class별 오류표 생성 |
| `AI-M3 Edge 계약` | Jetson 입력·출력·성능 연결 | HW/CONTROL contract test 통과 |
| `AI-M4 E2E 이벤트` | 대표 프레임→BE 조회 | model·원본·위치·신뢰도 연결 |
| `AI-M5 안정화` | drift·오탐 검토와 rollback | RC dataset 회귀시험 통과 |

모든 세부 목표일은 `[추후 추가 예정]`이며 Gate 통과를 대신하지 않는다.

## 3. 다른 파트와 경계

- HW/CONTROL로부터 calibration·timestamp가 있는 이미지와 Jetson 자원 예산을 받는다.
- BE에 model identity, class, geometry, confidence와 분석 근거를 전달한다.
- 결함 결과는 차량 steering·speed 명령으로 직접 변환하지 않는다.

## 4. 후속 작성 템플릿

각 AI Spec에는 목적, 사용자 결정, 입력·출력, 데이터 계약, 지표, 오류 비용, 성능 예산, 버전·재현성, Exit Criteria와 인계물을 채운다. 미작성 절은 `[추후 추가 예정]`으로 유지한다.
