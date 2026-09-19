---
layout: single
title: "Google Drive MCP를 팀에 배포하기: Internal, External, OAuth 검증의 경계"
date: 2026-09-19 13:00:00 +0900
categories: [dev]
tags: [Google Drive, MCP, OAuth, Google Cloud, Aside, 보안, 배포]
excerpt: >-
  혼자 쓰던 Google Drive MCP의 Cloud 프로젝트와 OAuth Client를 다른 사람도 쓰게 하려면 무엇을 배포해야 할까. 같은 조직의 내부 배포, 외부 협력자 시험, 불특정 다수 공개는 서로 다른 문제다. Client Secret을 나눠주는 방식의 한계와 현재 Preview에서 현실적인 배포 모델을 정리한다.
---

> **Aside에 Google Drive MCP 붙이기 시리즈**
> 1. [Aside에 공식 Google Drive MCP 연결하기: OAuth 콜백부터 Developer Preview까지](/dev/2026/09/19/aside에-공식-google-drive-mcp-연결하기-oauth-콜백부터-developer-preview까지/)
> 2. **Google Drive MCP를 팀에 배포하기: Internal, External, OAuth 검증의 경계** (현재 글)

> 이 글은 2026년 9월 19일 Google Drive MCP Developer Preview와 Aside 1.26.914를 기준으로 작성했다. Preview 가입 조건과 Google OAuth 정책은 바뀔 수 있다.

## 결론부터

혼자 사용하던 Google Drive MCP 연결을 다른 사람도 쓰게 하는 방법은 대상에 따라 세 가지로 나뉜다.

| 대상 | 권장 방식 | 현재 현실성 |
|---|---|---:|
| 같은 Google Workspace 조직의 소수 팀원 | `Internal` 유지, 사용자별 OAuth Client 발급 | 높음 |
| 외부 협력자 소수 | 별도 테스트 프로젝트, `External + Testing` | 제한적 |
| 불특정 다수 | 별도 Production 프로젝트, OAuth 앱 검증 | Preview에서는 낮음 |

현재 가장 현실적인 답은 두 가지다.

1. **조직 내부 배포**: 하나의 Cloud 프로젝트를 조직이 관리하되, OAuth Client는 사용자별 또는 기기별로 분리한다.
2. **외부 사용자용 설치 가이드**: 공용 Client Secret을 배포하지 않고 각 사용자가 자기 Cloud 프로젝트와 OAuth Client를 만들도록 안내한다.

하나의 Client ID와 Client Secret을 인터넷에 공개하는 방식은 배포가 아니다. Secret이 더 이상 비밀이 아니게 되고, 폐기와 감사의 단위도 사라진다.

---

## 먼저 구분할 것: 앱을 공유하는가, Drive를 공유하는가

OAuth 2.0(Open Authorization 2.0, 비밀번호를 넘기지 않고 권한을 위임하는 표준)의 구성요소를 분리해서 봐야 한다.

| 구성요소 | 의미 |
|---|---|
| Google Cloud 프로젝트 | API, 할당량, OAuth 앱, 감사의 관리 단위 |
| OAuth Client | Google이 인식하는 애플리케이션 신원 |
| Google 로그인 계정 | 실제 Drive 데이터의 사용자 |
| Access token | 해당 사용자가 앱에 위임한 단기 접근 권한 |
| Refresh token | 새 Access token을 받기 위한 장기 자격증명 |

같은 OAuth Client를 여러 사람이 사용해도 내 Drive가 그들에게 공유되는 것은 아니다. 각 사용자는 자신의 Google 계정으로 로그인하고, 자신의 Drive에 대한 토큰을 발급받는다.

```text
공통 OAuth Client
  + 사용자 A 로그인
    -> 사용자 A의 Drive 토큰

공통 OAuth Client
  + 사용자 B 로그인
    -> 사용자 B의 Drive 토큰
```

공유되는 것은 Drive 데이터가 아니라 다음 운영 단위다.

- Cloud 프로젝트의 API 사용량과 할당량
- OAuth 동의 화면의 앱 이름과 브랜드
- 등록된 OAuth 범위
- 검증 상태
- Client Secret의 사고 범위
- Secret 교체 시 영향을 받는 사용자 범위

따라서 "다른 사람이 쓰게 한다"는 질문은 사실 두 질문이다.

1. 그 사람이 OAuth 앱을 사용할 자격이 있는가
2. OAuth Client 자격증명을 어떤 단위로 발급하고 폐기할 것인가

---

## 배포 모델 1: 같은 Workspace 조직 내부

회사나 학교의 같은 Google Workspace 조직 구성원에게 제공하는 경우가 가장 단순하다.

Google Auth Platform의 Audience를 `Internal`로 유지하면 된다. Internal 앱은 프로젝트가 속한 Workspace 또는 Cloud Identity 조직의 사용자만 인증할 수 있다. 외부 Google 계정은 사용할 수 없다.

### 내부 배포의 기본 구조

```text
조직 소유 Google Cloud 프로젝트
  + Drive API
  + Drive MCP API
  + Internal OAuth 동의 화면
  + 사용자별 OAuth Client
      -> 사용자 각자의 Aside
          -> 사용자 각자의 Google Drive
```

내부 배포라고 해서 사용자를 Google Cloud 프로젝트의 IAM(Identity and Access Management, 프로젝트 관리 권한)에 추가할 필요는 없다. MCP를 사용하는 사람과 Cloud 프로젝트를 관리하는 사람은 다른 역할이다.

- Cloud 관리자: API, OAuth Client, Secret, 감사 정책 관리
- 최종 사용자: 자신의 Google 계정으로 OAuth 동의 후 Drive 사용

### 1. Audience를 Internal로 유지한다

Google Cloud Console에서 `Google Auth Platform > Audience`를 연다.

```text
User type: Internal
```

이 설정은 같은 조직 사용자의 접근은 허용하고 조직 밖 계정은 막는다. 조직 내부 앱은 일반적인 외부 OAuth 앱의 브랜드 검증 대상에서 제외되지만, Workspace 관리자의 앱 접근 제어 정책은 그대로 적용된다.

### 2. Developer Preview 사용자를 등록한다

OAuth Audience와 Google Workspace Developer Preview 자격은 별개다.

- `Internal`: 이 OAuth 앱에 로그인할 수 있는 조직 범위
- Developer Preview: Drive MCP 기능을 호출할 수 있는 프로그램 자격

Drive MCP가 Preview인 동안에는 사용할 Google Workspace 계정과 Cloud 프로젝트가 프로그램에 등록되어야 한다. 이미 가입한 멤버는 Developer Preview 페이지의 추가 이메일 또는 프로젝트 등록 요청 절차를 이용한다.

팀원에게 OAuth 로그인이 보인다고 해서 Drive MCP 호출까지 된다는 뜻은 아니다. Preview 등록이 빠지면 연결이나 동의는 성공해 보이지만 실제 도구 호출이 권한 오류로 끝날 수 있다.

### 3. Workspace 관리자의 OAuth 앱 정책을 확인한다

조직에서 서드파티 또는 내부 앱 접근을 제한하고 있다면 Workspace Admin Console에서 앱을 허용해야 한다.

확인할 항목은 다음과 같다.

- Google Workspace API 서비스 접근이 Restricted인지
- 내부 앱 자동 신뢰가 켜져 있는지
- 해당 OAuth Client가 Trusted 또는 Specific access로 등록되어 있는지
- 조직 단위(OU, Organizational Unit)별 차단 정책이 있는지

Google 검증을 통과한 앱이어도 Workspace 관리자는 차단할 수 있다. 반대로 내부 앱이라도 조직 정책이 제한적이면 사용자가 인증하지 못할 수 있다.

### 4. 사용자별 OAuth Client를 만든다

하나의 Cloud 프로젝트 안에 여러 Web application OAuth Client를 만들 수 있다.

예를 들면 다음처럼 이름을 나눈다.

```text
Aside Drive MCP - user-a
Aside Drive MCP - user-b
Aside Drive MCP - shared-mac-01
```

각 Client의 Authorized redirect URI에는 Aside의 콜백 주소를 등록한다.

```text
http://127.0.0.1:21420/mcp/oauth/callback
```

모든 사용자의 Aside가 같은 기본 로컬 주소를 사용하므로 콜백 URI 자체는 같아도 된다. 각자의 컴퓨터에서 `127.0.0.1`은 그 컴퓨터 자신을 가리킨다.

### 하나의 Client를 공유하지 않는 이유

| 하나의 Client를 전원이 공유 | 사용자별 또는 기기별 Client |
|---|---|
| 초기 설정이 간단함 | Client 생성 작업이 추가됨 |
| Secret 하나의 유출이 전원에게 영향 | 유출 범위가 한 사용자 또는 기기로 제한 |
| Secret 교체 시 전원 재설정 | 특정 Client만 교체 가능 |
| 누가 사용했는지 구분하기 어려움 | Client 단위 감사가 가능 |
| 퇴사자 한 명 때문에 전체 교체 가능 | 해당 사용자 Client만 폐기 가능 |

소규모 시험에서는 하나의 Client를 공유할 수 있다. 그러나 정기적으로 쓸 팀 연결이라면 사용자별 또는 기기별 분리가 낫다.

### 5. Client Secret은 개별 전달한다

Client Secret은 다음 경로로 보내지 않는다.

- 공개 Git 저장소
- 사내 위키의 평문 페이지
- 이메일 본문
- 여러 사람이 있는 메신저 채널
- 블로그나 화면 녹화

비밀번호 관리 도구의 일회성 공유나 관리자와 사용자 간 보안 채널로 개별 전달한다. 사용자는 Aside의 MCP Advanced settings에 직접 입력한다.

Secret을 전달했다고 Cloud 프로젝트 관리 권한까지 준 것은 아니다. 반대로 사용자가 Cloud 프로젝트에 들어갈 수 있다고 Secret을 공개해도 되는 것은 아니다.

### 6. 사용자 자신의 계정으로 인증한다

사용자는 Aside에서 다음 서버를 등록한다.

| 항목 | 값 |
|---|---|
| Name | `GoogleDrive` |
| Transport | Streamable HTTP |
| URL | `https://drivemcp.googleapis.com/mcp/v1` |
| Authentication | OAuth 또는 Auto |
| Client ID | 사용자에게 발급한 Client ID |
| Client Secret | 해당 Client의 Secret |

그 뒤 자신의 Workspace 계정으로 Google 동의를 진행한다. 관리자의 계정을 대신 쓰거나 공용 Google 계정을 공유하면 안 된다.

### 7. 새 Aside 채팅에서 직접 호출을 검증한다

연결 화면에서 성공했다고 끝내지 않는다.

1. 새 Aside 채팅 열기
2. `list_recent_files` 실행
3. `search_files`로 본인 파일 검색
4. 알고 있는 파일 ID로 `get_file_metadata` 실행
5. 테스트 문서를 `read_file_content`로 읽기

특정 폴더만 실패하면 파일 권한을 본다. 모든 호출이 실패하면 Preview 등록, OAuth 계정, Workspace 앱 정책을 먼저 본다.

---

## 배포 모델 2: 외부 협력자 소수에게 시험 제공

다른 Workspace 조직이나 개인 Google 계정 사용자가 대상이면 `Internal` 앱을 그대로 쓸 수 없다. 이 경우 별도의 테스트 프로젝트를 만드는 편이 안전하다.

```text
내부 운영 프로젝트
  -> Internal 유지

외부 시험 프로젝트
  -> External + Testing
```

Google은 테스트 환경과 프로덕션 환경에 서로 다른 프로젝트를 사용하도록 권장한다. 외부 시험 때문에 잘 작동하는 내부 앱의 Audience와 OAuth 구성을 바꾸지 않는 편이 낫다.

### External Testing 설정

1. 새 Google Cloud 프로젝트 생성
2. Drive API와 Drive MCP API 활성화
3. OAuth Audience를 `External`로 설정
4. Publishing status를 `Testing`으로 유지
5. 외부 사용자를 Test users에 추가
6. Web application OAuth Client 생성
7. Aside 콜백 URI 등록
8. Developer Preview 자격 확인

External Testing은 소규모 시험용이지 장기 배포 방식이 아니다.

| 제한 | 의미 |
|---|---|
| Test user 최대 100명 | 명시적으로 등록한 사용자만 로그인 가능 |
| Refresh token 수명 제한 | 주기적인 재인증이 발생할 수 있음 |
| 테스트 경고 화면 | 정식 검증 앱이 아님을 사용자에게 표시 |
| Preview 자격 | OAuth Test user 등록만으로 Drive MCP 자격이 생기지 않음 |

Google의 OAuth 상태 문서는 External Testing 앱에 최대 100명의 Test user 제한과 7일 Refresh token 만료 제한이 있다고 설명한다. 따라서 매주 안정적으로 동작해야 하는 팀 업무 연결에는 맞지 않는다.

### 외부 시험에서 특히 조심할 것

외부 사용자를 Test user에 추가해도 다음은 자동으로 해결되지 않는다.

- Google Workspace Developer Preview 등록
- 상대 조직 관리자의 OAuth 앱 허용
- Restricted scope 사용 승인
- Aside에 넣을 Client Secret의 안전한 전달

Preview 기능을 시험하는 동안은 상대방에게 설정값만 보내기 전에 그 계정이 실제로 Drive MCP를 호출할 자격이 있는지 먼저 확인해야 한다.

---

## 배포 모델 3: 불특정 다수에게 공개

이 단계부터는 "설정 공유"가 아니라 OAuth 애플리케이션 운영이다.

Audience를 External로 바꾸고 `Publish App`을 누르는 것만으로 검증이 끝나는 것은 아니다. Published와 Verified는 다른 상태다.

```text
Testing
  -> 등록된 Test user만 사용

Published, Unverified
  -> 외부 사용 가능
  -> 경고와 사용자 수 제한 가능

Published, Verified
  -> Google 검증을 통과한 프로덕션 앱
```

### 프로덕션 프로젝트를 분리한다

내부용 또는 테스트용 프로젝트를 그대로 공개 프로젝트로 전환하기보다 별도의 Production 프로젝트를 만든다.

이유는 다음과 같다.

- 테스트용 Redirect URI와 실험용 Client를 제거할 수 있다.
- OAuth 검증 대상 범위를 명확히 할 수 있다.
- 운영 담당자와 연락처를 분리할 수 있다.
- 문제가 생겨도 내부 연결에 영향을 주지 않는다.
- 검증 자료와 감사 로그를 프로덕션 앱 기준으로 관리할 수 있다.

### 공개 앱에 필요한 기본 자료

Google OAuth 검증을 준비하려면 대체로 다음이 필요하다.

- 실제 서비스 이름과 로고
- 운영 주체가 확인되는 홈페이지
- 개인정보처리방침
- 서비스 이용약관
- 사용자 지원 이메일
- 개발자 연락처 이메일
- 소유권을 확인할 수 있는 도메인
- 각 OAuth 범위가 필요한 이유
- OAuth 동의부터 실제 기능 사용까지 보여주는 데모 영상
- 사용자 데이터의 저장, 이용, 삭제 방법

단순한 개인 설정 프로젝트와 요구 수준이 달라진다. Google은 앱이 누구의 것인지, 왜 그 범위가 필요한지, 데이터를 어디에서 처리하는지 확인한다.

---

## 가장 큰 장벽: Drive OAuth 범위

공개 배포 난이도는 사용자 수보다 OAuth 범위가 결정한다.

Google Drive API의 대표 범위는 다음처럼 분류된다.

| 범위 | 의미 | Google 분류 |
|---|---|---|
| `drive.file` | 앱으로 만들거나 사용자가 앱에 연 파일 | Non-sensitive, 비민감 |
| `drive.readonly` | 사용자의 모든 Drive 파일 읽기와 다운로드 | Restricted, 제한됨 |
| `drive` | 사용자의 모든 Drive 파일 보기, 수정, 생성, 삭제 | Restricted, 제한됨 |

Google의 공식 Drive MCP 설정 문서는 `drive.readonly`와 `drive.file`을 안내한다. 이 중 `drive.readonly`는 Restricted scope다.

Restricted scope는 광범위한 사용자 데이터에 접근하므로 외부 프로덕션 앱에서 추가 검증이 필요하다. 데이터가 제3자 서버를 통과하거나 저장되는 구조라면 Google 지정 보안 평가가 요구될 수 있다.

### 실제 동의 화면이 문서보다 넓을 수 있다

내 환경에서는 `drive.readonly`와 `drive.file`을 설정했지만 실제 OAuth 동의 화면에 모든 Drive 파일을 보기, 수정, 생성, 삭제할 수 있다는 광범위한 권한이 표시됐다.

따라서 공개 배포 전에는 문서에 적힌 범위만 보지 말고 다음을 모두 확인해야 한다.

- Google Auth Platform의 Data Access 목록
- 실제 OAuth 승인 화면의 문구
- 발급 토큰에 포함된 실제 scope
- Drive MCP 호출이 요구하는 추가 scope
- Google 검증 화면이 분류한 민감도

실제 승인 요청이 전체 `drive` 범위를 포함한다면 공개 검증 난이도와 위험은 더 커진다.

---

## Aside에서 공용 Client Secret을 배포하면 안 되는 이유

Aside의 현재 Google Drive MCP 연결은 사용자의 컴퓨터에 다음 값을 입력한다.

```text
Client ID
Client Secret
```

Aside는 Secret을 일반 설정 파일이 아니라 보안 저장소에 보관한다. 저장 방식은 적절하지만, 외부 사용자에게 Secret을 전달하는 순간 배포자는 그 값의 복사를 통제할 수 없다.

### Secret이 로컬 앱에 배포되면 생기는 문제

- 사용자가 값을 복사할 수 있다.
- 악성 프로그램이 사용자 환경에서 탈취할 수 있다.
- 한 번 공개되면 누가 사용했는지 구분하기 어렵다.
- Secret 교체 시 모든 사용자가 재설정해야 한다.
- 같은 Client를 사용하는 정상 사용자까지 동시에 끊긴다.

즉 공용 Web application Client Secret을 불특정 다수의 데스크톱 앱에 넣는 구조는 장기적인 공개 배포 모델로 적합하지 않다.

### 콜백 URI가 같아도 되는 것과 Secret 공유는 다른 문제다

모든 Aside 사용자가 다음 콜백 URI를 등록하는 것은 문제가 아니다.

```text
http://127.0.0.1:21420/mcp/oauth/callback
```

각 컴퓨터의 `127.0.0.1`은 서로 다른 로컬 컴퓨터를 가리킨다. 콜백 주소가 같은 것은 Aside의 구현 규칙을 공유하는 것이다.

반면 Client Secret을 공유하는 것은 하나의 앱 자격증명을 공유하는 것이다. 두 문제를 섞으면 안 된다.

---

## 외부 배포의 현실적인 선택지

### 선택지 1. 각 사용자가 자신의 OAuth Client를 만든다

현재 Aside와 Developer Preview 조건에서는 가장 현실적이다.

배포자는 다음만 제공한다.

- Google Cloud 프로젝트 생성 방법
- 필요한 API 목록
- OAuth 범위
- Aside 콜백 URI
- Drive MCP 엔드포인트
- Aside 입력 방법
- 오류 해결법

사용자는 자신의 프로젝트에서 Client ID와 Secret을 만들고 자신의 Aside에 입력한다.

장점:

- 배포자가 Secret을 보관하거나 전달하지 않는다.
- 사용자가 자신의 Cloud 프로젝트와 OAuth 승인을 통제한다.
- 한 사용자의 Secret 사고가 다른 사용자에게 번지지 않는다.

단점:

- 사용자마다 Google Cloud 설정이 필요하다.
- Preview 등록 절차를 각자 거쳐야 할 수 있다.
- 비개발자에게는 진입 장벽이 높다.

### 선택지 2. 조직 관리자가 사용자별 Client를 발급한다

같은 Workspace 조직에서는 가장 균형이 좋다.

```text
중앙 프로젝트와 정책
  + 사용자별 Client
  + 개별 Secret 전달
  + 사용자별 폐기
```

프로젝트와 OAuth 앱 브랜드는 중앙에서 관리하면서 사고 범위는 사용자 단위로 제한할 수 있다.

### 선택지 3. 별도의 인증 중계 서비스를 만든다

불특정 다수를 대상으로 한다면 Client Secret을 서버에만 두고 사용자는 일반 OAuth 로그인만 하게 만드는 구조가 필요하다.

하지만 이것은 Aside 설정값 배포의 범위를 넘어선다.

- HTTPS Redirect URI를 가진 서비스
- 사용자별 OAuth 상태 관리
- 토큰 암호화 저장 또는 전달 구조
- 계정 연결 해제와 데이터 삭제 기능
- 개인정보처리방침과 운영 연락처
- Google OAuth 검증
- 제한된 범위 보안 평가 가능성

또한 현재 Google의 공식 Drive MCP가 Developer Preview이므로 일반 공개 서비스로 운영하기에 제약이 크다.

### 선택지 4. 일반 공개가 될 때까지 기다린다

Google Drive MCP가 정식 공개되고 Aside가 Google 관리형 연결을 제공하게 되면 사용자가 직접 Client Secret을 만들거나 전달할 필요가 없어질 수 있다.

현재 Preview에서 억지로 공개 배포 체계를 만드는 비용이 실제 효용보다 클 수 있다.

---

## 추천 아키텍처

### 회사 내부 팀

```text
조직 소유 Cloud 프로젝트
  Audience: Internal
  Preview: 사용할 계정 등록
  API: Drive + Drive MCP
  OAuth Client: 사용자별 또는 기기별
  Secret: 보안 채널로 개별 전달
  Workspace Admin: 앱 접근 정책 확인
```

### 외부 협력자 시험

```text
별도 테스트 Cloud 프로젝트
  Audience: External
  Status: Testing
  Test users: 필요한 계정만
  Preview: 계정과 프로젝트 자격 확인
  기간: 단기 시험
  운영 연결: 사용하지 않음
```

### 블로그 독자나 불특정 Aside 사용자

```text
공용 Client Secret: 배포하지 않음
  -> 사용자가 자기 프로젝트와 OAuth Client 생성
  -> 글은 설치 가이드만 제공
```

### 공개 프로덕션 서비스

```text
별도 Production 프로젝트
  + External Audience
  + 앱 브랜드와 도메인
  + 개인정보처리방침과 약관
  + OAuth 범위 검증
  + 필요시 보안 평가
  + Secret을 서버에만 보관하는 별도 인증 구조
```

---

## 운영 체크리스트

### 같은 조직 내부

- [ ] Cloud 프로젝트가 조직 소유인지 확인
- [ ] Audience를 `Internal`로 유지
- [ ] 사용할 계정의 Developer Preview 자격 확인
- [ ] Workspace 관리자의 OAuth 앱 정책 확인
- [ ] 사용자별 또는 기기별 OAuth Client 생성
- [ ] Aside 콜백 URI 정확히 등록
- [ ] Client Secret 개별 전달
- [ ] 새 Aside 채팅에서 실제 Drive 호출 검증
- [ ] 퇴사 또는 기기 폐기 시 해당 Client 삭제

### 외부 시험

- [ ] 내부용과 별도 테스트 프로젝트 사용
- [ ] Audience를 `External`로 설정
- [ ] Publishing status를 `Testing`으로 유지
- [ ] Test users만 추가
- [ ] 100명 한도와 Refresh token 제한 인지
- [ ] 상대 계정의 Developer Preview 자격 확인
- [ ] 상대 조직의 OAuth 앱 차단 정책 확인
- [ ] 시험 종료 후 Client와 Secret 폐기

### 외부 공개

- [ ] 별도 Production 프로젝트 사용
- [ ] 홈페이지와 소유 도메인 준비
- [ ] 개인정보처리방침과 약관 공개
- [ ] 실제 필요한 OAuth 범위만 요청
- [ ] Restricted scope 검증 준비
- [ ] 실제 데이터 처리 경로 문서화
- [ ] 필요시 보안 평가 예산과 일정 확보
- [ ] 공용 Client Secret을 사용자 컴퓨터에 배포하지 않기
- [ ] Preview 기능을 공개 서비스의 핵심 의존성으로 삼아도 되는지 재검토

---

## 결정 트리

```text
사용자가 같은 Workspace 조직인가?
  |
  +-- 예
  |    -> Internal 유지
  |    -> Preview 계정 등록
  |    -> 사용자별 OAuth Client
  |
  +-- 아니오
       |
       +-- 소수의 단기 시험인가?
       |    -> External + Testing
       |    -> Test user 등록
       |    -> 100명, 토큰 수명, Preview 제한 감수
       |
       +-- 불특정 다수인가?
            -> 공용 Secret 배포 금지
            -> 사용자 자체 OAuth Client 방식
            또는
            -> 별도 프로덕션 앱과 OAuth 검증
```

---

## 참고

- [Google Workspace Developer Preview Program](https://developers.google.com/workspace/preview)
- [Google Drive MCP 서버 구성](https://developers.google.com/workspace/drive/api/guides/configure-mcp-server)
- [Google OAuth 앱 상태와 Audience](https://developers.google.com/identity/protocols/oauth2/production-readiness/overview)
- [OAuth 동의 화면과 범위 구성](https://developers.google.com/workspace/guides/configure-oauth-consent)
- [Google Drive API OAuth 범위 분류](https://developers.google.com/workspace/drive/api/guides/api-specific-auth)
- [Restricted scope 검증](https://developers.google.com/identity/protocols/oauth2/production-readiness/restricted-scope-verification)
- [OAuth 프로덕션 정책 준수](https://developers.google.com/identity/protocols/oauth2/production-readiness/policy-compliance)
- [Workspace에서 OAuth 앱 접근 제어](https://knowledge.workspace.google.com/admin/apps/control-which-apps-access-google-workspace-data)

---

## 한 줄 회고

OAuth Client를 다른 사람도 쓰게 만드는 일은 Client ID와 Secret을 전달하는 작업이 아니었다. **사용자 범위, Preview 자격, 제한된 Drive 권한, Secret의 폐기 단위를 함께 설계하는 배포 작업**이었다.

내부 팀이라면 사용자별 Client가 답에 가깝다. 불특정 다수라면 공용 Secret을 나눠주는 대신 사용자가 자신의 OAuth Client를 만들게 하거나, 정식 OAuth 애플리케이션을 별도로 운영해야 한다.
