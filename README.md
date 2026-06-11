# pkgup

macOS에서 **Homebrew(formula/cask)** 와 **npm 글로벌 패키지** 를 주기적으로 업데이트하고,
**마스킹된 패키지는 확인 후에만** 업데이트하며, 결과를 **Slack** 으로 리포트하는 단일 zsh CLI.

## 특징

- formula · cask · npm 글로벌을 한 번에 업데이트
- 마스킹(hold) 시스템: 버전을 고정하고 싶은 패키지는 자동 업데이트에서 제외하고 확인받아 적용
- TTY 자동 감지로 대화식/비대화식(스케줄) 모드 자동 전환
- 업데이트 결과를 Slack incoming webhook으로 리포트
- 추가 런타임 의존성 없음 (brew / npm / node / curl / coreutils 만 사용)

## 동작 방식

| 실행 상황 | 모드 | 비마스킹 패키지 | 마스킹 패키지 |
|---|---|---|---|
| 터미널에서 `pkgup update` | `interactive` (자동 감지) | 자동 업데이트 | **y/N 확인 후 업데이트** |
| launchd / cron 스케줄 | `auto` (TTY 없음) | 자동 업데이트 | 건드리지 않고 `pending` 으로 리포트 |

cron/launchd는 비대화식이라 그 안에서는 확인을 받을 수 없습니다. 따라서 스케줄 실행은
마스킹 패키지 업데이트를 **보류(pending)** 후 Slack에 알리고, 사용자가 나중에 터미널에서
`pkgup update` 를 직접 실행해 확인/적용하는 흐름입니다.

## 요구사항

- macOS, zsh (Catalina 이후 기본 셸)
- Homebrew (formula/cask 관리 시)
- Node.js + npm (npm 글로벌 관리 시 — npm outdated 파싱에 node 사용)
- npm 글로벌 prefix가 **사용자 쓰기 가능** 위치여야 sudo 없이 동작
  (Homebrew node / nvm / `npm config set prefix ~/.npm-global` 권장)

## 설치

```sh
mkdir -p ~/.local/bin ~/.config/pkgup
install -m 755 pkgup ~/.local/bin/pkgup
cp config.example ~/.config/pkgup/config
$EDITOR ~/.config/pkgup/config        # SLACK_WEBHOOK_URL 등 설정

# PATH에 ~/.local/bin 추가 (.zshrc)
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && exec zsh
```

## 사용법

```sh
pkgup check                       # outdated/masked 목록만 출력 (변경 없음, dry-run)
pkgup update                      # 업데이트 (터미널이면 마스킹 패키지는 확인)
pkgup update --yes                # auto 모드 강제 (마스킹은 확인 없이 보류)
pkgup update --interactive        # interactive 모드 강제

pkgup mask add brew kubectl       # formula 마스킹
pkgup mask add cask docker        # cask 마스킹
pkgup mask add npm  pnpm          # npm 글로벌 마스킹
pkgup mask rm  brew kubectl       # 마스킹 해제
pkgup mask list                   # 마스킹 목록

pkgup schedule install            # 주간 자동 실행 launchd 에이전트 등록
pkgup schedule uninstall          # 에이전트 해제 + plist 삭제
pkgup schedule status             # 등록/로드 상태 확인

pkgup help                        # 도움말
```

## 마스킹 (hold)

마스킹 = "자동 업데이트 금지, 확인받아야 올림". 관리자 종류는 `brew` · `cask` · `npm` 세 가지.
클러스터와 버전 호환을 맞춰야 하는 `kubectl` · `helm` · `terraform` 같은 도구를 마스킹해 두면
자동 루프에서 임의로 올라가지 않고, 터미널에서 직접 확인할 때만 업데이트됩니다.

마스킹 목록은 `~/.config/pkgup/masks` 에 `manager:name` 한 줄씩 저장됩니다.

## 설정

`~/.config/pkgup/config` (zsh로 source 됨):

| 키 | 기본값 | 설명 |
|---|---|---|
| `SLACK_WEBHOOK_URL` | (없음) | Slack incoming webhook. 미설정 시 전송 건너뜀 (리포트는 로그/stdout에만) |
| `NPM_UPDATE_TARGET` | `latest` | `latest` = 최신(메이저 포함) / `wanted` = semver 범위 내 |
| `BREW_CLEANUP` | `false` | 업그레이드 후 `brew cleanup` 실행 |
| `BREW_CASK_GREEDY` | `false` | 자체 업데이트하는 cask까지 검사 (`brew outdated --greedy`) |
| `EXTRA_PATH` | (없음) | 스케줄 실행 시 PATH 보강 (예: nvm 노드 경로) |
| `SCHEDULE_WEEKDAY` | `0` | `schedule install` 실행 요일 (0/7 = 일요일) |
| `SCHEDULE_HOUR` | `10` | `schedule install` 실행 시각(시) |
| `SCHEDULE_MINUTE` | `0` | `schedule install` 실행 시각(분) |
| `PKGUP_LABEL` | `com.user.pkgup` | LaunchAgent 라벨 |

`PKGUP_HOME` 환경변수로 설정/상태 디렉터리 위치(기본 `~/.config/pkgup`)를 바꿀 수 있습니다.

## 주간 자동 실행

### pkgup schedule (권장)

설치한 `pkgup` 으로 launchd 에이전트를 직접 등록/해제합니다. plist를 손으로 만들거나
`launchctl` 을 칠 필요가 없습니다.

```sh
pkgup schedule install      # ~/Library/LaunchAgents/<label>.plist 생성 후 등록
pkgup schedule status       # 등록/로드 상태
pkgup schedule uninstall    # 해제 + plist 삭제

# 등록 직후 즉시 한 번 실행해 보고 싶으면 (install 출력에도 안내됨)
launchctl kickstart -k gui/$(id -u)/com.user.pkgup
```

실행 요일/시각은 config의 `SCHEDULE_WEEKDAY` / `SCHEDULE_HOUR` / `SCHEDULE_MINUTE`
(기본 일요일 10:00). 변경 후 `pkgup schedule install` 을 다시 실행하면 갱신됩니다.

> 에이전트의 PATH에는 `~/.npm-global/bin` 까지 포함됩니다. nvm 등 다른 노드 경로를
> 쓰면 config의 `EXTRA_PATH` 로 보강하세요.

### 수동 / cron (대안)

리포지토리의 `com.user.pkgup.plist` 는 수동 설치용 참고 파일입니다
(`pkgup schedule install` 은 동일한 내용을 자동 생성하므로 보통 필요 없습니다).

cron으로 하려면:

```sh
crontab -e
# 매주 일요일 10:00
0 10 * * 0 /Users/<you>/.local/bin/pkgup update --yes >> ~/Library/Logs/pkgup.cron.log 2>&1
```

macOS에서 cron은 Full Disk Access 권한이 필요할 수 있고, sleep 중 누락 시 보정 실행이
없습니다. launchd(=`pkgup schedule`)는 예약 시각에 잠들어 있었으면 깨어난 직후 한 번
실행해 주므로 권장합니다.

## 파일 위치

- 설정: `~/.config/pkgup/config` (secret 포함 — git 커밋 금지)
- 마스킹 목록: `~/.config/pkgup/masks` (`manager:name` 한 줄씩)
- 실행 로그: `~/.config/pkgup/logs/` (최근 20개 자동 유지)

## 참고 / 제약

- npm 업데이트 기본값은 `latest`(메이저 포함). semver 범위 내로만 올리려면
  config에서 `NPM_UPDATE_TARGET="wanted"`.
- cask는 기본적으로 자체 업데이트하는 앱은 검사에서 제외됩니다. 모두 포함하려면
  `BREW_CASK_GREEDY=true`.
- Slack webhook 미설정 시 전송만 건너뛰고 리포트는 로그/stdout에 남습니다.
- 개별 패키지 업그레이드 실패는 전체를 중단시키지 않고 리포트의 `Failed` 에 모입니다.

## 저장소 구조

```
.
├── CLAUDE.md              # Claude Code용 프로젝트 컨텍스트
├── README.md
├── pkgup                  # 메인 실행 스크립트 (zsh)
├── config.example        # 설정 템플릿
└── com.user.pkgup.plist   # 주간 실행용 launchd LaunchAgent
```
