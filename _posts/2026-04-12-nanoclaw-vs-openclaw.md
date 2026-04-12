---
layout: single
title: "NanoClaw vs OpenClaw — 15항목 비교 분석"
date: 2026-04-12 11:00:00 +0900
categories: [research]
tags: [AI Agent, NanoClaw, OpenClaw, Framework]
excerpt: "경량 보안 중심의 NanoClaw와 대규모 생태계의 OpenClaw을 15개 항목으로 비교 분석한 연구 노트"
---

> **작성일**: 2026년 4월 10일 · **출처**: 4-Agent Swarm Research

## 주요 통계

| 항목 | NanoClaw | OpenClaw |
|------|----------|----------|
| GitHub Stars | 22K | 350K |
| Core 코드량 | ~500줄 | 500K+줄 |
| Contributors | 50+ | 1,200+ |
| Active Users | 4.2K+ | 3.2M |

---

## Anthropic OAuth 정책 변경 (2026.04.04)

2026년 4월 4일 Anthropic이 Claude Pro/Max 구독의 OAuth 토큰을 서드파티 도구에서 차단했다. OpenClaw는 즉시 영향을 받았으나, NanoClaw는 Claude Code CLI 내부 실행 구조로 현재 구독 기반 OAuth가 작동 중이다. 다만 ToS상 향후 변경 가능성이 있다.

---

## 15개 비교 항목

### 1. 프로젝트 개요

**NanoClaw** — 경량 오픈소스 AI 에이전트 프레임워크. OpenClaw의 보안 취약점 해결을 위해 개발된 미니멀리스트 대안. 핵심 코드 약 500줄 TypeScript. 2026년 1월 31일 출시.

**OpenClaw** — 대규모 오픈소스 자율 AI 에이전트 플랫폼. 2025년 11월 "Clawdbot"으로 시작, 2026년 3월 리브랜딩. GitHub 최다 스타 프로젝트(350K+). Peter Steinberger 창시.

### 2. 개발 조직

**NanoClaw** — Qwibit AI (Gavriel Cohen & Lazer Cohen 형제 공동 창립). 내부에서 NanoClaw 인스턴스 "Andy"를 실제 업무에 활용 중.

**OpenClaw** — Peter Steinberger(오스트리아 개발자) 창시. 2026년 2월 OpenAI 합류 발표 후 독립 비영리 재단으로 전환. 1,200+ 커뮤니티 기여자 공동 운영.

### 3. 아키텍처

**NanoClaw** — 컨테이너 격리 아키텍처. 그룹별 독립 Linux 컨테이너에서 에이전트 실행. 단일 Node.js 오케스트레이터 + SQLite + 파일시스템 IPC. 읽기 전용 컨테이너 + MicroVM 이중 격리.

**OpenClaw** — 단일 장시간 실행 Node.js "Gateway" 프로세스가 모든 기능을 통합 처리. 플러그인 기반 폴리글랏 아키텍처(Python, JS, Java, Go, C#, Ruby).

### 4. 코드 규모

**NanoClaw** — 핵심 500줄 / 전체 약 8,000줄 TypeScript. 소스 파일 15개. Claude 컨텍스트 윈도우의 17%에 들어갈 정도로 작음. 프로덕션 의존성 3개.

**OpenClaw** — 500,000줄 이상. 수백 개 내장 스킬, 50+ 메시징 채널 어댑터, 1,200+ 커뮤니티 모듈. 보안 연구자들이 "430,000줄의 잠재적 공격 표면"으로 지적.

### 5. 보안 모델

**NanoClaw** — Security-by-Default. 컨테이너 격리 + Docker Sandboxes MicroVM. 2026년 3월 Docker와 공식 파트너십. 일부 한계: 네트워크 격리 미완성, API 자격증명 컨테이너 노출.

**OpenClaw** — 보안 우려 심각. 샌드박스 탈출 방어율 평균 17%. Palo Alto Networks가 "보안 악몽"으로 평가. ClawKeeper 등 서드파티 보안 레이어로 보완 중.

### 6. 성능

**NanoClaw** — 바이너리 0.8MB, 콜드 스타트 3ms, 유휴 RAM 1.2MB. 처리량 18,000 req/s (8GB RAM, 4코어 ARM).

**OpenClaw** — 최소 2GB RAM(권장 4GB). 멀티라운드 대화 시 워킹 메모리 기하급수적 증가로 레이턴시 이슈 가능. 42,000+ 활성 셀프호스팅 인스턴스 운영 중.

### 7. 지원 LLM

**NanoClaw** — Claude 전용. Anthropic Agent SDK 기반. 멀티 LLM 미지원이 주요 한계.

**OpenClaw** — 멀티 모델 지원. Claude, GPT-4, Ollama, Gemma 4, AWS Bedrock, Arcee 등.

### 8. 메시징 채널

**NanoClaw** — WhatsApp 내장 + 스킬로 Telegram, Discord, Slack, Gmail 추가 가능.

**OpenClaw** — 50+ 채널 지원. WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Teams, LINE, WeChat 등.

### 9. 로보틱스 통합

**NanoClaw** — 로보틱스 통합 기능 없음. 순수 소프트웨어 에이전트 프레임워크에 집중.

**OpenClaw** — ROSClaw 프레임워크로 ROS2 완전 통합. AgileX NERO 7-DoF, Unitree G1 등 다수 로봇 호환. 물리적 에이전트 구현 가능.

### 10. 생태계 규모

**NanoClaw** — 22K+ Stars, 50+ Contributors. Andrej Karpathy 추천 + Docker 파트너십으로 급성장 중.

**OpenClaw** — 350K+ Stars, 1,200+ Contributors, 3.2M 활성 사용자. ClawHub 44,000+ 커뮤니티 스킬. 180+ 스타트업 수익 창출.

### 11. 설치 & 설정

**NanoClaw** — `git clone → cd → claude → /setup`. 약 10~15분. Node.js 20+, Claude Code CLI, Docker 필요.

**OpenClaw** — 공식 가이드 약 5분 설정. Node.js 22+, Python 필요. 24시간 구동 시 항시 가동 하드웨어 필수.

### 12. 라이선스 & 비용

**NanoClaw** — MIT. Claude API 종량제 또는 구독 OAuth. 월 $20~$200 정액 운영 가능(현재, 향후 불확실).

**OpenClaw** — MIT. API 종량제 필수(구독 OAuth 차단). 자율 에이전트 특성상 일일 $1,000~$5,000 API 비용 발생 가능.

### 13. 학술 & 연구

**NanoClaw** — 전용 학술 논문 없음. 기술 블로그(TechCrunch, VentureBeat) 중심.

**OpenClaw** — 다수 학술 논문. 보안 분석(ArXiv 2603.10387), OpenClaw-RL, ROSClaw, ClawKeeper 등.

### 14. 실제 활용 사례

**NanoClaw** — Qwibit AI 내부 에이전트 "Andy"(세일즈 파이프라인, 태스크 할당), "Alfred" 듀얼 에이전트 아키텍처.

**OpenClaw** — Ecovacs Bajie 가정용 로봇, "Patch" 수퍼바이저(20개 병렬 코딩 인스턴스), AutoResearchClaw 논문 자동화 등.

### 15. 주요 한계점

**NanoClaw**: Claude 전용(멀티 LLM 미지원), 플러그인 생태계 미성숙, 네트워크 격리 미완성, 엔터프라이즈 통합 부족, 로보틱스 통합 없음, 구독 OAuth 향후 차단 리스크.

**OpenClaw**: 심각한 보안 취약점(방어 17%), 거대 코드베이스 감사 어려움, 메모리 폭증, 서드파티 연동 불안정, 구독 OAuth 차단으로 API 비용 급증.

---

## 선택 가이드

| NanoClaw을 선택할 때 | OpenClaw을 선택할 때 |
|---|---|
| 보안이 최우선인 환경 | 50+ 메시징 플랫폼 동시 통합 |
| 전체 코드를 직접 감사하고 싶을 때 | 로보틱스/물리적 에이전트 구현 |
| 최소 리소스 & 빠른 콜드 스타트 | 다양한 LLM 전환 사용 |
| Docker 기반 격리 필수 | 44,000+ 스킬 생태계 활용 |
| 구독 정액제 비용 통제 (현재) | API 종량제 비용 감수 가능 |

---

## 참고자료

- NanoClaw GitHub, nanoclaw.dev
- OpenClaw GitHub, openclaw.ai
- The Register, The New Stack, MindStudio
- ArXiv, TechCrunch, VentureBeat
