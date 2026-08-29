---
layout: single
title: "Tailscale 완전 삭제 후 재설치 — 순서를 틀리면 재부팅을 헛쓴다"
date: 2026-08-30 12:20:00 +0900
categories: [dev]
tags: [Tailscale, macOS, Homebrew, brew-services, systemextensionsctl, launchd, 트러블슈팅]
excerpt: "macOS에서 Tailscale을 완전히 걷어내는 절차. sudo 없는 brew services는 root 데몬을 못 보고, brew uninstall은 빈 디렉터리를 남기며, 시스템 확장은 앱을 지워도 안 빠진다. 순서를 지키지 않으면 재부팅을 헛쓰게 된다."
---

> **macOS Tailscale 정리 시리즈**
> 1. [MagicDNS가 4개월간 안 켜졌다 — 47바이트짜리 진범](/dev/2026/08/30/magicdns가-4개월간-안-켜졌다-47바이트짜리-진범/)
> 2. [macOS Tailscale은 변종이 3개다 — 겹치면 데몬이 안 뜬다](/dev/2026/08/30/macos-tailscale은-변종이-3개다-겹치면-데몬이-안-뜬다/)
> 3. **완전 삭제 후 재설치 — 순서를 틀리면 재부팅을 헛쓴다** ← 현재 글

## 결론부터

**함정 세 개가 순서대로 기다리고 있다.**

| # | 함정 | 증상 | 대응 |
|---|---|---|---|
| 1 | `brew services list`가 root 데몬을 못 봄 | `none`인데 프로세스는 돌고 있음 | `sudo`를 붙여서 조회·중지 |
| 2 | `brew uninstall`이 빈 디렉터리를 남김 | 버전 없이 이름만 출력됨 | `rmdir`로 직접 제거 |
| 3 | 시스템 확장이 앱 삭제로 안 빠짐 | 재부팅하면 `activated disabled`로 부활 | `systemextensionsctl uninstall` 명시 실행 |

3번을 모르고 재부팅했다가 한 번을 통째로 날렸다.

⚠️ **먼저 알아둘 것**: 아래 1단계를 실행하는 순간 테일넷 연결이 끊긴다. 원격 SSH나 서비스 접속이 그 터널에 의존하고 있다면 작업 시점을 골라야 한다.

---

## 0단계: 무엇이 실제로 돌고 있나

가장 먼저 할 일은 **어느 데몬이 진짜 터널을 제공하는지** 확정하는 것이다. 이걸 건너뛰면 살아 있는 유일한 터널을 지우고 시작하게 된다.

```bash
which -a tailscale tailscaled
brew list --versions tailscale
sudo brew services list | grep -i tailscale
ps aux | grep -i "[t]ailscale"
ls -l /Library/LaunchDaemons | grep -i tail
systemextensionsctl list
scutil --nc list
```

내 경우는 이렇게 나왔다.

```
/opt/homebrew/bin/tailscale
/usr/local/bin/tailscale
/opt/homebrew/bin/tailscaled

tailscale 1.98.2 1.96.4

root  687  /opt/homebrew/opt/tailscale/bin/tailscaled   26Jul26부터 149시간 CPU
```

**Homebrew 데몬이 진짜 주인이었다.** GUI 앱도 떠 있었지만 소켓을 뺏겨서 놀고 있었다.

---

## 함정 1: sudo 없는 brew services는 거짓말한다

같은 상황에서 `brew services list`는 이렇게 나온다.

```
$ brew services list | grep -i tailscale
tailscale  none  root
```

상태가 `none`이다. 그런데 프로세스는 넉 달째 돌고 있다. **모순처럼 보이지만 아니다.**

`brew services`는 sudo 없이 실행하면 **사용자 LaunchAgents 도메인만** 들여다본다. `/Library/LaunchDaemons`에 등록된 root 데몬은 시야 밖이다. 그래서 실제로는 brew가 관리 중인데 "관리 안 함"으로 보고한다.

```bash
sudo brew services list | grep -i tailscale   # 이게 진짜 상태
sudo brew services stop tailscale             # 이게 진짜 중지
```

`sudo`가 붙으면 `launchctl bootout`과 plist 삭제까지 처리해준다.

plist가 남으면 수동으로 걷어낸다.

```bash
sudo launchctl bootout system/homebrew.mxcl.tailscale
sudo rm -f /Library/LaunchDaemons/homebrew.mxcl.tailscale.plist
```

> **곁가지 함정**: 등록 방식이 두 가지다. `homebrew.mxcl.tailscale.plist`면 `brew services`가 만든 것이고, `com.tailscale.tailscaled.plist`면 `sudo tailscaled install-system-daemon`이 만든 것이다. 후자라고 짐작하고 `uninstall-system-daemon`을 실행하면 조용히 빗나간다. **파일명을 먼저 확인하자.**

중지 확인은 프로세스와 소켓 양쪽으로 한다.

```bash
ps aux | grep -i "[t]ailscaled"      # 비어야 함
ls -l /var/run/tailscaled.socket     # No such file이어야 함
```

---

## 함정 2: brew uninstall이 남기는 껍데기

```bash
brew uninstall --force tailscale
```

버전이 둘 이상 깔려 있을 수 있으므로 `--force`를 쓴다. 그런데 이걸 하고도 이런 출력이 나온다.

```
$ brew list --versions tailscale
tailscale
```

**버전 번호 없이 이름만 나온다.** Cellar를 보면 이유가 보인다.

```
$ ls -ld /opt/homebrew/Cellar/tailscale
drwxr-xr-x  2 user  admin  64  Aug 30 02:38  /opt/homebrew/Cellar/tailscale
```

64바이트, 즉 **빈 디렉터리**다. 안의 버전 디렉터리는 지워졌는데 껍데기가 남았다. 직접 지운다.

```bash
rmdir /opt/homebrew/Cellar/tailscale
brew list --versions tailscale   # 이제 빈 출력
```

Homebrew판이 남긴 DNS 설정도 같이 치운다. 이게 남으면 나중에 GUI 앱이 제대로 DNS를 꽂아도 `Not Reachable` 리졸버가 계속 끼어든다.

```bash
sudo rm -f /etc/resolver/search.tailscale
```

---

## 함정 3: 시스템 확장은 앱을 지워도 안 빠진다

여기가 재부팅을 헛쓴 지점이다.

앱을 휴지통에 버려도 macOS는 시스템 확장 등록을 `/Library/SystemExtensions`에 별도로 보관한다. 상태가 `terminating for uninstall`로 바뀌길래 "재부팅하면 완료되겠지" 하고 재부팅했더니 이렇게 돌아왔다.

```
enabled active  teamID      bundleID (version)                                    [state]
        *       W5364U7YZB  io.tailscale.ipn.macsys.network-extension (1.102.3)   [activated disabled]
```

**제거가 완료되지 않고 `activated disabled`로 복귀했다.** 명시적 uninstall이 필요하다.

```bash
osascript -e 'quit app "Tailscale"'
systemextensionsctl uninstall W5364U7YZB io.tailscale.ipn.macsys.network-extension
systemextensionsctl list
```

앱을 이미 지운 뒤라면 CLI가 거부할 수 있다. 그때는 GUI로 간다. `systemextensionsctl` 출력이 직접 안내하는 경로다.

**시스템 설정 → 일반 → 로그인 항목 및 확장 프로그램 → 네트워크 확장**

`waiting to uninstall on reboot`이 나오면 그때는 진짜로 재부팅이 필요하다. macOS가 강제하는 것이라 우회로가 없다.

---

## 나머지 잔재

```bash
# 시스템 영역
sudo rm -rf /Library/Tailscale        # state, ssh 키, 로그 (내 경우 43MB)
sudo rm -f  /usr/local/bin/tailscale  # 앱을 가리키는 래퍼

# 홈 디렉터리 — 두 세대 모두
rm -rf ~/Library/Containers/io.tailscale.ipn.mac* \
       ~/Library/Group\ Containers/*io.tailscale* \
       ~/Library/Caches/io.tailscale* \
       ~/Library/Preferences/io.tailscale.ipn.mac*.plist
```

`~/Library/Containers`의 글롭을 `io.tailscale.ipn.mac*`로 잡은 건 의도적이다. `macos`(App Store판)와 `macsys`(독립판)를 **한꺼번에** 잡기 위해서다. 하나만 지우면 다시 같은 함정에 빠진다.

**VPN 프로파일은 GUI에서 지운다.** 시스템 설정 → 네트워크에서 Tailscale 항목들을 선택해 제거한다.

> ⚠️ **`com.apple.tailspind`는 건드리지 말 것.** `launchctl list | grep -i tail`을 돌리면 같이 걸리는데, 이건 Apple의 진단 데몬이고 Tailscale과 무관하다. 이름에 `tail`이 들어갈 뿐이다.

---

## 재부팅 후 검증

```bash
systemextensionsctl list        # Tailscale 항목 없음
scutil --nc list                # Tailscale VPN 프로파일 없음
ls -ld /Library/Tailscale /usr/local/bin/tailscale   # 둘 다 No such file
brew list --versions tailscale  # 빈 출력
ls -la /etc/resolver/           # 비어 있음
ifconfig | grep -c "inet 100\." # 0
```

전부 통과하면 깨끗한 상태다.

---

## 재설치

[공식 패키지 서버](https://pkgs.tailscale.com/stable/#macos)에서 독립판 pkg를 받아 설치한다. **App Store판은 피하는 게 좋다.** Tailscale 자신도 독립판을 권장 변종으로 안내하고 있다.

설치 후 순서대로 승인한다.

1. 시스템 확장 승인 (시스템 설정 → 개인정보 보호 및 보안 → 허용)
2. `"Tailscale"이 VPN 구성을 추가하려고 합니다` → 허용
3. 메뉴바 아이콘 → Log in

CLI를 터미널에서 쓰려면 앱 안에서 연동을 설치한다. **Settings → CLI integration → Show me how → Install Now.** 이게 `/usr/local/bin/tailscale` 래퍼를 만들어준다. 손으로 심볼릭 링크를 걸어도 되지만, 공식 경로를 쓰는 편이 나중에 흔적 추적이 쉽다.

---

## 결과

정리 전후 비교다.

| 항목 | 정리 전 | 정리 후 |
|---|---|---|
| `scutil --dns \| grep -c 100.100.100.100` | `0` (4개월간) | `3` |
| 이름 해석 | `/etc/hosts` 우회 필요 | 순수 DNS로 해석 |
| `tailscale status` | `CLIError error 1` | 피어 목록 정상 |
| 설치 상태 | 세 변종의 잔재 | 독립판 하나 |
| ping TTL | `56` (가짜 응답) | `64` (진짜 피어) |

마지막 줄은 별도 글로 다뤘다. 정리 도중 터널이 죽었는데도 ping이 성공해서 오판했던 이야기다.

→ [100.64 대역에 ping이 되는데 연결이 안 된다면](/dev/2026/08/30/100-64-대역에-ping이-되는데-연결이-안-된다면/)

---

## 되돌리기

중간에 막히면 원상복구는 짧다. 앱이 끝내 안 뜨면 일단 되살려두고 나중에 차분히 하는 게 낫다.

```bash
brew install tailscale
sudo brew services start tailscale
```

DNS는 다시 반쪽이 되지만 터널은 즉시 복구된다. `/etc/hosts` 한 줄을 같이 넣어두면 이름도 임시로 해결된다.

---

## 체크리스트

- [ ] 0단계로 **진짜 주인 데몬**부터 확정
- [ ] `sudo brew services stop` (sudo 필수)
- [ ] launchd plist 파일명 확인 후 제거
- [ ] `brew uninstall --force` + `rmdir`로 빈 Cellar 정리
- [ ] `/etc/resolver/search.tailscale` 제거
- [ ] `systemextensionsctl uninstall` **명시 실행**
- [ ] 홈 컨테이너를 `mac*` 글롭으로 두 세대 동시 제거
- [ ] VPN 프로파일 GUI 삭제
- [ ] 재부팅 후 6개 항목 검증
- [ ] 독립판 pkg 재설치 + 승인 2단계

---

### 참고

- [Install Tailscale on macOS — Tailscale Docs](https://tailscale.com/docs/install/mac)
- [Authorizing the Tailscale system extension on macOS — Tailscale Docs](https://tailscale.com/docs/concepts/macos-sysext)
- [Tailscale CLI reference — Tailscale Docs](https://tailscale.com/docs/reference/tailscale-cli)
- [Tailscale 패키지 서버 (버전별 아카이브)](https://pkgs.tailscale.com/stable/#macos)
