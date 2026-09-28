---
layout: single
title: "Claude Code에 공식 Google Drive MCP 연결하기: user scope와 OAuth 콜백"
date: 2026-09-28 10:00:00 +0900
categories: [dev]
tags: [Claude Code, Google Drive, MCP, OAuth, Google Cloud, AI 에이전트]
excerpt: >-
  Aside에 연결했던 Google Drive MCP를 Claude Code에도 붙이려면 서버 URL만 복사해서는 안 된다. Claude Code 전용 OAuth Client, localhost 콜백, user scope 등록이 필요하다. 실제 CLI가 만든 인증 URL과 설정 파일을 확인해 정확한 명령과 검증 방법을 정리한다.
---

> **공식 Google Drive MCP 연결 시리즈**
> 1. [Aside에 공식 Google Drive MCP 연결하기: OAuth 콜백부터 Developer Preview까지](/dev/2026/09/19/aside에-공식-google-drive-mcp-연결하기-oauth-콜백부터-developer-preview까지/)
> 2. [Google Drive MCP를 팀에 배포하기: Internal, External, OAuth 검증의 경계](/dev/2026/09/19/google-drive-mcp를-팀에-배포하기-internal-external-oauth-검증의-경계/)
> 3. **Claude Code에 공식 Google Drive MCP 연결하기: user scope와 OAuth 콜백** (현재 글)

> 이 글은 2026년 9월 28일 Claude Code 2.1.283과 Google Drive MCP Developer Preview를 기준으로 작성했다. `mcp add` 명령, 실제 OAuth 인증 URL, 설정 저장 위치와 인증 전 서버 응답은 실측했다. 실제 Google 계정으로 Claude Code OAuth 승인을 끝까지 완료하는 단계는 아직 실행하지 않았다.

## 결론부터

이미 [Aside 연결에 사용한 Google Cloud 프로젝트](/dev/2026/09/19/aside에-공식-google-drive-mcp-연결하기-oauth-콜백부터-developer-preview까지/)가 있다면 프로젝트와 API 설정은 그대로 쓸 수 있다. Google Drive MCP(Model Context Protocol, 모델 컨텍스트 프로토콜) 서버도 같다. 다만 Claude Code용 OAuth(Open Authorization, 권한 위임 표준) Client는 따로 만드는 편이 낫다.

핵심값은 다음과 같다.

| 항목 | 값 |
|---|---|
| MCP 서버 URL | `https://drivemcp.googleapis.com/mcp/v1` |
| 전송 방식 | HTTP(Streamable HTTP) |
| Claude Code scope | `user` |
| OAuth Client 유형 | Web application |
| OAuth 콜백 URI | `http://localhost:3118/callback` |
| 서버 이름 | `google-drive` |

등록과 로그인 명령은 다음 두 개다.

```bash
claude mcp add \
  --transport http \
  --scope user \
  --client-id '<OAUTH_CLIENT_ID>' \
  --client-secret \
  --callback-port 3118 \
  google-drive \
  https://drivemcp.googleapis.com/mcp/v1

claude mcp login google-drive
```

여기서 `--client-secret` 뒤에 Secret 값을 붙이지 않는다. 이 옵션은 Claude Code가 Secret을 별도로 입력받게 하는 스위치다.

그리고 상태 표시만 믿으면 안 된다. `claude mcp get google-drive`가 `Connected`라고 해도 Google Drive 인증은 아직 안 끝났을 수 있다. 실제 파일 조회까지 해봐야 한다.

---

## Aside 설정을 그대로 복사하면 안 되는 이유

MCP 서버 URL은 같다. OAuth 콜백은 다르다.

| 클라이언트 | 콜백 URI |
|---|---|
| Aside | `http://127.0.0.1:21420/mcp/oauth/callback` |
| Claude Code 2.1.283 | `http://localhost:3118/callback` |

OAuth 서버는 Redirect URI를 문자열로 비교한다. 다음 차이는 모두 의미가 있다.

- `localhost`와 `127.0.0.1`
- 포트 `3118`과 `21420`
- 경로 `/callback`과 `/mcp/oauth/callback`

Aside용 Client에 Claude Code 콜백 URI를 추가하면 하나의 OAuth Client를 같이 쓰는 것도 기술적으로는 가능하다. 운영 측면에서는 분리를 권한다.

```text
하나의 Google Cloud 프로젝트
├── Aside용 OAuth Client
│   └── http://127.0.0.1:21420/mcp/oauth/callback
└── Claude Code용 OAuth Client
    └── http://localhost:3118/callback
```

분리하면 한쪽 Secret을 교체하거나 Client를 폐기해도 다른 도구의 연결은 끊기지 않는다. Google Cloud Console에서도 어느 Client가 어느 도구의 것인지 바로 구별할 수 있다.

OAuth 토큰도 복사하지 않는다. Aside가 발급받은 토큰은 Aside가 관리하고, Claude Code 토큰은 Claude Code가 새로 발급받아 관리한다.

---

## 사전 조건

Google Drive MCP는 이 글을 쓴 시점에 Developer Preview다. Claude Code 설정 전에 Google 쪽 조건부터 충족해야 한다.

1. [Google Workspace Developer Preview Program](https://developers.google.com/workspace/preview)에 계정과 Cloud 프로젝트를 등록한다.
2. Google Cloud 프로젝트에서 Google Drive API를 활성화한다.
3. 같은 프로젝트에서 Google Drive MCP API를 활성화한다.
4. OAuth 동의 화면을 구성한다.
5. 실제 로그인할 계정이 Preview와 OAuth Audience 조건에 맞는지 확인한다.

이 과정은 [시리즈 1편](/dev/2026/09/19/aside에-공식-google-drive-mcp-연결하기-oauth-콜백부터-developer-preview까지/)에 자세히 적었다. Google의 공식 설정 문서도 Drive MCP가 Developer Preview이며 OAuth 2.0을 사용한다고 안내한다.

---

## 1. Claude Code 버전과 명령을 확인한다

내 환경에서 확인한 버전은 다음과 같다.

```bash
claude --version
# 2.1.283 (Claude Code)
```

`claude` 명령을 찾지 못한다면 설치 경로가 `PATH`에 없는지 먼저 확인한다. 현재 설치본은 다음 위치에 있었다.

```text
~/.local/bin/claude
```

현재 셸에서만 경로를 추가하려면 다음처럼 실행한다.

```bash
export PATH="$HOME/.local/bin:$PATH"
claude --version
```

MCP 명령의 현재 옵션은 설치된 CLI에서 직접 확인하는 것이 가장 정확하다.

```bash
claude mcp --help
claude mcp add --help
claude mcp login --help
```

2.1.283의 `mcp add`에는 다음 옵션이 있다.

```text
--callback-port <port>
--client-id <clientId>
--client-secret
--scope <local|user|project>
--transport <stdio|sse|http>
```

---

## 2. Google Cloud에 Claude Code용 OAuth Client를 만든다

Google Auth Platform의 `Clients`에서 OAuth Client를 새로 만든다.

| 설정 | 값 |
|---|---|
| Application type | `Web application` |
| Name | 예: `Claude Code Google Drive MCP` |
| Authorized redirect URI | `http://localhost:3118/callback` |

승인된 자바스크립트 원본은 필요하지 않다.

### 왜 `localhost`인가

Claude Code 2.1.283에서 `--callback-port 3118`을 지정하고 실제 인증 URL을 출력해봤다. URL에는 다음 값이 들어갔다.

```text
redirect_uri=http://localhost:3118/callback
```

따라서 이 버전에서는 `127.0.0.1`이 아니라 `localhost`를 Google Cloud에 등록해야 한다. 이전 버전이나 이후 버전에서 구현이 바뀔 수 있으므로, `redirect_uri_mismatch`가 나면 추측하지 말고 Claude Code가 실제로 만든 인증 URL을 확인해야 한다.

생성 후 Client ID와 Client Secret이 나온다. 둘을 문서나 Git 저장소에 넣지 않는다.

---

## 3. user scope로 서버를 등록한다

Claude Code의 MCP scope는 세 가지다.

| scope | 적용 범위 | 이 연결에서의 판단 |
|---|---|---|
| `local` | 현재 프로젝트에서 현재 사용자만 사용 | 프로젝트마다 다시 등록해야 함 |
| `user` | 현재 사용자의 모든 프로젝트 | 권장 |
| `project` | 프로젝트 구성원과 설정 공유 | OAuth Client 설정을 저장소에 넣을 이유가 없음 |

Google Drive는 특정 코드 저장소의 도구가 아니라 개인 또는 회사 계정에 연결된 사용자 도구다. 따라서 `user`가 맞다.

```bash
claude mcp add \
  --transport http \
  --scope user \
  --client-id '<OAUTH_CLIENT_ID>' \
  --client-secret \
  --callback-port 3118 \
  google-drive \
  https://drivemcp.googleapis.com/mcp/v1
```

명령을 실행하면 Client Secret 입력을 요청한다. Secret을 다음처럼 명령줄에 직접 적지 않는다.

```bash
# 이렇게 하지 않는다
claude mcp add ... --client-secret '실제-Secret' ...
```

`--client-secret`은 값을 받는 옵션이 아니라 안전한 입력 절차를 켜는 옵션이다.

### 설정 파일에는 무엇이 남나

격리된 임시 사용자 환경에서 등록한 뒤 `~/.claude.json`을 확인했다. 사용자 범위의 서버 정의는 대략 다음 모양이었다.

```json
{
  "mcpServers": {
    "google-drive": {
      "type": "http",
      "url": "https://drivemcp.googleapis.com/mcp/v1",
      "oauth": {
        "clientId": "<OAUTH_CLIENT_ID>",
        "callbackPort": 3118
      }
    }
  }
}
```

Client Secret은 이 JSON에 없었다. Claude Code의 secure storage에 따로 저장되며, macOS에서는 Keychain을 사용한다.

이 구조도 `user` scope를 권하는 이유다. 프로젝트의 `.mcp.json`에 넣고 공유할 설정이 아니다.

---

## 4. Google 계정으로 로그인한다

서버 등록 후 로그인한다.

```bash
claude mcp login google-drive
```

브라우저를 자동으로 열 수 없는 환경에서는 다음 옵션을 쓴다.

```bash
claude mcp login --no-browser google-drive
```

Claude Code가 인증 URL을 출력한다. 브라우저에서 승인한 뒤 로컬 콜백이 실패하면, 주소창의 전체 Redirect URL을 Claude Code 프롬프트에 붙여 넣을 수 있다.

실제 생성된 인증 URL에서 확인한 OAuth 범위는 다음 세 개였다.

```text
https://www.googleapis.com/auth/drive
https://www.googleapis.com/auth/drive.readonly
https://www.googleapis.com/auth/drive.file
```

`drive`가 포함돼 있다는 점이 중요하다. 이 연결을 단순한 읽기 전용 연결로 보면 안 된다. 최종 승인 화면에 표시되는 권한을 읽고, 조직 정책상 허용되는지 확인한 뒤 승인해야 한다.

---

## 5. `Connected`만으로 성공 판정하지 않는다

여기서 가장 헷갈리기 쉬운 지점이 있었다.

임시 환경에 가짜 Client ID와 Secret을 넣고 서버만 등록한 뒤 다음 명령을 실행했다.

```bash
claude mcp get google-drive
```

Claude Code는 다음과 같이 표시했다.

```text
Status: Connected
OAuth: client_id configured, client_secret configured, callback_port 3118
```

하지만 Google 계정 인증은 하지 않은 상태였다.

이유는 Drive MCP 서버의 인증 경계가 도구 목록과 실제 파일 호출 사이에 있기 때문이다. 직접 요청했을 때의 결과는 다음과 같았다.

| 요청 | 인증 없는 응답 |
|---|---:|
| MCP `initialize` | 성공 |
| `tools/list` | 성공 |
| 실제 `get_file_metadata` 호출 | `401 Unauthorized` |

즉 `Connected`는 다음만 증명한다.

```text
Claude Code가 MCP 서버에 접속해 프로토콜 초기화를 할 수 있다.
```

다음은 증명하지 않는다.

```text
Google 계정 OAuth가 끝났고 실제 Drive 파일을 읽을 수 있다.
```

그래서 검증은 두 단계로 나눠야 한다.

```bash
# 1. 전송과 서버 설정 확인
claude mcp get google-drive
claude mcp list

# 2. OAuth 완료
claude mcp login google-drive
```

그다음 Claude Code 새 세션에서 실제 읽기 요청을 한다.

```text
Google Drive MCP로 내 최근 파일 3개를 조회해.
브라우저 자동화는 사용하지 마.
```

또는 이름을 알고 있는 테스트 문서를 검색한다.

```text
Google Drive MCP의 search_files를 사용해서
제목에 "테스트"가 들어간 파일을 찾아줘.
```

실제 Drive 데이터가 반환돼야 연결 완료다.

---

## 6. 새 세션에서 시험한다

MCP 서버를 추가한 뒤 이미 열려 있던 Claude Code 세션에는 새 도구가 바로 반영되지 않을 수 있다. 연결과 로그인을 마친 뒤 새 세션을 시작한다.

```bash
claude
```

처음부터 쓰기 도구로 시험하지 않는다. 다음 순서가 안전하다.

1. `list_recent_files`로 최근 파일 조회
2. `search_files`로 제목 검색
3. `get_file_metadata`로 테스트 파일 확인
4. `read_file_content`로 내용 읽기
5. 필요한 경우에만 별도 테스트 파일을 생성하거나 복사

Google Drive MCP에는 파일 생성과 복사 도구도 있다. 읽기 연결이라고 생각하고 기존 파일에 바로 쓰기 시험을 하면 안 된다.

---

## 7. 오류별 확인 순서

### `redirect_uri_mismatch`

Google Cloud의 Authorized redirect URI가 다음 값과 정확히 같은지 확인한다.

```text
http://localhost:3118/callback
```

`127.0.0.1`을 등록했거나 경로를 `/mcp/oauth/callback`으로 적었다면 Claude Code용 콜백이 아니다.

### `OAuth callback port already in use`

3118번 포트를 다른 프로세스가 쓰는지 확인한다.

```bash
lsof -nP -iTCP:3118 -sTCP:LISTEN
```

포트를 바꾸려면 두 값을 같이 바꾼다.

1. Google Cloud의 Authorized redirect URI
2. `claude mcp add`의 `--callback-port`

### `Unauthorized`

저장된 OAuth 자격증명을 지우고 다시 로그인한다.

```bash
claude mcp logout google-drive
claude mcp login google-drive
```

로그인한 Google 계정이 Preview에 등록한 계정과 같은지도 확인한다.

### `The caller does not have permission`

다음 네 가지를 확인한다.

- Google Workspace Developer Preview 승인
- Google Drive API 활성화
- Google Drive MCP API 활성화
- Preview에 등록한 계정과 프로젝트의 일치

이 오류는 특정 파일의 공유 문제보다 앞단에서 발생할 수 있다. `list_recent_files`처럼 파일 ID가 필요 없는 호출도 같은 오류가 나는지 확인하면 구분하기 쉽다.

### `claude: command not found`

```bash
export PATH="$HOME/.local/bin:$PATH"
```

이후 `claude --version`으로 다시 확인한다.

---

## 8. 재인증과 제거

OAuth 토큰만 다시 받으려면 다음 두 명령을 쓴다.

```bash
claude mcp logout google-drive
claude mcp login google-drive
```

서버 정의까지 지우려면 사용자 범위를 지정해 제거한다.

```bash
claude mcp remove google-drive --scope user
```

완전히 폐기할 때는 세 곳을 정리한다.

1. Claude Code의 MCP 서버와 OAuth 자격증명
2. Google 계정의 서드파티 앱 접근 권한
3. Google Cloud의 Claude Code용 OAuth Client

OAuth Client를 Aside와 분리해두면 이때 Aside 연결에는 영향이 없다.

---

## 최종 체크리스트

- [ ] Workspace Developer Preview 승인 완료
- [ ] Drive API와 Drive MCP API 활성화
- [ ] Claude Code 전용 Web application OAuth Client 생성
- [ ] `http://localhost:3118/callback` 등록
- [ ] `--scope user`로 MCP 서버 등록
- [ ] Client Secret을 명령 인자나 저장소에 기록하지 않음
- [ ] `claude mcp login google-drive` 실행
- [ ] 승인 화면에서 실제 Drive 권한 확인
- [ ] 새 Claude Code 세션에서 실제 파일 읽기 성공
- [ ] `Connected` 표시만으로 완료 판정하지 않음

---

## 아직 확인하지 않은 것

이 글에서 Claude Code 2.1.283의 명령 구문과 실제 인증 URL은 확인했다. 격리 환경에서 설정 파일과 Secret 분리 저장도 확인했다. Drive MCP 서버가 인증 전에는 도구 목록을 주고 실제 호출에서 401을 반환하는 것도 확인했다.

다음은 아직 Claude Code에서 실제 계정으로 끝까지 실행하지 않았다.

- Google 로그인과 동의 완료
- authorization code와 token 교환 성공
- Claude Code 안에서 실제 Drive 파일 조회
- 최종 동의 화면에 표시되는 권한 문구

따라서 이 글은 **실측한 설정 가이드**이지만 **Claude Code 연결 성공기**는 아니다. 실제 연결을 마치면 이 절과 제목의 표현을 갱신할 예정이다.

---

## 참고 자료

- [Google Drive MCP 서버 구성](https://developers.google.com/workspace/drive/api/guides/configure-mcp-server)
- [Google Drive MCP Reference](https://developers.google.com/workspace/drive/api/reference/mcp)
- [Google Workspace Developer Preview Program](https://developers.google.com/workspace/preview)
- [Claude Code: Connect to MCP servers](https://code.claude.com/docs/en/mcp-quickstart)
