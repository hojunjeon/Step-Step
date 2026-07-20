# Codex 에이전트 사용 가이드

이 문서는 이 저장소에서 Codex 스킬로 GitLab 이슈/브랜치/MR을 자동화하는 방법과, 그러기 위한 로컬 환경 설정을 정리합니다. 프로젝트 자체에 대한 설명은 [README.md](../README.md)를 참고하세요.

> GitLab 호스트: **SSAFY 자체 호스팅 인스턴스** `https://lab.ssafy.com` (gitlab.com 아님)

## 사용 가능한 Codex 스킬

### 이 프로젝트의 이슈 트래커: Jira가 메인
이 팀은 Epic/Sprint/Story를 **Jira**(`S15P11A103` 프로젝트)에서 관리합니다. GitLab 이슈는 선택사항이며, **브랜치명과 커밋 메시지는 항상 Jira 키(`S15P11A103-33` 같은 형식)를 기준으로** 짓습니다. Codex에게 "구현해줘" 라고 요청할 때는 관련 Jira 키를 먼저 알려주세요 — 없으면 Codex가 먼저 물어봅니다 ([AGENTS.md](../AGENTS.md) 참고).

### `create-issue-with-branch`
대화 중 "이슈 만들어줘" / "이 Jira 카드로 브랜치 파줘" 라고 요청하면, **Jira 키를 먼저 확인**한 뒤(없으면 물어봄) 그 키를 기준으로 브랜치를 만들어 로컬 체크아웃 + 원격 푸시까지 해줍니다. 필요하면 GitLab 이슈도 팀 템플릿(개요/상세내용/완료조건/담당자)으로 초안을 잡아 **명시적 승인 후에만** 추가로 생성합니다 (Jira 카드와는 별개의 보조 트래킹용).

- 브랜치 네이밍: `<type>/<Jira키>-<영문 짧은설명>` (예: `feature/S15P11A103-302-uart-driver`)
- GitLab 이슈를 같이 만드는 경우 라벨은 이 저장소에 실제 존재하는 `ai`, `bug`, `chore`, `docs`, `hardware`, `infra`, `refactor`, `release`, `robot`, `test` 중 하나 선택 (이 목록 밖의 라벨은 사용하지 않음)
- 정의: [.agents/skills/create-issue-with-branch/SKILL.md](../.agents/skills/create-issue-with-branch/SKILL.md)

### `create-mr-from-template`
대화 중 "PR 만들어줘" / "MR 올려줘" 라고 요청하면, 현재 브랜치의 커밋·diff를 바탕으로 팀 MR 템플릿(관련 Jira 카드/작업 개요/상세 내용/체크리스트/참고 사항)을 채워서 초안으로 먼저 보여주고, **명시적으로 승인한 뒤에만** 브랜치를 푸시하고 실제로 MR을 생성합니다.

- 관련 Jira 키는 브랜치 이름(`<type>/<Jira키>-...`)에서 자동으로 뽑아 MR 설명에 표시합니다 (Jira 스마트 커밋 연동을 위해 MR 제목/설명에도 키가 포함됨).
- GitLab 이슈도 같이 쓰는 경우에만 `Closes #이슈번호`를 추가로 넣습니다.
- 체크리스트는 기본적으로 전부 미체크 — 사용자가 명시적으로 확인해준 항목만 체크합니다.
- MR은 항상 로컬에 로그인된 사용자 본인의 `glab` 계정으로만 생성됩니다 (다른 사람 명의로 대신 올리지 않음).
- 정의: [.agents/skills/create-mr-from-template/SKILL.md](../.agents/skills/create-mr-from-template/SKILL.md)

> 새 스킬이 추가되면 이 섹션에 한 항목씩 추가해주세요.

### ⚠️ Codex 환경에서 `glab` 명령이 안 될 때

Codex(터미널 밖에서 자동으로 명령을 실행하는 모드)의 도구 실행 환경은 프로젝트 폴더는 사용자의 실제 로컬 저장소와 공유하지만, `glab` 로그인 토큰이 저장되는 `AppData\Local\glab-cli` 같은 사용자 프로필/자격증명 영역은 공유하지 않을 수 있습니다. 그래서 본인 터미널에서 `glab auth login`을 정상적으로 마쳐도, Codex가 실행하는 `glab issue create` / `glab mr create`는 계속 미인증(401)으로 실패할 수 있습니다.

이건 설정을 잘못한 게 아니라 구조적인 샌드박스 격리이며, `glab config set` 등으로 고쳐지지 않습니다. 이 경우 Codex는:
- 이슈/PAT 내용, 브랜치명 등은 그대로 초안으로 잡아주고
- `glab issue create` / `glab mr create` **실행 명령만 사용자에게 건네서 사용자 본인 터미널에서 실행**하도록 안내합니다
- 브랜치 생성·체크아웃·`git push`는 별도 자격증명 체계(SSH 등)라 Codex가 문제없이 대신 처리할 수 있습니다

**Personal Access Token을 Codex와의 채팅창에 붙여넣어서 우회하지 마세요.** 채팅 대화 기록에 토큰이 그대로 남아 노출 범위가 넓어집니다. 실수로 붙여넣었다면 `https://<host>/-/user_settings/personal_access_tokens`에서 즉시 Revoke 하세요.

## 환경 설정

이 스킬들을 쓰려면 각자 아래를 로컬에 설치/설정해야 합니다.

### 1. Git
이미 설치되어 있다면 생략. 없으면 https://git-scm.com/downloads

### 2. GitLab CLI (`glab`)
이슈/브랜치 자동화가 내부적으로 `glab` 명령을 사용합니다.

**Windows**
```powershell
winget install GLab.GLab
```
(또는 https://gitlab.com/gitlab-org/cli/-/releases 에서 직접 다운로드, scoop 사용 시 `scoop install glab`)

**설치 확인**
```bash
glab --version
```
설치 직후 "인식할 수 없는 명령"이 뜨면 PATH가 아직 안 갱신된 것 — 로그아웃 후 재로그인(또는 재부팅) 후 다시 확인.

### 3. `glab` 인증 (SSAFY 자체 호스팅 지정 필수)
```bash
glab auth login --hostname lab.ssafy.com
```
- Git protocol: HTTPS 추천
- 로그인 방식에서 **Token**을 선택하고, 안내되는 링크로 PAT 발급:
  `https://lab.ssafy.com/-/user_settings/personal_access_tokens/legacy/new?scopes=api,write_repository`
  (Scope는 `api`, `write_repository` 둘 다 필요 — 링크에 이미 체크되어 있음)
- 발급된 토큰을 프롬프트에 붙여넣기 (입력값은 화면에 안 보임, 정상)
- `✓ Logged in as <아이디>`가 뜨면 성공

**인증 확인**
```bash
glab auth status --hostname lab.ssafy.com
```
`--hostname` 없이 실행하면 기본값인 gitlab.com을 검사해서 실패로 나오니 반드시 붙여야 함. `✓ Logged in to lab.ssafy.com as ...`가 떠야 정상.

### 4. 저장소 연결
로컬 저장소의 `origin`이 실제 팀 GitLab 프로젝트(`lab.ssafy.com`)를 가리키고 있어야 이슈/브랜치 자동화가 동작합니다.

```bash
git clone https://lab.ssafy.com/<그룹>/<프로젝트>.git     # 처음 받는 경우
# 또는 이미 폴더가 있다면
git remote add origin https://lab.ssafy.com/<그룹>/<프로젝트>.git
```

### 5. 확인
아래가 모두 정상이면 스킬을 바로 쓸 수 있습니다.
```bash
glab --version && glab auth status --hostname lab.ssafy.com && git remote get-url origin
```

## 브랜치 / 커밋 / MR 규칙 요약
- 브랜치: `<type>/<Jira키>-<짧은설명>` (`type`: `feature`, `fix`, `chore`) — 예: `feature/S15P11A103-302-uart-driver`
- 커밋: `<type>(<Jira키>): <설명>` — 예: `feat(S15P11A103-302): 파손 분류기 추가`. Jira 스마트 커밋 연동이 이 형식으로 커밋을 감지해서 해당 Jira 카드에 자동 연결한다.
- GitLab 라벨(GitLab 이슈/MR을 같이 쓸 때만 사용, Jira 키와는 별개): `ai`, `bug`, `chore`, `docs`, `hardware`, `infra`, `refactor`, `release`, `robot`, `test`
- MR(Merge Request): 설명에 관련 Jira 키 표시, GitLab 이슈를 같이 쓰면 `Closes #이슈번호`도 추가. 최소 1인 리뷰 후 squash merge

## Codex 사용 시 커밋 규칙
Codex에게 구현을 맡길 때 적용되는 규칙 ([AGENTS.md](../AGENTS.md) 참고):
- 큰 작업을 한 번에 커밋하지 않고, 논리적 단위로 나눠서 중간중간 커밋
- 커밋에 `Co-Authored-By: Codex ...` 트레일러를 붙이지 않음 — author는 로컬 git 설정을 따르는 실제 작업자 이름만 남음
