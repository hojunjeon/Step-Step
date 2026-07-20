---
name: create-mr-from-template
description: Draft a GitLab merge request (what the team casually calls "PR") from the current branch's commits/diff using this team's MR template, show the draft for review, and — only after explicit approval — push and open the MR via `glab`. Use this whenever the user asks to open/create/raise a PR or MR ("PR 만들어줘", "PR 올려줘", "MR 만들어줘", "merge request 열어줘"), typically after finishing work on a `feature/`, `fix/`, or `chore/` branch. The MR is always opened under the user's own authenticated `glab` account — never fabricate completion of checklist items the user hasn't confirmed. Requires GitLab CLI (`glab`) installed and authenticated, and a local branch that's ahead of the base branch.
---

# MR(PR) 생성

목표는 "지금 브랜치에서 한 작업을 팀 MR 템플릿에 맞춰 정리하고, 사용자가 검토해서 승인한 뒤에만 실제로 올리는" 것이다. MR 생성은 팀 전체에 보이는 행위이고 리뷰어에게 알림이 가므로, **사용자의 명시적 승인 없이는 절대 `glab mr create`를 실행하지 않는다.**

## 0. 사전 점검

- `glab --version`, `glab auth status` — 설치/인증 안 되어 있으면 프로젝트 루트 `README.md`의 "환경 설정" 참고하도록 안내
- `git branch --show-current` — 지금 base 브랜치(main 등) 위에 있다면, MR을 만들 것이 아니라 먼저 작업 브랜치로 옮겨야 함을 알린다
- `git status` — 커밋 안 된 변경사항이 있으면 먼저 커밋할지 확인 (임의로 커밋하지 않는다)
- MR은 항상 로컬에 `glab auth status`로 로그인되어 있는 사용자 계정으로 올라간다. 다른 사람 명의로 대신 올리거나 계정을 바꿔서 진행하지 않는다.

**⚠️ 샌드박스 인증 격리 주의**: [create-issue-with-branch](../create-issue-with-branch/SKILL.md)와 동일한 문제가 여기도 적용된다. Codex의 도구 실행 환경이 사용자의 실제 터미널과 `glab` 자격증명을 공유하지 않을 수 있어, 사용자 터미널에서는 로그인된 상태여도 Codex가 실행하는 `glab auth status`는 미인증으로 나올 수 있다. 이 증상이면 3-2의 `glab mr create`는 Codex가 직접 실행하지 않고 명령을 사용자에게 주고 사용자 터미널에서 실행하도록 안내한다. 사용자의 Personal Access Token을 채팅으로 받아 우회하지 않는다.

## 1. 내용 채우기

### base(target) 브랜치 결정
이 저장소는 4단계 구조(`main` → `develop` → `ai/develop`·`server/develop`·`robot/develop` → 작업 브랜치)를 쓴다. 현재 브랜치 이름으로 target을 정한다:
- `ai/...`, `server/...`, `robot/...`로 시작하면 target은 해당 도메인 통합 브랜치(`ai/develop` 등)
- 그 외(공통 작업, `<type>/<Jira키>-...`)면 target은 `develop`
- 도메인 통합 브랜치(`ai/develop` 등)나 `develop` 자신에서 상위로 올리는 MR(`ai/develop`→`develop`, `develop`→`main`)은 스토리 단위가 아니라 스프린트 리뷰/데모 시점에 만드는 것이므로, 사용자가 명시적으로 그 상위 병합을 요청한 경우에만 진행한다. 확실치 않으면 target을 추측하지 말고 사용자에게 확인한다.

### 관련 Jira 키 찾기
현재 브랜치 이름이 `<domain>/<type>/<Jira키>-<설명>` 또는 공통 작업용 `<type>/<Jira키>-<설명>` 규칙(예: `ai/feature/S15P11A103-302-uart-driver`)을 따르면 거기서 Jira 키를 뽑는다. 패턴에 안 맞으면 사용자에게 관련 Jira 키를 물어본다 — 이 팀은 Jira가 메인 트래커이므로, 지어내거나 생략하지 말고 반드시 확인한다.

### MR 제목에 도메인 표시
현재 브랜치 이름이 `ai/`, `server/`, `robot/` 중 하나로 시작하면, MR 제목 맨 앞에 `[도메인]`을 붙인다 — 예: `[ai] feat(S15P11A103-302): UART 드라이버 초기 구현`. 도메인은 **AGENTS.md에 정의된 `ai`/`server`/`robot` 3개로만 한정**하고, 그 외 값을 임의로 만들어 붙이지 않는다. 공통 작업 브랜치(`<type>/<Jira키>-...`, 도메인 prefix 없음)나 도메인 통합 브랜치 자체에서 올리는 상위 MR(`ai/develop`→`develop` 등, 이 경우 도메인은 이미 브랜치명에 드러나므로 제목에 또 붙이지 않아도 됨)은 도메인 표시를 생략한다.

### GitLab 이슈 번호 (있는 경우만)
브랜치를 만들 때 GitLab 이슈도 같이 만들었다면 그 이슈 번호도 확인해서 `Closes #이슈번호`에 쓴다. GitLab 이슈 없이 Jira 키만 있는 게 일반적인 케이스다.

### 변경 내용 파악
```bash
git log <base브랜치>..HEAD --oneline
git diff <base브랜치>..HEAD --stat
```
이 결과와 대화 맥락을 근거로 "작업 개요"와 "상세 내용"을 채운다. 실제로 diff에 없는 내용을 지어내지 않는다.

### 템플릿
```
## 🧩 관련 Jira
S15P11A103-302
(GitLab 이슈도 있으면 한 줄 추가: Closes #이슈번호)

---

## 🎯 작업 개요
변경/추가된 기능 요약

---

## 🔍 상세 내용
- 변경된 코드 주요 요약
- 세부 구현 내용
- 참고한 자료나 문서

---

## ✅ 체크리스트
- [ ] 기능이 정상 동작한다
- [ ] 테스트 코드 통과
- [ ] 코드 리뷰 반영 완료
- [ ] 스타일 가이드 준수

---

## 💬 참고 사항
- 관련 문서/링크
- 디자인/기획 참고 위치
- 기타 전달 사항
```

**체크리스트는 기본적으로 전부 미체크 상태로 둔다.** "테스트 통과했다", "코드 리뷰 반영 끝났다" 처럼 사용자가 명시적으로 확인해준 항목만 체크하고, 확인 안 된 걸 임의로 체크하지 않는다.

Jira 키를 못 찾겠으면(브랜치명이 옛날 규칙이거나 수동으로 만든 브랜치 등) 사용자에게 물어보고, 정말 없다면 그 사실을 확인한 뒤 "관련 Jira" 섹션을 생략한다.

## 2. 검수 루프

채운 MR 초안 전체를 사용자에게 보여준다. 수정 요청이 오면 반영하고 다시 보여준다. **명시적 승인("ok", "올려줘", "좋아", "생성해줘" 등)을 받기 전까지는 3단계로 넘어가지 않는다.**

## 3. 승인 후 실행

### 3-1. 원격에 브랜치 반영
```bash
git push -u origin <현재브랜치>
```
이미 푸시되어 최신 상태면 생략.

### 3-2. MR 생성

**Codex 자신의 `glab auth status`가 정상이면** 아래 명령을 Codex가 직접 실행한다. 멀티라인 본문은 임시 파일에 먼저 쓰고 셸 치환으로 넘긴다:
```bash
glab mr create \
  --title "<제목>" \
  --description "$(cat <임시파일경로>)" \
  --source-branch <현재브랜치> \
  --target-branch <base브랜치>
```
`<제목>`은 위 "MR 제목에 도메인 표시" 규칙대로 도메인이 있으면 `[ai] ...`/`[server] ...`/`[robot] ...`로 시작한다.

필요하면 `--label "<라벨>"`도 추가 — GitLab 이슈에 붙였던 라벨과 동일하게 맞춘다. 이 저장소의 실제 라벨은 아래 10개뿐이다 (2026-07-15 기준 `glab label list`로 확인된 정확한 문자열, 이모지 포함 — `--label`엔 이모지까지 그대로 쓴다):

| 라벨 (그대로 사용) | 용도 |
|---|---|
| `🧠 ai` | 모델 학습 및 추론 |
| `🐛 bug` | 버그 이슈 |
| `🧹 chore` | 빌드, 기타설정 |
| `📋 docs` | 문서 작업 |
| `🍓 hardware` | 하드웨어 연결, 핀/보드 설정 |
| `🔌 infra` | 인프라 설정 |
| `🔨 refactor` | 리팩토링 |
| `😊 release` | 릴리즈 |
| `🤖 robot` | 주행/센서/모터 관련 |
| `🧪 test` | 테스트 |

이 목록 밖의 라벨(`embedded`, `backend`, `server` 등)은 존재하지 않는다 — 제목의 `[도메인]` 표시와 `--label`은 별개다. `server` 도메인처럼 이름이 같은 라벨이 없는 경우, 라벨은 억지로 끼워맞추지 말고 내용에 맞는 다른 라벨(예: `🔌 infra`, `📋 docs`)을 고르거나 생략한다.

⚠️ **이모지 없이 `--label "ai"`처럼 넘기면 GitLab이 위 라벨과 다른 문자열로 인식해서 이모지 없는 새 라벨을 만들어버린다** (실제로 이 문제로 라벨이 중복 생성된 적 있음). 위 표는 2026-07-15 스냅샷이라 이후 바뀌었을 수 있으니, 확신이 안 서면 `--label`을 쓰기 전에 `glab label list`로 정확한 문자열을 다시 조회해서 그대로 복사해 쓴다. 조회가 안 되거나 매칭되는 라벨을 못 찾겠으면 라벨 없이 진행한다 — 이모지를 추측해서 지어내지 않는다.

**샌드박스 인증 격리 증상이 있다면**, 위 명령을 사용자가 자기 터미널에 붙여넣을 수 있는 형태로 안내한다 (PowerShell이면 `$body = @'...'@` here-string 사용). 사용자가 실행 결과(MR URL)를 알려주면 그걸로 마무리한다.

### 3-3. 결과 보고
생성된 MR URL을 사용자에게 알린다. 예:
> MR 생성됨: https://lab.ssafy.com/GROUP/PROJECT/-/merge_requests/34 — `[ai] feat(S15P11A103-302): UART 드라이버 초기 구현` (`ai/feature/S15P11A103-302-uart-driver` → `ai/develop`)

## 실패 처리
- `glab mr create` 실패(권한 없음, 이미 열린 MR 존재, 브랜치 미푸시 등) → 원인 그대로 사용자에게 전달, 임의로 옵션을 바꿔가며 재시도하지 않는다
  - 특히 "Another open merge request already exists for this source branch" (409) → 이미 MR이 만들어져 있다는 뜻이므로 에러가 아니라 정상 상태. 기존 MR URL을 다시 확인시켜주고 재시도를 권하지 않는다
- base 브랜치와 차이가 없으면(커밋 없음) MR을 만들 이유가 없다고 알리고 중단한다
- `glab auth status`가 사용자 터미널과 Codex 쪽에서 다르게 나오면 → 샌드박스 인증 격리(위 0단계 참고). 바로 "사용자 터미널에서 실행" 경로로 전환한다
