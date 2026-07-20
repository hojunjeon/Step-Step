# 점자블록 AIoT 안전관리 시스템

> 자율주행 로버 기반 점자블록 AIoT 안전관리 프로젝트 — SSAFY AIoT 트랙

`AI VISION` · `AUTONOMOUS ROVER` · `EDGE + SERVER`

## 한 줄 정의
소형 자율주행 로버가 점자블록 구간을 이동하며 상태를 촬영·분석하고, 훼손 정도를 **정상·주의·교체**로 판정해 파손 이미지와 좌표를 서버에 기록하는 AIoT 점검 시스템.

## 배경
점자블록은 시각장애인의 보행을 돕는 시설이지만, 파손·오염·탈락이나 잘못된 배치로 유도 기능이 약화될 수 있다. 현재는 사람이 직접 넓은 구간을 돌며 점검해야 해 반복 점검이 어렵고 기록이 표준화되지 않는다. 이 프로젝트는 점검 과정을 로버·비전 모델·거리 센서·서버로 데이터화한다.

## 핵심 기능 (MVP)
- 점자블록 구간 주행 및 카메라 촬영
- 점자블록 영역/형태 인식, `normal_block` / `damaged_block` 상태 분류
- 정상·주의·교체 등급 산정
- LiDAR·초음파 기반 장애물 감지 및 로컬 안전 정지
- 파손 이미지·좌표 서버 전송 및 이벤트 저장

## 확장 기능
신호등·횡단보도 앞 점자블록의 각도·배치 적정성을 GPU 서버에서 분석 판정.

## 팀 구성
| 이름 | 역할 |
|---|---|
| 전호준 | 팀장 · HW |
| 황인서 | HW |
| 박다빈 | AI · 백엔드 |
| 박민규 | 백엔드 |
| 이주인 | AI |
| 김초아 | AI |

## 하드웨어
Jetson Orin Nano · Raspberry Pi 5 · YDLIDAR X4-PRO · USB/CSI 카메라 · PCA9685 제어보드 · SG-90 서보 · DC 모터 · HC-SR04

## 저장소 구조
```
ai/       # 데이터셋, 학습, 추론, 배포
robot/    # 카메라, 주행, 센서, 모터, 안전 정지
server/   # API, 모델 연동, 저장소
hardware/ # 회로도, 핀맵
tests/
docs/
```

## 개발 프로세스
1주 단위 Scrum · Jira Epic/Story · `main` → `develop` → 도메인 브랜치(`ai/develop`·`server/develop`·`robot/develop`) → 작업 브랜치 · PR 최소 1인 리뷰 (자세한 규칙은 [AGENTS.md](AGENTS.md) 참고)

## Codex로 이슈/브랜치/MR 작업하기
이 저장소는 Codex 스킬로 이슈 생성 → 브랜치 → MR을 자동화합니다. 스킬 목록, 환경 설정, 라벨/브랜치 컨벤션은 [docs/codex-guide.md](docs/codex-guide.md) 참고.

## 커밋 규칙
[AGENTS.md](AGENTS.md) 참고 — 논리 단위로 커밋, `Co-Authored-By` 트레일러 미사용.
