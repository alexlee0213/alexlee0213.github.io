---
layout: single
title: "평문 SSH 키 하나가 맥과 GitHub을 동시에 열고 있었다"
date: 2026-09-18 10:00:00 +0900
categories: [dev]
tags: [SSH, 보안, 키관리, GitHub, authorized_keys, 트러블슈팅]
excerpt: "다른 관리자가 root를 가진 공용 서버에 MCP 브리지를 붙이려다, 그 서버에 passphrase 없는 개인키가 있는 걸 발견했다. 같은 키가 내 맥의 admin 셸과 GitHub 계정을 동시에 열고 있었다. 키 주석은 믿을 게 못 되고, 지문으로 grep 하면 지워지지 않는다."
---
> **자격증명 신뢰 경계 정리 시리즈**
> 1. **평문 SSH 키 하나가 맥과 GitHub을 동시에 열고 있었다** ← 현재 글
> 2. [노트북의 원격 로그인, 꺼도 되나 — 인바운드와 아웃바운드를 가르는 기준](/dev/2026/09/18/노트북의-원격-로그인-꺼도-되나-인바운드와-아웃바운드를-가르는-기준/)
> 3. [키를 지우고도 git은 쓴다 — SSH 에이전트 포워딩과 tmux의 죽은 소켓](/dev/2026/09/18/키를-지우고도-git은-쓴다-ssh-에이전트-포워딩과-tmux의-죽은-소켓/)
> 4. [에이전트 포워딩을 반나절 만에 껐다 — 위협 모델에서 빠뜨린 것](/dev/2026/09/18/에이전트-포워딩을-반나절-만에-껐다-위협-모델에서-빠뜨린-것/) *(3편에 대한 반론)*
>
> 관련 독립 글
> - [`bash -lc`가 만든 허상 — 비대화형 셸에서 nvm이 사라진다](/dev/2026/09/18/bash-lc가-만든-허상-비대화형-셸에서-nvm이-사라진다/)

## 결론부터

**다른 사람이 root를 가진 기계에 개인키를 두면, 그 키가 여는 모든 문이 그 사람들에게도 열린다.** 당연한 말 같지만, 실제로 확인해보기 전까지는 그 키가 몇 개의 문을 여는지 본인도 모른다.

점검해야 할 것은 네 가지다.

| 확인 항목 | 명령 | 위험 신호 |
|---|---|---|
| 개인키가 평문인가 | `ssh-keygen -y -P "" -f <키>` | 성공하면 passphrase 없음 |
| 그 기계의 root가 누구인가 | `getent group sudo` | 본인 외 사용자 존재 |
| 키에 제한이 걸려 있나 | `authorized_keys` 앞부분 | `from=`/`command=`/`restrict` 없음 |
| 이 키가 여는 다른 문 | `ssh -T git@github.com` 등 | 예상 못 한 서비스에서 인증 성공 |

그리고 이 글의 두 가지 오진은 이것이다.

- **키 주석(comment)은 신원이 아니다.** 같은 주석을 단 서로 다른 키가 흔하다.
- **지문으로 `grep` 하면 안 지워진다.** 지문은 해시라서 파일 안의 base64 키 문자열과 매칭되지 않는다.

---

## 발단

공용 개발 서버(`shared-server`)에서 돌리는 에이전트가 내 노트북의 CLI를 호출하게 하려고, 서버의 MCP 설정에 항목 하나를 추가했다. 서버가 `ssh`로 노트북에 붙어 명령을 실행하는 구조다.

```json
{
  "command": "ssh",
  "args": ["-T", "-o", "BatchMode=yes",
           "user@my-laptop", "/Users/user/.local/bin/some-cli", "mcp"]
}
```

연결은 한 번에 됐다. `BatchMode=yes`로 붙었다는 건 **암호 입력 없이 키만으로 인증됐다**는 뜻이다. 편리한데, 동시에 그게 문제의 신호이기도 하다. 그 키가 어디에 어떤 상태로 있는지 확인하지 않은 채였다.

---

## 1차 점검: 그 기계의 root는 누구인가

이 서버는 회사 소유고 나 말고도 사용자가 있다.

```bash
$ getent group sudo
sudo:x:27:userA,user,userB
```

**나를 포함해 세 명, 즉 다른 관리자 두 명이 root다.** 이 사실만으로 판단 기준이 바뀐다. 이 기계의 디스크에 있는 내 파일은 전부 그 두 사람이 읽을 수 있다고 가정해야 한다. `chmod 600`은 root 앞에서 아무 의미가 없다.

그래서 물어야 할 질문은 "키가 안전하게 보관돼 있나"가 아니라 **"그 키가 유출됐다고 가정하면 무엇이 열리나"** 다.

---

## 오진 1: 주석이 같으니 같은 키겠지

노트북의 `authorized_keys`를 열어보니 키가 딱 하나 있었고, 주석은 `user@example.com`이었다. 서버의 개인키 주석도 `user@example.com`. 노트북 자신의 `~/.ssh/id_ed25519.pub` 주석도 `user@example.com`.

여기서 "전부 같은 키를 돌려쓰고 있구나, 최악이다"라고 결론 내릴 뻔했다. **틀렸다.**

주석은 `ssh-keygen`이 만들 때 붙이는 자유 문자열일 뿐이고, 기본값이 `사용자@호스트`라서 여러 기계에서 비슷하게 생성된다. 신원을 확인하려면 **지문**을 봐야 한다.

```bash
# 각 파일의 지문
$ ssh-keygen -lf ~/.ssh/id_ed25519.pub
256 SHA256:bbbb…B2 user@example.com (ED25519)

# authorized_keys 는 여러 줄일 수 있으므로 줄마다 계산
$ while IFS= read -r l; do [ -n "$l" ] && printf '%s\n' "$l" | ssh-keygen -lf -; done < ~/.ssh/authorized_keys
256 SHA256:aaaa…A1 user@example.com (ED25519)
```

**주석은 같은데 지문이 다르다.** 노트북 자신의 키(`bbbb…B2`)와, 노트북에 들어올 수 있는 키(`aaaa…A1`)는 서로 다른 키였다. `aaaa…A1`의 개인키를 찾아보니 공용 서버에 있었다.

이건 좋은 소식이었다. 키가 용도별로 분리돼 있으면 하나를 폐기해도 나머지가 안 깨진다. **지문을 안 봤으면 멀쩡한 키까지 폐기할 뻔했다.**

---

## 확인: 평문인가

```bash
$ ls -l ~/.ssh/id_ed25519
-rw------- 1 user user 411 ...

$ ssh-keygen -y -P "" -f ~/.ssh/id_ed25519 >/dev/null 2>&1 \
    && echo "passphrase 없음" || echo "passphrase 있음"
passphrase 없음
```

`-P ""`로 열리면 암호가 안 걸린 평문 키다. `sudo cat` 한 번이면 그대로 복사해 어디서든 쓸 수 있다.

그리고 노트북의 `authorized_keys`에는 아무 제한이 없었다.

```
ssh-ed25519 AAAAC3Nza... user@example.com
```

앞에 `from=`도 `command=`도 `restrict`도 없다. 즉 이 키를 가진 사람은 **내 노트북에서 admin 셸을 그대로 얻는다.**

---

## 확장: 이 키가 여는 다른 문

여기까지는 예상 범위였다. 문제는 그다음이다. 이 키가 **다른 곳에도 등록돼 있는지**는 따로 확인해야 한다. 키 파일은 어디에 등록됐는지 스스로 알려주지 않는다.

가장 빠른 방법은 서비스에 직접 물어보는 것이다.

```bash
# 공용 서버에서 실행
$ ssh -T -o BatchMode=yes git@github.com
Hi example-user! You've successfully authenticated, but GitHub does not provide shell access.
```

**GitHub에 등록돼 있었다.** 즉 다른 관리자 두 명은 이 키로 내 계정으로 push할 수 있는 상태였다. 노트북 문만 닫아서는 해결되지 않는 별개의 통로다.

다른 호스트도 같은 방식으로 훑는다.

```bash
$ for h in bitbucket.org gitlab.com; do
    printf -- '--- %s\n' "$h"
    ssh -T -o BatchMode=yes git@$h 2>&1 | grep -viE '^Warning|known hosts' | head -2
  done
--- bitbucket.org
git@bitbucket.org: Permission denied (publickey).
--- gitlab.com
git@gitlab.com: Permission denied (publickey).
```

여기선 둘 다 미등록이었다.

내가 운영하는 다른 서버들도 확인해야 한다. 이건 각 서버의 `authorized_keys`를 지문으로 대조한다.

```bash
FP="SHA256:aaaa…A1"
for h in server-a server-b server-c; do
  printf -- '--- %s: ' "$h"
  ssh -o BatchMode=yes "$h" '
    n=0
    while IFS= read -r l; do
      [ -z "$l" ] && continue
      fp=$(printf "%s\n" "$l" | ssh-keygen -lf - 2>/dev/null | awk "{print \$2}")
      [ "$fp" = "'"$FP"'" ] && n=$((n+1))
    done < ~/.ssh/authorized_keys
    echo "$n 건"'
done
```

세 대 모두 0건. **이 키가 여는 문은 노트북과 GitHub 두 곳뿐**이라는 게 확정됐다. 범위가 확정돼야 안심하고 지울 수 있다.

---

## 오진 2: 지문으로 grep 하면 안 지워진다

정리를 시작했다. 노트북의 `authorized_keys`에서 해당 키를 빼려고 이렇게 했다.

```bash
$ grep -v "aaaa…A1" ~/.ssh/authorized_keys > /tmp/new && mv /tmp/new ~/.ssh/authorized_keys
```

그리고 검증했더니 **여전히 들어와졌다.**

```bash
$ ssh -o ControlPath=none user@my-laptop 'echo STILL_IN'
STILL_IN
```

원인은 단순하다. **지문은 공개키를 SHA256으로 해시한 값이고, 파일에 적힌 건 base64로 인코딩된 키 자체다.** 둘은 문자열로 겹치지 않는다. `grep`이 아무것도 못 찾았으니 `grep -v`는 모든 줄을 그대로 통과시켰다.

검증을 안 했으면 "지웠다"고 믿은 채 넘어갔을 것이다. **이 실수가 위험한 이유는 조용히 실패하기 때문이다** — 에러도 안 나고, 파일도 정상이고, 명령도 성공한다.

올바른 방법은 줄마다 지문을 계산해 비교하는 것이다.

```bash
cp ~/.ssh/authorized_keys ~/.ssh/authorized_keys.bak-$(date +%Y%m%d-%H%M%S)

FP="SHA256:aaaa…A1"
while IFS= read -r l; do
  [ -z "$l" ] && continue
  fp=$(printf '%s\n' "$l" | ssh-keygen -lf - 2>/dev/null | awk '{print $2}')
  [ "$fp" = "$FP" ] || printf '%s\n' "$l"
done < ~/.ssh/authorized_keys > /tmp/ak.new
mv /tmp/ak.new ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys
```

그리고 반드시 **새 연결로** 검증한다. 기존 연결은 이미 인증을 마쳤으므로 아무것도 증명하지 못한다.

```bash
$ ssh -o ControlPath=none -o BatchMode=yes user@my-laptop 'echo IN'
user@my-laptop: Permission denied (publickey,password,keyboard-interactive).
```

---

## 지우기 전 마지막 점검: 서명키인가

GitHub에서 키를 삭제하기 전에 하나 확인할 게 있다. **그 키가 커밋 서명에 쓰이고 있으면 삭제가 비가역적이다.** 인증키는 새로 만들면 그만이지만, SSH 서명키를 내리면 그 키로 서명한 과거 커밋들의 "Verified" 배지가 풀린다.

```bash
$ git config --global --get-regexp 'gpg|sign|user\.'
user.email user@example.com
user.name  Example User
```

`gpg.format=ssh`도 `user.signingkey`도 `commit.gpgsign`도 없다 → **순수 인증용**이다. 지워도 되돌릴 수 있는 종류의 변경이다.

---

## 정리 실행

```bash
# 1. GitHub 등록 키 목록 (지문 대조용)
$ gh ssh-key list
alex-laptop     ssh-ed25519 AAAA...  2025-07-31  900000001  authentication
shared-server   ssh-ed25519 AAAA...  2025-09-15  900000002  authentication

# 2. 대상의 지문을 계산해 확인 — 이름만 보고 지우지 않는다
$ echo "ssh-ed25519 AAAA... shared-server" | ssh-keygen -lf -
256 SHA256:aaaa…A1 shared-server (ED25519)

# 3. 삭제
$ gh api -X DELETE /user/keys/900000002

# 4. 검증 (공용 서버에서)
$ ssh -T -o BatchMode=yes git@github.com
git@github.com: Permission denied (publickey).

# 5. 개인키 폐기 (여는 문이 하나도 없음을 확인한 뒤)
$ shred -u ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
```

`gh ssh-key list`에 `admin:public_key` 스코프가 없다면 이렇게 받는다.

```bash
gh auth refresh -h github.com -s admin:public_key
```

---

## 폐기 vs 보관

같은 정리 과정에서 안 쓰는 옛 RSA 키(`dddd…D4`)도 나왔다. 이건 **지우지 않고 보관 이동**했다. 기준이 다르기 때문이다.

| | 공용 서버의 키 (`aaaa…A1`) | 노트북의 옛 키 (`dddd…D4`) |
|---|---|---|
| 있는 곳 | 타인 root가 있는 기계 | 내 노트북 |
| 노출 위험 | 즉시 | 없음 |
| 등록처 확인 범위 | 전수 확인 완료 | 일부만 확인 |
| 조치 | **즉시 `shred`** | `~/.ssh/retired/`로 이동 |

`~/.ssh/config`에 `IdentitiesOnly yes`가 없으면 ssh는 기본 이름의 키(`id_rsa` 등)를 **모든 호스트에 자동으로 제시한다**. `known_hosts`에 60개 넘는 호스트가 쌓여 있는데 그중 다섯 곳만 확인한 상태에서 개인키를 지우는 건, 되돌릴 수 없는 결정을 불완전한 정보로 내리는 것이다.

디렉터리 밖으로 옮기면 ssh가 더는 제시하지 않으므로 **목적(불필요한 키 제시 중단)은 즉시 달성**되고, 되돌릴 여지만 남는다. 폐기는 몇 주 지켜본 뒤에 해도 늦지 않다.

```bash
mkdir -p ~/.ssh/retired && chmod 700 ~/.ssh/retired
mv ~/.ssh/id_rsa ~/.ssh/id_rsa.pub ~/.ssh/retired/
# 왜 옮겼는지, 무엇을 확인했고 무엇을 못 확인했는지 메모를 같이 남긴다
```

---

## 판별 커맨드 세트

```bash
# 1. 이 기계의 root는 누구인가
getent group sudo

# 2. 개인키가 평문인가
ssh-keygen -y -P "" -f ~/.ssh/id_ed25519 >/dev/null 2>&1 && echo "평문"

# 3. 들어올 수 있는 키의 지문 (주석 말고 지문을 본다)
while IFS= read -r l; do [ -n "$l" ] && printf '%s\n' "$l" | ssh-keygen -lf -; done < ~/.ssh/authorized_keys

# 4. 이 키가 여는 외부 서비스
for h in github.com bitbucket.org gitlab.com; do
  printf -- '--- %s\n' "$h"; ssh -T -o BatchMode=yes git@$h 2>&1 | head -1
done

# 5. 삭제 후 검증은 반드시 새 연결로
ssh -o ControlPath=none -o BatchMode=yes <대상> 'echo IN'
```

---

## 체크리스트

- [ ] 그 기계에 나 말고 root가 있는지 먼저 확인
- [ ] 키 주석이 아니라 **지문**으로 신원 판단
- [ ] 개인키의 passphrase 유무 확인
- [ ] `authorized_keys`의 옵션 제한(`from=`/`command=`/`restrict`) 유무 확인
- [ ] 그 키가 등록된 **모든 서비스·서버** 전수 조사 후 삭제
- [ ] 서명키로 쓰이는지 확인 (서명키 삭제는 비가역)
- [ ] 파일 편집은 지문 계산으로, `grep` 금지
- [ ] 삭제 후 **새 연결**로 거부되는지 검증
- [ ] 노출 위험이 없는 키는 폐기 대신 보관 이동

---

## 다음 편

키를 정리하고 나니 다음 질문이 남았다. 애초에 이 노트북이 외부에서 SSH 접속을 받아야 할 이유가 있나?

끄면 뭐가 불편해지는지를 추측이 아니라 실측으로 확인해봤다.

→ [2편: 노트북의 원격 로그인, 꺼도 되나](/dev/2026/09/18/노트북의-원격-로그인-꺼도-되나-인바운드와-아웃바운드를-가르는-기준/)

---

### 참고

- [sshd_config — authorized_keys 옵션(`from`, `command`, `restrict`)](https://man.openbsd.org/sshd#AUTHORIZED_KEYS_FILE_FORMAT)
- [ssh-keygen(1)](https://man.openbsd.org/ssh-keygen)
- [GitHub Docs — SSH 키 관리](https://docs.github.com/authentication/connecting-to-github-with-ssh)
