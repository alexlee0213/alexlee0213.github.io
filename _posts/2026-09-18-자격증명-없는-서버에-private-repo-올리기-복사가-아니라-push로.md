---
layout: single
title: "자격증명 없는 서버에 private repo 올리기 — 복사가 아니라 push로"
date: 2026-09-18 17:30:00 +0900
categories: [dev]
tags: [git, SSH, rsync, 부트스트랩, 자동화, 보안, 워크플로]
excerpt: "서버에 GitHub 자격증명을 두지 않기로 했다면, 새 private repo는 어떻게 그 서버에 올리나. 파일을 복사하는 대신 노트북에서 push한다. receive.denyCurrentBranch=updateInstead가 열쇠고, 반복되는 절차라 한 줄 스크립트로 묶었다."
---
> **자격증명 신뢰 경계 정리 시리즈**
> 1. [평문 SSH 키 하나가 맥과 GitHub을 동시에 열고 있었다](/dev/2026/09/18/평문-ssh-키-하나가-맥과-github을-동시에-열고-있었다/)
> 2. [노트북의 원격 로그인, 꺼도 되나 — 인바운드와 아웃바운드를 가르는 기준](/dev/2026/09/18/노트북의-원격-로그인-꺼도-되나-인바운드와-아웃바운드를-가르는-기준/)
> 3. [키를 지우고도 git은 쓴다 — SSH 에이전트 포워딩과 tmux의 죽은 소켓](/dev/2026/09/18/키를-지우고도-git은-쓴다-ssh-에이전트-포워딩과-tmux의-죽은-소켓/)
> 4. [에이전트 포워딩을 반나절 만에 껐다 — 위협 모델에서 빠뜨린 것](/dev/2026/09/18/에이전트-포워딩을-반나절-만에-껐다-위협-모델에서-빠뜨린-것/)
> 5. **자격증명 없는 서버에 private repo 올리기 — 복사가 아니라 push로** ← 현재 글

## 결론부터

[4편](/dev/2026/09/18/에이전트-포워딩을-반나절-만에-껐다-위협-모델에서-빠뜨린-것/)에서 서버의 GitHub 자격증명을 전부 없앴다. 그러면 남는 질문이 하나다. **새 private repo는 어떻게 그 서버에 올리나.**

**파일을 복사하는 대신 git으로 밀어넣는다.** 노트북이 접속을 걸고, 서버는 받기만 한다.

```bash
# 노트북에서 — 서버에 빈 repo를 만들고
ssh shared-server-git "git init -q -b main <원격경로> \
  && git -C <원격경로> config receive.denyCurrentBranch updateInstead"

# 밀어넣는다
git remote add shared-server-repo shared-server-git:<원격경로>
git push shared-server-repo --all
git push shared-server-repo --tags
```

반복되는 작업이라 스크립트로 묶었다. 결국 이 한 줄이 된다.

```bash
repo-to-server.sh ~/path/to/repo
```

---

## 그전에 — 대부분은 이 문제에 해당하지 않는다

먼저 갈라야 할 것이 있다. 서버에 코드를 가져오는 상황은 remote 형태로 두 갈래인데, **한쪽은 애초에 자격증명이 필요 없다.**

| remote | 서버에서 |
|---|---|
| `https://github.com/...` (공개) | **그냥 `git clone` 하면 된다.** 인증 없음 |
| `git@github.com:...` (private) | 불가. 노트북을 거쳐야 한다 |

이 구분이 중요한 이유는, 이 서버가 GPU 박스라 **하는 일의 대부분이 공개 ML 프로젝트를 설치하는 것**이기 때문이다. pyenv, 논문 재현 repo, 벤치마크 툴킷 — 전부 공개 HTTPS다. 자격증명을 없앴다고 이것들까지 막히지 않는다.

실제로 막히는 건 조직 private repo 몇 개뿐이고, 그건 개수가 고정돼 있다. **"repo가 수시로 들락거려서 관리가 번거롭다"는 걱정은 읽기 방향에서는 성립하지 않는다.** 들락거리는 쪽은 인증이 필요 없고, 인증이 필요한 쪽은 들락거리지 않는다.

이 분기를 명시해두지 않으면 반대 방향의 사고가 난다 — 에이전트나 나중의 내가 공개 repo까지 "자격증명이 없어서 안 된다"고 막아서는 것이다.

---

## 왜 복사가 아니라 push인가

### 1. `origin`이 따라온다

이게 결정적이다. 노트북에서 클론한 repo의 `.git/config`에는 이렇게 들어 있다.

```ini
[remote "origin"]
    url = git@github.com:org/private-repo.git
```

디렉터리를 통째로 복사하면 **서버에도 이 설정이 그대로 생긴다.** 그런데 서버는 이 remote를 쓸 수 없다. 결과는 예정된 혼란이다.

```bash
$ git pull
git@github.com: Permission denied (publickey).
```

자격증명을 의도적으로 없앴다는 걸 아는 사람에게는 정상 신호다. 하지만 **그 맥락을 모르는 사람이나 도구에게는 고장으로 보인다.** 실제로 서버에서 돌던 코딩 에이전트가 이 오류를 만나 "키가 없으니 새로 만들어 GitHub에 등록하자"는 제안까지 갔다. 하루 전에 없앤 것을 되살리는 제안이었다.

복사 후 `git remote remove origin`을 매번 붙이면 되긴 한다. 다만 **잊기 좋은 종류의 후속 작업**이고, 잊었을 때 조용히 나쁜 상태가 된다.

`git init`으로 시작하면 이 문제가 아예 없다. 새로 만든 repo는 remote가 0개다. 지울 것이 없으면 잊을 것도 없다.

### 2. 이후 동기화에 쓸 수 없다

처음은 scp, 이후 갱신은 git이라면 도구를 두 개 쓰는 셈이다. 게다가 scp로 덮어쓰면 **서버에서 만든 커밋이 날아간다.** 실험 결과를 커밋해두고 노트북에서 코드를 고친 뒤 다시 복사하면, 그 결과가 사라진다.

`git init` + push는 처음과 이후가 같은 메커니즘이다. 익힐 것이 하나고, 덮어쓰기 사고가 구조적으로 막힌다(non-fast-forward는 거부된다).

### 3. 무거운 것까지 따라온다

`.venv/`, `data/`, 체크포인트, `wandb/`. ML repo에서 `.gitignore` 대상은 수 GB가 되기 쉽다. `git push`는 추적 대상만 보낸다. **데이터를 보내지 않는 것이 기본값인 편이 맞다** — 공개 데이터셋은 서버에서 직접 받는 게 대역폭도 아끼고, 트랙별 단일 캐시 같은 규칙에도 맞는다.

필요할 때만 따로 보내면 된다.

```bash
rsync -av <로컬데이터>/ shared-server-git:<원격경로>/data/
```

---

## `receive.denyCurrentBranch=updateInstead`가 핵심이다

절차 중에 이 한 줄이 실질적인 일을 한다.

git은 기본적으로 **체크아웃된 브랜치로의 push를 거부한다**(`refuse`). 작업 중인 트리와 HEAD가 어긋나버리기 때문이다. 그래서 보통은 bare repo를 따로 두고 거기로 밀어넣는다.

그런데 이 서버는 그 repo에서 **직접 실행하고 편집한다.** bare repo는 쓸 수 없다.

`updateInstead`는 정확히 이 경우를 위한 값이다. **작업트리가 깨끗하면 push를 받아들이고 작업트리까지 함께 갱신한다.** 더러우면 거부한다 — 편집 중인 내용을 덮어쓰지 않는다.

실제로 확인해보면 이렇다.

```bash
# 서버: 받기 전
$ cat file.txt
before

# 노트북에서 push 후 — 서버:
$ cat file.txt
after
$ git status --porcelain | wc -l
0
```

`git fetch` 후 수동으로 `merge`할 필요가 없다. push 한 번으로 서버의 작업 디렉터리가 최신이 된다.

---

## 일상 루프

정리하면 이렇다. **접속은 항상 노트북이 건다.**

```bash
# 서버 — 실행·편집·커밋까지만
git add -A && git commit -m "실험 결과"

# 노트북 — 회수하고 GitHub에 반영
git pull shared-server-repo main
git push origin main

# 노트북 — 코드를 고쳤다면 내려보내기
git push shared-server-repo main
```

서버는 커밋까지가 책임이다. 그 앞도 뒤도 노트북이 한다.

---

## 스크립트로 묶기

4편에서 이렇게 적었다.

> **마찰이 큰 통제는 결국 지켜지지 않는다.**

deploy key를 배제한 근거였는데, 같은 기준을 내가 만든 절차에도 적용해야 공평하다. 네 줄짜리 절차를 repo마다 손으로 치면, 언젠가 귀찮아서 `scp -r`을 쓰게 된다. 그러면 `origin`이 따라오고 처음으로 돌아간다.

그래서 한 줄로 줄였다.

```bash
repo-to-server.sh ~/Develop/path/to/repo      # 온보딩 또는 갱신
repo-to-server.sh -n ~/Develop/path/to/repo   # dry-run
```

### 무엇을 자동화했나

**경로 치환.** 노트북과 서버의 홈 경로가 다르다(`/Users/user` ↔ `/home/user`). 매번 손으로 바꾸면 오타가 난다.

**상태 판정을 한 번의 접속으로.** 원격 경로가 없는지(부트스트랩), git repo가 아닌지(중단), 이미 있는지(갱신)를 한 번에 가져온다. ssh 왕복을 세 번 하지 않는다.

```bash
REMOTE_STATE="$(ssh -o BatchMode=yes "$SSH_ALIAS" "
  p='$REMOTE_PATH'
  if [ ! -e \"\$p\" ]; then echo 'MISSING'
  elif [ ! -d \"\$p/.git\" ]; then echo 'NOT_A_REPO'
  else
    dirty=\$(git -C \"\$p\" status --porcelain | wc -l)
    echo \"EXISTS \$dirty \$(git -C \"\$p\" rev-parse --short HEAD)\"
  fi")"
```

**멱등성.** 이미 올라와 있는 repo에 다시 돌려도 안전하다. 부트스트랩 대신 갱신으로 처리한다. *"이 repo 올렸던가?"* 를 기억할 필요가 없다는 것이 생각보다 큰 차이다. 기억해야 하는 절차는 결국 안 쓰게 된다.

**작업트리 오염 사전 경고.** push가 거부된 뒤 원인을 찾는 대신, 접속 시점에 알려준다.

```
경고: 서버 작업트리에 커밋 안 된 변경 1건이 있다.
      체크아웃된 브랜치로의 push는 거부된다 — 서버에서 커밋하거나 stash하라.
```

**검증 출력.** 양쪽 HEAD를 대조하고, 서버의 remote 개수를 찍는다.

```
맥 HEAD:          6af3f8e
서버 HEAD:        6af3f8e
서버 remote:      0개 (GitHub 흔적 없음 — 의도된 상태)
서버 작업트리:     0건 변경
```

**`remote 0개`가 이 구성의 성공 지표다.** 숫자가 0이 아니면 어디선가 `origin`이 딸려온 것이고, 그게 바로 피하려던 상태다. 매번 눈에 보이게 해두면 사고를 조기에 잡는다.

---

## 주의점 세 가지

**1. push 전에 서버 작업트리를 깨끗하게.** `updateInstead`의 전제다.

**2. 순서는 pull 먼저, push 나중.** 서버에 미회수 커밋이 있는데 노트북에서 밀면 non-fast-forward로 거부된다. **거부되는 것이 정상이고 안전장치다.** `--force`로 뚫으면 서버의 실험 결과가 사라진다.

**3. 데이터는 git이 아니다.** `.gitignore` 대상은 넘어가지 않는다. 의도된 동작이고, 필요하면 `rsync`로 따로 보낸다.

---

## 여담: 이걸 뭐라고 부를 것인가

이 절차를 처음에 "온보딩"이라고 불렀는데, 쓰면서 갸웃했다.

인프라 맥락에서 *"onboard a repo to a platform"*(기존 시스템에 새 대상을 편입시킨다)은 통용되는 용법이라 틀리지는 않는다. 다만 한국어 기술 문서에서 "온보딩"은 **사람과 사용자에 강하게 묶여 있다** — 신규 입사자 온보딩, 사용자 온보딩. 대상이 repo면 한 박자 걸린다.

정확히는 두 층이 섞여 있다.

- **부트스트랩** — 없는 상태에서 초기 상태를 세우는 것 (`git init`)
- **온보딩** — 기존 워크플로에 편입시키는 것 (remote 등록, `updateInstead` 설정)

스크립트가 둘 다 하니 어느 쪽으로 불러도 반은 맞다. 다만 **기술 용어로는 "부트스트랩"이 더 좁고 정확하다.** 문서 제목에는 평서형("새 repo를 서버에 처음 올리기")이 오해가 가장 적었다.

사소해 보이지만, 나중에 검색해서 이 문서를 다시 찾는 건 결국 내가 그때 쓴 단어다.

---

## 체크리스트

- [ ] remote가 공개(HTTPS)인지 private(SSH)인지 먼저 확인 — 공개면 서버에서 바로 clone
- [ ] private이면 `scp -r` 대신 `git init` + `push`
- [ ] 서버 repo에 `receive.denyCurrentBranch=updateInstead` 설정
- [ ] `--all`과 `--tags`를 함께 (브랜치 하나만 가면 나중에 곤란하다)
- [ ] 서버 repo의 **remote 개수가 0인지** 확인 — 이게 성공 지표
- [ ] 데이터는 `rsync`로 별도, 공개 데이터셋은 서버에서 직접
- [ ] 절차가 세 줄을 넘으면 스크립트로 묶기 — 마찰이 큰 통제는 지켜지지 않는다

---

### 참고

- [git-config(1) — receive.denyCurrentBranch](https://git-scm.com/docs/git-config#Documentation/git-config.txt-receivedenyCurrentBranch)
- [git-push(1) — --all, --tags](https://git-scm.com/docs/git-push)
- [rsync(1)](https://download.samba.org/pub/rsync/rsync.1)
