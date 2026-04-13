---
layout: single
title: "Slack MCP 커넥터 동작 원리 — Claude 데스크톱 앱 ↔ Slack 연동 흐름"
date: 2026-04-13 09:37:23 +0900
categories: [research]
tags: [MCP, Slack, OAuth, Claude, 커넥터]
excerpt: "Claude 데스크톱 앱에서 Slack MCP 커넥터를 통해 메시지를 전송하는 전체 동작 흐름을 분석한 기술 문서입니다. OAuth 인증부터 실제 메시지 전송까지의 모든 단계를 시퀀스 다이어그램과 함께 설명합니다."
---
## 1. 개요

### 1.1 문서 목적

본 문서는 Claude 데스크톱 앱에서 Slack MCP(Model Context Protocol) 커넥터를 통해 Slack으로 메시지를 전송하는 전체 동작 흐름을 기술적으로 문서화한 것입니다. OAuth 인증부터 실제 메시지 전송까지의 모든 단계를 시퀀스 다이어그램과 함께 설명합니다.

### 1.2 배경

Claude 데스크톱 앱에서 Slack으로 메시지를 전송할 때, 발신자가 기존 '사용자 A'에서 '사용자 B'로 변경된 것이 확인되었습니다. 이 이슈를 계기로 Slack MCP 커넥터의 동작 원리를 조사하고, 발신자 계정이 결정되는 메커니즘을 정리하였습니다.

### 1.3 대상 독자

개발팀 내부 구성원을 대상으로 하며, MCP 프로토콜 및 OAuth 2.0 인증 흐름에 대한 기본적인 이해를 전제로 합니다.

---

## 2. 시스템 구성 요소

Slack MCP 커넥터의 동작에는 다음 5개의 컴포넌트가 관여합니다.

| 컴포넌트 | 위치 | 역할 |
|---------|------|------|
| Claude 데스크톱 앱 | 내 PC | 사용자 인터페이스, 모델과의 통신, MCP 도구 호출 중계 |
| 웹 브라우저 | 내 PC | OAuth 인증 시 Slack 로그인 및 권한 동의 처리 |
| Anthropic MCP 커넥터 서버 | Anthropic 클라우드 | OAuth 토큰 보관, MCP 프로토콜 처리, Slack API 호출 대행 |
| Slack OAuth 서버 | slack.com | OAuth 2.0 인증 처리, Authorization Code 및 Access Token 발급 |
| Slack Web API 서버 | api.slack.com | 실제 메시지 전송, 사용자 검색, 채널 조회 등 실행 |

**핵심 포인트:** MCP 커넥터 서버는 내 PC에서 돌아가는 로컬 프로세스가 아니라, Anthropic이 관리하는 원격 클라우드 서버입니다. OAuth 토큰도 Anthropic 클라우드에 저장되므로, 다른 PC에서 같은 Claude 계정으로 로그인해도 동일한 Slack 커넥터가 연결된 채로 작동합니다.

---

## 3. Phase 1: OAuth 인증 흐름

커넥터를 최초로 추가하거나 재인증할 때 실행되는 OAuth 2.0 인증 흐름입니다.

### 3.1 시퀀스 다이어그램

![Phase 1: OAuth 인증 흐름 시퀀스 다이어그램](/assets/images/slack-mcp-phase1-oauth.png)

### 3.2 단계별 상세 설명

1. **커넥터 추가 요청:** 사용자가 Claude 데스크톱 앱에서 Slack 커넥터 추가 버튼을 클릭합니다.
2. **OAuth 인증 페이지 오픈:** 앱이 웹 브라우저를 열어 Slack OAuth 인증 페이지로 이동시킵니다. URL에는 client_id(Anthropic 앱 ID), scope(권한 범위), redirect_uri(Anthropic 콜백 URL), state(CSRF 방지 토큰)가 포함됩니다.
3. **로그인 및 권한 동의:** 브라우저에 이미 Slack에 로그인된 계정이 있으면 해당 계정으로 권한 동의 화면이 표시됩니다. 로그인되지 않은 경우 먼저 로그인을 진행합니다. 이 시점에서 인증 계정(= 향후 메시지 발신자)이 결정됩니다.
4. **Authorization Code 발급:** 사용자가 '허용(Allow)'을 클릭하면, Slack이 Authorization Code를 발급하고 redirect_uri로 리디렉트합니다. 이 코드가 Anthropic MCP 커넥터 서버로 전달됩니다.
5. **Access Token 교환:** Anthropic 서버가 Authorization Code, client_id, client_secret를 Slack OAuth 서버에 전송하여 Access Token을 발급받습니다. 이 토큰에는 인증된 사용자 ID와 워크스페이스 정보가 포함됩니다.
6. **토큰 저장:** Anthropic 서버가 토큰을 암호화하여 저장하고, Claude 사용자 계정과 매핑합니다. 이후 모든 Slack API 호출에 이 토큰이 사용됩니다.

### 3.3 핵심 포인트

- 브라우저에 로그인된 Slack 계정이 곧 토큰 소유자이며, 이후 모든 API 호출의 발신자가 됩니다.
- Code → Token 교환은 서버 간 직접 통신으로 이루어져 클라이언트에 토큰이 노출되지 않습니다.
- 재인증 시 다른 Slack 계정으로 로그인하면 발신자가 변경됩니다.

---

## 4. Phase 2~3: 도구 로드 및 메시지 전송

앱 시작 시 도구 목록을 로드하고, 사용자 요청에 따라 실제 Slack API를 호출하는 흐름입니다.

### 4.1 시퀀스 다이어그램

![Phase 2~3: 도구 로드 및 메시지 전송 시퀀스 다이어그램](/assets/images/slack-mcp-phase23-messaging.png)

### 4.2 Phase 2: 도구 목록 로드

Claude 데스크톱 앱이 시작될 때마다 MCP 커넥터 서버에 tools/list 요청을 보냅니다. 서버는 slack_send_message, slack_search_users, slack_read_channel 등 사용 가능한 도구 목록과 각 도구의 파라미터 스키마를 JSON 형태로 응답합니다. 이 도구 목록은 Claude AI 모델의 시스템 프롬프트에 포함되어, 모델이 사용자 요청에 맞는 도구를 선택할 수 있게 됩니다.

### 4.3 Phase 3: 메시지 전송 흐름

예시: 사용자가 "홍길동에게 슬랙 메시지 보내줘: 내일 미팅 가능?"이라고 입력한 경우:

1. **모델 추론 (1차):** Claude 모델이 사용자 메시지를 분석하고, 홍길동의 user_id를 모르므로 먼저 slack_search_users 도구를 호출하기로 결정합니다.
2. **도구 호출 (1차):** 앱이 MCP 서버로 slack_search_users({query: "홍길동"}) 요청을 전달하고, MCP 서버가 저장된 토큰으로 Slack API(users.list)를 호출하여 홍길동의 user_id를 반환합니다.
3. **모델 추론 (2차):** 홍길동의 user_id를 확인한 모델이 slack_send_message 도구 호출을 결정합니다.
4. **도구 호출 (2차):** MCP 서버가 Slack API(chat.postMessage)를 호출합니다. 이때 토큰 소유자(사용자 B)의 신원으로 메시지가 전송되므로, Slack에서는 사용자 B가 보낸 DM으로 표시됩니다.
5. **응답 생성:** 모델이 도구 실행 결과를 바탕으로 자연어 응답("홍길동에게 메시지를 보냈습니다")을 생성하여 사용자에게 표시합니다.

### 4.4 핵심 포인트

- 모든 Slack API 호출은 Phase 1에서 저장된 토큰의 소유자 신원으로 실행됩니다.
- 모델은 필요에 따라 여러 번의 도구 호출을 순차적으로 수행할 수 있습니다 (검색 → 전송 등).
- Claude 데스크톱 앱은 모델과 MCP 서버 사이의 중계자 역할만 수행합니다.

---

## 5. 발신자 계정 변경 이슈 분석

### 5.1 현상

Claude 데스크톱 앱에서 Slack으로 메시지를 전송할 때, 발신자가 기존 '사용자 A'에서 '사용자 B'로 변경된 것이 확인되었습니다.

### 5.2 확인된 현재 인증 정보

| 항목 | 값 |
|------|-----|
| Display Name | (익명) |
| Real Name | (익명) |
| Username | (익명) |
| Email | (익명) |
| Workspace | (비공개) |

### 5.3 원인 추정

Slack MCP 커넥터의 OAuth 재인증이 수행되었으며, 재인증 시점에 브라우저에 사용자 A가 아닌 사용자 B 계정으로 Slack에 로그인되어 있었던 것으로 추정됩니다. Phase 1의 3단계에서 설명한 바와 같이, 브라우저에 로그인된 계정이 곧 토큰 소유자가 되므로, 인증 계정이 변경된 것입니다.

### 5.4 복구 방법

1. Claude 데스크톱 앱에서 현재 Slack 커넥터를 해제(연결 끊기)합니다.
2. 웹 브라우저에서 원래 사용자(사용자 A) 계정으로 Slack에 로그인합니다.
3. Claude 데스크톱 앱에서 Slack 커넥터를 다시 추가하여 재인증합니다.

---

## 6. 참고 사항

### 6.1 보안 고려사항

- OAuth Access Token은 Anthropic 클라우드에 암호화되어 저장됩니다. 로컬 PC에는 토큰이 저장되지 않습니다.
- 커넥터를 해제하면 Anthropic 서버에서 토큰이 삭제됩니다.
- 토큰의 권한 범위(scope)는 커넥터 추가 시 요청된 범위로 제한되며, Slack 워크스페이스 관리자가 앱 관리 페이지에서 확인할 수 있습니다.

### 6.2 로컬 MCP 서버와의 차이

Claude Code 등에서 npx 명령으로 로컬에 MCP 서버를 직접 띄우는 방식도 있습니다. 이 경우 토큰이 로컬 PC에 저장되고, MCP 서버가 로컬에서 실행됩니다. Claude 데스크톱 앱의 공식 커넥터는 Anthropic 호스팅 방식이며, 본 문서는 이 방식을 기준으로 작성되었습니다.

### 6.3 토큰 갱신 및 만료

Slack OAuth 토큰은 일반적으로 만료되지 않지만, Slack 워크스페이스 관리자가 앱을 제거하거나 사용자가 토큰을 취소하면 무효화됩니다. 토큰이 무효화되면 커넥터 재인증이 필요합니다.