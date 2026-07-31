---
layout: single
title: "PDF는 GitHub에, 메타데이터는 Zotero에 — 링크 첨부 워크플로우"
date: 2026-07-31 14:40:14 +0900
categories: [dev]
tags: [Zotero, GitHub, 워크플로우, PDF, 협업, macOS]
excerpt: "PDF 파일 자체는 Git으로 버전 관리하고 Zotero에는 서지정보와 링크만 두는 방법. Stored copy와 Linked file의 차이, 기준 폴더 설정, 맥에서 링크로 첨부하는 구체적 절차, 팀 협업 셋업을 정리했다."
---
PDF 파일 원본은 GitHub 저장소로 버전 관리하고, Zotero에는 서지정보와 "링크"만 두고 싶을 때가 있다. 파일 실체와 메타데이터의 관리 주체를 분리하는 구성이다. 이 글은 그 방법을 정리한다.

## 핵심 개념: Stored copy vs Linked file

Zotero가 파일을 첨부하는 방식은 두 가지다.

- **Stored copy(가져오기)**: 파일을 Zotero 저장소로 복사한다. Zotero 클라우드 용량을 차감하고, 파일이 Zotero 안에 갇힌다.
- **Linked file(링크)**: 파일 원본은 그대로 두고 경로만 참조한다. Zotero는 서지정보와 링크만 관리하고, 파일 자체는 절대 동기화하지 않는다.

"PDF는 GitHub, 정보만 Zotero"라는 목표에는 **Linked file 방식**이 정답이다.

## 1단계: GitHub 저장소를 로컬에 클론

관리할 PDF를 담을 저장소를 로컬에 clone한다. 예를 들어 `~/research-pdfs`. 이 폴더에 PDF를 넣고 `git add / commit / push`로 버전 관리한다. 하위 폴더로 정리해도 된다.

## 2단계: Zotero에 "링크 기준 폴더" 설정 (핵심)

Zotero 설정 → Advanced → Files and Folders → **Linked Attachment Base Directory** 를 클론한 폴더(`~/research-pdfs`)로 지정한다.

이렇게 하면 링크가 **상대경로**로 저장된다. 그 덕분에 다른 사람이나 다른 PC에서도 각자 자기 클론 폴더를 기준 폴더로 지정하기만 하면 링크가 그대로 연결된다. 협업의 열쇠가 이 설정이다.

## 3단계: PDF를 "링크"로 첨부 (macOS)

그냥 드래그하면 복사(stored)되므로 반드시 링크로 넣어야 한다. 두 가지 방법이 있다.

**메뉴 방식(확실함)**: 자료 항목을 우클릭 → Add Attachment → **Attach Link to File...** → 기준 폴더 안의 PDF 선택. 바로 아래 "Attach Stored Copy of File..."는 복사되므로 고르지 않는다.

**드래그 방식(빠름)**: Finder에서 PDF를 자료 위로 끌 때 **⌘Command + ⇧Shift** 를 누른 채 놓으면 링크로 첨부된다.

## 4단계: 상대경로 확인

첨부한 PDF를 클릭하고 오른쪽 패널에서 경로를 본다. 기준 폴더 기준의 상대경로(예: `game-ai/paper.pdf`)로 보이면 성공이다. 절대경로(`/Users/...`)로 보이면 파일이 기준 폴더 밖에 있거나 설정이 안 맞는 것이니 다시 확인한다.

## 동기화 동작

Zotero 설정의 Sync에서 데이터 동기화는 켜되, 파일 동기화는 링크 파일에 적용되지 않는다. 즉 서지정보·태그·주석 데이터만 Zotero가 클라우드로 올리고, 실제 PDF는 GitHub이 담당한다. Zotero 무료 용량(300MB)은 사실상 소진되지 않는다.

## 대안: URL만 링크

로컬에 파일을 두지 않고 GitHub 웹에서 바로 열게 하려면, 자료 우클릭 → Add Attachment → **Attach Link to URI...** → GitHub의 raw PDF 링크를 붙여넣는다. 로컬 저장이 0이고 항상 최신 버전이지만, 오프라인 열람과 Zotero 내 하이라이트는 안 된다.

## 팀 협업 셋업

팀으로 쓸 때는 세 가지만 맞추면 된다. ① 모두가 같은 저장소를 clone하고, ② 각자 같은 기준 폴더 규칙으로 Linked Attachment Base Directory를 설정하고, ③ 서지정보는 Zotero 그룹 라이브러리에 올린다. 새 논문이 생기면 폴더에 PDF를 넣고 push, 팀원은 pull하면 파일이 로컬에 생기고 링크가 그대로 열린다.

읽고 주석까지 하려면 링크 파일 방식, 순수 참조만 원하면 URL 링크 방식이다. 둘을 섞어 써도 된다.
