---
layout: page
title: "MCP 전송 방식 비교: STDIO vs Streamable HTTP"
summary: "MCP(Model Context Protocol)의 두 가지 전송 방식인 STDIO와 Streamable HTTP의 작동 원리, 아키텍처 차이, 인증 방식, 사용 사례를 비교 분석한 연구 노트"
date: 2026-04-12
tags: [MCP, Model Context Protocol, STDIO, HTTP, SSE, Streamable HTTP, Claude, AI, 인증, OAuth]
---

## 개요

MCP(Model Context Protocol)는 AI 모델과 외부 도구/서비스를 연결하는 개방형 프로토콜이다. MCP에서 클라이언트와 서버 간 통신은 **전송 계층(Transport Layer)**을 통해 이루어지며, 현재 공식 사양에서 지원하는 두 가지 전송 방식은 **STDIO**와 **Streamable HTTP**이다.

본 문서에서는 두 방식의 작동 원리, 아키텍처 차이, 인증 구현, 그리고 적합한 사용 사례를 비교 분석한다.

---

## 1. STDIO 전송 방식

### 작동 원리

STDIO는 **로컬 단일 클라이언트** 연결을 위한 전송 방식이다.

- MCP 클라이언트(예: Claude Code)가 MCP 서버를 **같은 머신에서 자식 프로세스(subprocess)로 직접 실행**한다.
- 서버는 **stdin**으로 JSON-RPC 메시지를 읽고, **stdout**으로 응답을 보내며, **stderr**로 디버그/로그 정보를 출력한다.
- 메시지는 줄바꿈으로 구분된 JSON-RPC 객체로, 한 줄에 하나의 완전한 JSON 객체가 들어간다.

### 핵심 특성

| 특성 | 설명 |
|------|------|
| 클라이언트 수 | 1개 (1:1 관계) |
| 상태 관리 | 프로세스 수명 동안 상태 유지 (Stateful) |
| 지연시간 | 매우 낮음 (네트워크 오버헤드 없음) |
| 네트워크 | 불필요 (같은 머신) |
| 서버 수명 | 클라이언트에 종속 |

### 설정 예시

```json
{
  "mcpServers": {
    "my-local-tool": {
      "command": "npx",
      "args": ["@my-org/mcp-server"],
      "type": "stdio",
      "env": {
        "API_KEY": "your-key"
      }
    }
  }
}
```

### 적합한 사용 사례

- 로컬 파일 시스템 접근 도구
- 로컬 데이터베이스 쿼리 도구
- 시스템 유틸리티
- 네트워크 오버헤드가 없어야 하는 경우

---

## 2. Streamable HTTP 전송 방식

### 작동 원리

Streamable HTTP는 2025년 3월 MCP 사양(2025-03-26)에서 도입된 **원격/다중 클라이언트** 연결 방식이다.

MCP 서버가 **독립적인 HTTP 서버**로 실행되며 하나의 엔드포인트(예: `https://example.com/mcp`)를 노출한다.

#### 통신 흐름

1. **클라이언트 → 서버 (POST):** 클라이언트가 JSON-RPC 메시지를 HTTP POST로 전송
2. **서버 응답 — 두 가지 모드:**
   - **단순 응답:** `Content-Type: application/json`으로 즉시 JSON-RPC 응답 반환
   - **스트리밍 응답:** `Content-Type: text/event-stream`(SSE)으로 여러 이벤트를 순차 전송
3. **서버 → 클라이언트 알림 (GET):** 클라이언트가 GET 요청으로 SSE 스트림을 열어 서버 푸시 알림 수신
4. **세션 관리:** 서버가 초기화 시 `MCP-Session-Id` 헤더를 발급, 클라이언트는 이후 모든 요청에 포함

#### 메시지 흐름 다이어그램

```
Client → POST /mcp (InitializeRequest)
Server → HTTP 200 + InitializeResponse + MCP-Session-Id: abc123...

Client → POST /mcp (Request) + MCP-Session-Id: abc123...
Server → Option A: HTTP 200 + application/json (단순 응답)
         Option B: HTTP 200 + text/event-stream (SSE 스트림)

Client → GET /mcp + MCP-Session-Id: abc123...
Server → HTTP 200 + text/event-stream (서버 알림용 SSE 스트림)
```

### 핵심 특성

| 특성 | 설명 |
|------|------|
| 클라이언트 수 | 다수 동시 연결 가능 |
| 상태 관리 | 세션 기반 또는 무상태(Stateless) |
| 확장성 | 로드 밸런서 뒤에서 수평 확장 가능 |
| 네트워크 | HTTP/HTTPS 필요 |
| 연결 복구 | `Last-Event-ID`로 자동 복구 |

### 설정 예시

```json
{
  "mcpServers": {
    "remote-service": {
      "type": "http",
      "url": "https://mcp.example.com/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_TOKEN"
      }
    }
  }
}
```

### 적합한 사용 사례

- 클라우드 SaaS 서비스 (Slack, Jira, GitHub 등)
- 원격 API 및 마이크로서비스
- 다수 사용자가 동시 접속하는 서비스
- 로드 밸런싱이 필요한 대규모 배포

---

## 3. 이전 HTTP+SSE 방식의 폐기 배경

MCP 초기 사양(2024-11-05)에서는 HTTP+SSE 전송 방식을 사용했으나, 다음의 한계로 Streamable HTTP로 대체되었다:

| 비교 항목 | 기존 HTTP+SSE | Streamable HTTP |
|-----------|--------------|-----------------|
| 엔드포인트 | 2개 (요청/응답 분리) | 1개 (통합) |
| 스트리밍 | SSE 전용 | SSE 또는 단순 HTTP 선택 |
| HTTP/2·3 호환 | 제한적 | 완전 지원 |
| 에러 처리 | 다중 채널 | HTTP 상태 코드 단일화 |
| 연결 복구 | 수동 구현 | Last-Event-ID 내장 |
| 로드 밸런싱 | 어려움 | 설계 단계부터 지원 |

---

## 4. 종합 비교

| 항목 | STDIO | Streamable HTTP |
|------|-------|-----------------|
| 배포 모델 | 로컬 자식 프로세스 | 원격 HTTP 서버 |
| 클라이언트 수 | 1개 (1:1) | 다수 동시 연결 |
| 상태 관리 | 프로세스 내 암묵적 유지 | 세션 ID 기반 또는 무상태 |
| 네트워크 | 불필요 (같은 머신) | HTTP/HTTPS 필요 |
| 확장성 | 제한적 | 수평 확장·로드 밸런싱 가능 |
| 지연시간 | 매우 낮음 | 네트워크 의존 |
| 서버 수명 | 클라이언트에 종속 | 독립적 운영 |
| 연결 복구 | 프로세스 재시작 | Last-Event-ID로 재연결 |
| 보안 | 프로세스 격리 | HTTPS, Origin 검증, 인증 |

---

## 5. 인증 방식

### STDIO

STDIO 방식은 클라이언트가 서버를 직접 자식 프로세스로 실행하므로, 같은 머신의 같은 사용자 권한 하에서 동작한다. **프로세스 격리 자체가 암묵적 신뢰 관계**를 형성하므로 별도 인증이 필요하지 않다.

### Streamable HTTP

네트워크를 통해 통신하므로 인증이 실질적으로 필수이다. MCP 사양은 특정 인증 방식을 강제하지 않으며(SHOULD, 권장 수준), 서버 구현자가 자유롭게 선택할 수 있다:

| 인증 방식 | 설명 | 적합한 경우 |
|-----------|------|------------|
| 인증 없음 | 내부 VPN/프라이빗 네트워크 의존 | 사내 전용 도구 (비권장) |
| 정적 API 키 | 사전 발급된 키를 Bearer 헤더에 포함 | 소규모 팀, 내부 도구 |
| OAuth 2.1 | 토큰 발급/갱신/만료/스코프 관리 | 외부 공개 서비스, SaaS |

---

## 6. 향후 전망 (2026 로드맵)

MCP 로드맵에 따르면 다음과 같은 발전이 계획되어 있다:

- **세션 상태 분리:** 전송 계층에서 데이터 모델 계층으로 세션 상태 이동
- **쿠키 유사 메커니즘:** HTTP 쿠키와 유사한 세션 식별 메커니즘 도입
- **무상태 확장 강화:** 로드 밸런싱과 수평 확장을 더욱 용이하게 지원

---

## 참고 자료

- [MCP Specification: Transports (2025-11-25)](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)
- [Exploring the Future of MCP Transports](https://blog.modelcontextprotocol.io/posts/2025-12-19-mcp-transport-future/)
- [The 2026 MCP Roadmap](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/)
- [Why MCP Deprecated SSE and Went with Streamable HTTP](https://blog.fka.dev/blog/2025-06-06-why-mcp-deprecated-sse-and-go-with-streamable-http/)
