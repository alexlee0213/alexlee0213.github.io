---
layout: single
title: "OpenClaw에서 Google Calendar iCal URL로 일정 조회 및 아침 알림 cron 설정하기"
date: 2026-04-16 01:45:38 +0900
categories: [research]
tags: [OpenClaw, Google Calendar, iCal, cron, Telegram, automation]
excerpt: "Google 계정 OAuth 연동 없이 private iCal URL만 사용해 OpenClaw에서 오늘 일정을 조회하고, 매일 아침 8시 텔레그램으로 자동 발송하도록 설정한 과정을 정리한 연구 노트."
---
이 문서는 **Google 계정 OAuth 연동 없이**, Google Calendar의 **private iCal(.ics) URL**을 사용해 OpenClaw에서 오늘 일정을 읽고, 매일 아침 텔레그램으로 자동 발송하도록 설정한 과정을 정리한 가이드입니다.

## 목표

- Google Calendar의 private iCal URL 사용
- OpenClaw에서 오늘/내일/이번 주 일정 조회
- 여러 캘린더(personal, family, work) 동시 조회
- 매일 오전 8시(Asia/Seoul) 텔레그램으로 오늘 일정 자동 발송

---

## 1. 왜 iCal URL 방식을 썼는가

이 방식은 다음 상황에 잘 맞습니다.

- Google OAuth 연동을 피하고 싶을 때
- 읽기 전용 조회만 필요할 때
- "오늘 일정 알려줘" 같은 간단한 사용 흐름이 목적일 때

장점:
- 설정이 비교적 단순함
- Google 계정 권한 부여 없이 조회 가능
- 일정 수정 권한이 없어서 범위가 좁고 안전함

주의:
- **private iCal URL은 비밀번호처럼 취급**해야 함
- Google iCal 피드는 반영이 약간 늦을 수 있음
- 반복 일정/예외 일정 처리는 구현 품질이 중요함

---

## 2. 구현한 파일들

OpenClaw workspace 기준:

- `calendar/calendar_ics.py`
  - iCal fetch + 파싱 + 일정 필터링 스크립트
- `calendar/calendar-ics.example.json`
  - 설정 예시 파일
- `calendar/calendar-ics.local.json`
  - 실제 private iCal URL을 넣는 로컬 설정 파일
- `calendar/README.md`
  - 구현 요약 및 사용법
- `calendar/OPENCLAW-INTEGRATION.md`
  - OpenClaw에서 호출할 때의 연결 메모
- `skills/ical-calendar/SKILL.md`
  - OpenClaw가 이 기능을 로컬 skill로 인식하도록 한 문서
- `calendar/today_schedule.sh`
  - 간단 실행 래퍼

---

## 3. 핵심 파서에서 처리한 것

`calendar/calendar_ics.py`에서 다음을 지원하도록 구현했습니다.

### 시간대 처리

#### UTC → KST 변환

예:

```text
DTSTART:20260415T063000Z
```

- `Z` 접미사는 UTC 의미
- `Asia/Seoul` 기준으로 +9시간 변환
- 결과: `15:30`

#### TZID 처리

예:

```text
DTSTART;TZID=Asia/Seoul:20260415T153000
```

- 이미 `Asia/Seoul` 기준이므로 그대로 해석

### 종일 일정 처리

예:

```text
DTSTART;VALUE=DATE:20260415
DTEND;VALUE=DATE:20260416
```

- 시간 없는 DATE 타입은 종일 일정으로 처리
- `DTEND`가 다음 날인 것은 ICS 표준의 exclusive end 관례

### 반복 일정 처리

지원 범위:
- `DAILY`
- `WEEKLY` + `BYDAY`
- `MONTHLY`
- `YEARLY`
- `INTERVAL`
- `COUNT`
- `UNTIL`
- `EXDATE`
- `RECURRENCE-ID` override

### 복수 캘린더

- 여러 iCal URL을 동시에 읽도록 구성
- 예: personal / family / work

### 중복 제거

- UID, 제목, 시간 범위, 장소 등을 바탕으로 중복 이벤트 제거

### 조회 범위

- 오늘 (`today`)
- 내일 (`tomorrow`)
- 이번 주 (`week`)

---

## 4. 설정 파일 작성

실제 URL은 아래 파일에 넣습니다.

- `~/.openclaw/workspace/calendar/calendar-ics.local.json`

예시:

```json
{
  "calendars": [
    {
      "name": "personal",
      "url": "https://calendar.google.com/calendar/ical/.../basic.ics",
      "enabled": true
    },
    {
      "name": "family",
      "url": "https://calendar.google.com/calendar/ical/.../basic.ics",
      "enabled": true
    },
    {
      "name": "work",
      "url": "https://calendar.google.com/calendar/ical/.../basic.ics",
      "enabled": true
    }
  ]
}
```

설명:
- `name`: 캘린더 표시 이름
- `url`: Google Calendar private iCal URL
- `enabled`: 조회 포함 여부

보안 원칙:
- 이 파일은 로컬에만 보관
- 위키, 메모리 파일, 일반 문서에 URL을 복사하지 않기
- Git 커밋에 포함하지 않기

---

## 5. 수동 조회 명령

### 오늘 일정

```bash
python3 ~/.openclaw/workspace/calendar/calendar_ics.py \
  --config ~/.openclaw/workspace/calendar/calendar-ics.local.json
```

### 내일 일정

```bash
python3 ~/.openclaw/workspace/calendar/calendar_ics.py \
  --config ~/.openclaw/workspace/calendar/calendar-ics.local.json \
  --range tomorrow
```

### 이번 주 일정

```bash
python3 ~/.openclaw/workspace/calendar/calendar_ics.py \
  --config ~/.openclaw/workspace/calendar/calendar-ics.local.json \
  --range week
```

실제 조회 예시:

```text
YYYY-MM-DD 일정
- 09:10-09:30 아침 스크럼 [work]
- 15:00-16:30 주간 미팅 [work]
- 18:00-19:00 치과 w/t <가족 구성원> [family]
```

이처럼 캘린더 이름까지 함께 표시되도록 했습니다.

---

## 6. OpenClaw skill 연결

로컬 skill 파일:

- `skills/ical-calendar/SKILL.md`

이 skill에는 다음 내용이 들어 있습니다.

- 언제 이 skill을 써야 하는지
  - 예: "오늘 일정 알려줘", "내일 일정 보여줘", "이번 주 일정 브리핑해줘"
- 어떤 명령으로 조회할지
- 보안 원칙
- 응답 스타일

즉, OpenClaw가 일정 관련 요청을 받았을 때 이 로컬 ICS 조회 경로를 타도록 정리한 것입니다.

---

## 7. 아침 텔레그램 알림 cron 설정

매일 오전 8시 텔레그램으로 오늘 일정을 보내도록 OpenClaw cron job을 등록했습니다.

### 실제로는 터미널에서 직접 실행할 일이 거의 없다

아래에 `openclaw cron add ...` 명령을 자세히 적어뒀지만, 실제 운영에서는 이 명령을 직접 입력할 기회가 거의 없습니다. 이유는 간단합니다 — **텔레그램으로 자연어 요청만 보내면 OpenClaw가 알아서 cron 잡을 만들어주기 때문**입니다.

예시:

> "이제 이 캘린더 오늘의 일정 알림을 매일 오전 8시에 텔레그램으로 발송하도록 설정해줘"

이렇게만 보내면 OpenClaw는 다음을 자동으로 처리합니다.

- 적절한 잡 이름 생성 (`telegram-calendar-daily-8am` 같은 형태)
- 스케줄(`0 8 * * *`) · 타임존(`Asia/Seoul`) 설정
- 채널/대상(`telegram` / 내 user ID)
- 세션 방식(`isolated`) 및 `--light-context`
- `calendar_ics.py` 호출 메시지까지 작성

따라서 아래의 CLI 명령은 **"내부적으로 이런 형태의 잡이 등록된다"는 레퍼런스** 정도로 봐 두면 충분하고, 필요하면 잡 내용을 조정할 때만 참고하면 됩니다.

### 내부적으로 등록되는 잡

생성한 잡:
- 이름: `telegram-calendar-daily-8am`
- 스케줄: `0 8 * * *`
- 타임존: `Asia/Seoul`
- 타겟 채널: Telegram
- 타겟 사용자: `<YOUR_TELEGRAM_USER_ID>`
- 세션 방식: `isolated`

추가 명령 예시 (참고용 — 실제로는 위의 텔레그램 자연어 요청으로 자동 생성됨):

```bash
openclaw cron add \
  --name "telegram-calendar-daily-8am" \
  --description "Send today's calendar schedule to Telegram every day at 8 AM Asia/Seoul" \
  --agent main \
  --session isolated \
  --announce \
  --channel telegram \
  --to <YOUR_TELEGRAM_USER_ID> \
  --cron "0 8 * * *" \
  --tz "Asia/Seoul" \
  --light-context \
  --message "Read and summarize today's schedule using the local ical-calendar skill and ~/.openclaw/workspace/calendar/calendar_ics.py with ~/.openclaw/workspace/calendar/calendar-ics.local.json. Reply in Korean. Include time overlap warnings if any. If there are no events, say so clearly."
```

확인 명령:

```bash
openclaw cron list
openclaw cron status
```

등록 확인 결과 예시:

- job name: `telegram-calendar-daily-8am`
- target: `telegram`
- next run: 다음 날 오전 8시

---

## 8. cron에서 왜 `isolated` 세션을 썼는가

`main` 세션이 아니라 `isolated` 세션을 사용한 이유:

- 스케줄 작업이 기존 채팅 맥락에 오염되지 않게 하기 위해
- 매일 같은 목적의 작은 작업을 가볍게 수행하기 위해
- 토큰 사용량을 줄이기 위해
- 일정 알림이 일반 대화 문맥에 휘둘리지 않도록 하기 위해

그리고 `--light-context`를 같이 써서 더 가볍게 실행되도록 했습니다.

---

## 9. 운영 중 확인한 문제와 해결

### 문제 1. 설정 파일 변경 후 재시작이 필요한가?

아니오.

`calendar_ics.py`는 실행할 때마다 설정 파일을 다시 읽습니다.
따라서 `calendar-ics.local.json` 수정 후 재시작 없이 즉시 반영됩니다.

### 문제 2. URL 넣었는데도 `일정 없음`

가능한 원인:
- placeholder URL 그대로 남아 있음
- `enabled: true`가 아님
- 잘못된 캘린더 URL 입력
- 실제로 오늘 일정이 없음
- Google iCal 피드 반영 지연
- 반복 일정/예외 일정 특이 케이스

### 문제 3. cron 추가 시 옵션 차이

처음에는 `--expr`를 사용하려 했지만, 실제 CLI는 `--cron` 옵션을 사용해야 했습니다.

또 `main` 세션 cron은 `--system-event`를 요구해서, 최종적으로는 `--session isolated`로 설정했습니다.

---

## 10. 보안 정리

이 방식에서 중요한 보안 포인트:

1. **private iCal URL은 비밀번호처럼 취급**
2. 채팅, 위키, memory 파일에 URL 기록 금지
3. Git 커밋 금지
4. 기본 응답에는 일정 내용만 포함
5. URL 원문이나 토큰은 출력하지 않음
6. 로컬 설정 파일만 사용

즉, OAuth보다 간단하지만 **링크 자체가 비밀값**이라는 점을 항상 기억해야 합니다.

---

## 11. 이 문서를 바탕으로 다시 설정할 때 최소 순서

1. `calendar/calendar-ics.local.json` 생성 또는 수정
2. personal / family / work의 private iCal URL 입력
3. 수동 조회 테스트
   - 오늘 일정 확인
4. OpenClaw cron 등록
5. `openclaw cron list` / `openclaw cron status`로 검증
6. 다음 날 오전 8시 텔레그램 알림 확인

---

## 12. 추가 개선 아이디어

원하면 다음도 확장 가능:

- 주말에는 알림 제외
- 오전 8시 외 다른 시간 추가
- "지금부터 남은 일정만" 요약
- 겹치는 일정 강조 표시 강화
- 내일 일정 저녁 알림 추가
- 복수 채널(텔레그램 + 슬랙) 발송
- 더 복잡한 RRULE 지원 강화

---

## 13. 결론

OpenClaw에서 Google Calendar iCal URL 기반 일정 조회는 충분히 실용적입니다.

이 방식은:
- OAuth 없이 동작하고
- 읽기 전용이라 단순하며
- 개인용 일정 브리핑 자동화에 잘 맞습니다.

특히 이번 구성은 다음을 만족합니다.

- 오늘 일정 수동 조회 가능
- multiple calendars 지원
- 반복 일정 처리 지원
- 매일 오전 8시 텔레그램 자동 브리핑 가능

필요 시 이 문서를 바탕으로 다른 장치나 다른 OpenClaw 인스턴스에도 동일한 구성을 재현할 수 있습니다.
