---
layout: single
title: "Freshdesk AI (Freddy AI) 리서치 노트"
date: 2026-04-15 09:46:38 +0900
categories: [research]
tags: [Freshdesk, Freddy AI, Customer Service, AI, SaaS, 경쟁분석]
excerpt: "Freshworks의 Freshdesk AI 제품군(Freddy AI Agent·Copilot·Insights)의 기능, 가격, 경쟁사 비교, 도입 사례를 정리한 중간 수준 리서치 노트."
---
# Freshdesk AI (Freddy AI) 리서치 노트

**작성일:** 2026년 4월 15일
**대상 제품:** Freshworks의 Freshdesk AI 제품군 (Freddy AI Suite)

---

## 1. 제품 개요

Freshdesk AI는 Freshworks가 고객 서비스 플랫폼 Freshdesk에 내장한 AI 브랜드 **"Freddy AI"**의 제품군입니다. 2024년 2월 Freddy AI Copilot 출시, 2025년 6월 Freddy AI Agent 출시를 거치며 **"자율 에이전트 + 상담사 어시스턴트 + 관리자 인사이트"** 3축 구조로 자리 잡았습니다. 2025년 11월 Refresh25 행사에서는 옴니채널·음성·에이전틱 워크플로우 기능이 추가 발표되었습니다.

Freddy AI는 크게 세 가지 구성요소로 나뉩니다.

- **Freddy AI Agent** — 고객을 자율적으로 응대하는 봇(티켓 자동 해결)
- **Freddy AI Copilot** — 상담사의 실시간 업무를 보조하는 어시스턴트
- **Freddy AI Insights** — 관리자를 위한 데이터 기반 인사이트·예측

---

## 2. 제품 기능 및 AI 역량

### 2.1 Freddy AI Agent (자율 해결)

- 고객 문의를 웹, 이메일, 메신저, WhatsApp 등에서 **대화형으로 자율 응대**
- 지식 기반(KB)과 연동해 근거 기반 답변 생성 (RAG 기반)
- 자율 해결률(Autonomous Resolution Rate) 공식 수치 **30–45%** 수준, 고난이도 케이스는 사람에게 에스컬레이션
- "세션(Session)" 단위 과금 — 24시간 내 고객–봇 간 대화 1건을 1세션으로 정의

### 2.2 Freddy AI Copilot (상담사 보조)

상담 화면에 내장된 "에이전트 어시스트" 세트입니다.

- **Response Assistance** — 티켓 맥락을 반영해 브랜드 톤에 맞는 답변 초안 자동 생성
- **Summarization** — 긴 티켓/스레드를 수 초 만에 요약, 핸드오프·에스컬레이션 시 컨텍스트 유지
- **Live Translation** — 60개 이상 언어 실시간 번역으로 다국어 상담 지원
- **Tone & Rewrite** — 어조 변경(친근/공식), 확장/축약, 문법 교정
- **Similar Ticket Context** — 유사 티켓 자동 매칭으로 해결 경험 재사용
- **Sentiment Detection** — 실시간 감정 분석으로 위험 티켓 우선 대응

### 2.3 Freddy AI Insights (관리자용)

- 티켓 트렌드·이상 징후 자동 탐지
- CSAT 예측, SLA 위반 위험 티켓 사전 알림
- 팀/에이전트 퍼포먼스 비교 분석, 개선 액션 제안

### 2.4 공통 AI 백엔드

- 멀티-LLM 하이브리드(자체 모델 + 주요 파운데이션 모델 결합)로 비용·정확도 최적화
- Enterprise plan에서 **커스텀 역할·데이터 보안 경계** 제공

---

## 3. 가격 및 플랜 (2026년 4월 기준, 연간 결제)

Freshdesk AI 비용은 **① 기본 Freshdesk 요금제 + ② Copilot 에이전트당 추가 + ③ Agent 세션 사용량** 3가지 축으로 구성됩니다.

### 3.1 기본 Freshdesk 플랜

| 플랜 | 가격 (USD/agent/월, 연간) | AI 포함 범위 |
|---|---|---|
| Free | $0 (최대 10명) | 기초 기능, AI 제한적 |
| Growth | ~$15 | 기본 자동화 |
| **Pro** | **$49** | Freddy Copilot 업그레이드 옵션, Agent 세션 500개 1회 제공 |
| **Enterprise** | **$79** | 커스텀 역할·고급 보안, Agent 세션 500개 1회 제공 |

### 3.2 Freddy AI Copilot 애드온

- **$29/agent/월** (연간 결제)
- **$35/agent/월** (월간 결제)
- 필요한 상담사 라이선스에만 선택적으로 부여 가능

### 3.3 Freddy AI Agent (세션 기반)

- **$100 / 1,000 세션 팩** (공식 가격)
- 일부 자료에는 **$49 / 100 세션** 형태의 소량 팩도 언급됨
- 세션은 **청구 주기 말에 소멸**, 롤오버 불가
- 1세션 = 24시간 이내 고객-봇 간 고유 대화 1건

### 3.4 가격 핵심 포인트

- Intercom Fin의 "해결당 과금"과 달리, **사용량 기반이긴 하나 세션 단위**라 비용 예측이 상대적으로 용이
- Pro/Enterprise 가입 시 **500 세션 크레딧 1회 지급**으로 PoC에 유리
- 단, **AI 세션이 이월되지 않는 점**은 수요 변동성이 큰 팀에 부담

---

## 4. 경쟁사 비교 (2026)

| 항목 | Freshdesk (Freddy AI) | Zendesk AI | Intercom Fin |
|---|---|---|---|
| **자율 해결률** | 30–45% (45–50% 주장) | 중간 수준 (Answer Bot 기반) | **55–65%** (업계 최고) |
| **AI 가격 모델** | 기본 플랜에 일부 포함 + Copilot 애드온 + 세션 팩 | **에이전트당 $50/월 애드온** | **해결당 $0.99** 종량제 |
| **학습 데이터 규모** | 자체 + 파운데이션 모델 | **180억 건**의 상담 데이터 학습 | 자체 제품 데이터 + 대화 히스토리 |
| **강점** | 가격 대비 기능, SMB 접근성 | 엔터프라이즈 티켓팅, 데이터 규모 | 자율 해결 정확도, 모던 SaaS UX |
| **약점** | 해결률 Intercom 대비 낮음, 세션 이월 불가 | AI 애드온 비쌈, UI 복잡 | 고볼륨 시 비용 폭증 가능 |

### 4.1 비용 비교 예시 (20명 팀, 월 1,000건 AI 해결)

- **Freshdesk Growth + Freddy AI**: 약 **$300/월** (AI 기본 포함)
- **Zendesk + AI 애드온**: 에이전트당 $50 AI 추가로 수직 상승
- **Intercom**: 기본 $580 + AI 해결 $990 = 약 **$1,570/월**

→ 고볼륨 기준 Freshdesk가 **최대 75%가량 저렴**하다는 분석이 다수

### 4.2 포지셔닝 요약

- **Zendesk** = 엔터프라이즈 티켓팅의 표준, 데이터 자산 기반 AI
- **Freshdesk** = **가성비·중견/중소기업(SMB) 최적**, All-in-one
- **Intercom** = 대화형 자율 해결 최고 수준, SaaS·제품 주도 성장 기업 친화

---

## 5. 도입 사례 및 성과

### 5.1 대표 고객 사례

- **Hobbycraft (소매)** — Freddy 챗봇이 문의의 **최대 30% 자동 응답**
- **Big Bus Tours (관광)** — Freddy Copilot으로 상담사 생산성 증가
- **AG Barr (음료 제조)** — 문의의 **절반을 사람 없이 해결**
- **PhonePe (핀테크)** — 인앱 거래 컨텍스트를 끌어와 개인화된 셀프서비스 구현
- **Sonder (호스피탈리티)** — 글로벌 게스트 지원 오퍼레이션을 AI로 확장

### 5.2 Freshworks 내부 도입 성과

- Level-1 티켓 **45% 디플렉션(자동 처리)**
- 신규 상담사 램프업 기간 **6개월 → 3개월로 단축**

### 5.3 Copilot 생산성 지표 (SMB 고객 기준)

- First Response Time **41.56% 개선**
- Resolution Time **36.39% 개선**
- 응답 속도·일관성 개선 체감 상담사 **67%**
- 요약 기능으로 시간 절감 체감 상담사 **56%**

### 5.4 일반 ROI 지표

- 전체 운영 효율 **25–40% 상승**
- 일부 사례에서 응답 시간 **최대 83% 감소**

---

## 6. 최근 업계 동향 (2025–2026)

- **2025년 6월** — Freddy AI Agent(자율 해결) 정식 출시. 과거 챗봇 수준을 넘어 "에이전틱 워크플로우"로 전환
- **2025년 11월 Refresh25** — Freshworks가 옴니채널·음성·자율 액션(시스템 간 전이, 리퀘스트 풀 체인) 기능 확장 발표
- 업계 전반적으로 **"해결당 가치 과금" vs "세션/시트 과금"** 경쟁이 심화. Freshdesk는 **세션 팩**이라는 중간 지대를 택해 비용 예측성을 어필
- **한국/APAC 동향:** 다국어 Copilot, 60+ 언어 지원, 카카오톡·WhatsApp 등 지역 채널 통합 강화가 2026년 핵심 업데이트 포인트

---

## 7. 종합 평가

**장점**
- AI가 기본 플랜에 상당 부분 포함되어 초기 도입 비용 낮음
- Copilot의 요약·번역·답변 초안이 상담사 생산성 체감 효과 뚜렷
- SMB~미드마켓에 최적화된 가격·UX

**한계**
- 자율 해결률은 Intercom Fin 대비 열세
- AI Agent 세션이 이월되지 않아 수요 변동이 큰 팀은 낭비 발생 가능
- 초대형 엔터프라이즈에서는 Zendesk의 데이터 자산·확장성에 밀리는 편

**추천 대상**
- 중견·중소 SaaS 또는 커머스 기업으로 **AI 비용 폭증 없이 상담 자동화를 확장**하고자 하는 팀
- 다국어·옴니채널 대응이 필요한 글로벌 SMB
- Intercom 수준의 최고 자율 해결률이 필수가 아닌 경우

---

## 출처 (Sources)

- [AI for Customer Service | Freddy AI Copilot | Freshworks](https://www.freshworks.com/freshdesk/omni/freddy-ai-copilot/)
- [Enhance Customer Support with Freddy AI: Self-Service, Copilot, and Insights](https://support.freshdesk.com/support/solutions/articles/50000010359-overview-of-freddy-ai-for-ticketing)
- [Freshdesk Pricing & Plans | Freshworks](https://www.freshworks.com/freshdesk/pricing/)
- [A guide to Freshdesk Freddy Copilot pricing in 2026 | eesel AI](https://www.eesel.ai/blog/freshdesk-freddy-copilot-pricing-2025)
- [A complete guide to Freshdesk AI pricing in 2026 | eesel AI](https://www.eesel.ai/blog/freshdesk-ai-pricing)
- [Freshdesk Freddy AI: Guide to Features, Pricing & Limitations (2026)](https://myaskai.com/blog/freshdesk-freddy-ai-agent-complete-guide-2026)
- [Zendesk vs Intercom vs Freshdesk: Feature Comparison 2026](https://www.saasgenie.ai/blogs/freshdesk-vs-zendesk-vs-intercom)
- [Freshdesk Freddy vs Intercom Fin Pricing Compared [2026]](https://myaskai.com/compare/freshdesk-vs-intercom-ai-pricing)
- [AI Agent Pricing Comparison 2026: Cost Guide | Fin.ai](https://fin.ai/learn/ai-customer-service-agent-pricing-comparison)
- [Freshdesk AI case studies: Real results from 2026 implementations | eesel AI](https://www.eesel.ai/blog/freshdesk-ai-case-studies)
- [How AI is unlocking ROI in customer service | Freshworks](https://www.freshworks.com/How-AI-is-unlocking-ROI-in-customer-service/)
- [Freshworks simplifies service with AI in customer support - SiliconANGLE](https://siliconangle.com/2025/11/19/freshworks-simplifies-service-ai-in-customer-support-refresh25/)
