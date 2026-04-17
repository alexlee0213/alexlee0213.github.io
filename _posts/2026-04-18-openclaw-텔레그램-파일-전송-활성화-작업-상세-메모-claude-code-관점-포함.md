---
layout: single
title: "OpenClaw 텔레그램 파일 전송 활성화 작업 상세 메모, Claude Code 관점 포함"
date: 2026-04-18 04:40:00 +0900
categories: [dev]
tags: [OpenClaw, Telegram, Claude Code, 파일전송, AGENTS.md]
excerpt: "OpenClaw에서 텔레그램 파일 전송이 실패하는 원인을 분석하고, managed outbound media path와 운영 규칙 보강으로 해결한 과정을 정리한 기술 메모."
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

## 핵심 관찰
실제로는 서로 다른 두 전송 경로가 존재했다.

1. **Direct outbound sending**
   - 명시적인 outbound send 경로를 통해 전송
   - 로컬 PDF를 텔레그램 document로 보내는 데 성공함

2. **Assistant normal reply delivery**
   - assistant가 reply 본문에 `MEDIA:` 라인을 넣는 방식
   - reply normalization 및 safety policy의 영향을 받음
   - host-local absolute path를 거부할 수 있음

즉, 파일 전송 실패의 핵심은 텔레그램 지원 부족이 아니라 OpenClaw가 어떤 reply path를 타느냐에 있었다.

## 기술적 분석 결과
### 1. Telegram document sending 자체는 정상
직접 outbound 테스트를 통해 로컬 PDF를 텔레그램으로 성공적으로 전송했다. 이로부터 다음 사실을 확인했다.
- Telegram API document send는 동작함
- OpenClaw의 Telegram sender는 PDF document 업로드를 지원함
- MIME/type 처리에도 문제가 없음

따라서 "텔레그램이 PDF를 못 보낸다"는 가설은 틀렸다.

### 2. Assistant normal reply는 absolute host-local MEDIA path를 차단
OpenClaw의 normal reply media normalization 경로에는 host-local absolute path 제한이 있었다.

실질적으로 다음과 같은 형태가 차단 대상이었다.
- `MEDIA:/home/<user>/...`
- `MEDIA:file:///home/<user>/...`

이 말은 assistant가 문법상 맞는 `MEDIA:` 라인을 만들더라도, normal reply pipeline이 절대 로컬 경로를 정책적으로 거부하면 실제 전송은 실패할 수 있다는 뜻이다.

### 3. Managed outbound media path는 실전에서 유효
실험 과정에서 다음 경로 아래의 파일은 전송에 적합했다.
- `~/.openclaw/media/outbound/...`

그리고 assistant reply를 bare line 하나로만 구성하여 다음처럼 보내면 실제 파일 전달이 동작했다.
- `MEDIA:/home/<user>/.openclaw/media/outbound/...`

이건 중요한 포인트다. 모든 로컬 absolute path가 동일하게 취급되는 것이 아니라, managed outbound media root 아래 파일은 실제 전달 경로에서 유효하게 작동했다.

## AGENTS.md 운영 규칙 보강
실패를 줄이고 assistant의 동작을 더 안정화하기 위해 AGENTS.md에 운영 규칙을 추가했다.

### 1. 파일 전송 규칙
assistant normal reply에서 임의의 local absolute path를 직접 쓰지 않도록 규칙을 넣었다.

핵심 의도는 다음과 같다.
- 일반 host-local absolute path를 피함
- 가능하면 message tool 또는 direct outbound 사용
- inline `MEDIA:` 가 불가피하면 managed outbound media storage를 사용

### 2. Act Before Replying 규칙
assistant가 tool로 바로 조회 가능한 요청에도 실제 실행 없이 "아직 확인 안 했다"는 말만 반복하는 루프가 발생했다. 이 문제의 근본 원인은 `openclaw.json`에 `"allow": ["message"]` 설정을 추가한 것이었다. message tool만 허용하면서 다른 tool 실행이 억제되고, 대답만 하는 현상이 생긴 것이다.

해당 설정을 제거한 뒤, AGENTS.md에 다음 운영 원칙을 추가하여 재발을 방지했다.
- tool로 해결 가능한 요청이면 같은 턴에서 먼저 실행
- 실제 시도하지 않았으면 "아직 확인 안 했다"고 말하지 않음
- blocker가 있으면 구체적으로 설명
- 같은 non-result 답변 반복 금지

설정 실수에서 비롯된 문제였지만, 운영 규칙 보강으로 visible failure mode를 줄이는 효과도 있었다.

## 파일 전송 검증 단계
### 텍스트 파일 검증
다음 파일을 생성했다.
- `~/.openclaw/media/outbound/test/hello.txt`

그리고 reply 본문을 아래 한 줄만 포함하도록 구성하여 전송 테스트를 했다.
- `MEDIA:/home/<user>/.openclaw/media/outbound/test/hello.txt`

이 방식은 managed outbound directory를 가리키는 bare `MEDIA:` line이 실제 파일 전달을 유발한다는 점을 검증했다.

### PDF 검증
같은 방식으로 여러 PDF를 생성하고 전송 테스트를 수행했다. 예시는 다음과 같다.
- `this-is-a-pdf.pdf`
- `yellow-black-text.pdf`

이 검증을 통해 managed outbound media 경로를 이용한 텔레그램 파일 전송이 실무적으로 동작함을 확인했다.

## Claude Code와의 관련성
질문은 "Claude Code에서 OpenClaw 설정을 수정한 작업"이라는 맥락을 포함하고 있었다. 실제로 이 작업은 다음 두 범주로 나눌 수 있다.

1. **설정 변경 작업**
   - OpenClaw config에서 `message` 허용 추가
   - agent instructions 보강

2. **운영 및 런타임 이해 기반 수정**
   - managed outbound media directory 사용
   - known-bad reply pattern 회피
   - 실제 Telegram delivery path 검증

즉, 설정 파일만 바꿨다고 끝나는 작업이 아니라, 설정, 런타임 이해, 출력 형식 discipline을 함께 맞춰야 실제 결과가 나왔다.

## 권장 운영 방식
### 권장 우선순위
1. **가장 좋은 방법:** message tool 또는 direct outbound send path 사용
2. **차선책:** `~/.openclaw/media/outbound/...` 아래 파일을 두고 bare `MEDIA:/absolute/path` line 으로 응답
3. **지양:** 임의의 host-local absolute path를 일반 assistant reply에 섞는 방식

### Managed outbound best practice
- 생성 파일은 다음처럼 namespace를 둔 디렉터리에 저장
  - `~/.openclaw/media/outbound/test/...`
  - `~/.openclaw/media/outbound/<date-or-task>/...`
- 가능하면 `MEDIA:` line만 단독으로 보냄
- caption이 필요하면 파일 전송 후 별도 메시지로 보냄

## 남은 과제
아직 해결되지 않은 이슈도 있었다.
- `tools.allow: ["message"]` 를 넣었는데도 현재 텔레그램 direct chat 세션에서 `message` tool이 실제 노출되지 않는 문제

이건 다음 영역에서 더 깊은 추적이 필요함을 뜻한다.
- tool profile assembly
- per-session tool filtering
- current-chat reply policy

## 최종 결론
이번 작업으로 텔레그램 파일 전송을 실사용 가능한 수준으로 정리할 수 있었다.

가장 중요한 결론은 다음과 같다.
- 텔레그램 파일 전송 자체는 정상 동작한다
- 실패의 주된 원인은 OpenClaw normal reply media restriction이다
- `message` allow 설정은 타당했지만 현재 세션 tool exposure 문제를 완전히 해결하진 못했다
- `~/.openclaw/media/outbound/...` 기반 managed outbound media path는 실질적인 우회책으로 유효했다
- agent operating rules를 강화하면 반복적인 non-action 문제를 크게 줄일 수 있다

즉, 성공한 해결책은 단일 magic setting이 아니라, 설정 변경, 런타임 이해, 그리고 출력 형식을 정확히 맞춘 운영 방식의 결합이었다.