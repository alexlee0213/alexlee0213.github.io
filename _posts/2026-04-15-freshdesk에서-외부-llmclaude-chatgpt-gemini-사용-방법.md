---
layout: single
title: "Freshdesk에서 외부 LLM(Claude · ChatGPT · Gemini) 사용 방법"
date: 2026-04-15 09:48:00 +0900
categories: [research]
tags: [Freshdesk, Claude, ChatGPT, Gemini, LLM, MCP, iPaaS, 연동]
excerpt: "Freshdesk 내장 Freddy AI 대신 Anthropic Claude, OpenAI ChatGPT, Google Gemini 같은 외부 LLM을 연동하는 4가지 경로(Marketplace 앱, iPaaS Webhook, MCP, Bot Builder Custom API)를 비교 정리."
---
# Freshdesk에서 외부 LLM(Claude · ChatGPT · Gemini) 사용 방법

**작성일:** 2026년 4월 15일
**질문:** Freshdesk 내장 Freddy AI 대신, Anthropic Claude / OpenAI ChatGPT / Google Gemini 같은 외부 LLM을 쓸 수 있는가?

---

## 결론 요약

> **네, 가능합니다.** 단, Freshdesk는 공식적으로 **"Bring-Your-Own-LLM(BYO-LLM)" 기능을 Freddy AI에 직접 꽂는 방식은 아직 제공하지 않습니다.**
> 대신 ① **Marketplace 앱**, ② **Freshdesk API/Webhook을 통한 외부 오케스트레이션**, ③ **MCP(Model Context Protocol) 기반 에이전트 연동**, ④ **봇 빌더의 Custom API 노드** 4가지 경로로 Claude·ChatGPT·Gemini를 Freshdesk에 붙일 수 있습니다.

Freddy AI 자체는 내부적으로 멀티-LLM 파트너(자체 모델 + 주요 파운데이션 모델)를 쓰지만, **사용자가 자신의 Anthropic·OpenAI·Google API 키를 Freddy에 주입하는 형태의 공식 BYO-LLM 옵션은 2026년 4월 현재 Freshdesk에서 확인되지 않습니다.** 따라서 "Freddy를 우리 Claude로 바꾸고 싶다"는 요구는 아래 우회 경로를 통해 구현해야 합니다.

---

## 1. Freshworks Marketplace 앱 (가장 쉬운 경로)

Marketplace에 이미 공개된 써드파티 앱을 설치하고 **자신의 API 키만 등록**하면 됩니다. 상담사 화면이나 티켓 상세에 "AI" 버튼이 추가되는 형태입니다.

| 앱 | 제공 기능 | LLM |
|---|---|---|
| **ChatGPT Assistant** (Freshworks 공식 Marketplace) | 티켓 요약, 답변 생성, 톤 조정 | OpenAI |
| **TicketGPT** | 상담사용 컨텍스트 기반 자동 답변 생성 (GPT-3.5 권장) | OpenAI |
| **ChatGPT AI Integration by IntegrateCloud** (무료) | Freshdesk 데이터로 학습된 개인화 챗봇 | OpenAI |
| **ChatGPT Assistant (Multilingual + Custom Fields)** | 다국어·커스텀 필드 대응 | OpenAI |

**한계:** 현재 Marketplace의 네이티브 앱은 대부분 **OpenAI(ChatGPT) 중심**이며, Claude/Gemini 전용 공식 앱은 드뭅니다. Claude·Gemini는 아래 2~4번 방식이 현실적입니다.

---

## 2. Freshdesk API/Webhook + iPaaS(노코드 자동화)

가장 유연하고 **3사 LLM(Claude·ChatGPT·Gemini) 모두 지원** 가능한 방식입니다. Freshdesk의 티켓 이벤트를 webhook으로 받아 외부 LLM에 보내고, 결과를 다시 Freshdesk API로 돌려쓰는 구조입니다.

### 2.1 지원되는 iPaaS 플랫폼

| 플랫폼 | 지원 LLM | 특이점 |
|---|---|---|
| **Make (Integromat)** | ChatGPT/DALL·E/Whisper, Claude, Gemini | 시각적 시나리오, 분기/에러 핸들링 우수 |
| **n8n** | Claude, OpenAI, Gemini | 셀프호스팅 가능 → **데이터 주권** 확보 |
| **Zapier** | ChatGPT, Claude, Gemini (8,000+ 앱) | 가장 대중적, 설정 간편 |
| **Albato** | Claude Anthropic, OpenAI | Webhook 트리거 네이티브 지원 |
| **viaSocket** | Claude | 노코드 워크플로우 |
| **Pipedream** | OpenAI 등 | 코드 삽입 가능 |
| **Latenode / Appy Pie / Integrately** | OpenAI 중심 | 간단 1-클릭 통합 |

### 2.2 대표적인 패턴

1. **티켓 자동 분류/라우팅** — 신규 티켓 webhook → LLM에게 카테고리 분류 요청 → Freshdesk API로 태그·우선순위 업데이트
2. **답변 초안 자동 작성** — 티켓 생성 시 LLM이 사내 KB를 RAG로 참조해 답변 초안 생성 → Private Note로 상담사에게 전달
3. **감정/우선순위 분석** — 고객 메시지 → LLM 감정 분석 → 부정적이면 즉시 매니저 에스컬레이션
4. **다국어 요약/번역** — Claude/Gemini의 장문 번역을 활용해 글로벌 티켓 처리

---

## 3. MCP (Model Context Protocol) 기반 에이전트 연동

2025년 이후 **Claude·ChatGPT 같은 외부 에이전트가 Freshdesk를 "도구"처럼 호출**하는 방식이 보편화되고 있습니다.

- **Composio의 Freshdesk MCP 서버** — Claude Code, Claude Agent SDK, 또는 자체 에이전트에서 Freshdesk 티켓 생성·조회·업데이트·응답을 MCP 툴 콜로 수행
- OAuth·토큰 관리·API breaking change 대응을 MCP가 대신 처리
- **Freshdesk에 내장되는 것이 아니라, Claude 쪽에서 Freshdesk를 조종하는 구조** — 내부 운영팀이 Claude Desktop/Claude Code로 "A고객 티켓 요약해줘, 답변 초안 써줘"처럼 사용할 때 유용

**주의:** 이 방식은 **고객 응대용 챗봇을 대체하는 용도라기보다, 내부 상담사/운영팀이 외부 LLM으로 Freshdesk를 원격 조작**하는 시나리오에 적합합니다.

---

## 4. Freddy Bot Builder의 Custom API 노드 활용

Freshdesk/Freshchat의 봇 빌더 안에 **"Connect APIs"** 기능이 있어, 봇 대화 흐름 중 임의의 HTTP 엔드포인트를 호출할 수 있습니다.

- 봇 대화 중 특정 의도(Intent) 매칭 시 → **Anthropic/OpenAI/Google API 엔드포인트로 직접 POST**
- 응답을 변수에 저장해 봇의 다음 응답에 삽입
- 자사 API 게이트웨이를 앞단에 두고 **다중 LLM A/B 테스트**, **프롬프트 버전 관리**, **PII 마스킹** 등도 가능

**활용 예:** Freddy의 기본 응답은 유지하되, 특정 복잡한 질문(예: 기술 지원 FAQ)만 Claude Sonnet API로 라우팅해 품질을 높이는 하이브리드 구성.

---

## 5. 운영 관점 체크리스트

외부 LLM을 붙일 때 반드시 점검해야 할 항목입니다.

- **API 키 관리** — Freshdesk Marketplace 앱/iPaaS에 직접 키를 저장하지 말고, 사내 Secret Manager(AWS Secrets Manager, Vault 등) 경유 권장
- **개인정보(PII) 처리** — 고객 이메일·주문정보가 외부 LLM 프롬프트로 흘러갈 수 있음. **마스킹 또는 지역 리전 LLM(예: Claude on Bedrock Tokyo, Gemini on Vertex Seoul) 사용** 고려
- **비용 폭증 방지** — webhook마다 LLM 호출이 트리거되므로 **레이트 리밋·캐싱·세션 묶음** 필수
- **로깅·감사** — 누가 어떤 프롬프트로 고객 답변을 생성했는지 추적 가능해야 함
- **지연(latency)** — 상담사 UI에 인라인으로 붙일 경우 2~3초 이내 응답 필요. 대형 모델은 스트리밍 응답 구현 권장
- **품질 관리** — "상담사가 반드시 검수 후 발송" 규칙을 기본값으로, 완전 자동 발송은 좁은 범위부터 단계적 롤아웃

---

## 6. 시나리오별 추천

| 상황 | 추천 방식 |
|---|---|
| "우리는 이미 OpenAI 계정 있음. 빠르게 써보고 싶다" | **① Marketplace의 ChatGPT Assistant 설치** |
| "Claude로 상담사 답변 초안을 자동화하고 싶다" | **② Make/n8n으로 webhook → Claude Messages API → Private Note 생성** |
| "Gemini의 긴 컨텍스트로 고객 전체 이력을 분석하고 싶다" | **② n8n(셀프호스팅) → Gemini 1.5 Pro** |
| "내부 운영팀이 Claude Desktop에서 티켓 관리" | **③ Composio Freshdesk MCP** |
| "Freddy는 유지하되, 특정 플로우만 외부 LLM으로 교체" | **④ Freddy Bot Builder의 Connect APIs 노드** |
| "규제·보안 요구가 엄격한 엔터프라이즈" | **② n8n 셀프호스팅 + Bedrock/Vertex의 리전 LLM** |

---

## 7. 핵심 한계 (2026년 4월 기준)

1. Freshdesk/Freddy는 **공식 BYO-LLM 기능을 제공하지 않습니다.** Freddy 내부에서 "내 Claude 키로 돌려줘"는 불가.
2. Marketplace 기본 앱은 **OpenAI 편중** — Claude·Gemini는 iPaaS 또는 커스텀 개발이 사실상 필요.
3. 외부 LLM 호출은 Freddy AI Copilot UI와 **완전히 매끄럽게 통합되지는 않음** — 상담사 입장에서는 별도 버튼/앱/탭이 되기 쉬움.
4. 대규모 트래픽에서 **Freddy AI Agent(세션 팩 $100/1,000)보다 외부 API가 더 비싸질 수 있음.** TCO 비교 필수.

---

## 출처 (Sources)

- [Freshworks ChatGPT Assistant Integration | Freshworks Marketplace](https://www.freshworks.com/apps/chatgpt_assistant/)
- [Freshworks TicketGPT Freshdesk Integration | Freshworks Marketplace](https://www.freshworks.com/apps/ticket-gpt_1/)
- [Freshworks ChatGPT AI integration By IntegrateCloud | Freshworks Marketplace](https://www.freshworks.com/apps/chatgpt_ai_integration_by_integratecloud/)
- [Freshdesk and OpenAI | Make](https://www.make.com/en/integrations/freshdesk/openai-gpt-3)
- [Freshdesk and OpenAI: Automate Workflows with n8n](https://n8n.io/integrations/freshdesk/and/openai/)
- [Claude and Freshdesk: Automate Workflows with n8n](https://n8n.io/integrations/claude/and/freshdesk/)
- [Claude AI (Anthropic) and Freshworks (Freshdesk) integration | Albato](https://albato.com/connect/claude_ai_anthropic-with-freshworks)
- [Integrate Freshdesk with Anthropic Claude | viaSocket](https://viasocket.com/integrations/freshdesk/anthropic-claude)
- [Freshdesk ChatGPT (OpenAI) Integration | Zapier](https://zapier.com/apps/freshdesk/integrations/chatgpt)
- [Freshdesk MCP Integration with Claude Code | Composio](https://composio.dev/toolkits/freshdesk/framework/claude-code)
- [How to integrate Freshdesk MCP with Claude Agent SDK | Composio](https://composio.dev/toolkits/freshdesk/framework/claude-agents-sdk)
- [Connect your chatbot with your favorite apps through APIs | Freshdesk Support](https://support.freshdesk.com/support/solutions/articles/50000001021-setting-up-your-api-library-in-the-bot-builder)
- [People-First AI for Customer & Employee Experience | Freshworks](https://www.freshworks.com/platform/ai-capability/)
- [How to set up and use a Freshdesk ChatGPT integration in 2026 | eesel AI](https://www.eesel.ai/blog/freshdesk-chatgpt)
