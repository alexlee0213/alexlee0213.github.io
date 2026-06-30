---
layout: single
title: "Claude Code에서 OAuth 기본 + 필요할 때만 API 키 전환하기 (.zshrc claude-api 함수, cmux 대응)"
date: 2026-06-30 23:24:29 +0900
categories: [dev]
tags: [Claude, Claude Code, zsh, cmux, iTerm2, ANTHROPIC_API_KEY, OAuth, macOS, dotfiles]
excerpt: "평소엔 OAuth 구독 인증으로 claude를 쓰고, 특정 작업에서만 claude-api 명령으로 API 키를 임시 적용하는 .zshrc 설정. iTerm2·일반 zsh·cmux 모두에서 동작하도록 cmux 래퍼/shim까지 우회하는 방법을 절차적으로 정리했다."
---
평소에는 **OAuth 로그인(구독 인증)** 으로 `claude`를 쓰고, 특정 작업에서만 `claude-api` 명령으로 **`ANTHROPIC_API_KEY`(API 키 인증)** 를 임시 적용하는 방법을 절차적으로 정리한다.
iTerm2 · 일반 zsh · cmux(터미널 멀티플렉서) 모두에서 동일하게 동작한다.

> 이 글 하나로 **설정(how)** 과 **원리·원인 분석(why)** 을 모두 다룬다. (8·9장에 cmux 관련 원인·사이드 이펙트 포함)

---

## 0. 왜 이렇게 쓰나 (동작 원리)

Claude Code의 인증 우선순위:

- 환경변수 **`ANTHROPIC_API_KEY`가 설정돼 있으면** → 그 키로 인증/과금 (API Console 사용량으로 청구)
- 설정돼 있지 **않으면** → `claude login`으로 저장된 **OAuth 구독 인증** 사용

따라서 "기본은 OAuth, 가끔 API 키"를 안전하게 구현하려면:

1. 키를 **셸 환경에 상시 export하지 않는다** (그러면 모든 `claude`가 API 과금됨)
2. 대신 `claude-api`라는 **별도 명령**을 만들어, 그 명령을 실행한 그 순간/그 프로세스에만 키를 주입하고 끝나면 자동으로 사라지게 한다

```text
claude       → 환경에 키 없음 → OAuth 구독 인증
claude-api   → 서브셸에서만 ANTHROPIC_API_KEY 주입 → API 키 인증 (종료 시 소멸)
```

---

## 1. 사전 요구사항 확인

```zsh
# 셸이 zsh인지 확인 (macOS 기본)
echo $SHELL            # → /bin/zsh 또는 /usr/bin/zsh

# Claude Code 설치 및 경로 확인
claude --version
which claude           # 예: /opt/homebrew/bin/claude
```

> `which claude`가 `/var/folders/.../cmux-cli-shims/.../claude` 처럼 나오면 cmux 환경이다. 이 가이드의 함수는 그 경우도 자동 처리하므로 그대로 진행하면 된다. (원인은 8장 참고)

---

## 2. OAuth 로그인(기본 인증) 설정

평소 사용할 구독 인증을 먼저 로그인해 둔다.

```zsh
claude login          # 브라우저로 OAuth 로그인 (Pro/Max 구독 계정)
```

로그인 후 확인:

```zsh
claude                # 실행 → /status 입력 → 인증 방식이 구독(OAuth)으로 표시되는지 확인
```

이 상태에서 `ANTHROPIC_API_KEY`만 없으면 항상 구독 인증으로 동작한다.

---

## 3. API 키 발급 및 안전하게 보관

### 3-1. API 키 발급
[Anthropic Console](https://console.anthropic.com/) → **API Keys** 에서 키 생성 (`sk-ant-...`).

### 3-2. 키 파일 생성 (권한 제한)

키를 `.zshrc`에 직접 넣지 않고 **별도 파일**에 보관한다. (셸 히스토리·설정 백업에 키가 노출되지 않도록)

```zsh
# 비밀 디렉터리 생성 (소유자만 접근)
mkdir -p ~/.secrets && chmod 700 ~/.secrets

# 키 저장 (sk-ant-... 부분을 실제 키로 교체)
echo 'sk-ant-...' > ~/.secrets/anthropic_key

# 파일 권한 제한 (소유자 읽기/쓰기만)
chmod 600 ~/.secrets/anthropic_key
```

### 3-3. 저장 확인

```zsh
ls -l ~/.secrets/anthropic_key     # → -rw------- (600) 확인
wc -c  ~/.secrets/anthropic_key    # 키 길이(바이트) 확인, 0이 아니어야 함
```

> ⚠️ 키 끝에 줄바꿈/공백이 들어가지 않도록 주의. `echo`로 넣으면 끝에 개행이 하나 붙지만, 함수가 `$(<file)`로 읽을 때 zsh가 마지막 개행을 제거하므로 문제없다.

---

## 4. `.zshrc`에 `claude-api` 함수 추가

`~/.zshrc` 하단에 아래 블록을 추가한다 (편집기로 열거나 그대로 붙여넣기).

```zsh
# ===== Claude Code: API 키로 임시 실행 =====
# 평상시 `claude`는 OAuth 구독 인증, `claude-api`는 API 키로 실행 후 자동 해제
claude-api() {
  local key_file="$HOME/.secrets/anthropic_key"
  local key

  # 1) 키 파일 존재 확인
  if [[ ! -f "$key_file" ]]; then
    echo "❌ 키 파일이 없습니다: $key_file" >&2
    echo "   mkdir -p ~/.secrets && chmod 700 ~/.secrets" >&2
    echo "   echo 'sk-ant-...' > ~/.secrets/anthropic_key && chmod 600 ~/.secrets/anthropic_key" >&2
    return 1
  fi

  # 2) 키 내용 읽기 + 빈 값 확인
  key="$(<"$key_file")"
  if [[ -z "$key" ]]; then
    echo "❌ 키 파일이 비어 있습니다: $key_file" >&2
    return 1
  fi

  # 3) 진짜 claude 바이너리 찾기 (cmux 래퍼/shim 우회)
  #    cmux는 `claude`를 함수/shim으로 가로채 앱 데몬이 대신 실행하므로
  #    서브셸에서 export한 키가 전달되지 않는다. PATH의 shim을 제외하고
  #    실제 바이너리를 직접 실행한다.
  #    (cmux가 없는 iTerm2/일반 zsh에서는 매칭이 없어 평소대로 동작)
  local real_claude
  real_claude="$(whence -ap claude | grep -v 'cmux-cli-shims' | head -1)"
  if [[ ! -x "$real_claude" ]]; then
    echo "❌ 실제 claude 바이너리를 찾지 못했습니다 (PATH 확인 필요)" >&2
    return 1
  fi

  # 4) 서브셸에서 실행 → 현재 셸 환경 오염 없이, 종료 시 키 자동 소멸
  (
    export ANTHROPIC_API_KEY="$key"
    # 세션 내부에서 claude를 재호출해도 cmux shim에 안 걸리도록 PATH에서 제거 (없으면 무해)
    path=("${(@)path:#*cmux-cli-shims*}")
    "$real_claude" "$@"
  )
}
```

### 4-1. 함수 동작 요약 (단계별)

| 단계 | 내용 |
|------|------|
| 1 | `~/.secrets/anthropic_key` 존재 확인, 없으면 안내 후 종료 |
| 2 | 파일에서 키 읽기, 빈 값이면 종료 |
| 3 | `whence -ap`로 PATH의 실제 바이너리만 나열 → cmux shim 제외 → 진짜 `claude` 선택 |
| 4 | 서브셸에서만 키 export 후 실제 바이너리 실행 (격리 + 자동 소멸) |

### 4-2. 핵심 셸 구문별 역할

| 구문 | 역할 |
|------|------|
| `whence -ap claude` | PATH의 **실제 바이너리만** 나열 (함수/alias 무시) |
| `grep -v 'cmux-cli-shims'` | cmux shim 경로 제외 → 진짜 바이너리 선택 |
| `head -1` | 첫 후보 선택 (`/opt/homebrew/bin/claude` 등) |
| `( ... )` 서브셸 | 키·PATH 변경을 격리, 명령 종료 시 자동 소멸 |
| `export ANTHROPIC_API_KEY="$key"` | 이 서브셸(그리고 그 자식 claude)에만 키 주입 |
| `path=("${(@)path:#*cmux-cli-shims*}")` | 서브셸 PATH에서 shim 디렉터리 제거 (중첩 호출 대비, 없으면 무해) |
| `"$real_claude" "$@"` | 절대경로로 진짜 바이너리 직접 실행, 인자 그대로 전달 |

> 경로를 하드코딩하지 않고 PATH에서 찾으므로 Intel맥(`/usr/local/bin`)·버전 변경에도 자동 대응한다.

---

## 5. 적용 및 검증

### 5-1. 문법 검사 후 반영

```zsh
zsh -n ~/.zshrc        # 문법 오류 없는지 먼저 확인
source ~/.zshrc        # 새 함수 정의를 현재 셸에 반영 (또는 새 터미널 열기)
```

### 5-2. 함수가 잡혔는지 확인

```zsh
type claude-api        # → "claude-api is a shell function from ~/.zshrc"
```

### 5-3. 실제 인증 전환 확인

```zsh
# (A) 평소 — OAuth 구독 인증
claude
#   → /status 입력 → 구독/OAuth 로 표시

# (B) API 키 인증
claude-api
#   → /status 입력 → API 키 인증으로 표시
```

`claude-api` 종료 후 부모 셸에 키가 남지 않는지 확인:

```zsh
echo "${ANTHROPIC_API_KEY:-(unset)}"   # → (unset)  ← 정상 (서브셸에서만 존재했으므로)
```

---

## 6. 사용법

```zsh
# 평소 작업 (구독 인증)
claude

# 이번 작업만 API 키로 (인자도 그대로 전달됨)
claude-api
claude-api --help
claude-api -p "이 코드 리뷰해줘"
```

`claude-api`에 넘긴 모든 인자(`"$@"`)는 실제 `claude`로 그대로 전달된다.

---

## 7. 환경별 동작 (iTerm2 / 일반 zsh / cmux)

| 환경 | `claude-api` | 평소 `claude` |
|------|--------------|----------------|
| iTerm2 / 일반 zsh | shim 없음 → 진짜 바이너리 직행, 키 상속 ✅ | OAuth 구독 |
| cmux | shim 우회 → 진짜 바이너리 직접 실행, 키 상속 ✅ | OAuth 구독 (cmux 통합 유지) |

환경을 분기하지 않고 한 함수로 모두 커버한다. cmux가 있으면 우회하고, 없으면 우회 로직이 매칭되지 않아 무해하다.

---

## 8. cmux에서 키가 안 먹혔던 이유 (원인 분석)

3장의 단순 버전 함수(아래)는 iTerm2에서는 되는데 cmux에서는 키가 적용되지 않았다. 그 원인이 4장에서 `whence`/shim 우회 로직을 넣은 이유다.

```zsh
# (문제가 있던 단순 버전)
claude-api() {
  key="$(<~/.secrets/anthropic_key)"
  ( export ANTHROPIC_API_KEY="$key"; claude "$@" )   # 서브셸 안에서만 키 유효
}
```

- **iTerm2**: `claude` → `/opt/homebrew/bin/claude` 직행. 서브셸의 자식 프로세스라 `ANTHROPIC_API_KEY` 상속 → ✅
- **cmux**: cmux가 `claude`를 **래퍼 함수 + shim**으로 가로챈다.

```text
claude () { _cmux_claude_wrapper_command "$@" }
  → /Applications/cmux.app/.../cmux-claude-wrapper   (shim, PATH 맨 앞)
  → cmux.app(데몬)이 claude를 대신 spawn
```

claude를 실제로 띄우는 주체가 **현재 셸이 아니라 cmux.app**이고, 앱은 기동 시점의 환경(키 없음)으로 claude를 실행한다. 그래서 서브셸에서 export한 키가 claude까지 도달하지 못한다 → ❌

### 8-1. 진단 근거 (실제 확인한 출력)

cmux pane에서 확인한 내용:

```zsh
type claude
# claude is a shell function
# claude () { _cmux_claude_wrapper_command "$@" }

which -a claude
# /var/folders/.../cmux-cli-shims/.../claude   ← shim (PATH 맨 앞)
# /opt/homebrew/bin/claude                     ← 진짜 바이너리
```

→ `claude`가 함수로 가로채지고, shim이 진짜 바이너리보다 앞에 잡힌다는 게 확인된다.

### 8-2. cmux shim 내부 동작

cmux shim(`$CMUX_CLAUDE_WRAPPER_SHIM`)은 `exec`로 환경을 넘기지만, 그다음 `cmux-claude-wrapper` 바이너리가 cmux.app 데몬에 실행을 위임하면서 셸 환경과의 연결이 끊긴다.

흥미롭게도 shim의 **fallback 경로**(래퍼 바이너리가 없을 때)는 오히려 PATH에서 shim을 제거하고 `exec claude` 하도록 되어 있다. 4장의 우회안은 이 fallback과 같은 아이디어(= **shim 제거 + 진짜 바이너리 직접 실행**)를 항상 적용한 것이다.

---

## 9. 사이드 이펙트

- `claude-api`로 띄운 claude는 **cmux의 claude 전용 통합 밖에서** 돈다.
  - cmux UI의 세션 추적·상태 표시·알림 등이 그 세션엔 안 붙을 수 있음.
  - 단, **claude 본체 기능(MCP·슬래시 명령·서브에이전트·`~/.claude` 설정)과 멀티플렉서 자체(split/창/스크롤백)는 전부 정상.**
- 평소 `claude`(구독 인증)는 손대지 않으므로 cmux 통합이 그대로 유지된다.
- cmux의 claude 통합을 적극적으로 쓴다면, 우회 대신 cmux.app 환경에 키를 넣는 방식도 있으나 **키 상시 노출** 트레이드오프가 있어 비권장.
- 세션 내부에서 다시 `claude`를 호출하는 경우에도 4장 함수의 `path=(... :#*cmux-cli-shims*)` 덕분에 shim에 재차 걸리지 않는다.

---

## 10. 트러블슈팅

| 증상 | 원인 / 해결 |
|------|-------------|
| `claude-api: command not found` | `.zshrc`가 안 읽힘. `source ~/.zshrc` 또는 새 터미널. interactive 셸인지 `echo $-`로 `i` 확인 |
| `❌ 키 파일이 없습니다` | 3장 키 파일 생성 안 됨 |
| `❌ 키 파일이 비어 있습니다` | 키 파일에 실제 키가 안 들어감 (`cat ~/.secrets/anthropic_key`로 확인) |
| `claude-api`인데 여전히 구독 인증 | cmux 환경에서 우회가 안 됨 → `whence -ap claude` 결과에 shim이 섞여 있는지, 함수에 우회 로직(4장 3단계)이 포함됐는지 확인 |
| 인증 오류(401 등) | 키가 만료/오타. Console에서 키 재발급 후 파일 갱신 |

---

## 11. 보안 주의사항

- 키는 **`.zshrc`가 아니라 `~/.secrets/anthropic_key`(권한 600)** 에만 둔다. 설정 파일을 dotfiles 레포에 올려도 키가 노출되지 않는다.
- `~/.secrets`는 git에 커밋하지 않는다 (`.gitignore`에 `.secrets/` 추가 권장).
- 키를 셸 환경에 상시 export하지 않는다 → 평소 `claude`가 의도치 않게 API 과금되는 것을 방지.
- `claude-api`는 서브셸에서만 키를 노출하므로, 명령 종료 즉시 키가 메모리에서 사라진다.
- 키 폐기가 필요하면 Console에서 해당 키를 revoke 후 `~/.secrets/anthropic_key`만 교체하면 된다.

---

## 부록: 키 파일 대신 1Password/직접 입력으로 받기 (선택)

파일 보관이 싫으면 3장 대신 함수 안에서 동적으로 받아올 수도 있다. 예: 1Password CLI

```zsh
# key="$(<"$key_file")" 대신
key="$(op read 'op://Private/Anthropic API/credential')"
```

또는 매번 직접 입력(가장 안전, 가장 번거로움):

```zsh
read -rs "key?ANTHROPIC_API_KEY: "; echo
```
