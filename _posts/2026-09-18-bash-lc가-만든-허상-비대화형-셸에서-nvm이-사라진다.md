---
layout: single
title: "bash -lc가 만든 허상 — 비대화형 셸에서 nvm이 사라진다"
date: 2026-09-18 11:30:00 +0900
categories: [dev]
tags: [bash, nvm, SSH, 셸, PATH, 트러블슈팅]
excerpt: "원격 서버 점검을 AI 에이전트에게 맡겼더니 "서버 5개가 npx를 못 찾아 죽어 있다"는 보고가 왔다. 전부 멀쩡했다. 범인은 에이전트가 점검에 쓴 bash -lc였다. 로그인 셸과 대화형 셸은 다른 축이고, Ubuntu의 .bashrc는 비대화형이면 첫 줄에서 돌아선다."
---
> 이 글은 **자격증명 신뢰 경계 정리 시리즈**의 관련 독립 글이다.
> - [1편: 평문 SSH 키 하나가 맥과 GitHub을 동시에 열고 있었다](/dev/2026/09/18/평문-ssh-키-하나가-맥과-github을-동시에-열고-있었다/)
> - [2편: 노트북의 원격 로그인, 꺼도 되나](/dev/2026/09/18/노트북의-원격-로그인-꺼도-되나-인바운드와-아웃바운드를-가르는-기준/)
> - [3편: 키를 지우고도 git은 쓴다 — SSH 에이전트 포워딩과 tmux의 죽은 소켓](/dev/2026/09/18/키를-지우고도-git은-쓴다-ssh-에이전트-포워딩과-tmux의-죽은-소켓/)

## 결론부터

**로그인 셸과 대화형 셸은 서로 다른 축이다.** `bash -lc`는 로그인이지만 비대화형이고, Ubuntu 기본 `~/.bashrc`는 비대화형이면 맨 앞에서 `return` 한다. nvm·pyenv·rbenv처럼 `.bashrc`에서 PATH를 세우는 도구들은 그 지점에서 통째로 사라진다.

| 호출 방식 | 로그인 | 대화형 | `~/.profile` | `~/.bashrc` 본문 |
|---|:---:|:---:|:---:|:---:|
| `ssh host 'cmd'` | ✗ | ✗ | ✗ | ✗ |
| `bash -lc 'cmd'` | ✓ | ✗ | ✓ | **조기 중단** |
| `bash -ic 'cmd'` | ✗ | ✓ | ✗ | ✓ |
| `ssh host` 후 타이핑 | ✓ | ✓ | ✓ | ✓ |

그래서 원격 점검의 제1 원칙은 이것이다.

> **사용자가 실제로 쓰는 경로와 같은 셸로 확인하라.** 다른 셸에서 본 결과는 그 셸에 대한 사실일 뿐이다.

---

## 증상: 서버 절반이 죽어 있다

원격 서버의 설정을 손본 김에 AI 에이전트에게 전체 상태 점검을 시켰다. 결과가 험악했다.

```
$ ssh remote-server 'bash -lc "some-tool list"'
...
service-a: npx -y @vendor/a - ✘ Failed to connect — ENOENT: Executable not found in $PATH: "npx"
service-b: npx @vendor/b@latest - ✘ Failed to connect — ENOENT: Executable not found in $PATH: "npx"
service-c: npx -y @vendor/c - ✘ Failed to connect — ENOENT: Executable not found in $PATH: "npx"
service-d: npx -y @vendor/d - ✘ Failed to connect — ENOENT
service-e: npx -y @vendor/e - ✘ Failed to connect — ENOENT
```

다섯 개가 같은 이유로 죽어 있었다. `npx`가 PATH에 없다.

에이전트는 **"이번 작업과 무관한 기존 문제를 발견했다"고 보고했다.** 비대화형 셸에서 node/nvm PATH가 안 잡히는 전형적인 증상이라며, 고칠까요 하고 물었다. 나는 고쳐달라고 했다.

**전부 틀린 보고였다.**

---

## 오진: PATH를 고치려 들기

고치러 들어간 에이전트가 상태를 확인했다.

```
$ ssh remote-server 'bash -lc "command -v node npx; node -v"'
bash: line 1: node: command not found

$ ssh remote-server 'ls -d ~/.nvm/versions/node/*'
/home/user/.nvm/versions/node/v22.19.0
/home/user/.nvm/versions/node/v24.15.0
```

node는 설치돼 있는데 로그인 셸에서 안 보인다. 여기서 "역시 PATH 문제"라는 확신이 굳어졌고 해결책이 세 개쯤 나왔다 — 절대 경로로 바꾸기, 설정 파일에 `env.PATH` 박기, 심볼릭 링크 만들기.

다행히 고치기 전에 **어디서 로드되는지**를 먼저 확인했다.

```
$ ssh remote-server 'grep -n "nvm\|NVM" ~/.bashrc ~/.profile ~/.bash_profile 2>/dev/null'
/home/user/.bashrc:155:export NVM_DIR="$HOME/.nvm"
/home/user/.bashrc:156:[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
/home/user/.bashrc:157:[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

`.bashrc`의 155번째 줄. 그리고 Ubuntu 기본 `.bashrc`의 맨 앞은 이렇게 생겼다.

```bash
# If not running interactively, don't do anything
case $- in
    *i*) ;;
      *) return;;
esac
```

**비대화형이면 여기서 돌아선다.** 155번째 줄까지 갈 일이 없다.

---

## 진짜 원인: 점검에 쓴 셸이 문제였다

대화형 셸로 같은 걸 물어보면 답이 달라진다.

```
$ ssh remote-server 'bash -ic "command -v node npx; node -v"'
/home/user/.nvm/versions/node/v24.15.0/bin/node
/home/user/.nvm/versions/node/v24.15.0/bin/npx
v24.15.0
```

있다. 그러면 원래 점검도 대화형으로 돌려봐야 한다.

```
$ ssh remote-server 'bash -ic "some-tool list"'
service-a: npx -y @vendor/a - ✔ Connected
service-b: npx @vendor/b@latest - ✔ Connected
service-c: npx -y @vendor/c - ✔ Connected
service-d: npx -y @vendor/d - ✔ Connected
service-e: npx -y @vendor/e - ✔ Connected
```

**전부 정상이었다.** 나는 터미널에서 대화형 셸로 이 도구를 쓴다. 그 경로에서는 처음부터 아무 문제가 없었다. 고장 난 것은 서버가 아니라 **점검 방법**이었다.

---

## 로그인과 대화형은 다른 축이다

이 오진의 뿌리는 두 개념을 하나로 뭉뚱그린 데 있다.

- **로그인 셸(`-l`)**: `/etc/profile`과 `~/.profile`(또는 `~/.bash_profile`)을 읽는다. "로그인 절차를 밟는 셸"이다.
- **대화형 셸(`-i`)**: `~/.bashrc`를 읽는다. "사람이 타이핑하는 셸"이다.

둘은 독립적이라 네 가지 조합이 전부 존재한다. `bash -lc`는 그중 **로그인이면서 비대화형**이라는, 실사용에서는 잘 안 나오는 조합이다.

혼동을 키우는 건 Ubuntu의 `~/.profile`이 안에서 `~/.bashrc`를 불러준다는 점이다.

```bash
# ~/.profile
if [ -n "$BASH_VERSION" ]; then
    if [ -f "$HOME/.bashrc" ]; then
        . "$HOME/.bashrc"
    fi
fi
```

그래서 "로그인 셸이니까 `.bashrc`도 읽히겠지"라고 생각하게 되는데, **읽히기는 하지만 첫 줄에서 되돌아 나온다.** 파일을 여는 것과 내용을 실행하는 것은 다르다.

---

## 왜 nvm이 특히 잘 걸리나

`nvm`은 PATH를 셸 함수로 관리하는 도구고, 설치 스크립트가 초기화 코드를 **`.bashrc` 끝에** 붙인다. 즉 구조적으로 "대화형 셸에서만 존재하는 node"가 된다.

같은 방식으로 PATH를 세우는 도구는 전부 같은 성질을 갖는다 — `pyenv`, `rbenv`, `sdkman`, 대부분의 버전 매니저가 여기 해당한다.

반대로 `/usr/local/bin`처럼 시스템 PATH에 실체가 있는 도구는 어느 셸에서든 보인다. **"이 명령이 어디서 오는가"에 따라 재현 조건이 달라진다.**

---

## 같은 뿌리, 다른 증상

이 함정은 한 번 밟고 끝나지 않는다. 같은 작업에서 형태를 바꿔 또 나왔다.

SSH 에이전트 포워딩을 쓰려고 서버 `.bashrc`에 소켓 경로를 고정하는 코드를 넣었는데,

```bash
# ~/.bashrc
ln -sf "$SSH_AUTH_SOCK" "$HOME/.ssh/agent.sock"
export SSH_AUTH_SOCK="$HOME/.ssh/agent.sock"
```

이걸 `ssh host 'command'`로 검증하니 **인증이 실패했다.** 비대화형이라 `.bashrc`가 안 돌았고, 링크가 갱신되지 않아 이전 세션의 죽은 소켓을 가리키고 있었던 것이다. 설정은 멀쩡했다.

```
$ ssh remote-server 'ls -l ~/.ssh/agent.sock'
... agent.sock -> /tmp/ssh-OLD/agent.12345     # 이미 사라진 세션의 소켓

$ ssh remote-server 'bash -ic "true"; ls -l ~/.ssh/agent.sock'
... agent.sock -> /tmp/ssh-NEW/agent.67890     # 갱신됨
```

자세한 구성은 [3편](/dev/2026/09/18/키를-지우고도-git은-쓴다-ssh-에이전트-포워딩과-tmux의-죽은-소켓/)에 적었다. 요점은 **`.bashrc`에 의존하는 모든 것은 비대화형 점검에서 거짓 실패를 만든다**는 것이다.

---

## 판별 커맨드 세트

```bash
# 1. 이 명령이 대화형에서만 보이는가
ssh host 'command -v <명령>'             # 비대화형
ssh host 'bash -ic "command -v <명령>"'  # 대화형
# → 앞은 실패, 뒤는 성공이면 .bashrc 의존

# 2. 어디서 PATH가 세워지는가
ssh host 'grep -n "<도구명>" ~/.bashrc ~/.profile ~/.bash_profile 2>/dev/null'

# 3. .bashrc가 비대화형에서 돌아서는지 (Ubuntu 기본값)
ssh host 'head -10 ~/.bashrc'

# 4. 현재 셸이 대화형인지 ($- 에 i 가 있는가)
echo $-
```

4번이 제일 빠른 자가 진단이다. `himBHs`처럼 `i`가 있으면 대화형, `hBc`처럼 없으면 아니다.

---

## 교훈

**진단 도구가 재현 조건을 바꾸면, 보이는 것은 시스템이 아니라 도구다.**

원격 점검은 편의상 `ssh host 'cmd'`나 `bash -lc`로 한 방에 끝내고 싶어진다. 사람이든 에이전트든 마찬가지다. 그런데 정작 그 도구를 쓰는 사람은 터미널에 앉아 대화형 셸을 쓴다. 두 환경이 다르면 **점검자가 본 고장은 사용자에게 존재하지 않는 고장**이다.

실제 비용도 있었다. 에이전트는 없는 문제를 보고했고, 나는 고치라고 승인했다. 그대로 갔으면 멀쩡한 설정에 불필요한 `env.PATH` 하드코딩이 박히고 "고쳤다"는 보고가 돌아왔을 것이다. **없는 문제를 고치면 진짜 부작용이 남는다.**

에이전트에게 원격 점검을 맡길 때 특히 조심할 이유가 여기 있다. 에이전트는 비대화형으로 명령을 실행하는 것이 기본값이라 이 함정에 구조적으로 더 자주 빠지고, 그 결과를 자신 있게 보고한다. 승인하기 전에 **"어떤 셸로 확인했나"** 한 번만 되물으면 대부분 걸러진다.

---

## 체크리스트

- [ ] 원격 점검 결과가 나쁘면 **대화형 셸(`bash -ic`)로 재확인**
- [ ] `-l`(로그인)과 `-i`(대화형)를 같은 것으로 취급하지 않기
- [ ] 문제의 명령이 `.bashrc` 의존인지 확인 (`grep -n` 한 줄)
- [ ] Ubuntu `.bashrc`의 비대화형 조기 `return` 인지
- [ ] 고치기 전에 "사용자의 실제 경로에서도 깨지나" 확인
- [ ] `.bashrc`에 의존하는 설정은 비대화형 검증에서 거짓 실패한다는 것 기억
- [ ] 에이전트가 원격 점검 결과를 보고하면 **어떤 셸로 확인했는지** 되묻기

---

### 참고

- [Bash Reference Manual — Bash Startup Files](https://www.gnu.org/software/bash/manual/bash.html#Bash-Startup-Files)
- [Bash Reference Manual — Is this Shell Interactive?](https://www.gnu.org/software/bash/manual/bash.html#Is-this-Shell-Interactive_003f)
- [nvm — Installing and Updating](https://github.com/nvm-sh/nvm#installing-and-updating)
