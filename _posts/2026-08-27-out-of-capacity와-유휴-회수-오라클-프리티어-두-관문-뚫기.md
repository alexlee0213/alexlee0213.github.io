---
layout: single
title: "Out of capacity와 유휴 회수 — 오라클 프리티어 두 관문 뚫기"
date: 2026-08-27 10:30:00 +0900
categories: [dev]
tags: [오라클클라우드, OCI, 프리티어, OutOfCapacity, PAYG, AlwaysFree, ARM]
excerpt: "오라클 프리티어에서 ARM 인스턴스를 만들려 하면 십중팔구 Out of host capacity를 만난다. 겨우 만들어도 유휴 상태면 회수된다. 두 관문의 정확한 조건과 우회법, 그리고 결국 둘 다 해결하는 PAYG 전환의 손익을 정리했다."
---

> **오라클 클라우드 프리티어 실전 시리즈**
> 1. [2026년 프리티어, 조용히 반토막 났다](/dev/2026/08/27/2026년-오라클-클라우드-프리티어-조용히-반토막-났다/)
> 2. **Out of capacity와 유휴 회수 — 두 관문 뚫기** ← 현재 글
> 3. [포트를 열었는데 왜 접속이 안 되나 — 오라클의 이중 방화벽](/dev/2026/08/27/포트를-열었는데-왜-접속이-안-되나-오라클의-이중-방화벽/)
> 4. [ARM과 AMD, 둘 다 쓸 수 있다](/dev/2026/08/27/arm과-amd-둘-다-쓸-수-있다-오라클-프리티어-셰이프-정리/)
> 5. [Claude API 지연 실측 — 에이전트 루프에서 서버 리전이 중요한 이유](/dev/2026/08/27/claude-api-지연-실측-에이전트-루프에서-서버-리전이-중요한-이유/)

## 요약

- `Out of host capacity`는 **ARM(A1) 셰이프에만** 사실상 집중된다. AMD 마이크로는 거의 항상 잡힌다.
- 우회 순서: **폴트 도메인 미지정 → 스펙 낮추기 → AD 변경 → 시간대 바꿔 재시도 → 자동화 스크립트**.
- 유휴 회수 조건은 **CPU·네트워크·메모리 세 가지가 모두 20% 미만**일 때다. **AND 조건이라 하나만 넘겨도 안전하다.**
- 메모리 20% 조건은 **A1(ARM)에만** 적용된다.
- 두 문제를 한 번에 없애는 방법은 **PAYG 전환**이다. 한도 내면 여전히 $0이고, 덤으로 ARM 4 OCPU/24GB를 유지한다.

---

## 관문 1. Out of host capacity

### 왜 생기나

무료 ARM 물량은 리전별로 한정돼 있고, 인기가 많아 늘 부족하다. 콘솔에서 인스턴스 생성을 누르면 이런 응답이 온다.

```json
{
  "code": "InternalError",
  "message": "Out of host capacity."
}
```

이건 **내 계정 문제가 아니라 리전 재고 문제**다. 오라클 문서도 "홈 리전에 Always Free 셰이프가 일시적으로 부족한 상태"라고 설명한다.

### 대처법 — 쉬운 순서대로

#### ① 폴트 도메인을 지정하지 않는다

가장 간단하면서 효과가 있다. 생성 화면에서 폴트 도메인을 명시하지 않으면 오라클이 여유 있는 곳으로 알아서 배치한다.

> When you start or create a VM, don't specify a Fault Domain. If a Fault Domain isn't specified, the best possible Fault Domain is assigned to the VM.

#### ② 스펙을 낮춰서 시도한다

2 OCPU / 12GB가 안 잡히면 **1 OCPU / 6GB로 먼저 확보**한다. 자리를 잡은 뒤 나중에 리사이즈하거나, 1코어짜리를 하나 더 만들어 총 2코어를 채우는 방법도 있다.

작은 조각이 큰 조각보다 훨씬 잘 잡힌다.

#### ③ 가용 도메인(AD)을 바꾼다

AD가 여러 개인 리전이면 AD-1 / AD-2 / AD-3을 차례로 시도한다.

> ⚠️ **서울 리전은 AD가 1개뿐**이라 이 방법이 통하지 않는다. 도쿄·오사카 등 멀티 AD 리전에서만 유효하다.

#### ④ 시간대를 바꿔 반복한다

재고는 수시로 풀린다. 다른 사용자가 인스턴스를 삭제하거나 오라클이 용량을 증설하는 순간이 있다. 새벽 시간대에 잘 잡힌다는 경험담이 많다.

#### ⑤ 자동 재시도 스크립트

가장 확실한 방법이다. 사람이 콘솔을 계속 누르는 대신 API로 주기적으로 생성을 시도해서, 재고가 풀리는 순간 낚아챈다.

사실상 표준으로 쓰이는 도구가 [hitrov/oci-arm-host-capacity](https://github.com/hitrov/oci-arm-host-capacity)다.

동작 방식은 단순하다.

1. OCI API 키를 발급받아 스크립트에 설정
2. cron으로 몇 분마다 인스턴스 생성 API 호출
3. 실패하면 무시, 성공하면 종료

이미 인스턴스가 있으면 이런 응답이 오는데, 이건 정상이다.

```json
{
  "code": "LimitExceeded",
  "message": "The following service limits were exceeded: standard-a1-memory-count, standard-a1-core-count."
}
```

> ⚠️ **GitHub Actions로 무한 실행하지 말 것.** GitHub 약관 위반이다. 저장소 README에도 명시돼 있다. 본인 PC나, 이미 확보해둔 AMD 마이크로 인스턴스에서 돌리자.

여기서 유용한 순서가 나온다. **AMD 마이크로는 재고 부족이 거의 없으므로 먼저 2대를 확보하고, 그중 1대에서 ARM 확보 스크립트를 상시 구동**하는 것이다. 발판을 먼저 만드는 셈이다.

#### ⑥ PAYG로 전환한다

결정적 해법이다. 유료 계정은 인스턴스 생성 우선순위가 높아서 capacity 에러를 거의 겪지 않는다.

> With PAYG, you'll continue to enjoy all the free benefits without any additional cost, but you'll also receive priority for launching instances and are less likely to face "Out of host capacity" errors.

손익은 이 글 마지막에서 따로 정리한다.

---

## 관문 2. 유휴 인스턴스 회수

겨우 인스턴스를 만들었다고 끝이 아니다. 무료 계정은 **놀고 있는 인스턴스를 오라클이 중지·삭제**한다.

### 정확한 조건

7일 연속으로 아래 **세 가지가 모두** 해당되면 회수 대상이 된다.

| 지표 | 임계값 |
|---|---|
| CPU 사용률 (95 백분위수) | 20% 미만 |
| 네트워크 사용률 | 20% 미만 |
| 메모리 사용률 | 20% 미만 — **A1 셰이프에만 적용** |

### 핵심: AND 조건이다

여기가 실전에서 가장 중요한 포인트다. 세 조건이 **모두** 충족되어야 회수 대상이 되므로, **하나만 넘겨도 안전하다.**

ARM(A1)은 메모리 조건이 있어서 오히려 쉽다. 12GB 중 **2.4GB 이상만 상시 점유**하면 통과한다. Docker 컨테이너 몇 개를 띄워두면 자연히 해결된다.

- PostgreSQL이나 Redis를 띄워두기
- JVM 애플리케이션에 `-Xms` 값을 넉넉히 주기
- 실제로 뭔가를 돌리기 (가장 정직한 방법)

CPU를 억지로 태우는 스크립트를 돌리는 사람도 있는데, 전력 낭비인 데다 메모리 조건 하나로 해결되므로 권하지 않는다.

> AMD 마이크로(E2.1.Micro)는 **메모리 조건이 적용되지 않는다.** CPU와 네트워크만 보므로, 1GB짜리 인스턴스를 놀리고 있다면 회수 위험이 ARM보다 오히려 높다.

### 회수되면 어떻게 되나

먼저 **중지**되고, 이후 삭제 절차로 넘어간다. 중지된 상태라면 셰이프 재고가 있을 때 다시 시작할 수 있다.

> If your idle Always Free compute instance is stopped, you can restart it as long as the associated compute shape is available in your region.

문제는 "재고가 있을 때"라는 조건이다. ARM은 재고가 없어서 재시작이 막힐 수 있고, 그러면 관문 1로 되돌아간다.

### 근본 해결책

오라클이 공식적으로 안내하는 방법이 PAYG 전환이다.

> You can keep idle compute instances from being stopped by converting your account to Pay As You Go (PAYG). With PAYG, you will not be charged as long as your usage for all OCI resources remains within the Always Free limits.

---

## 결론: PAYG 전환의 손익

두 관문 모두 PAYG로 귀결된다. 정리하면 이렇다.

| | Always Free 계정 | PAYG 계정 |
|---|---|---|
| ARM 한도 | 2 OCPU / 12GB | **4 OCPU / 24GB 유지** |
| 인스턴스 생성 | capacity 에러 빈발 | **우선순위 높음** |
| 유휴 회수 | 있음 | **없음** |
| 고객 지원 | ❌ | ✅ |
| **비용** | $0 | **한도 내면 $0** |

[1편](/dev/2026/08/27/2026년-오라클-클라우드-프리티어-조용히-반토막-났다/)에서 언급한 2026년 6월 사양 축소가 **PAYG 계정에는 적용되지 않는다**는 점이 특히 크다. 전환만으로 ARM 할당량이 두 배가 된다.

### 하지만 공짜는 아니다

PAYG는 **무료 한도를 넘는 순간 자동 과금**된다. 실수로 유료 셰이프를 만들거나, 오브젝트 스토리지 20GB를 넘기거나, 볼륨을 200GB 이상 잡으면 청구서가 날아온다.

전환한다면 아래는 필수다.

**① 예산 알림 설정**

Billing → Budgets에서 예산을 만들고 임계값 알림을 건다. $1만 넘어도 메일이 오도록 해두면 사고를 조기에 잡을 수 있다.

**② Always Free-eligible 배지 확인 습관**

리소스를 만들 때마다 콘솔에 배지가 표시되는지 본다. 배지가 없으면 유료다.

**③ 한도 모니터링**

Governance & Administration → Tenancy Management → **Limits, Quotas and Usage**에서 현재 사용량을 주기적으로 확인한다.

특히 주의할 항목:

| 항목 | 무료 한도 |
|---|---|
| 블록 볼륨 (부팅+블록) | 200GB |
| 오브젝트 스토리지 | 20GB |
| 볼륨 백업 | 5개 |
| 아웃바운드 트래픽 | 10TB/월 |

### 그래서 전환할까

**실서비스를 올릴 거라면 전환하는 쪽을 권한다.** 유휴 회수로 서비스가 갑자기 멈추는 것보다, 예산 알림을 걸어두고 PAYG로 안정적으로 운영하는 게 낫다.

반면 **학습·실험 용도로 가끔 켜본다면 무료 계정 그대로**가 마음 편하다. 과금 사고 가능성이 아예 0이다.

---

## 다음 편

인스턴스를 만들었다면 다음은 웹 서비스를 띄우는 단계인데, 여기서 거의 모든 사람이 한 번은 막힌다. 보안 목록에서 80번 포트를 열었는데도 접속이 안 되는 문제다.

→ [3편: 포트를 열었는데 왜 접속이 안 되나 — 오라클의 이중 방화벽](/dev/2026/08/27/포트를-열었는데-왜-접속이-안-되나-오라클의-이중-방화벽/)

---

### 참고

- [Always Free Resources — Oracle Docs](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)
- [Resolving Out of Host Capacity error — Oracle Docs](https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/troubleshooting-out-of-host-capacity.htm)
- [hitrov/oci-arm-host-capacity](https://github.com/hitrov/oci-arm-host-capacity)
- 본문의 임계값과 한도는 **2026년 8월 기준**이다. 유휴 회수 임계값은 과거 10%였다가 20%로 바뀐 이력이 있으니, 오래된 글과 수치가 다르면 공식 문서를 우선하자.
