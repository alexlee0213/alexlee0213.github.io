---
layout: single
title: "Zotero를 Claude에 연결하기 — MCP 설치·설정 완전 가이드 (macOS)"
date: 2026-07-31 14:40:58 +0900
categories: [dev]
tags: [Zotero, MCP, Claude, ClaudeCode, 연구도구, macOS, 설치가이드]
excerpt: "zotero-mcp를 이용해 로컬 Zotero 라이브러리를 Claude에 연결하는 방법. 설치, 로컬 API 활성화(Zotero 9 기준), Claude Desktop·Claude Code 등록, 설정 마법사 옵션, 그리고 쓰기 작업을 위한 하이브리드 모드까지 정리했다."
---
Zotero 라이브러리를 MCP(Model Context Protocol)로 Claude에 연결하면, Claude가 내 소장 논문을 직접 검색·열람·요약할 수 있다. 이 글은 커뮤니티 서버 `54yyyu/zotero-mcp`를 로컬 방식으로 설치하는 전 과정을 정리한다. 로컬 방식은 API 키가 필요 없고 PDF 전문까지 접근할 수 있다.

## 사전 준비물

- Zotero 7 이상 (로컬 API + 전문 접근)
- Python 3.10 이상
- Claude Desktop (MCP 클라이언트)

## 1. 설치

`uv`로 설치하는 방식이 가장 깔끔하다.

```bash
brew install uv
uv tool install zotero-mcp-server
zotero-mcp version   # 설치 확인
```

pip를 선호하면 `pip install zotero-mcp-server`, pipx면 `pipx install zotero-mcp-server`로도 된다.

## 2. Zotero에서 로컬 API 켜기 (중요)

여기서 문서마다 표현이 갈리는데, 최신 Zotero(9.x)에는 별도의 "Enable local API" 체크박스가 없다. **토글 하나**가 곧 로컬 API 스위치다.

**Settings → Advanced → General** 에서 **"Allow other applications on this computer to communicate with Zotero"** 를 체크한다. 이걸 켜면 로컬 HTTP API(포트 23119)가 활성화된다.

그 토글조차 안 보이면 Config Editor로 직접 켠다. Settings → Advanced → Config Editor에서 `extensions.zotero.httpServer.enabled`를 true, `extensions.zotero.httpServer.port`를 23119로 설정하고 재시작한다.

동작 확인은 Zotero를 켜둔 채 브라우저에서 `http://localhost:23119/api/users/0/items`에 접속해 JSON 응답(또는 빈 배열)이 나오는지 보면 된다. 그리고 **사용 중에는 Zotero 앱이 항상 실행되어 있어야 한다** — 로컬 API는 앱이 서버 역할을 한다.

## 3. Claude Desktop 자동 설정

```bash
zotero-mcp setup
```

이 명령이 `~/Library/Application Support/Claude/claude_desktop_config.json`에 서버를 자동 등록한다. 수동으로 하려면 다음을 추가한다.

```json
{
  "mcpServers": {
    "zotero": {
      "command": "zotero-mcp",
      "env": { "ZOTERO_LOCAL": "true" }
    }
  }
}
```

Claude Desktop을 완전히 종료(⌘Q) 후 재시작하면 `zotero_search_items` 같은 도구가 나타난다.

## 4. Claude Code에도 연결하기

Claude Code를 쓴다면 바이너리는 이미 설치돼 있으니 등록만 하면 된다.

```bash
claude mcp add zotero -s user -e ZOTERO_LOCAL=true -- zotero-mcp
claude mcp list   # 확인
```

참고로 Claude Code에는 함께 설치되는 `zotero-cli`가 더 효율적일 수 있다. 셸에서 직접 호출하므로 MCP 도구 스키마를 매번 로드하는 것보다 토큰을 훨씬 덜 먹고 스크립트·자동화에 잘 맞는다. `zotero-cli search "..."` 처럼 쓴다. 제작자도 대화형 클라이언트엔 MCP, 셸 파이프라인엔 CLI를 권장한다.

## 5. 설정 마법사 옵션 해설

`zotero-mcp setup`을 돌리면 시맨틱 검색 관련 옵션을 몇 개 묻는다. 의미를 정리하면 이렇다.

**DB 갱신 주기** — 새로 추가한 논문이 "의미 기반 검색"에 잡히도록 임베딩 DB를 언제 갱신할지 고른다. Manual(수동), Auto(서버 시작 시마다), Daily(하루 1회), Every N days 중 대부분은 **Daily**가 자동이면서 매번 느려지지 않아 최적이다. 참고로 일반 키워드 검색은 항상 실시간이라 이 설정과 무관하다.

**PDF 페이지 캡** — 시맨틱 검색 색인 시 PDF 한 개당 몇 페이지까지 텍스트를 뽑을지 정한다. 기본 10은 초록·서론 위주만 커버한다. 일반 학술 논문 위주라면 **15~20**으로 올리면 논문 대부분을 통째로 커버하면서 속도도 괜찮다. 이 값은 색인 깊이에만 영향을 주며, 특정 논문 전문을 직접 읽는 기능은 값과 무관하게 전체를 가져온다.

**임베딩 모델** — 텍스트를 벡터로 바꾸는 모델을 고른다. 판단 축은 품질·비용·프라이버시·설치 난이도다.

- **Default (all-MiniLM-L6-v2)**: 무료·완전 로컬·설정 불필요. 대부분 이걸로 시작하면 된다.
- **OpenAI / Gemini**: 품질이 우수하지만 API 키가 필요하고, 유료이며, 내 논문 텍스트가 외부 서버로 전송된다.
- **Ollama**: 로컬이면서 Default보다 좋은 모델을 쓸 수 있으나 Ollama를 따로 설치·실행해야 한다.

민감 자료가 많으면 Default나 Ollama, 최고 품질을 원하고 데이터 전송이 괜찮으면 OpenAI/Gemini다. 모델을 바꾸면 기존 임베딩과 호환이 안 되므로 `zotero-mcp update-db --force-rebuild`로 DB를 다시 빌드해야 한다.

## 로컬 방식의 한계

로컬 JS API 특성상 태그 편집·자료 추가 등 쓰기(write) 작업은 제대로 안 될 수 있다. 검색·읽기·주석 추출은 문제없다. 쓰기까지 필요하면 아래 하이브리드 모드를 설정한다.

## 6. 쓰기까지 하려면 — 하이브리드 모드

로컬 API는 읽기만 빠르고 쓰기가 안 되므로, 쓰기는 **Zotero 웹 API로 우회**시킨다. 로컬 읽기 + 웹 쓰기를 합친 것이 하이브리드 모드다. 필요한 준비물은 두 가지다.

**① Zotero API 키** — [zotero.org/settings/keys](https://www.zotero.org/settings/keys)에서 새 키를 생성하되, **쓰기 권한("Allow write access")을 반드시 허용**한다. 읽기 전용 키로는 쓰기가 안 된다.

**② Library ID** — 같은 페이지에 표시되는 **숫자 userID**. (그룹 라이브러리에 쓰려면 그룹 ID를 쓰고 타입도 group으로 지정한다.)

가장 쉬운 설정은 setup 마법사를 다시 도는 것이다. 기본 setup이 하이브리드 모드를 자동 구성하며 API 키만 넣으면 된다.

```bash
zotero-mcp setup
```

수동으로 하려면 `ZOTERO_LOCAL`은 그대로 두고 API 자격증명을 추가한다. **로컬 true + API 키가 함께 있으면 자동으로 하이브리드로 동작**한다.

```json
{
  "mcpServers": {
    "zotero": {
      "command": "zotero-mcp",
      "env": {
        "ZOTERO_LOCAL": "true",
        "ZOTERO_API_KEY": "발급받은_키",
        "ZOTERO_LIBRARY_ID": "숫자_userID",
        "ZOTERO_LIBRARY_TYPE": "user"
      }
    }
  }
}
```

수정 후 Claude Desktop을 완전히 재시작한다. Claude Code라면 `claude mcp add` 시 같은 `-e` 옵션들을 함께 넘기면 된다.

알아둘 점 몇 가지. 웹 API로 쓰면 Zotero 클라우드에 반영되므로 **로컬 앱이 동기화(Sync 로그인)돼 있어야** 로컬과 웹 라이브러리가 일치한다. PDF 첨부까지 웹으로 올리는 작업은 무료 300MB 용량에 영향을 줄 수 있다(메타데이터·태그 쓰기는 용량과 무관). 회사 계정이라면 키에 **필요한 최소 권한만** 부여하는 게 안전하다. 그리고 터미널에서 같은 환경변수를 export해두면 config 값을 덮어쓴다는 점도 기억하자.

이렇게 설정하면 "DOI로 논문 추가하고 오픈액세스 PDF 붙이기", "특정 자료들에 태그 달기", "새 컬렉션 만들어 자료 넣기", "중복 자료 병합", "메타데이터 수정" 같은 쓰기 작업을 Claude에게 시킬 수 있다.

## 트러블슈팅

- 결과가 안 나옴 → Zotero 앱이 켜져 있고 로컬 API 토글이 켜졌는지 확인
- 전문이 안 나옴 → Zotero 7+ 인지 확인
- 도구가 Claude에 안 뜸 → Claude Desktop 완전 재시작, config 확인
- 쓰기가 안 됨 → 하이브리드 설정과 API 키의 쓰기 권한 확인
- DB 오류 → `zotero-mcp update-db --force-rebuild`

다음 글에서는 이렇게 연결한 Zotero를 실제 연구 워크플로우에서 어떻게 활용하는지, 장점과 실전 팁을 다룬다.
