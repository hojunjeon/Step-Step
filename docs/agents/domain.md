# Domain Docs

## Before exploring

- 루트 `CONTEXT.md`를 읽습니다.
- 존재한다면 관련 `docs/adr/` 문서를 읽습니다.
- 파일이 없으면 별도로 문제 삼지 않고 계속 진행합니다.

## Layout

이 저장소는 단일 컨텍스트 구조를 사용합니다.

- `CONTEXT.md`: 도메인 용어와 개념
- `docs/adr/`: 아키텍처 결정 기록
- `src/`: 구현 코드

출력에서는 `CONTEXT.md`에 정의된 용어를 일관되게 사용합니다.
기존 ADR과 충돌하는 제안은 그 충돌을 명시합니다.
