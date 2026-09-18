---
layout: single
title: "키를 지우고도 git은 쓴다 — SSH 에이전트 포워딩과 tmux의 죽은 소켓"
date: 2026-09-18 11:00:00 +0900
categories: [dev]
tags: [SSH, ForwardAgent, tmux, git, ControlMaster, 보안]
excerpt: "공용 서버에 키를 두지 않고도 git을 쓰는 방법은 에이전트 포워딩이다. 그런데 tmux는 SSH 세션보다 오래 살기 때문에, 판 안의 프로세스가 이미 죽은 소켓을 붙들고 조용히 인증에 실패한다. 고정 경로 한 겹으로 해결된다."
---
> **자격증명 신뢰 경계 정리 시리즈**
> 1. [평문 SSH 키 하나가 맥과 GitHub을 동시에 열고 있었다](/dev/2026/09/18/평문-ssh-키-하나가-맥과-github을-동시에-열고-있었다/)
> 2. [노트북의 원격 로그인, 꺼도 되나 — 인바운드와 아웃바운드를 가르는 기준](/dev/2026/09/18/노트북의-원격-로그인-꺼도-되나-인바운드와-아웃바운드를-가르는-기준/)
> 3. **키를 지우고도 git은 쓴다 — SSH 에이전트 포워딩과 tmux의 죽은 소켓** ← 현재 글
>
> 관련 독립 글
> - [`bash -lc`가 만든 허상 — 비대화형 셸에서 nvm이 사라진다](/dev/2026/09/18/bash-lc가-만든-허상-비대화형-셸에서-nvm이-사라진다/)

## 결론부터

**에이전트 포워딩을 쓰면 서버 디스크에 자격증명이 하나도 남지 않는다.** 서버의 git은 접속해 있는 동안 내 노트북의 키를 빌려 쓴다.

다만 tmux를 쓴다면 한 줄이 더 필요하다.

```bash
# 서버의 ~/.bashrc
if [ -n "$SSH_AUTH_SOCK" ] && [ "$SSH_AUTH_SOCK" != "$HOME/.ssh/agent.sock" ]; then
  ln -sf "$SSH_AUTH_SOCK" "$HOME/.ssh/agent.sock"
fi
export SSH_AUTH_SOCK="$HOME/.ssh/agent.sock"
```

이게 없으면 tmux 안에서 며칠째 돌던 프로세스가 **이미 사라진 소켓을 붙들고 조용히 인증에 실패한다.**

증상이 나오면 진단은 한 줄이다.

```bash
ls -l ~/.ssh/agent.sock    # 가리키는 대상이 살아 있는가
```

---

## 문제 정의

[1편](/dev/2026/09/18/평문-ssh-키-하나가-맥과-github을-동시에-열고-있었다/)에서 공용 서버의 개인키를 폐기했다. 그런데 그 서버에서는 실제로 개발을 한다 — private repo 여러 개에 매일 커밋이 올라간다. 인증 수단이 없으면 일이 안 된다.

조건은 이렇다.

- 서버에 다른 관리자 두 명이 root를 갖고 있다 → **디스크에 저장하는 것은 전부 읽힌다고 가정**
- git 작업은 전부 내가 SSH로 접속해 있는 동안 일어난다 (크론잡·무인 작업 없음)
- repo가 여러 개고 조직도 섞여 있다

세 가지 선택지를 놓고 비교했다.

| | 에이전트 포워딩 | deploy key | fine-grained PAT |
|---|---|---|---|
| 서버에 저장 | **없음** | 개인키 | 토큰 |
| 적용 범위 | 계정 전체 | repo 1개씩 | 지정한 repo |
| 끊긴 동안 동작 | 안 됨 | 됨 | 됨 |
| 탈취 시 피해 | 붙어 있는 동안만 | 그 repo 하나 | 만료일까지, 지정 repo |
| 폐기 | 에이전트에서 내림 | 키 삭제 | 클릭 한 번 |

**무인 작업이 없다는 사실이 결정적이었다.** repo가 여러 개라 deploy key는 관리가 번거롭고, PAT는 저장이 필요하다. 저장하지 않는 선택지가 있는데 굳이 저장할 이유가 없다.

---

## 구성

### 1. GitHub 전용 키를 따로 만든다

그동안 키 하나로 GitHub 인증과 서버 로그인을 겸하고 있었다. 그대로 포워딩하면 **탈취 시 GitHub과 서버 접속이 동시에 열린다.** 용도를 나누면 최악의 경우도 GitHub 하나로 묶인다.

```bash
ssh-keygen -t ed25519 -C "github-only (laptop, agent-forward)" -f ~/.ssh/id_github
gh ssh-key add ~/.ssh/id_github.pub --title "laptop github-only"
```

```
# ~/.ssh/config
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_github
  IdentitiesOnly yes          # 다른 키를 제시하지 않게 못 박는다
```

`IdentitiesOnly yes`가 중요하다. 없으면 ssh가 기본 이름의 키들을 순서대로 제시해서, 용도 분리가 실질적으로 무너진다.

작동을 확인한 뒤 옛 키는 GitHub에서 내린다. 이제 **GitHub = `id_github`, 서버 로그인 = 별도 키**로 완전히 갈린다.

```bash
$ ssh -v -T git@github.com 2>&1 | grep -i 'Server accepts key'
debug1: Server accepts key: /Users/user/.ssh/id_github ED25519 SHA256:cccc…C3
```

### 2. 포워딩을 켠다 — 대상 호스트에만

```
# ~/.ssh/config
Host shared-server
  ForwardAgent yes
  ...
```

**`Host *`에 걸면 안 된다.** 에이전트를 넘긴다는 건 그 기계의 root에게 "내 키로 서명해줄게"라고 약속하는 것이다. 신뢰를 판단한 호스트에만 준다.

### 3. 별칭이 여러 개면 전부 설정한다

이게 놓치기 쉬운 지점이다. 같은 기계에 공인 IP용·VPN용 별칭을 따로 두는 구성이 흔한데,

```
Host shared-server          # 공인 IP 경유
Host shared-server-vpn      # VPN 경유
```

한쪽에만 `ForwardAgent yes`를 넣으면 **평소 쓰는 쪽이 다른 별칭일 때 아무 일도 일어나지 않는다.** 실제로 어느 쪽을 쓰는지는 마스터 연결을 보면 안다.

```bash
$ for h in shared-server shared-server-vpn; do printf '%-20s ' "$h"; ssh -O check "$h" 2>&1; done
shared-server         Control socket connect(...): No such file or directory
shared-server-vpn     Master running (pid=52431)      # ← 실제로 쓰는 쪽
```

같은 기계의 두 경로이므로 신뢰 판단도 동일하다. 둘 다 켜는 게 맞다.

---

## 개인키는 소켓을 건너오지 않는다

구성을 끝낸 뒤, 서버 쪽에서 스스로 점검해봤다. 이 머신에는 인증에 쓸 만한 것이 정말로 하나도 없다.

```bash
$ ls ~/.ssh/id_* 2>/dev/null
(없음)

$ git config --get credential.helper
(미설정)

$ env | grep -E 'GH_TOKEN|GITHUB_TOKEN'
(없음)
```

그런데 `git push`가 된다. 연결을 따라가면 이렇게 생겼다.

```
SSH_AUTH_SOCK=/home/user/.ssh/agent.sock
  → symlink → /tmp/ssh-XXXXXX/agent.NNNN        (포워딩된 에이전트 소켓)
      → ssh-add -l
        256 SHA256:cccc…C3 github-only (laptop, agent-forward) (ED25519)
```

remote가 `git@github.com:...` 형태의 SSH이므로, git은 인증할 때 **로컬 키 파일을 읽지 않고 소켓 너머에 서명을 요청한다.**

1. GitHub이 챌린지를 보낸다
2. 서버의 `ssh`가 소켓으로 "이것을 서명해달라"를 전달한다
3. 노트북의 에이전트가 키로 서명한 뒤 **서명값만** 돌려준다
4. 서버는 그 서명값을 GitHub에 전달한다

**개인키는 이 왕복의 어느 구간에도 등장하지 않는다.** 서버가 하는 일은 서명 연산을 원격에 위탁하는 것뿐이다. 그래서 디스크에 자격증명이 없는 채로 인증이 성립한다.

이 구조가 주는 실질적 차이는 폐기 방식에서 드러난다. 키를 서버에 두는 방식이었다면 유출 여부를 알 수 없으니 "이미 복사됐다"고 가정하고 키 자체를 갈아야 한다. 에이전트 방식에서는 **노트북에서 `ssh-add -D` 한 번이면 그 순간부터 서버는 아무것도 못 한다.**

---

## 함정 1: tmux는 SSH 세션보다 오래 산다

여기서 크게 막혔다. 설정을 다 했는데 서버의 git이 계속 실패했다.

에이전트 포워딩은 `SSH_AUTH_SOCK` 환경변수로 동작한다. SSH가 접속할 때마다 `/tmp/ssh-XXXXXX/agent.NNNN` 같은 **새 소켓**을 만들고, 그 경로를 환경변수에 넣어준다. 세션이 끝나면 소켓도 사라진다.

그런데 tmux 서버는 SSH 세션과 수명이 다르다.

```bash
$ tmux ls
work: 1 windows (created Wed Sep 16 12:17:57 2026)     # 이틀 전

$ ps -eo pid,etime,args | grep '[a]gent-process'
   6433  1-23:13:32 agent-process                       # 이틀째 실행 중
```

이 판(pane) 안의 프로세스는 **9월 16일 그 SSH 세션의 소켓 경로**를 붙들고 있다. 그 세션은 이미 죽었고 소켓 파일도 없다. 재접속해도 기존 판의 환경변수는 갱신되지 않는다.

확인해보면 아예 변수 자체가 없기도 하다.

```bash
$ tr '\0' '\n' < /proc/6433/environ | grep '^SSH_AUTH_SOCK='
(없음)
```

### 해결: 고정 경로를 한 겹 둔다

소켓을 직접 가리키지 말고, **항상 같은 경로의 심볼릭 링크**를 보게 한다. 로그인할 때마다 링크만 새 소켓으로 갈아끼우면 기존 프로세스가 따라온다.

```bash
# 서버의 ~/.bashrc
if [ -n "$SSH_AUTH_SOCK" ] && [ "$SSH_AUTH_SOCK" != "$HOME/.ssh/agent.sock" ]; then
  ln -sf "$SSH_AUTH_SOCK" "$HOME/.ssh/agent.sock"
fi
export SSH_AUTH_SOCK="$HOME/.ssh/agent.sock"
```

새로 여는 판은 `.bashrc`를 타므로 자동으로 이 경로를 쓴다. **`~/.tmux.conf`는 건드릴 필요가 없다** — 여러 기계에 동기화해두는 파일이라면 특히 그대로 두는 게 낫다.

단, **이미 돌고 있던 프로세스는 한 번 재시작**해야 새 경로를 잡는다. 그 뒤로는 재접속을 반복해도 유지된다.

---

## 함정 2: 새 터미널을 열어도 새 연결이 아니다

`ForwardAgent`는 **접속할 때 협상되는 설정**이다. 이미 맺어진 연결에 소급 적용되지 않는다. 그러니 설정을 바꿨으면 새로 접속해야 한다.

그런데 `ControlMaster auto` + `ControlPersist`를 쓰고 있으면, 새 터미널에서 `ssh`를 쳐도 **기존 마스터 연결을 재사용한다.** 새 연결처럼 보이지만 실제로는 에이전트 없는 옛 터널을 타고 들어가서, 원인 모를 `Permission denied`만 반복해서 보게 된다.

```bash
# 마스터를 명시적으로 끊는다
ssh -O exit shared-server-vpn
ssh -O exit shared-server

# 그 다음 새로 접속
ssh shared-server-vpn
```

`ControlPersist` 만료를 기다릴 수도 있지만, 그 사이 접속이 한 번이라도 있으면 타이머가 갱신된다. 명시적으로 끊는 편이 확실하다.

---

## 함정 3: 비대화형 ssh로 진단하면 오판한다

링크를 만드는 코드는 `.bashrc`에 있다. 그런데 **`ssh host 'command'` 형태의 비대화형 접속은 `.bashrc`를 타지 않는다.** 그래서 이렇게 테스트하면 링크가 갱신되지 않은 채 옛 대상을 가리키고 있어 실패한다.

```bash
$ ssh shared-server 'SSH_AUTH_SOCK=$HOME/.ssh/agent.sock ssh -T git@github.com'
git@github.com: Permission denied (publickey).      # ← 설정 문제가 아니다

$ ssh shared-server 'bash -ic "true"; SSH_AUTH_SOCK=$HOME/.ssh/agent.sock ssh -T git@github.com'
Hi example-user! You've successfully authenticated...
```

같은 함정을 다른 형태로 밟은 이야기를 독립 글로 따로 적었다. 셸이 로그인이냐 대화형이냐에 따라 환경이 갈리는 문제는 반복해서 나타난다.

→ [`bash -lc`가 만든 허상 — 비대화형 셸에서 nvm이 사라진다](/dev/2026/09/18/bash-lc가-만든-허상-비대화형-셸에서-nvm이-사라진다/)

---

## 검증

단계별로 확인 지점을 나눠두면 어디가 끊겼는지 바로 안다.

```bash
# 1. 노트북: 에이전트에 키가 올라와 있는가
ssh-add -l

# 2. 서버: 변수가 전달됐는가
echo "$SSH_AUTH_SOCK"

# 3. 서버: 에이전트가 실제로 보이는가
ssh-add -l

# 4. 서버: GitHub 인증이 되는가
ssh -T git@github.com

# 5. 서버: 진짜 repo로 최종 확인
git -C <repo> ls-remote --heads origin
```

`git status`는 로컬 동작이라 인증과 무관하다. **`ls-remote`나 `pull`이 진짜 검증**이다.

그리고 한 번은 **반대 방향으로도** 확인해두는 게 좋다. 이 구성의 목적은 "git이 되는 것"이 아니라 "자격증명 없이 git이 되는 것"이기 때문이다.

```bash
ls ~/.ssh/id_* 2>/dev/null              # 개인키가 없어야 한다
git config --get credential.helper      # 미설정이어야 한다
env | grep -E 'GH_TOKEN|GITHUB_TOKEN'   # 없어야 한다
git -C <repo> ls-remote origin          # 그런데 성공해야 한다
```

앞의 셋 중 하나라도 값이 나오면, 어딘가에 저장된 자격증명이 있고 에이전트가 아니라 그쪽이 인증을 처리하고 있을 수 있다.

---

## 한계와 완화

정직하게 두 가지를 적어둔다.

**1. 접속해 있는 동안에는 대리 인증이 가능하다.** 서버의 root는 `$SSH_AUTH_SOCK`을 가로채 에이전트에 서명을 요청할 수 있다. 구분이 중요하다 — **키 탈취는 불가능하고, 대리 인증은 가능하다.** 훔쳐서 나중에 쓰는 것은 막히지만, 내가 붙어 있는 동안 내 이름으로 서명시키는 것은 막히지 않는다.

완화책은 **사용할 때마다 확인을 요구하는 키**다. macOS라면 Secure Enclave 기반 에이전트(Touch ID 요구), 하드웨어 키가 있다면 `ssh-keygen -t ed25519-sk`. 개인키가 추출 불가능해지고, 세션을 가로채도 내 손가락 없이는 조용히 쓸 수 없다.

다만 자동화가 git을 자주 호출하는 워크플로에서는 확인 프롬프트가 늘어난다. 커밋은 인증이 필요 없고 `fetch`/`push`/`clone`만 해당하니 세션당 몇 번 수준이긴 하다. 보안과 매끄러움의 교환이라는 걸 알고 고르면 된다.

**2. 접속이 끊긴 동안에는 push가 실패한다.** 에이전트가 사라지면 서버에 인증 수단이 없다. 이건 버그가 아니라 설계다 — 코드가 밖으로 나가는 순간에 내가 그 자리에 있게 된다.

무인 push가 꼭 필요해지면 그때 **범위를 좁힌 fine-grained PAT**를 추가하면 된다. 그것도 "서버 접속까지 열어주던 무제한 키"보다는 훨씬 작은 노출이다.

**3. 에이전트는 SSH 인증만 처리한다.** 이걸 모르면 나중에 막힌다. `gh` CLI, PR 생성, HTTPS remote는 전부 **토큰 기반**이라 에이전트가 관여하지 않는다.

```bash
$ gh auth status
(실패 — 저장된 토큰이 없다)
```

즉 "git은 되는데 `gh`는 안 되는" 상태가 정상이다. `gh`까지 필요해지면 앞서 말한 fine-grained PAT를 그 용도로 발급하면 된다. **git 인증과 API 인증은 별개의 통로**라는 것만 기억하면 헷갈리지 않는다.

---

## 증상 대응표

| 증상 | 먼저 볼 것 |
|---|---|
| `Permission denied (publickey)` | `ls -l ~/.ssh/agent.sock` — 죽은 링크인가 |
| 링크는 살아 있는데 실패 | `ssh-add -l` — 노트북 에이전트에 키가 올라와 있는가 |
| 설정을 바꿨는데 그대로 | `ssh -O exit <별칭>` — 마스터 재사용 중인가 |
| 별칭 하나만 되고 다른 건 안 됨 | 두 별칭 모두 `ForwardAgent yes`인가 |
| 오래된 tmux 판에서만 실패 | 그 프로세스 재시작 |
| git은 되는데 `gh`가 안 됨 | 정상이다 — 에이전트는 SSH만 처리한다 |

복구는 대부분 같다. **아무 쪽으로든 서버에 대화형으로 새로 로그인하면 링크가 갱신되면서 풀린다.**

---

## 체크리스트

- [ ] GitHub 전용 키를 분리하고 `IdentitiesOnly yes` 설정
- [ ] `ForwardAgent yes`는 신뢰를 판단한 호스트에만 (`Host *` 금지)
- [ ] 같은 기계의 별칭이 여러 개면 전부 설정
- [ ] 서버 `.bashrc`에 고정 경로 심볼릭 링크 추가
- [ ] 설정 변경 후 `ssh -O exit`로 마스터 끊기
- [ ] 이미 돌고 있던 장수 프로세스는 한 번 재시작
- [ ] 검증은 `ls-remote`/`pull`로 (`git status`는 로컬이라 무의미)
- [ ] 진단할 때 비대화형 ssh로 오판하지 않기
- [ ] 서버에 개인키·credential helper·토큰이 **없는지** 역방향 확인
- [ ] `gh`/HTTPS remote는 별개 통로라는 것 인지

---

### 참고

- [ssh_config(5) — ForwardAgent, ControlMaster, IdentitiesOnly](https://man.openbsd.org/ssh_config)
- [ssh-agent(1)](https://man.openbsd.org/ssh-agent)
- [GitHub Docs — Deploy keys와 fine-grained PAT](https://docs.github.com/authentication)
