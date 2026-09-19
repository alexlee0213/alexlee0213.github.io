---
layout: single
title: "Aside에 공식 Google Drive MCP 연결하기: OAuth 콜백부터 Developer Preview까지"
date: 2026-09-19 12:00:00 +0900
categories: [dev]
tags: [Aside, Google Drive, MCP, OAuth, Google Cloud, AI 에이전트]
excerpt: >-
  Aside에 Google의 공식 Drive MCP 서버를 붙였다. API와 OAuth만 설정하면 끝날 줄 알았지만 Developer Preview 승인, 고정 콜백 URI, 재인증, 새 채팅 반영이라는 네 개의 관문이 더 있었다. 실제 오류 흐름과 처음부터 다시 한다면 밟을 순서를 정리한다.
---

> **Aside에 Google Drive MCP 붙이기 시리즈**
> 1. **Aside에 공식 Google Drive MCP 연결하기: OAuth 콜백부터 Developer Preview까지** (현재 글)
> 2. [Google Drive MCP를 팀에 배포하기: Internal, External, OAuth 검증의 경계](/dev/2026/09/19/google-drive-mcp를-팀에-배포하기-internal-external-oauth-검증의-경계/)

> 이 글은 2026년 9월 19일, Aside 1.26.914와 Google Drive MCP Developer Preview를 기준으로 작성했다. Preview 기능과 Aside 화면 이름은 이후 바뀔 수 있다.

## 결론부터

Aside에 Google의 공식 Drive MCP(Model Context Protocol, 모델 컨텍스트 프로토콜) 서버를 연결할 수 있다. 연결에 필요한 핵심값은 다음과 같다.

| 항목 | 값 |
|---|---|
| 서버 URL | `https://drivemcp.googleapis.com/mcp/v1` |
| 전송 방식 | Streamable HTTP |
| 인증 | OAuth 2.0 |
| OAuth 클라이언트 유형 | Web application |
| Aside 콜백 URI | `http://127.0.0.1:21420/mcp/oauth/callback` |
| 권장 서버 이름 | `GoogleDrive` |

하지만 URL 하나 넣고 Google 로그인만 하면 끝나는 연결은 아니었다.

1. Google Workspace Developer Preview Program에 계정과 Cloud 프로젝트를 등록한다.
2. Google Cloud 프로젝트에서 Google Drive API와 Drive MCP API를 활성화한다.
3. OAuth 동의 화면과 Drive 범위를 설정한다.
4. Aside의 고정 콜백 URI로 Web application OAuth 클라이언트를 만든다.
5. Aside의 MCP 설정에 URL, Client ID, Client Secret을 입력하고 인증한다.
6. Preview 승인 전 발급한 토큰이 있다면 `Reconnect`로 재인증한다.
7. 새 Aside 채팅에서 Drive 도구가 나타나는지 확인한다.

이 순서를 지키면 된다. 나는 1번을 나중에 발견해서, 정상처럼 보이는 설정을 두고 권한 오류를 오래 추적했다.

---

## 왜 Drive MCP를 붙였나

연결 전에는 Aside가 Google Drive를 다루는 방법이 브라우저 자동화였다. 웹 화면을 열고 폴더를 이동하고, 메뉴를 누르고, 파일 선택기를 통해 업로드했다. 사람이 하는 일을 대신 클릭하는 방식이다.

브라우저 자동화가 나쁜 것은 아니다. Drive 웹에서만 가능한 작업도 있고, 사용자가 보는 화면과 같은 상태를 확인할 수 있다는 장점도 있다. 다만 파일 검색이나 메타데이터 조회처럼 API에 잘 맞는 작업까지 화면으로 처리하면 느리고 UI 변경에 취약하다.

공식 Drive MCP를 연결하면 에이전트가 다음과 같은 일을 도구 호출로 수행할 수 있다.

- 최근 파일 목록 조회
- 제목과 본문을 대상으로 파일 검색
- 파일 메타데이터와 권한 확인
- Google Docs, Sheets, Slides 등의 자연어 표현 읽기
- 파일 다운로드
- 파일 생성과 복사

내 환경에서는 연결 후 다음 8개 도구가 확인됐다.

```text
copy_file
create_file
download_file_content
get_file_metadata
get_file_permissions
list_recent_files
read_file_content
search_files
```

중요한 경계도 있다. 이 연결은 Drive 전용이다. Google Docs의 구조와 서식을 직접 수정하는 기능까지 필요하다면 별도의 Docs MCP나 다른 편집 경로가 필요하다.

---

## 연결 구조

구성요소는 네 개다.

```text
Aside
  -> Google의 공식 Drive MCP 서버
      -> Google OAuth 2.0
          -> 사용자의 Google Drive
```

여기서 Google Cloud의 OAuth 클라이언트는 별도 앱을 개발하기 위한 것이 아니다. Aside라는 MCP 클라이언트가 어떤 애플리케이션 신원으로 Google 로그인을 요청할지 정의하는 인증 설정이다.

헷갈렸던 지점은 클라이언트 유형이었다. Aside는 데스크톱 앱이지만 Google Cloud에서 만드는 OAuth 클라이언트는 `Desktop app`이 아니라 `Web application`이어야 한다. 승인 후 브라우저가 Aside의 로컬 콜백 주소로 authorization code를 돌려주기 때문이다.

```text
http://127.0.0.1:21420/mcp/oauth/callback
```

`127.0.0.1`은 Google 서버가 내 컴퓨터에 직접 접속한다는 뜻이 아니다. 로그인과 동의를 마친 브라우저가 로컬에서 실행 중인 Aside로 결과를 전달한다.

### 콜백 포트 `21420`은 누가 정하는가

`21420`은 Google Cloud에서 사용자가 임의로 고르는 포트가 아니다. 현재 확인한 Aside 1.26.914에서는 Aside 로컬 데몬의 기본 포트가 `21420`으로 정해져 있고, OAuth 콜백 경로도 `/mcp/oauth/callback`으로 정해져 있다.

```text
호스트: 127.0.0.1
기본 포트: 21420
콜백 경로: /mcp/oauth/callback

최종 URI:
http://127.0.0.1:21420/mcp/oauth/callback
```

설정 방향을 반대로 이해하면 안 된다.

```text
Aside가 사용할 콜백 주소를 먼저 결정한다.
  -> 사용자가 Google Cloud의 Authorized redirect URI를 그 주소에 맞춘다.
```

Google Cloud에 임의의 포트를 등록하면 Aside가 그 값을 읽어 자동으로 맞추는 구조가 아니다. 예를 들어 다음 주소를 등록해도 Aside는 기본적으로 `3000` 포트에서 인증 결과를 기다리지 않는다.

```text
http://127.0.0.1:3000/mcp/oauth/callback
```

정상 흐름은 다음과 같다.

1. Google Cloud OAuth 클라이언트에 Aside의 정확한 콜백 URI를 등록한다.
2. 같은 OAuth 클라이언트의 Client ID와 Client Secret을 Aside MCP 설정에 입력한다.
3. Aside에서 `Connect` 또는 `Reconnect`를 실행한다.
4. Google 인증을 마친 브라우저가 authorization code를 로컬 콜백으로 전달한다.
5. `21420` 포트에서 실행 중인 Aside가 code를 받아 토큰 교환을 완료한다.

콜백 URI 등록만으로 연결되는 것은 아니다. Google Cloud와 Aside 양쪽에 같은 OAuth 클라이언트 정보가 맞아야 하고, Aside에서 실제 연결 절차를 시작해야 한다.

내부적으로 Aside는 `ASIDE_PORT` 환경변수가 있으면 그 값을 사용할 수 있고, 없으면 `21420`을 사용한다. 그러나 일반적인 Aside 앱 실행에서는 사용자가 따로 정할 설정값으로 보지 않는 편이 맞다. 포트를 바꾸면 OAuth 콜백뿐 아니라 로컬 데몬 연결에도 영향을 줄 수 있으므로, 특별히 Aside 실행 환경을 직접 관리하는 경우가 아니라면 기본값을 그대로 사용한다.

---

## 내가 실제로 겪은 순서

### 1. API와 OAuth부터 설정했다

처음에는 Google Cloud 프로젝트를 만들고 두 API를 켰다.

- Google Drive API
- Google Drive MCP API

OAuth 동의 화면을 구성하고 `drive.readonly`, `drive.file` 범위를 추가한 뒤, Aside 콜백 URI를 가진 Web application 클라이언트를 만들었다. Aside에는 MCP 서버 URL, Client ID, Client Secret을 입력했다.

연결 자체는 성공해 보였다. Aside가 서버를 인식했고 8개 도구도 가져왔다.

### 2. 실제 호출은 `The caller does not have permission`

Drive 폴더 ID로 메타데이터를 조회했지만 다음 오류가 났다.

```text
The caller does not have permission
```

처음에는 폴더 공유 문제라고 생각했다. 그러나 같은 계정으로 브라우저에서는 폴더가 열렸고, 특정 폴더와 무관한 `list_recent_files`도 같은 오류를 냈다.

이 두 사실을 같이 보면 폴더의 접근제어목록(ACL, Access Control List)보다 앞단의 문제다.

- 브라우저에서 같은 계정으로 대상 폴더 접근 가능
- 특정 폴더를 요구하지 않는 최근 파일 조회도 실패

### 3. 빠진 것은 Developer Preview 등록이었다

Google의 공식 Drive MCP 서버는 당시 일반 공개 기능이 아니라 Google Workspace Developer Preview Program 대상이었다. API를 활성화하고 OAuth 동의를 끝냈더라도, Workspace 계정과 Google Cloud 프로젝트를 Preview 프로그램에 등록해 승인받아야 했다.

여기서 중요한 구분이 있다.

| 설정 | 역할 |
|---|---|
| Drive API 활성화 | 기존 Google Drive API 사용 허용 |
| Drive MCP API 활성화 | Drive MCP 서비스 호출 허용 |
| OAuth 클라이언트 | Aside가 사용자 동의를 받을 앱 신원 |
| Developer Preview 등록 | 해당 계정과 프로젝트가 Preview MCP를 쓸 자격 |

앞의 셋이 맞아도 마지막 하나가 빠지면 호출은 실패할 수 있다.

### 4. 승인 후 오류가 `Unauthorized`로 바뀌었다

Google에서 프로젝트 등록 완료 메일을 받은 직후 다시 호출했다. 이번에는 오류가 이렇게 바뀌었다.

```text
Unauthorized
```

오류가 바뀐 것은 유용한 단서였다. Preview 자격 문제는 해소됐지만, 승인 전에 받은 기존 OAuth 토큰은 새 상태를 반영하지 못했다.

Aside 설정에서 기존 서버의 actions 메뉴를 열고 `Reconnect`를 선택해 다시 로그인했다. 이때 상단의 일반 `Connect` 버튼은 새 MCP 서버를 추가하는 대화상자를 여는 버튼이었다. 이미 등록된 서버를 재인증할 때는 기존 항목의 `Reconnect`를 써야 했다.

### 5. 재인증 후 직접 조회에 성공했다

재연결 후 `get_file_metadata`로 기존 Drive 폴더를 조회했고, 폴더 이름과 하위 항목이 정상적으로 반환됐다. 별도 공유도, Drive 웹 화면도 필요하지 않았다.

최종 오류 흐름은 다음과 같았다.

```text
Preview 미등록
  -> The caller does not have permission

Preview 승인, 기존 토큰 유지
  -> Unauthorized

Aside에서 Reconnect 후 재인증
  -> 정상 조회
```

---

## 처음부터 다시 한다면 밟을 순서

실제 경험 순서가 아니라, 재작업을 줄이는 권장 순서다.

### 1. Developer Preview부터 신청한다

먼저 [Google Workspace Developer Preview Program](https://developers.google.com/workspace/preview)에 가입한다.

신청에는 대체로 다음 정보가 필요하다.

- 사용할 Google Workspace 계정
- Google Cloud 프로젝트 이름
- 프로젝트 ID
- 프로젝트 번호

프로젝트가 아직 없다면 먼저 하나 만든 뒤 신청한다. 새 프로젝트를 별도로 만드는 것은 필수가 아니다. 기존 프로젝트를 써도 된다. 다만 나는 다음 이유로 전용 프로젝트를 권한다.

- OAuth 클라이언트와 Secret의 용도가 명확해진다.
- API 사용량과 오류를 따로 볼 수 있다.
- 나중에 연결을 폐기할 때 영향 범위를 줄일 수 있다.
- 조직 감사 시 무엇을 위한 프로젝트인지 설명하기 쉽다.

Preview 등록은 계정과 프로젝트를 함께 본다. 프로젝트만 맞고 로그인 계정이 다르거나, 반대인 경우에도 실패 원인이 될 수 있다.

### 2. 두 API를 활성화한다

Google Cloud Console의 API Library에서 다음 두 서비스를 활성화한다.

```text
Google Drive API
Google Drive MCP API
```

전용 프로젝트를 새로 만들었다면 API도 그 프로젝트에서 다시 활성화해야 한다. 다른 프로젝트에서 켠 API는 OAuth 클라이언트가 속한 새 프로젝트로 따라오지 않는다.

### 3. OAuth 동의 화면을 구성한다

Google Auth Platform에서 다음을 설정한다.

### Branding

- 앱 이름: 예를 들어 `Aside Google Drive MCP`
- 사용자 지원 이메일
- 개발자 연락처 이메일

### Audience

- 조직의 Google Workspace 계정만 쓸 경우: `Internal`
- 개인 Gmail이나 조직 밖 계정도 쓸 경우: `External`

`External`의 테스트 단계라면 실제 로그인할 계정을 Test users에 추가해야 한다.

### Data Access

Google의 Drive MCP 설정 문서는 다음 두 범위를 수동으로 추가하도록 안내한다.

```text
https://www.googleapis.com/auth/drive.readonly
https://www.googleapis.com/auth/drive.file
```

- `drive.readonly`: 사용자가 접근할 수 있는 Drive 파일 읽기
- `drive.file`: 앱이 만들거나 앱으로 연 파일에 대한 접근

단, 실제 동의 화면은 반드시 읽어야 한다. 내 연결에서는 위 두 범위를 설정했지만 OAuth 승인 화면에 전체 `drive` 범위에 해당하는 것으로 보이는 광범위한 권한이 함께 나타났고, 모든 Drive 파일을 보기, 수정, 생성, 삭제할 수 있다고 표시됐다.

따라서 이 연결을 단순한 읽기 전용 연결이라고 설명하면 안 된다. 승인 화면에 표시된 최종 권한을 기준으로 판단해야 한다.

### 4. Web application OAuth 클라이언트를 만든다

Google Auth Platform의 `Clients`에서 새 OAuth 클라이언트를 만든다.

| 항목 | 설정 |
|---|---|
| Application type | `Web application` |
| Name | 예: `Aside Google Drive MCP` |
| Authorized redirect URI | `http://127.0.0.1:21420/mcp/oauth/callback` |

생성 후 Client ID와 Client Secret이 나온다.

보안 수칙은 단순하다.

- Client Secret을 글, 스크린샷, Git 저장소에 넣지 않는다.
- JSON 설정 파일에 평문으로 복사해 두지 않는다.
- 사용하지 않는 이전 Secret은 새 연결 검증 후 비활성화하거나 삭제한다.
- 여러 클라이언트가 같은 Cloud 프로젝트를 쓰더라도 OAuth 클라이언트는 도구별로 분리한다.

마지막 항목은 필수 조건은 아니지만 운영이 편하다. Aside와 다른 도구가 같은 Secret을 쓰면 하나를 교체할 때 다른 쪽도 같이 끊긴다.

### 5. Aside에 MCP 서버를 등록한다

Aside에서 `Settings > Plugins & MCPs > MCPs`로 이동한다. 버전에 따라 메뉴 문구는 조금 다를 수 있다.

새 MCP 서버를 다음처럼 등록한다.

| Aside 항목 | 입력값 |
|---|---|
| Name | `GoogleDrive` |
| Transport | `Streamable HTTP` |
| URL | `https://drivemcp.googleapis.com/mcp/v1` |
| Authentication | OAuth 또는 Auto |
| Client ID | Google Cloud에서 만든 Client ID |
| Client Secret | Advanced settings의 Secret 입력란 |

서버 이름은 영문, 숫자, 밑줄, 하이픈만 쓰는 편이 안전하다. `GoogleDrive`처럼 공백 없이 두면 도구 이름도 식별하기 쉽다.

Aside에서는 Client ID가 일반 MCP 구성에 포함되지만 Client Secret은 별도의 보안 저장소에 보관된다. Secret을 일반 설정 JSON이나 메모에 옮겨 적는 방식보다 UI의 Secret 입력란을 쓰는 이유다.

### 6. Google 계정으로 인증한다

연결을 시작하면 브라우저에서 Google 로그인과 권한 동의 화면이 열린다.

여기서 확인할 것은 세 가지다.

1. Preview에 등록한 계정으로 로그인했는가
2. 의도한 Workspace 조직의 계정인가
3. 동의 화면의 실제 권한이 예상 범위와 일치하는가

내 경우 세 번째에서 공식 안내보다 넓어 보이는 권한이 나타났다. 편의상 넘길 항목이 아니다. Drive 전체 접근이 부담스럽다면 연결을 중단하고 조직 정책과 OAuth 로그를 먼저 확인하는 편이 맞다.

### 7. 연결 상태와 도구 목록을 확인한다

연결 직후 Aside가 공식 도구 8개를 가져오는지 확인한다.

```text
copy_file
create_file
download_file_content
get_file_metadata
get_file_permissions
list_recent_files
read_file_content
search_files
```

도구 목록이 보이는 것은 서버 메타데이터를 읽었다는 뜻이지, 실제 Drive 데이터 호출까지 성공했다는 뜻은 아니다. 반드시 읽기 호출로 검증해야 한다.

### 8. 새 채팅에서 스모크 테스트한다

새로 연결한 MCP 도구는 이미 열려 있던 Aside 채팅에 즉시 생기지 않을 수 있다. MCP 서버를 연결한 뒤 새 채팅을 열어 다음 순서로 시험한다.

1. `list_recent_files`로 최근 파일 조회
2. `search_files`로 제목 검색
3. 알고 있는 파일이나 폴더 ID로 `get_file_metadata`
4. 문서 하나를 `read_file_content`로 읽기

쓰기 도구는 읽기 검증 후 별도의 테스트 파일로 확인한다. 기존 문서를 대상으로 바로 쓰기 테스트를 하지 않는다.

---

## 오류별 판별법

### `The caller does not have permission`

무조건 폴더 공유부터 바꾸지 않는다. 먼저 오류 범위를 넓혀 본다.

```text
특정 폴더만 실패하는가?
  -> 폴더 ACL이나 공유 계정 확인

list_recent_files도 실패하는가?
  -> Preview 등록, API 활성화, Cloud 프로젝트, OAuth 계정 확인
```

특정 리소스가 필요 없는 호출까지 같은 오류라면 개별 폴더 권한 문제가 아닐 가능성이 높다.

### `Unauthorized`

OAuth 연결이 없거나 기존 토큰이 더 이상 유효하지 않은 상태다.

- 기존 GoogleDrive 서버의 actions 메뉴 열기
- `Reconnect` 선택
- 의도한 Google 계정으로 다시 로그인
- 새 채팅에서 재시험

Preview 승인이나 OAuth 설정 변경 뒤에는 기존 토큰을 계속 쓰지 말고 재인증하는 편이 빠르다.

### 도구는 보이는데 현재 채팅에서 호출할 수 없다

새 Aside 채팅을 연다. MCP 도구 목록은 채팅이 시작될 때 결정될 수 있어 기존 세션에 자동 반영되지 않는다.

### `Don't set a verbosity for the snippets and exclude them.`

`search_files`나 `list_recent_files`에서 다음 두 옵션을 동시에 줬을 때 난다.

```text
excludeContentSnippets: true
snippetVerbosity: ...
```

스니펫을 제외한다면 `snippetVerbosity`를 생략한다.

### 특정 폴더만 안 열린다

이때는 실제 폴더 권한을 본다.

- MCP에 인증한 계정과 브라우저에서 연 계정이 같은가
- 공유 드라이브 정책이 외부 앱 접근을 막는가
- 바로가기 대상 원본에 권한이 있는가
- 조직 관리자가 OAuth 앱을 제한했는가

먼저 계정과 전역 호출을 확인한 다음 폴더 공유를 바꾸는 순서가 안전하다.

---

## 설정에서 배운 것

### 1. 연결 성공과 도구 성공은 다르다

Aside가 서버 URL을 받아들이고 도구 목록을 가져왔어도, 실제 도구 호출은 권한 문제로 실패할 수 있다. MCP 연결 검증은 최소 세 층으로 나눠야 한다.

| 층 | 확인 방법 |
|---|---|
| 서버 등록 | Aside에 MCP 항목이 남아 있는가 |
| 도구 탐색 | 8개 도구가 보이는가 |
| 데이터 접근 | `list_recent_files`나 `get_file_metadata`가 성공하는가 |

첫째와 둘째만 보고 연결이 끝났다고 판단하면 안 된다.

### 2. OAuth 오류는 메시지 변화가 상태 전이를 말해준다

`permission`에서 `Unauthorized`로 바뀐 것은 같은 실패의 반복이 아니었다.

- `permission`: 프로젝트 또는 계정의 Preview 자격을 의심
- `Unauthorized`: 자격은 열렸지만 토큰 재발급이 필요함

오류 문구가 달라졌다면 이전 가설을 그대로 밀지 말고, 어느 인증 단계까지 통과했는지 다시 그리는 편이 빠르다.

### 3. 전용 Cloud 프로젝트는 선택이지만 값어치가 있다

기술적으로는 기존 프로젝트를 재사용할 수 있다. 그러나 인증 자격 증명, API 사용량, 감사 범위를 분리하려면 전용 프로젝트가 낫다. MCP를 제거할 때 프로젝트 단위로 정리할 수 있다는 것도 장점이다.

### 4. 공식 범위 설명보다 실제 동의 화면이 우선이다

문서에 `drive.readonly`와 `drive.file`이 적혀 있어도, 사용자가 승인하는 최종 화면이 더 넓다면 실제 보안 판단은 그 화면을 기준으로 해야 한다.

이 연결은 내 환경에서 모든 Drive 파일의 보기, 수정, 생성, 삭제가 가능하다고 표시됐다. 최소 권한이라고 부를 수 없었다. 업무용 Drive라면 특히 조직 관리자와 범위를 확인할 필요가 있다.

### 5. Secret은 설정값이 아니라 수명주기가 있는 자격증명이다

Client Secret은 한 번 입력하고 잊는 문자열이 아니다.

- 어디에 저장되는가
- 어떤 클라이언트가 같이 쓰는가
- 언제 교체하는가
- 폐기 후 기존 연결이 정말 끊겼는가

이 네 가지를 같이 관리해야 한다. Aside에서는 Advanced settings의 Secret 입력란을 사용하고, 블로그나 설정 덤프에는 절대 포함하지 않는다.

---

## 최종 체크리스트

### Google 쪽

- [ ] Developer Preview에 Workspace 계정과 Cloud 프로젝트 등록
- [ ] Google Drive API 활성화
- [ ] Google Drive MCP API 활성화
- [ ] OAuth Audience 설정
- [ ] `drive.readonly`, `drive.file` 범위 추가
- [ ] Web application OAuth 클라이언트 생성
- [ ] 콜백 URI를 `http://127.0.0.1:21420/mcp/oauth/callback`로 등록
- [ ] 실제 동의 화면의 권한 범위 확인

### Aside 쪽

- [ ] 서버 이름 `GoogleDrive`
- [ ] Streamable HTTP 선택
- [ ] 서버 URL `https://drivemcp.googleapis.com/mcp/v1`
- [ ] Client ID 입력
- [ ] Client Secret은 Advanced settings에 입력
- [ ] Preview 승인 후 기존 연결이라면 `Reconnect`
- [ ] 새 채팅 열기
- [ ] 최근 파일, 검색, 메타데이터, 본문 읽기 순으로 검증

### 보안과 정리

- [ ] Client Secret을 문서, 스크린샷, Git에 남기지 않기
- [ ] 사용하지 않는 이전 Secret 폐기
- [ ] 테스트 파일 외에는 쓰기 도구로 검증하지 않기
- [ ] 조직의 OAuth 앱 정책과 감사 로그 확인
- [ ] 연결을 제거할 때 OAuth 권한과 Secret도 함께 폐기

---

## 참고

- [Google Drive MCP 서버 구성](https://developers.google.com/workspace/drive/api/guides/configure-mcp-server)
- [Google Drive MCP 도구 레퍼런스](https://developers.google.com/workspace/drive/api/reference/mcp)
- [Google Workspace MCP 서버 구성](https://developers.google.com/workspace/guides/configure-mcp-servers)
- [Google Workspace Developer Preview Program](https://developers.google.com/workspace/preview)
- [Google 및 Google Cloud MCP 서버 인증 설정](https://docs.cloud.google.com/mcp/set-up-authentication-mcp-servers)

---

## 한 줄 회고

이번 연결에서 가장 오래 걸린 부분은 OAuth가 아니었다. **서버가 등록됐다는 사실, 도구 목록이 보인다는 사실, 실제 Drive 파일을 읽을 수 있다는 사실을 서로 다른 성공 조건으로 보지 않은 것**이었다.

다음에 원격 MCP를 붙일 때는 연결 버튼보다 먼저 세 가지를 확인할 것이다.

```text
사전 프로그램 가입이 필요한가?
OAuth 클라이언트를 수동 등록해야 하는가?
실제 데이터 호출까지 검증했는가?
```

MCP 연결은 URL을 저장하는 작업이 아니라, 서버 자격, 애플리케이션 신원, 사용자 권한의 세 층을 맞추는 작업이었다.
