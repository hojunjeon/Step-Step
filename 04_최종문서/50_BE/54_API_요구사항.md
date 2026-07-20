# API 요구사항

> 상태: `[추후 추가 예정]` 템플릿. endpoint와 payload 상세는 BE 상세설계에서 작성한다.

## 1. API 그룹

- Device 등록·health 조회
- Route·version 조회와 patrol 시작 정보
- Event 멱등 접수·상태 조회
- Media 업로드·무결성 확인
- Patrol·event·지도·관리 구간 조회
- AI analysis 요청·결과 조회
- Review·수정·보수 후보 workflow

## 2. 공통 계약

| 항목 | 요구 |
|---|---|
| Versioning | 명시적 API/schema version |
| Auth | device와 human principal 분리 |
| Idempotency | create/upload 요청 key 지원 |
| Error | stable code, message, retryable, trace ID |
| Pagination | 대량 조회의 stable cursor `[결정 필요]` |
| Time/Geo | timezone·좌표계·정확도 명시 |
| Audit | 변경 actor와 before/after 보존 |

## 3. 완료 조건

OpenAPI 또는 동등한 machine-readable contract, example, validation, auth·error·idempotency contract test가 연결돼야 `BE-M0`을 통과한다.
