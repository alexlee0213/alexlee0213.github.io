---
layout: single
title: "OpenClaw에 memory-wiki로 지식 저장소 구축하기 — Linux 서버 + Telegram/Discord 운영 환경 기준"
date: 2026-04-15 09:43:01 +0900
categories: [research]
tags: [OpenClaw, memory-wiki, AI Agent, Knowledge Base, Telegram, Discord, Linux, Self-hosted]
excerpt: "OpenClaw의 memory-wiki 플러그인을 사용해 provenance-rich 지식 저장소(vault)를 구축하는 절차와, 헤드리스 Linux 서버에서 Telegram/Discord로만 교신하는 운영 환경에서 Obsidian 연동이 실제로 필요한지에 대한 현장 판단 정리."
---
## TL;DR

OpenClaw에는 지속성 있는 지식 저장소를 만들기 위한 `memory-wiki` 플러그인이 있다. 이 플러그인은 기존 memory 시스템을 **대체하지 않고 옆에 붙어서**, 누적된 지식을 provenance(출처·근거)가 풍부한 wiki 페이지로 "컴파일"한다. 운영 환경이 헤드리스 Linux 서버이고 교신 채널이 Telegram/Discord로 한정된 경우, **Obsidian 연동은 필요 없고 `renderMode: "native"`가 더 적합**하다는 결론.

---

## 1. memory-wiki 개념 정리

OpenClaw의 메모리 시스템은 세 층으로 구성된다.

- `MEMORY.md` — 장기 지식
- `memory/YYYY-MM-DD.md` — 일일 노트
- `DREAMS.md` — 선택적 consolidation

`memory-wiki`는 이 시스템 **옆에** 얹히는 레이어다. recall / promotion / dreaming은 기존 memory 플러그인이 계속 담당하고, memory-wiki는 claims, dashboards, contradictions, freshness 같은 구조화된 지식 레이어를 추가한다. 즉 memory-wiki는 "지식 조립기" 역할에 가깝다.

## 2. 두 가지 Vault 모드

**Isolated 모드** — wiki가 독립된 큐레이션 저장소로 동작한다. memory-core에 의존하지 않으며, 수동 관리형 지식 베이스를 원할 때 적합하다.

**Bridge 모드** — 활성 memory 플러그인(예: QMD backend)이 내보내는 public memory artifacts와 memory events를 읽어서 자동으로 wiki를 조립한다. 공식 문서는 "QMD를 recall용 active memory로, memory-wiki를 bridge 모드로" 쓰는 하이브리드 구성을 권장한다.

현장 권장 순서는 **isolated로 시작 → 익숙해지면 bridge로 승격**이다.

## 3. 초기화

```bash
openclaw wiki init
openclaw wiki status
openclaw wiki doctor
```

`init`이 실행되면 vault 디렉터리에 표준 구조가 자동 생성된다.

```
<vault>/
  AGENTS.md
  WIKI.md
  index.md
  inbox.md
  entities/      # 사람, 조직, 프로젝트 등 개체
  concepts/      # 용어, 정의, 개념
  syntheses/     # 종합 정리 페이지
  sources/       # 원본 출처
  reports/       # 주기적 리포트
  _attachments/
  _views/        # dashboard 뷰
  .openclaw-wiki/
```

`doctor`는 구성 문제를 진단해주므로 설치 직후 한 번 돌려볼 것.

## 4. Isolated 모드 최소 설정 (시작용)

```json5
{
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "isolated",
          vault: {
            path: "/var/lib/openclaw/wiki/main",
            renderMode: "native"
          },
          render: {
            createDashboards: true
          }
        }
      }
    }
  }
}
```

vault 경로는 `~/.openclaw/...` 대신 서비스 계정이 접근 가능한 `/var/lib/openclaw/...` 같은 고정 경로를 권장한다. 서버 운영 관점에서 권한·백업 관리가 수월하다.

## 5. Bridge 모드 완전 설정 (권장 하이브리드)

```json5
{
  memory: {
    backend: "qmd"
  },
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "bridge",
          vault: {
            path: "/var/lib/openclaw/wiki/main",
            renderMode: "native"
          },
          bridge: {
            enabled: true,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true
          },
          ingest: {
            autoCompile: true,
            maxConcurrentJobs: 1,
            allowUrlIngest: true
          },
          search: {
            backend: "shared",
            corpus: "all"
          },
          context: {
            includeCompiledDigestPrompt: false
          },
          render: {
            preserveHumanBlocks: true,
            createBacklinks: true,
            createDashboards: true
          }
        }
      }
    }
  }
}
```

`bridge` 블록이 핵심이다. `readMemoryArtifacts`, `indexDreamReports`, `indexDailyNotes`가 켜져 있어야 active memory가 생성하는 결과물이 wiki로 흘러 들어온다.

## 6. 콘텐츠 워크플로 — ingest → compile → search

**주입 (ingest)**

```bash
openclaw wiki ingest ./notes/alpha.md
```

`allowUrlIngest: true`이면 URL도 바로 넣을 수 있다.

**컴파일 (compile)**

```bash
openclaw wiki compile
```

원자료를 기반으로 엔티티·개념·synthesis 페이지를 생성하고 기계가 읽을 수 있는 digest를 만든다. `autoCompile: true`로 두면 ingest 후 자동 실행된다.

**검색과 조회**

```bash
openclaw wiki search "your query"
openclaw wiki get entity.alpha
```

넓은 recall은 `memory_search corpus=all`로, 출처·신뢰도 기반의 정밀한 조회는 `wiki_search` / `wiki_get`으로 분리해 쓰는 것이 공식 가이드다.

**유지보수**

```bash
openclaw wiki lint     # 모순·저신뢰 주장·오래된 페이지 점검
openclaw wiki status   # 현재 상태 요약
openclaw wiki apply    # 제안된 변경사항 적용
```

Bridge 모드에서 수동 동기화가 필요할 때:

```bash
openclaw wiki bridge import
```

## 7. 운영 팁

- `render.createDashboards: true` — `_views/` 아래에 모순 탐지, 저신뢰 claim, stale 페이지 대시보드가 자동 생성되어 품질 관리가 쉬워진다.
- `preserveHumanBlocks: true` — 사람이 손으로 편집한 영역을 컴파일 시 덮어쓰지 않게 보호한다. 사람이 자주 손대는 vault라면 필수.
- 임베딩 provider가 설정되어 있어야 hybrid search가 제대로 동작한다. OpenClaw 메인 설정의 embedding 관련 항목을 함께 확인할 것.

---

## 8. 현장 판단 — Linux 서버 + Telegram/Discord 환경에서 Obsidian 연동이 필요한가?

### 결론: 필요 없다

**이유 1. Obsidian 연동의 목적 자체가 데스크톱 GUI 워크플로다.**
`obsidian.enabled: true`와 `renderMode: "obsidian"`은 사람이 데스크톱 GUI에서 vault를 직접 열람·편집하는 상황을 가정한다. 구체적으로는 Obsidian CLI를 통해 "vault 열기", "daily note 점프", "페이지 포커스" 같은 데스크톱 액션을 트리거하고, 링크·프런트매터 포맷을 Obsidian 규약에 맞춰 렌더링한다.

**이유 2. 헤드리스 서버에서는 트리거될 GUI가 없다.**
서버에는 Obsidian GUI가 없으니 "열기" 류 액션이 실행될 곳이 없고, `openAfterWrites` 같은 옵션은 no-op이 되거나 실패 로그만 남긴다.

**이유 3. Telegram/Discord 출력 가독성이 떨어진다.**
Obsidian 전용 문법(`[[wikilinks]]`, 특유의 프런트매터 등)이 그대로 채팅 메시지에 노출되면 가독성이 나빠진다. `renderMode: "native"`가 표준 Markdown 링크를 사용하므로 채널 렌더링에 훨씬 깔끔하다.

**이유 4. 운영 부담 증가.**
Obsidian CLI 의존성이 추가로 붙는다. 헤드리스 서버에서 의미 없는 의존성을 유지할 이유가 없다.

### 권장 설정

`renderMode`를 `"native"`로 두고 `obsidian` 블록은 통째로 생략한다. 앞의 **섹션 4·5의 설정 예시가 이미 이 방침을 따른다.**

### 나중에 로컬 편집이 필요해지면?

두 가지 선택지가 있다.

1. **Syncthing / rsync / Git으로 vault 디렉터리만 로컬에 동기화** — 서버 설정은 `native` 그대로 유지하고, 로컬 Obsidian으로 열어본다. **가볍고 되돌리기 쉬워서 1순위 추천.**
2. `renderMode`를 `"obsidian"`으로 전환 — 실제로 데스크톱 중심 워크플로로 이동할 때.

서버 운영 중에는 1번이 거의 항상 답이다.

---

## 9. 권장 시작 경로

1. **isolated + native**로 최소 구성 기동
2. `wiki init → ingest → compile → search` 흐름 숙지
3. 품질 관리 루틴으로 `lint` / dashboards 습관화
4. 필요 시 memory backend를 QMD로 맞추고 **bridge 모드로 승격**
5. 로컬 편집 수요가 생기면 **vault만 동기화** (Obsidian 연동은 마지막 선택지)

전환 중에도 기존 `MEMORY.md` / 일일 노트는 그대로 유지되므로 손실 위험이 없다.

---

## 참고 자료

- [openclaw/openclaw GitHub](https://github.com/openclaw/openclaw)
- [OpenClaw Official Documentation](https://docs.openclaw.ai)
- [Memory Wiki Plugin Docs](https://docs.openclaw.ai/plugins/memory-wiki)
- [Memory concepts (GitHub)](https://github.com/openclaw/openclaw/blob/main/docs/concepts/memory.md)
