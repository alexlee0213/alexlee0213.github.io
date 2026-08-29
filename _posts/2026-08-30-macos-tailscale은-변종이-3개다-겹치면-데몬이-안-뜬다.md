---
layout: single
title: "macOS Tailscale은 변종이 3개다 — 겹치면 데몬이 안 뜬다"
date: 2026-08-30 12:10:00 +0900
categories: [dev]
tags: [Tailscale, macOS, 시스템확장, SystemExtension, Homebrew, 트러블슈팅, 포렌식]
excerpt: "CLIError error 1, tailscaled.socket 없음, inet 100.x 없음. 증상 세 개가 전부 같은 원인을 가리킨다. macOS Tailscale은 App Store판·독립판·오픈소스 CLI 세 가지고, 앱 변종이 겹치면 데몬 역할을 하는 시스템 확장이 아예 뜨지 않는다."
---

> **macOS Tailscale 정리 시리즈**
> 1. [MagicDNS가 4개월간 안 켜졌다 — 47바이트짜리 진범](/dev/2026/08/30/magicdns가-4개월간-안-켜졌다-47바이트짜리-진범/)
> 2. **macOS Tailscale은 변종이 3개다 — 겹치면 데몬이 안 뜬다** ← 현재 글
> 3. [완전 삭제 후 재설치 — 순서를 틀리면 재부팅을 헛쓴다](/dev/2026/08/30/tailscale-완전-삭제-후-재설치-순서를-틀리면-재부팅을-헛쓴다/)

## 결론부터

**macOS GUI 버전의 Tailscale은 CLI와 GUI와 데몬이 전부 한 바이너리다.** 데몬 역할은 네트워크 시스템 확장이 맡는다. 그래서 확장이 안 뜨면 CLI도 같이 죽는다.

| 변종 | 번들 ID | 데몬 정체 | 설치 경로 |
|---|---|---|---|
| App Store판 | `io.tailscale.ipn.macos` | `IPNExtension` | Mac App Store |
| 독립판(Standalone) | `io.tailscale.ipn.macsys` | `io.tailscale.ipn.macsys.network-extension` | pkg 설치 파일 |
| 오픈소스 CLI | 없음 | 별도 `tailscaled` 프로세스 | `brew install tailscale` |

앞의 두 개가 겹치면 사고가 난다. 그리고 그 사고를 우회하려고 세 번째를 깔면 [1편에서 다룬 DNS 반쪽 문제](/dev/2026/08/30/magicdns가-4개월간-안-켜졌다-47바이트짜리-진범/)로 이어진다.

---

## 증상 세 개는 사실 하나다

```
$ tailscale version
The Tailscale CLI failed to start: The operation couldn't be completed. (Tailscale.CLIError error 1.)

$ ls -l /var/run/tailscaled.socket
ls: /var/run/tailscaled.socket: No such file or directory

$ ifconfig | grep "inet 100\."
(출력 없음)
```

앱을 재실행해도 안 고쳐진다. 재부팅해도 그대로다.

셋 다 같은 원인이다. Tailscale 이슈 트래커에 정확한 설명이 있다.

> The Tailscale UI and CLI is just a user-facing process that talks to the network extension; you need to understand why the Tailscale *network extension* is not starting.

UI도 CLI도 껍데기다. 실제로 일하는 건 네트워크 시스템 확장이고, 그게 안 뜨면 전부 무너진다. **그러니 이 증상을 만나면 앱을 재설치하기 전에 확장 상태부터 봐야 한다.**

```bash
systemextensionsctl list
```

```
1 extension(s)
--- com.apple.system_extension.network_extension
enabled active  teamID      bundleID (version)                                    [state]
                W5364U7YZB  io.tailscale.ipn.macsys.network-extension (1.102.3)   [terminating for uninstall]
```

`enabled`와 `active` 컬럼에 `*`가 없고 상태가 정상이 아니면 그게 범인이다.

---

## 내가 깐 게 어느 변종인지 확인하기

세 가지를 구분하는 방법은 이렇다.

```bash
# 번들 ID로 앱 변종 판별
plutil -p /Applications/Tailscale.app/Contents/Info.plist | grep -E "CFBundleIdentifier|CFBundleShortVersionString"

# App Store판이면 이 파일이 있다
ls -d /Applications/Tailscale.app/Contents/_MASReceipt

# 서명 주체 확인 (독립판은 Developer ID)
codesign -dv --verbose=2 /Applications/Tailscale.app 2>&1 | grep -E "Identifier|Authority"

# 오픈소스 CLI 설치 여부
which -a tailscale tailscaled
brew list --versions tailscale
```

독립판이면 이렇게 나온다.

```
"CFBundleIdentifier" => "io.tailscale.ipn.macsys"
"CFBundleShortVersionString" => "1.102.3"

Identifier=io.tailscale.ipn.macsys
Authority=Developer ID Application: Tailscale Inc. (W5364U7YZB)
```

`_MASReceipt`가 있으면 App Store판, 없고 Developer ID 서명이면 독립판이다.

---

## 알려진 버그: 유령 App Store 확장

Tailscale 이슈 #17891의 제목이 상황을 그대로 요약한다.

> macOS Tahoe (26.x) + Tailscale: Orphaned App Store System Extension Completely Breaks PKG Install

재현 절차도 명확하다.

1. App Store판을 설치해서 한동안 쓴다 (시스템 확장이 등록됨)
2. 독립판 pkg로 갈아탄다
3. 확장 승인도 하고, VPN 구성 허용도 하고, UI도 정상으로 보인다
4. **그런데 데몬이 영영 안 뜬다**

리포터의 관찰이 특히 뼈아프다.

> The Homebrew (`brew install tailscale`) version works fine on the same machine, which suggests the problem is specific to the macOS PKG + Network Extension + historical App Store install combination

즉 **같은 머신에서 Homebrew판은 잘 돈다.** 그래서 많은 사람이 원인 규명 대신 Homebrew로 도망간다. 나도 그랬다.

---

## 타임스탬프 포렌식

4개월 전 무슨 일이 있었는지는 기억나지 않았지만, 파일 시스템은 기억하고 있었다.

```bash
ls -ldT ~/Library/Containers/io.tailscale.ipn.mac* \
        ~/Library/Group\ Containers/*io.tailscale* \
        /usr/local/bin/tailscale \
        /Library/Tailscale \
        /Library/LaunchDaemons/homebrew.mxcl.tailscale.plist
```

시각순으로 늘어놓으니 경위가 그대로 복원됐다.

| 시각 | 흔적 | 해석 |
|---|---|---|
| `Apr 27 02:26` | `~/Library/Containers/io.tailscale.ipn.macos` | App Store판 설치 |
| `Apr 27 02:52` | `/usr/local/bin/tailscale` (68바이트) | 독립판 pkg 설치 + CLI 연동 |
| `Apr 27 02:53` | `/Library/Tailscale` 생성 | 독립판 데몬 상태 디렉터리 |
| `Apr 27 09:34` | `homebrew.mxcl.tailscale.plist` | 7시간 뒤 포기하고 Homebrew |

**26분 만에 App Store판에서 독립판으로 갈아탔고, 안 되니까 7시간 뒤 Homebrew를 깔았다.** 위의 알려진 버그와 정확히 일치하는 경로다.

`/usr/local/bin/tailscale`이 68바이트인 것도 단서였다. 열어보면 래퍼 스크립트다.

```sh
#!/bin/sh
/Applications/Tailscale.app/Contents/MacOS/tailscale "$@"
```

이 파일은 손으로 만든 게 아니라 **앱의 Settings → CLI integration → Install Now가 만든다.** 즉 독립판 세팅을 진행하다 만 흔적이다.

VPN 구성 프로파일에도 두 세대가 남아 있었다.

```
$ scutil --nc list
* (Connecting)    ... VPN (io.tailscale.ipn.macsys) "Tailscale 2"
* (Disconnected)  ... VPN (io.tailscale.ipn.macos)  "Tailscale"
```

`Tailscale`과 `Tailscale 2`. 뒤에 숫자가 붙은 게 나중에 설치한 독립판이다. **프로파일 이름에 숫자가 붙어 있으면 이전 세대가 남아 있다는 신호다.**

---

## 여기서 얻을 교훈

**이중 설치는 원인이 아니라 증상이었다.**

처음에는 "Homebrew CLI와 앱이 겹쳐 있으니 정리하자"가 목표였다. 그런데 파고들수록 순서가 반대였다. 앱이 4월부터 고장나 있었고, 그 우회책으로 Homebrew를 깐 것이다. Homebrew를 걷어내는 것만으로는 아무것도 해결되지 않는다. **오히려 유일하게 작동하던 터널을 없애는 셈이다.**

일반화하면 이렇다.

> 여러 도구가 중복 설치돼 있으면, 왜 중복이 생겼는지부터 물어야 한다. 대개 앞의 것이 고장나서 뒤의 것을 깐 것이고, 앞의 것을 고치지 않고 뒤의 것만 지우면 상황이 나빠진다.

---

## 체크리스트

- [ ] `systemextensionsctl list`로 확장 상태 확인 (`enabled`/`active` 컬럼)
- [ ] `plutil -p .../Info.plist`로 번들 ID 확인 (`macos` 대 `macsys`)
- [ ] `_MASReceipt` 유무로 App Store판 여부 판별
- [ ] `scutil --nc list`에 VPN 프로파일이 2개 이상인지 확인
- [ ] `ls -ldT`로 설치 흔적 타임스탬프 정렬
- [ ] `which -a tailscale`로 오픈소스 CLI 공존 여부 확인

---

## 다음 편

원인을 알았으니 이제 걷어낼 차례다. 그런데 이 정리는 **순서가 중요하다.** 순서를 틀리면 좀비 launchd 항목이 남거나, 재부팅을 한 번 헛쓰게 된다. 실제로 나는 재부팅을 헛썼다.

→ [3편: 완전 삭제 후 재설치 — 순서를 틀리면 재부팅을 헛쓴다](/dev/2026/08/30/tailscale-완전-삭제-후-재설치-순서를-틀리면-재부팅을-헛쓴다/)

---

### 참고

- [Three ways to run Tailscale on macOS — Tailscale Docs](https://tailscale.com/docs/concepts/macos-variants)
- [Authorizing the Tailscale system extension on macOS — Tailscale Docs](https://tailscale.com/docs/concepts/macos-sysext)
- [Issue #17891: Orphaned App Store System Extension Completely Breaks PKG Install](https://github.com/tailscale/tailscale/issues/17891)
- [Issue #13848: Daemon not running on macOS](https://github.com/tailscale/tailscale/issues/13848)
