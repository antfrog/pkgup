# CLAUDE.md — pkgup

macOS에서 Homebrew(formula/cask)와 npm 글로벌 패키지를 주기적으로 업데이트하고,
마스킹된 패키지는 확인 후 업데이트하며, 결과를 Slack으로 리포트하는 단일 zsh CLI.
사용자 대상 문서는 `README.md` 참고.

## 저장소 구조

```
.
├── CLAUDE.md              # 이 파일
├── README.md             # 설치/사용/스케줄 문서 (변경 시 함께 갱신)
├── pkgup                  # 메인 실행 스크립트 (zsh, 유일한 로직)
├── config.example        # 설정 템플릿 (실제 config 는 커밋 금지)
└── com.user.pkgup.plist   # 수동 설치용 참고 plist (schedule install 은 동일 내용을 자동 생성)
```

런타임 설정/상태는 레포가 아니라 `~/.config/pkgup/` 에 위치:
`config`(secret 포함), `masks`(`manager:name` 한 줄씩), `logs/`.

## 개발 명령

```sh
zsh -n pkgup          # 문법 검사 (커밋 전 필수)
pkgup check           # dry-run: outdated/masked 출력, 변경 없음
pkgup update -i       # interactive 강제 (마스킹 확인 프롬프트 동작 검증)
pkgup update --yes    # auto 모드 강제 (스케줄과 동일 경로 검증)
```

- 셸 스크립트라 단위 테스트 하네스는 없음. `zsh -n` → `pkgup check` →
  격리 환경에서 `pkgup update` 순으로 수동 검증.
- shellcheck 은 bash/sh 대상이라 zsh 전용 문법(`${(@P)}`, `${(f)}` 등)에서
  오탐을 냄. shellcheck 지적을 그대로 적용하지 말 것.

## 아키텍처 — 반드시 지킬 불변식

이 스크립트를 수정할 때 아래 계약을 깨지 말 것:

1. **두 가지 실행 모드.** TTY 감지로 `interactive`(터미널) / `auto`(스케줄)를 자동 결정.
   - 비마스킹 패키지: 두 모드 모두 자동 업데이트.
   - 마스킹 패키지: interactive 에서는 y/N 확인 후 업데이트, **auto 에서는 절대
     업데이트하지 않고 `R_pending` 으로 리포트만** 한다. 스케줄 실행이 마스킹
     패키지를 조용히 올리는 일은 없어야 한다.
   - **스케줄러 비의존:** auto 모드는 launchd(plist 제공)든 cron이든 동일하게 동작한다.
     스크립트는 오직 TTY 유무로만 모드를 정하므로 특정 스케줄러를 가정하는 코드를
     넣지 말 것. cron 설정/주의사항(macOS Full Disk Access 등)은 README 참고.
2. **마스킹 단일 소스.** `~/.config/pkgup/masks` 의 `manager:name` 줄이 유일한 진실.
   `brew pin` 등 외부 상태에 의존하지 말 것. 관리자 종류는 `brew|cask|npm` 셋뿐
   (`validate_mgr` 에서 강제).
3. **의존성 최소화.** brew / npm / node / curl / coreutils 외 런타임 의존성 추가 금지.
   - npm outdated 파싱은 jq 가 아니라 node (`npm outdated -g --json | node -e ...`).
   - Slack 페이로드는 순수 zsh `json_escape` 로 만든다. python/jq 도입 금지.
4. **collect-then-loop 패턴.** `update_*` 함수는 process substitution 출력을 먼저
   배열로 모은 뒤 별도 `for` 루프에서 처리한다. `while ...; done < <(...)` 안에서
   직접 처리하면 stdin 이 묶여 interactive `read` 프롬프트가 깨진다. 이 구조 유지.
5. **set -e 가드.** `set -e -u -o pipefail` 사용 중. 실패 가능 명령은 반드시 가드:
   조건문(`if`/`&&`/`||`) 안에 두거나 `cmd || true`, `cmd || rc=$?` 로 처리.
   `brew outdated` / `npm outdated` 는 대상이 있으면 **비0 종료**하므로 항상 `|| true`.
6. **PATH 부트스트랩.** launchd/cron 은 PATH 가 빈약하다. `main()` 의 PATH 보강
   (`/opt/homebrew/bin`, `/usr/local/bin`, `~/.npm-global/bin`) 과 plist 의
   `EnvironmentVariables` 를 유지. nvm 등은 config 의 `EXTRA_PATH` 로 주입.
   - `pkgup schedule install/uninstall/status` 가 launchd 에이전트를
     `~/Library/LaunchAgents/${PKGUP_LABEL}.plist` 로 생성/로드(modern `launchctl
     bootstrap|bootout|enable`, 도메인 `gui/$UID`)한다. `write_agent_plist()` 가
     만드는 plist와 번들 `com.user.pkgup.plist` 는 **내용이 일치해야** 한다 — 한쪽의
     PATH/스케줄/키를 바꾸면 다른 쪽도 갱신할 것. 자기 경로는 `${0:A}` 로 해석한다.
7. **리포트 배열 동기화.** 결과는 `R_updated_brew/cask/npm`, `R_failed`,
   `R_pending`, `R_skipped` 에 누적되어 `build_report` 가 소비한다. 항목을 추가하면
   `build_report` 의 `report_section` 호출도 함께 갱신.

## 코딩 규칙

- 셸은 zsh 전용. bash 호환을 위해 zsh 기능을 제거("port to bash")하지 말 것.
- 코드 주석은 영어, 사용자 노출 문자열/로그는 간결하게. 한·일 등 다국어 문구 허용.
- 서브커맨드나 config 키를 추가하면 `usage()`, `README.md`, `config.example` 을
  같은 PR에서 함께 갱신.

## 하지 말 것

- 실제 `config`(webhook 등 secret 포함)를 커밋하지 말 것. `config.example` 만 추적.
  (`.gitignore` 에 `config` 와 `*.local` 추가 권장.)
- 새 패키지 관리자를 추가할 때 `validate_mgr` 만 고치지 말 것 — `update_*` 디스패치,
  `handle_masked`, `cmd_check` 까지 일관되게 반영해야 한다.
- 마스킹 패키지를 auto 모드에서 업데이트하도록 바꾸지 말 것(불변식 1).
