# team-ai-agent 프로젝트 규칙

## Jira 키 확인 (구현 작업 시작 전 필수)
- "구현해줘", "만들어줘" 같은 개발 요청을 받으면, **작업을 시작하기 전에 관련 Jira 이슈 키(예: `S15P11A103-33`)를 반드시 확인한다.**
- 사용자가 메시지에 키를 먼저 안 줬으면, 작업 착수 전에 물어봐서 받는다. 대화 맥락상 어느 Jira Story인지 명백해도, 정확한 키 번호는 추측하지 말고 확인받는다.
- 받은 키는 브랜치명과 그 작업의 모든 커밋 메시지에 일관되게 사용한다 (아래 "브랜치 / 커밋 규칙" 참고). 키 없이 임의로 브랜치를 만들거나 커밋하지 않는다.
- 이미 만들어둔 브랜치에서 이어서 작업하는 경우, 브랜치명에서 Jira 키를 그대로 재사용한다 (다시 물어볼 필요 없음).
- 팀 프로세스/도구 설정 변경처럼 특정 Jira Story에 매핑되지 않는 메타성 작업은 예외로 키 없이 진행할 수 있다 (사용자가 명시적으로 예외라고 확인해준 경우).

## 브랜치 전략 (4단계)
```
main
 └─ develop
     ├─ ai/develop      ← 작업 브랜치: ai/<type>/<Jira키>-...
     ├─ server/develop  ← 작업 브랜치: server/<type>/<Jira키>-...
     └─ robot/develop   ← 작업 브랜치: robot/<type>/<Jira키>-... (hardware/ 포함)
```
- `main`: 배포/데모 기준 안정 브랜치
- `develop`: 스프린트 통합 브랜치
- `ai/develop` · `server/develop` · `robot/develop`: 도메인 통합 브랜치, `develop`에서 분기. 저장소 최상위 폴더 기준(`ai/`→ai, `server/`→server, `robot/`+`hardware/`→robot)
  - 도메인 통합 브랜치 이름을 `ai/develop`처럼 도메인을 앞에 두는 이유: git은 브랜치를 경로처럼 저장해서, `develop`이라는 브랜치와 `develop/ai`라는 브랜치를 동시에 가질 수 없다(참조 경로 충돌 — 실제로 시도했다가 `cannot lock ref` 에러로 확인됨). `ai/develop`과 `ai/feature/...`는 둘 다 `ai/` 아래의 서로 다른 형제 경로라 문제없이 공존한다.
- 작업 브랜치: `<domain>/<type>/<Jira키>-<영문 짧은설명>`, 해당 도메인 통합 브랜치(`<domain>/develop`)에서 분기
  - 예: `ai/feature/S15P11A103-302-damage-classification`, `server/feat/S15P11A103-310-image-upload-api`, `robot/feat/S15P11A103-320-motor-control`
  - 특정 도메인 폴더에 속하지 않는 공통 작업(`docs/`, `tests/`, 루트 설정, CI 등)은 도메인 prefix 없이 `<type>/<Jira키>-<영문 짧은설명>` 형식으로 `develop`에서 바로 분기한다.
  - 한 작업이 여러 도메인 폴더를 동시에 건드리면 가능한 한 도메인별로 브랜치/PR을 나눈다. 불가피하게 함께 가야 하면 변경 비중이 더 큰 도메인을 prefix로 쓴다.

## 병합(PR) 흐름
- 작업 브랜치 → 도메인 통합 브랜치(`ai/develop` 등): Story 단위로 PR, 최소 1인 리뷰
- 도메인 통합 브랜치 → `develop`: 스프린트 리뷰/데모 준비 시점에 PR
- `develop` → `main`: 데모/배포 시점에 PR
- MR/PR 제목: 소스 브랜치가 `ai/`·`server/`·`robot/`로 시작하면 제목 맨 앞에 `[ai]`/`[server]`/`[robot]`을 붙인다. 이 3개 외의 도메인은 임의로 만들지 않는다. 공통 작업 브랜치나 도메인 통합 브랜치 자체의 상위 병합(`ai/develop`→`develop` 등, 도메인이 이미 브랜치명에 드러남)은 생략 가능

## 커밋 규칙
- `<type>(<Jira키>): <설명>` (예: `feat(S15P11A103-302): 파손 분류기 추가`) — Jira 스마트 커밋 연동이 이 형식으로 커밋을 감지해서 해당 Jira 카드에 자동으로 연결한다. 도메인은 브랜치명으로 이미 구분되므로 커밋 메시지에는 넣지 않는다.
- `type`은 Conventional Commits 기준: `feat`/`fix`/`docs`/`test`/`refactor`/`chore`

## 구현 작업 시 커밋 방식
- "구현해줘" 같은 개발 요청을 받으면, 작업을 논리적 단위(기능/모듈 단위)로 나눠서 진행 중간중간 커밋한다. 다 끝난 뒤 한 번에 거대한 커밋 하나로 몰아넣지 않는다.
- 커밋 메시지에 `Co-Authored-By: Codex ...` 트레일러를 붙이지 않는다. 커밋 author는 로컬 `git config user.name` / `user.email`을 그대로 사용하며, 실제 작업한 사람 이름만 남긴다.
- 커밋 전에는 항상 `git status` / `git diff`로 변경 범위를 확인하고, 관련 없는 파일은 같이 커밋하지 않는다.

## 로컬 Codex 문서 보호
- 다음 Codex 전용 로컬 파일과 디렉터리는 개인 환경에서만 사용하며 절대 stage하거나 commit하지 않는다:
  - `AGENTS.md`
  - `.agents/`
  - `docs/codex-guide.md`
  - `README.codex.md`
- 원격 공유용 Claude Code 파일인 `CLAUDE.md`, `.claude/`, `docs/claude-code-guide.md`, `README.md`는 Codex 전환을 이유로 수정하거나 삭제하지 않는다.
