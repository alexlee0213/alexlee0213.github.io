---
layout: single
title: "Telegram Channel Improvements (2026-04)"
date: 2026-04-15 00:00:00 +0900
categories: [dev]
tags: [NanoClaw, Telegram, MCP, Claude]
excerpt: "NanoClaw의 Telegram 채널 개선 작업 정리 — 토픽(포럼) 스레드 라우팅, typing indicator 개선, 파일 전송 기능 추가, 세션 소스 캐시 이슈."
---
## 1. Topic(Forum) Thread Routing

Telegram 그룹의 토픽(포럼) 기능에서 메시지가 오면, 같은 토픽으로 응답을 보내도록 수정.

### 문제

- 메시지 수신 시 `message_thread_id`를 캡처하고 있었지만, 응답 시 전달하지 않아서 모든 응답이 General 토픽으로 감.

### 변경사항

**DB 계층** (`src/db.ts`)
- `messages` 테이블에 `thread_id TEXT` 컬럼 추가 (마이그레이션 포함)
- `storeMessage()`에서 `thread_id` 저장
- `getMessagesSince()`, `getNewMessages()` SELECT 쿼리에 `thread_id` 포함

**Channel 인터페이스** (`src/types.ts`)
- `Channel.sendMessage()`에 `threadId?: string` 파라미터 추가

**메시지 처리** (`src/index.ts`)
- `activeThreadId: Record<string, string | undefined>` 맵 추가 — 그룹별 현재 활성 thread 추적
- `processGroupMessages()`: 마지막 메시지의 `thread_id`로 `activeThreadId` 갱신
- message loop의 pipe 경로: 후속 메시지 도착 시에도 `activeThreadId` 갱신
- 에이전트 응답 시 `channel.sendMessage(chatJid, text, activeThreadId[chatJid])` 호출

**IPC 경로** (에이전트가 `mcp__nanoclaw__send_message` 사용 시)
- `container/agent-runner/src/ipc-mcp-stdio.ts`: IPC JSON에 `threadId` 포함 (환경변수 `NANOCLAW_THREAD_ID`에서)
- `container/agent-runner/src/index.ts`: MCP 서버에 `NANOCLAW_THREAD_ID` 환경변수 전달
- `src/ipc.ts`: `IpcDeps.sendMessage`에 `threadId` 파라미터 추가, IPC 메시지의 `threadId` 전달
- `src/index.ts`: IPC sendMessage 콜백에서 `threadId ?? activeThreadId[jid]` fallback 사용

**라우터** (`src/router.ts`)
- `routeOutbound()`에 `threadId?: string` 파라미터 추가

### 주의사항

- `activeThreadId`는 인메모리 상태이며, NanoClaw 재시작 시 초기화됨. 재시작 후 첫 메시지에서 다시 설정됨.
- 여러 토픽에서 동시에 메시지가 오면, 마지막 메시지의 thread_id가 사용됨.

---

## 2. Typing Indicator 개선

### 문제

- Telegram typing indicator는 약 5초 후 자동 만료되는데, 한 번만 전송하고 있었음.

### 변경사항 (`src/channels/telegram.ts`)

- `typingIntervals` 맵으로 그룹별 interval 관리
- `setTyping(true)`: 즉시 전송 후 4초마다 반복 (`setInterval`)
- `setTyping(false)`: interval 해제
- `onContainerDone` 콜백 (`src/group-queue.ts` → `src/index.ts`): 컨테이너 완료 시 자동으로 typing 해제

---

## 3. 파일 전송 기능

Telegram 채팅으로 문서, 사진, 오디오, 비디오 파일을 전송할 수 있도록 추가.

### 변경사항

**Channel 인터페이스** (`src/types.ts`)
```typescript
sendFile?(
  jid: string,
  filePath: string,
  options?: { caption?: string; threadId?: string },
): Promise<void>;
```

**Telegram 구현** (`src/channels/telegram.ts`)
- grammY의 `InputFile`을 사용하여 로컬 파일 전송
- 확장자 기반 자동 분류:
  - 이미지 (.jpg, .jpeg, .png, .gif, .webp) → `sendPhoto`
  - 비디오 (.mp4, .mov, .avi, .mkv, .webm) → `sendVideo`
  - 오디오 (.mp3, .ogg, .wav, .flac, .m4a, .opus) → `sendAudio`
  - 기타 → `sendDocument`
- caption 및 threadId(토픽) 지원

**MCP 도구** (`container/agent-runner/src/ipc-mcp-stdio.ts`)
- `send_file` 도구 추가:
  - `file_path`: 컨테이너 내부 절대 경로 (예: `/workspace/group/output.pdf`)
  - `caption`: 선택적 캡션 텍스트
- 파일 존재 여부 사전 검증
- `threadId` 자동 포함

**IPC 처리** (`src/ipc.ts`)
- `IpcDeps`에 `sendFile` 메서드 추가
- `file` 타입 IPC 메시지 처리
- 컨테이너 경로 → 호스트 경로 변환: `/workspace/group/` → `groups/{folder}/`
- 권한 검증 (메시지와 동일한 로직)

**호스트 연결** (`src/index.ts`)
- IPC watcher의 `sendFile` 콜백: `channel.sendFile()` 호출
- `sendFile` 미지원 채널은 캡션을 텍스트 메시지로 fallback

### 사용 예시 (에이전트 컨테이너 내부)

```
mcp__nanoclaw__send_file(
  file_path="/workspace/group/report.pdf",
  caption="분석 보고서입니다"
)
```

---

## 4. 세션 소스 캐시 이슈

### 발견된 문제

컨테이너 시작 시 `/app/src`가 호스트의 `data/sessions/{group}/agent-runner-src/`에서 마운트됨. 이로 인해 `container/build.sh`로 이미지를 리빌드해도 캐시된 이전 소스가 사용되어 새 MCP 도구가 반영되지 않음.

### 임시 해결

```bash
# 모든 세션의 캐시된 소스를 최신으로 업데이트
for dir in data/sessions/*/agent-runner-src; do
  cp container/agent-runner/src/ipc-mcp-stdio.ts "$dir/ipc-mcp-stdio.ts"
  cp container/agent-runner/src/index.ts "$dir/index.ts"
done
```

### 관련 커밋

| 커밋 | 설명 |
|------|------|
| `ec3293c` | Telegram typing fix, model config, OCR 등 일괄 |
| `cb37d35` | feat: route Telegram responses to the correct topic thread |
| `fabb31a` | fix: pass thread_id through IPC send_message path |
| (미커밋) | feat: Telegram file sending via send_file MCP tool |
