---
name: create-issue-with-branch
description: Confirm the related Jira issue key (this team's source of truth for work items — Epic/Sprint/Story live in Jira, project `S15P11A103`) before starting any implementation work, then create a matching branch (and optionally a GitLab issue) named after that key. Use this whenever the user asks to create/open a branch, start implementing something ("구현해줘", "만들어줘", "이거 작업 시작하자"), or explicitly asks for a GitLab issue ("이슈 만들어줘", "이슈 등록해줘"). If the user hasn't given a Jira key yet, ask for it before doing anything else — never guess or invent one. Requires GitLab CLI (`glab`) installed and authenticated, and a local git repo whose `origin` points at a real GitLab project.
---

# Jira 키 확인 + 브랜치(+ 선택적 GitLab 이슈) 생성

목표는 "구현 작업을 시작하기 전에 반드시 관련 Jira 키를 확보하고, 그 키가 반영된 브랜치를 만들어서 바로 작업을 시작할 수 있게" 하는 것이다. 이 팀은 Epic/Sprint/Story를 **Jira**(`S15P11A103` 프로젝트)에서 관리하며, GitLab 이슈는 선택사항이다. 브랜치 생성은 로컬 작업이라 부담이 적지만, GitLab 이슈 생성은 팀 전체에게 보이는 행위이므로 **사용자의 명시적 승인 없이는 절대 `glab issue create`를 실행하지 않는다.**

## 0. Jira 키 확보 (가장 먼저, 반드시)

사용자 메시지에 `S15P11A103-33` 같은 Jira 키가 이미 포함돼 있으면 그걸 쓴다. **없으면 다른 어떤 단계로도 넘어가지 말고 먼저 물어본다** — 대화 맥락상 어느 카드인지 짐작이 가도 번호를 추측해서 지어내지 않는다.

이미 그 Jira 키로 만든 브랜치에서 이어서 작업하는 상황이면(브랜치명에 키가 이미 들어있음) 다시 물어볼 필요 없이 재사용한다.

## 0-1. 사전 점검 (문제가 있으면 여기서 멈추고 안내)

GitLab 이슈도 같이 만들 경우에만 필요:
- `glab --version` — 설치 안 되어 있으면 설치 안내 (프로젝트 루트 `README.md`의 "환경 설정" 참고하라고 안내)
- `glab auth status` — 인증 안 되어 있으면 `glab auth login` 안내 (gitlab.com인지 자체 호스팅 인스턴스인지 확인)
- `git remote get-url origin` — origin이 없거나 GitLab 호스트가 아니면, 어느 프로젝트에 이슈를 만들지 사용자에게 확인

브랜치만 만드는 거면 `git remote get-url origin`만 확인하면 충분하다 (glab 불필요).

**⚠️ 샌드박스 인증 격리 주의**: Codex의 도구 실행 환경(Bash/PowerShell)은 사용자의 실제 인터랙티브 터미널과 `glab` 자격증명 저장소(`AppData\Local\glab-cli`)를 공유하지 않을 수 있다. 사용자가 자기 터미널에서 `glab auth login --hostname <host>`으로 로그인해도, Codex가 실행하는 `glab auth status`는 여전히 미인증으로 나올 수 있다 — 이건 설정 실수가 아니라 구조적 격리이므로 `glab config set` 등으로 고치려 하지 않는다. 이 증상이 보이면 GitLab 이슈 생성 명령은 Codex가 직접 하지 않고, **명령을 사용자에게 주고 사용자의 터미널에서 실행하도록 안내**한다. 이걸 우회하겠다고 사용자의 Personal Access Token을 채팅으로 받아 Codex 쪽에서 로그인하는 것은 절대 하지 않는다 (토큰이 대화 기록에 노출됨).

## 1. 브랜치명 결정 (Jira 키 + 도메인 기준)

이 저장소는 4단계 브랜치 구조를 쓴다: `main` → `develop` → 도메인 통합 브랜치(`ai/develop`·`server/develop`·`robot/develop`) → 작업 브랜치. 작업 브랜치는 항상 해당 도메인 통합 브랜치에서 분기한다.

규칙: `<domain>/<type>/<Jira키>-<영문kebab짧은설명>`

- `domain`은 작업이 주로 건드릴 최상위 폴더 기준: `ai`(`ai/`) · `server`(`server/`) · `robot`(`robot/`, `hardware/` 포함). Jira 카드 내용이나 대화 맥락으로 애매하면 사용자에게 물어본다 — 짐작해서 정하지 않는다.
- 특정 도메인 폴더에 속하지 않는 공통 작업(`docs/`, `tests/`, 루트 설정, CI 등)은 도메인 없이 `<type>/<Jira키>-<영문kebab짧은설명>`으로 만들고, `develop`에서 바로 분기한다.
- `type`은 버그/오류 수정이면 `fix`, 설정/CI/문서 등 잡무면 `chore`, 그 외 일반 기능 작업이면 `feature`
- `<Jira키>`는 0단계에서 확보한 키 그대로 (예: `S15P11A103-302`)
- 짧은설명은 Jira 카드 제목을 2~4단어 영문 kebab-case로 직접 요약해서 만든다 (한글 제목을 그대로 쓰지 않는다)
- 예: `ai/feature/S15P11A103-302-uart-driver`, `server/fix/S15P11A103-411-mqtt-reconnect`, `chore/S15P11A103-500-ci-cache` (공통 작업)

## 2. GitLab 이슈도 만들지 확인

기본적으로는 브랜치만 만들면 충분하다 (Jira가 메인 트래커이므로). 사용자가 GitLab 이슈도 명시적으로 요청했거나, GitLab 쪽에서도 추적이 필요한 상황이면 아래 순서로 진행한다.

### 2-1. 이슈 내용 채우기
사용자의 요청(현재 대화 맥락 포함)에서 정보를 뽑아 아래 템플릿을 채운다. 정보가 부족하면 그냥 지어내지 말고 빈 항목으로 두거나 짧게 되묻는다. 설명 맨 위에 관련 Jira 키를 링크로 남긴다.

```
### 🔗 관련 Jira
- S15P11A103-302

### 🧩 개요
- 구현 또는 수정할 기능 간단 요약

### 🎯 상세 내용
- 세부 요구사항 및 동작 방식
- 참고 링크 또는 관련 자료

### ✅ 완료 조건
- [ ] 기능이 정상 동작한다
- [ ] 테스트 통과
- [ ] 코드 리뷰 완료

### 👤 담당자
- @담당자명
```

**라벨**: 이 저장소에 실제로 존재하는 라벨은 아래 10개뿐이다 (2026-07-15 기준 `glab label list`로 확인된 정확한 문자열, 이모지 포함). **이 목록 밖의 라벨(예: `embedded`, `backend`, `feature`)은 만들거나 지어내지 않는다. `--label` 인자에는 아래 문자열을 이모지까지 그대로 쓴다.**

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

⚠️ **이모지 없이 `--label "ai"`처럼 넘기면 GitLab이 위 라벨과 다른 문자열로 인식해서 이모지 없는 새 라벨을 만들어버린다** (실제로 이 문제로 라벨이 중복 생성된 적 있음). 위 표는 2026-07-15에 확인한 스냅샷이라 이후 라벨이 추가/변경/삭제됐을 수 있으니, 확신이 안 서면 아래 순서로 다시 확인한다:
1. `glab label list` (또는 `glab api projects/:id/labels`)로 현재 라벨 목록을 그대로 조회
2. 표와 다르면 조회된 **정확한 문자열(이모지 포함)을 그대로 복사**해서 `--label`에 사용
3. 조회가 안 되거나(샌드박스 인증 격리 등) 매칭되는 라벨을 못 찾겠으면, 라벨 없이 진행하거나 사용자에게 정확한 라벨 문자열을 물어본다 — 이모지를 추측해서 지어내지 않는다

내용을 보고 이 중 가장 맞는 라벨 하나를 고른다. 애매하면 사용자에게 물어본다.

**담당자**: 대화에서 언급됐으면 채우고, 아니면 요청한 사용자 본인으로 채우거나 비워둔다.

### 2-2. 검수 루프
채운 내용을 그대로 사용자에게 보여준다 ("이런 내용으로 이슈를 만들려고 합니다" 형식). 수정 요청이 오면 반영하고 다시 보여준다. **명시적인 승인("ok", "좋아", "생성해줘", "이대로 만들어줘" 등)을 받기 전까지는 다음 단계로 넘어가지 않는다.**

### 2-3. 이슈 생성
**Codex 자신의 `glab auth status`가 정상(로그인됨)으로 확인되면** 아래 명령을 Codex가 직접 실행한다. 멀티라인 한글 본문은 커맨드라인 인자로 직접 넘기면 따옴표/인코딩 문제가 생기기 쉬우므로, 본문을 임시 파일로 먼저 쓰고 셸 치환으로 넘긴다:

```bash
glab issue create --title "<제목>" --description "$(cat <임시파일경로>)" --label "<라벨>"
```
(라벨이 없으면 `--label` 생략)

**0-1단계에서 샌드박스 인증 격리 증상이 확인됐다면**, 위 명령을 직접 실행하지 말고 사용자가 자기 터미널에 그대로 붙여넣을 수 있는 형태로 안내한다 (PowerShell이면 `$body = @'...'@` here-string 사용). 사용자가 실행 결과(이슈 URL)를 붙여주면 그걸로 다음 단계를 이어간다.

## 3. 브랜치 생성 + 체크아웃 + 원격 푸시

base 브랜치는 도메인이 있으면 `<domain>/develop`(예: `ai/develop`), 공통 작업이면 `develop`이다 — `main`에서 바로 따지 않는다.

```bash
git fetch origin
git checkout -b <브랜치명> origin/<base브랜치>   # 예: origin/ai/develop, 없으면 사용자에게 base가 아직 없다고 알림
git push -u origin <브랜치명>
```
`origin/<base브랜치>`가 없으면(도메인 통합 브랜치가 아직 원격에 없는 경우) 임의로 `main`에서 따지 말고, base 브랜치부터 만들어야 한다는 점을 사용자에게 알린다.

로컬 체크아웃과 원격 푸시까지 해서, 사용자가 바로 커밋을 쌓기 시작할 수 있고 팀원들도 GitLab에서 바로 브랜치를 볼 수 있게 한다.

## 4. 결과 보고
체크아웃된 브랜치명과(만들었다면) 이슈 URL을 사용자에게 알린다. 앞으로 이 브랜치에서 커밋할 때는 `<type>(<Jira키>): <설명>` 형식(예: `feat(S15P11A103-302): UART 드라이버 초기 구현`)을 쓰면 Jira 스마트 커밋 연동이 자동으로 해당 카드에 연결한다는 점도 같이 안내한다. 예:
> `ai/feature/S15P11A103-302-uart-driver` 브랜치(base: `ai/develop`)로 체크아웃 & 푸시 완료. 커밋할 때 `feat(S15P11A103-302): ...` 형식을 쓰면 Jira 카드에 자동으로 연결됩니다.

## 실패 처리
- `glab issue create` 실패(라벨 없음, 권한 없음 등) → 원인 그대로 사용자에게 전달, 임의로 재시도하거나 라벨/제목을 바꿔서 재시도하지 않는다
- 브랜치 생성/푸시 실패 → 이슈까지는 만들어졌다면 그 사실은 명확히 알리고, 무엇이 실패했는지(권한, 충돌 등) 그대로 전달한다
- `glab auth status`가 사용자 터미널에서는 로그인됨으로 뜨는데 Codex가 실행하면 미인증/401로 뜸 → 샌드박스 인증 격리(위 0-1단계 참고). 재로그인 안내나 config 수정을 반복 시도하지 말고, 바로 "사용자 터미널에서 실행" 경로로 전환한다
