---
layout: single
title: "OpenClaw 텔레그램 파일 전송 활성화 작업 상세 메모, Claude Code 관점 포함"
date: 2026-04-18 04:40:00 +0900
categories: [dev]
tags: [OpenClaw, Telegram, Claude Code, 파일전송, AGENTS.md]
excerpt: "OpenClaw에서 텔레그램 파일 전송이 실패하는 원인을 단계별 실험으로 추적하고, managed outbound media path 발견과 설정 실수 수정, 운영 규칙 보강까지의 전 과정을 정리한 기술 메모."
---
# OpenClaw 텔레그램 파일 전송 활성화 작업 상세 메모, Claude Code 관점 포함

## 개요
이 문서는 OpenClaw에서 텔레그램으로 파일을 실질적으로 전송할 수 있게 만들기 위해 수행한 작업을 정리한 것이다. 특히 Claude Code 스타일의 설정 변경 및 운영 흐름과 연결되는 부분을 중심으로 설명한다.

목표는 단순히 텔레그램으로 파일을 보내는 데서 끝나는 것이 아니라, 어떤 전송 경로가 실패했고 왜 실패했는지, OpenClaw 내부 정책이 어떻게 작동하는지, 그리고 어떤 설정 및 운영 규칙이 실제 신뢰성을 높였는지를 명확하게 정리하는 것이었다.

## 초기 문제
처음에는 assistant reply 본문에 `MEDIA:/absolute/path` 형식의 절대 경로를 넣어 파일을 보내려 했지만, 텔레그램 파일 전송이 불안정하거나 실패하는 현상이 있었다.

핵심 증상은 다음과 같았다.
- assistant normal reply 경로에서 파일 전송이 자주 실패함
- 특히 `MEDIA:/home/...` 같은 절대 경로가 문제를 일으킴
- 반면 direct outbound 경로에서는 파일 전송이 성공함

이 차이는 텔레그램 자체 제약이 아니라 OpenClaw 내부 delivery path 차이일 가능성을 강하게 시사했다.

## 핵심 관찰: 두 가지 전송 경로
실제로는 서로 다른 두 전송 경로가 존재했다.

1. **Direct outbound sending**
   - 명시적인 outbound send 경로를 통해 전송
   - 로컬 PDF를 텔레그램 document로 보내는 데 성공함

2. **Assistant normal reply delivery**
   - assistant가 reply 본문에 `MEDIA:` 라인을 넣는 방식
   - reply normalization 및 safety policy의 영향을 받음
   - host-local absolute path를 거부할 수 있음

즉, 파일 전송 실패의 핵심은 텔레그램 지원 부족이 아니라 OpenClaw가 어떤 reply path를 타느냐에 있었다.

## 실험 1: Telegram document sending 직접 테스트

### 가설
"텔레그램 자체가 파일 전송을 지원하지 않거나 제한하는 것이 아닌가?"

### 실험 방법
direct outbound 경로를 통해 로컬 PDF 파일을 텔레그램으로 직접 전송하는 테스트를 수행했다.

### 결과
- Telegram API document send는 정상 동작함
- OpenClaw의 Telegram sender는 PDF document 업로드를 지원함
- MIME/type 처리에도 문제가 없음

### 결론
"텔레그램이 PDF를 못 보낸다"는 가설은 **틀렸다**. 문제는 텔레그램이 아니라 OpenClaw 내부 경로에 있었다.

## 실험 2: Assistant normal reply의 MEDIA path 차단 확인

### 가설
"assistant normal reply pipeline이 특정 경로 패턴을 정책적으로 차단하는 것이 아닌가?"

### 실험 방법
assistant reply 본문에 다양한 형태의 `MEDIA:` 라인을 넣어 전송을 시도했다.

테스트한 경로 패턴:
- `MEDIA:/home/<user>/documents/test.pdf` — 임의의 host-local absolute path
- `MEDIA:file:///home/<user>/documents/test.pdf` — file URI scheme

### 결과
위 형태는 모두 차단 대상이었다. assistant가 문법상 맞는 `MEDIA:` 라인을 만들더라도, normal reply pipeline이 절대 로컬 경로를 정책적으로 거부하여 실제 전송이 실패했다.

### 결론
OpenClaw의 normal reply media normalization 경로에는 host-local absolute path 제한이 존재했다. 이것이 파일 전송 실패의 직접적 원인이었다.

## 실험 3: Managed outbound media path 발견

### 가설
"OpenClaw가 관리하는 특정 디렉터리 아래 파일은 차단 정책에서 예외일 수 있다."

### 실험 방법
`~/.openclaw/media/outbound/` 경로 아래에 파일을 배치하고, assistant reply를 bare line 하나로만 구성하여 전송을 시도했다.

```
MEDIA:/home/<user>/.openclaw/media/outbound/test/hello.txt
```

### 결과
**전송 성공.** 모든 로컬 absolute path가 동일하게 취급되는 것이 아니라, managed outbound media root(`~/.openclaw/media/outbound/`) 아래 파일은 실제 전달 경로에서 유효하게 작동했다.

### 결론
이것이 핵심 발견이었다. OpenClaw는 자체 managed outbound directory 아래의 파일에 대해서는 전송을 허용했다.

## 실험 4: 텍스트 파일 전송 검증

### 실험 방법
다음 파일을 생성했다.
- `~/.openclaw/media/outbound/test/hello.txt`

그리고 reply 본문을 아래 한 줄만 포함하도록 구성하여 전송 테스트를 했다.
- `MEDIA:/home/<user>/.openclaw/media/outbound/test/hello.txt`

### 결과
managed outbound directory를 가리키는 bare `MEDIA:` line이 실제 파일 전달을 유발하는 것을 확인했다. 텍스트 파일 전송 성공.

## 실험 5: PDF 전송 검증

### 실험 방법
같은 방식으로 여러 PDF를 생성하고 managed outbound 경로에 배치한 뒤 전송 테스트를 수행했다.

테스트 파일:
- `this-is-a-pdf.pdf`
- `yellow-black-text.pdf`

### 결과
모든 PDF 전송 성공. managed outbound media 경로를 이용한 텔레그램 파일 전송이 텍스트뿐 아니라 PDF에서도 실무적으로 동작함을 확인했다.

## 실험 6: `tools.allow` 설정 변경 — 실패한 실험

### 가설
"`openclaw.json`에서 `message` tool을 명시적으로 허용하면 파일 전달을 더 올바른 경로로 유도할 수 있지 않을까?"

### 실험 방법
OpenClaw 설정 파일 `~/.openclaw/openclaw.json`의 `tools` 섹션에 다음을 추가했다.

```json
"tools": {
  "profile": "coding",
  "allow": ["message"]
}
```

의도는 기존 coding profile을 유지하면서, message tool을 명시적으로 허용하는 것이었다.

### 결과: 예상치 못한 부작용
이 설정은 **심각한 부작용을 일으켰다.** `"allow": ["message"]`가 message tool만 허용하는 whitelist로 작동하면서, 다른 모든 tool의 실행이 억제되었다. 그 결과:

- assistant가 tool로 바로 조회 가능한 요청에도 실제 실행 없이 대답만 반복
- "아직 확인 안 했다"는 말만 하는 루프가 발생
- tool 기반의 작업이 전반적으로 마비됨

또한 원래 의도했던 message tool 노출도 현재 채팅 세션에서는 실현되지 않았다. static config만으로는 session-level tool exposure 문제를 해결할 수 없었다.

### 조치
해당 설정을 즉시 제거했다. 이 실험을 통해 `tools.allow`가 additive가 아니라 exclusive whitelist로 작동한다는 것을 확인했다.

## AGENTS.md 운영 규칙 보강
실험 결과를 바탕으로, 실패를 줄이고 assistant의 동작을 더 안정화하기 위해 AGENTS.md에 두 가지 운영 규칙을 추가했다.

### 1. Act Before Replying 규칙
실험 6의 `"allow": ["message"]` 설정으로 인해 assistant가 tool 실행 없이 대답만 반복하는 문제가 발생했다. 해당 설정을 제거한 뒤, 재발 방지를 위해 다음 규칙을 AGENTS.md에 추가했다.

```markdown
## Act Before Replying

If a user asks for information that can be retrieved with an available tool, use the tool first in the same turn. Do not respond with intention, apology, or delay before attempting the tool call.

Do not say you have not checked, searched, or executed something unless you actually attempted it in this turn or hit a real blocker.

If blocked, state the exact blocker clearly, such as a missing tool, permission restriction, network failure, missing input, or invalid configuration.

Do not repeat non-results across multiple turns. After one failure to act, either perform the action immediately or explain the precise blocker.
```

### 2. MEDIA 파일 전송 규칙 (Sending files via chat channels)
실험 2~5에서 확인한 reply-media normalizer의 동작 방식을 기반으로, 구체적인 파일 전송 규칙을 AGENTS.md에 정리했다.

```markdown
## Sending files via chat channels (MEDIA directives)

Reply-media normalizer (`reply-media-paths.runtime-B671n_FS.js`) whitelists **absolute paths under these managed roots only**:

- `~/.openclaw/media/outbound/…`
- `~/.openclaw/media/tool-*/…`

Everything else goes through `mediaAccess` policy checks and is usually **silently dropped** (search logs for `dropping blocked reply media`).

Rules by file source:

- **Image/music/video generation tools** (`image.generate` etc.): auto-saved under `~/.openclaw/media/tool-*/`. The tool result already contains a `MEDIA:<absolute-path>` line — just include that line verbatim in your reply. No extra staging.
- **Browser outputs** (`browser.screenshot`, downloads): saved under `~/.openclaw/media/browser/` which is **NOT** whitelisted. Copy the file first:
  `cp <browser-path> ~/.openclaw/media/outbound/<YYYY-MM-DD>/<filename>`
  then reference the new absolute path with `MEDIA:`.
- **`exec`-created files** (PDF, MD, DOCX, arbitrary): write the file directly under `~/.openclaw/media/outbound/<YYYY-MM-DD>/<filename>` and emit:
  `MEDIA:/home/alexlee/.openclaw/media/outbound/<YYYY-MM-DD>/<filename>`

Hard rules:

- **Use absolute paths** to a managed root. Do not use relative `MEDIA:outbound/…` — relatives are resolved against the workspace dir, not the media dir, and behave differently.
- **Never** use `MEDIA:file://…` — explicitly rejected (`"Host-local MEDIA file URLs are blocked in normal replies."`).
- **Send the `MEDIA:` line bare** — it must be the entire reply, on its own, with no accompanying caption, no leading `[[reply_to_current]]`, no backticks or markdown wrapping. Verified: `MEDIA:/home/alexlee/.openclaw/media/outbound/…` alone delivers; the same path with a directive tag or prefix text has been observed to silently drop.
- If you need caption text, send the file first as a bare-line reply, then send the caption as a separate follow-up message.
- Namespace filenames with a date or session prefix to avoid collisions.
```

## Claude Code와의 관련성
이 작업은 Claude Code에서 OpenClaw 설정을 수정하는 맥락에서 진행되었다. 실제로 다음 두 범주로 나눌 수 있다.

1. **설정 변경 작업**
   - OpenClaw config에서 `message` 허용 시도 및 실패 확인
   - agent instructions(AGENTS.md) 보강

2. **런타임 이해 기반 실험**
   - 전송 경로별 차단 정책 실험 (실험 1~3)
   - managed outbound media directory 발견 및 검증 (실험 3~5)
   - `tools.allow` whitelist 동작 방식 확인 (실험 6)

즉, 설정 파일만 바꿨다고 끝나는 작업이 아니라, 설정, 런타임 이해, 출력 형식 discipline을 함께 맞춰야 실제 결과가 나왔다.

## 주의사항
- `tools.allow`는 additive가 아닌 exclusive whitelist로 작동한다. 특정 tool만 추가하려는 목적으로 사용하면 다른 모든 tool이 차단된다.
- session-level tool exposure는 static config와 별개이므로, config 변경 후 반드시 실제 세션에서 검증이 필요하다.
- `MEDIA:` 전송의 구체적인 규칙은 위 AGENTS.md 운영 규칙 보강 섹션을 참고한다.

## 최종 결론
이번 작업으로 텔레그램 파일 전송을 실사용 가능한 수준으로 정리할 수 있었다.

6번의 실험을 통해 확인된 핵심 사실:
- 텔레그램 파일 전송 자체는 정상 동작한다 (실험 1)
- 실패의 주된 원인은 OpenClaw normal reply media restriction이다 (실험 2)
- `~/.openclaw/media/outbound/...` 기반 managed outbound media path는 유효한 해결책이다 (실험 3~5)
- `tools.allow: ["message"]` 설정은 오히려 tool 실행을 마비시키는 부작용을 일으켰다 (실험 6)
- agent operating rules 보강은 설정 실수로 인한 failure mode 재발을 방지하는 데 효과적이었다

성공한 해결책은 단일 magic setting이 아니라, 단계적 실험을 통한 런타임 이해와 그에 맞춘 운영 방식의 결합이었다.
