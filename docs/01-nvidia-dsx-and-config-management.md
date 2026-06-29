# 01. NVIDIA DSX 와 구성 관리 체계 (검증된 사실)

> 출처·검증 등급은 [`03-sources-and-verification.md`](03-sources-and-verification.md) 참조. 본 문서의 사실은 모두 3표 적대적 검증을 통과한 confirmed claim 기반이며, 추정/해석 부분은 명시한다.

## 1. NVIDIA DSX 란 무엇인가

**DSX 는 단일 제품도, 단일 OS 도 아니다.** AI Factory 를 **설계(design) · 시뮬레이션(simulation) · 구축(build) · 운영(operations)** 하는 **풀스택 프레임워크/포트폴리오**다. 칩·시스템·네트워크(Spectrum-X/InfiniBand/NVLink)·스토리지·전력·시설·파트너 기술까지 "AI Factory 를 어떻게 짓고 최적화하는가" 를 정의하는 *playbook* 으로 마케팅된다 (2026-06 발표).

포트폴리오 구성요소(확인된 것):

| 컴포넌트 | 역할 |
|---|---|
| **DSX Reference Design** (예: Vera Rubin DSX) | AI Factory 참조 설계 |
| **Omniverse DSX (Sim)** | 디지털 트윈 — 배포 전 레이아웃·전력·열·운영정책·하드웨어/워크로드 변경을 **실 가동 중단 없이 사전 시뮬레이션/검증** |
| **DSX OS** | **운영(operations) 담당 오픈소스 모듈 소프트웨어** (아래 2절) |
| DSX MaxLPS / DSX Flex | 전력·인프라 관리 계열 |
| DSX Exchange | 생태계/교환 |

### DSX OS

DSX 의 **운영 레이어**. "멀티테넌트 AI Factory 를 운영·확장" 하기 위한 **오픈소스 모듈 소프트웨어 + 관련 NVIDIA 기술** 묶음이다. 제공 기능(NVIDIA 자체 표현):

- lifecycle management, intelligent scheduling, runtime consistency
- health automation, resiliency
- multi-tenant operations, platform services

실제 공개 GitHub 레포로 일부가 릴리스됨 (마케팅이 아닌 가장 강한 신호):
`NVIDIA/dsx-exchange`, `NVIDIA/infra-controller-core`, `NVIDIA/doca-platform`, `NVIDIA/fleet-intelligence-agent`, `NVIDIA/nvcf`.

> **레이어 주의**: DSX OS 도 "모델을 운영" 하는 게 아니라 "**factory(물리 인프라)를 운영**" 한다. AIPub 이 위치한 ML 애플리케이션/플랫폼 레이어와는 명확히 구분된다. 이 경계가 02 문서 적용안의 출발점이다.

### DGX OS / BCM / Mission Control 과의 관계

요청서가 물었던 비교 대상이지만, 검증 코퍼스에서 직접적인 "DSX vs DGX/BCM/Mission Control 계층도" 를 단정할 1차 근거는 부족했다. 다만 확인된 사실:
- **Mission Control** 2.0(GB200/GB300 NVL72) 문서에서 **고속 fabric 관리가 fabric 종류별로 분리**됨이 확인된다 — **NMX** 가 NVLink, **UFM** 가 InfiniBand 를 각각 독립 도구로 관리.
- 즉 NVIDIA 스택에는 "모든 fabric 을 아우르는 단일 config source-of-truth" 가 아니라 **fabric 별 별도 관리 도구**가 존재한다 (open question으로 남김 — 03 문서).

---

## 2. 실제 구성 관리 주체 5종과 desired-state / drift / reconcile 동작

요청서의 "Config Manager (MVCM)" 는 실제 제품으로 확인되지 않았다. 대신 아래 5종이 실재하며, **공통적으로 "기준 정보(source-of-truth) → 실제 상태 diff → drift 분류 → (일부) 교정"** 패턴을 구현한다.

### 2.1 NMX — NVLink fabric 컨트롤 플레인 (가장 reconcile 에 가까움)

NVLink(NVL5/GB200) fabric 용. **4개 서브시스템의 telemetry→analytics→control 폐루프**:

- **NMX-T** (Telemetry): 텔레메트리 수집·집계·전송
- **NMX-M** (Manager): 이벤트 구동 마이크로서비스. **텔레메트리에 ML/추론을 돌려 패턴을 감지하고, NMX-C 를 통해 네트워크/컴퓨트 엔티티의 "구성을 바꿔" 동작을 제어**
- **NMX-C** (Controller): SDN 컨트롤 플레인. nvlSM(NVLink Subnet Manager)·GFM(Global Fabric Manager) 를 구동하며 **실제 구성 변경을 enact**. 관리형 NVL5 스위치 또는 전용 head node 에 배포
- **NMX Oasis**: 데이터 레이크

→ **observe → analyze → act 의 닫힌 루프**. NVIDIA 스택 내에서 "drift 감지 → 자동 교정" 에 가장 근접한 구조이며, AIPub reconcile 컨트롤러의 직접적 개념 템플릿.

### 2.2 UFM — InfiniBand fabric 관리

InfiniBand fabric 의 프로비저닝·모니터링·관리·예방적 트러블슈팅 플랫폼. **자동 네트워크 디스커버리 + 검증**을 수행하며, **13개 fabric validation tests**, system validation, network performance tests 를 디스커버된 토폴로지에 대해 실행. (UFM Cyber-AI 는 시간에 따른 성능 저하/이상 클러스터를 AI로 탐지.)

### 2.3 NVUE (Cumulus Linux) — 진짜 선언적 desired-state 모델

NVIDIA 스택에서 **쿠버네티스식 선언적 config + drift 감지에 가장 가까운 모델**이며, AIPub 에 가장 직접 전이 가능:

- **객체지향·스키마 구동(OpenAPI/YANG 유사) 모델** — 시스템 전체 상태를 하나의 큰 트리로 표현, `nv set` / `nv unset` 로 desired config 선언, `nv config apply` 로 적용
- **4개 상태를 명확히 분리**:
  - `Pending` — 미커밋(= 의도된 변경)
  - `Applied` — 커밋됨
  - `Operational` — 실제 running 상태
  - `Startup` — 영속화 (`/etc/nvue.d/startup.yaml`)
- **`nv config diff`** — 임의의 두 상태(pending/applied/startup/revision)를 비교 → **intended vs actual 차이(=drift)를 표면화**

> "drift detection" 이라는 표현은 일부 해석이다 — 문서는 "configurations 간 차이" 라 부른다. 단 pending=의도, applied/operational=실제이므로 추론은 타당.

### 2.4 NetQ — Ethernet(Spectrum-X/Cumulus) 운영·검증 툴셋

NVLink Switch·Cumulus fabric 의 실시간 가시성·트러블슈팅·상관분석·검증 툴셋:

- **Preventive Validation**: 수동 config 오류가 프로덕션에 반영되기 전에 차단 (특히 **NetQ + NVIDIA Air 디지털 트윈 + CI/CD 파이프라인** 조합)
- **Snapshot and Compare**: 변경 전/후 네트워크 구성을 diff → 변경 위험 제거 (drift 감지 "유사" 기능, 단 연속 reconcile 루프가 아닌 시점 비교)
- **Topology validation**: LLDP 텔레메트리로 도출한 **실제 토폴로지를 사용자 제공 토폴로지 blueprint 파일과 비교** → intended vs actual 차이 감지 (= source-of-truth 기반 drift 감지의 구체 사례)

### 2.5 UFM Cable Validation Tool (CVT) — 가장 명확한 "기준정보→분류된 drift" 사례

물리 케이블링 검증. 코퍼스에서 **"reference-data → diff → 분류된 drift"** 를 가장 명확하게 보여줌:

- **Topology File(P2P/topo/dot)** 이 클러스터의 의도된 배선에 대한 **"authoritative source"** (= source-of-truth)
- **에이전트 기반 워크플로우**: 계획 토폴로지 로드 → 도달 가능한 관리형 스위치에 에이전트 설치·실행 → validation 트리거 → 각 에이전트가 실제 vs 계획 검사
- **drift 를 이산 카테고리로 분류** → 즉시 actionable:
  - `Wrong-neighbor` (토폴로지 파일과 다른 장비에 연결)
  - `Wrong-port` (장비는 맞으나 포트가 틀림)
  - `Unknown-neighbor` (파일에 없는 장비)
  - `Extra-cable`
- **단, CVT 는 감지 전용** — drift 를 표면화할 뿐 자동 교정/reconcile 은 하지 않는다.

---

## 3. 배포 전 검증 — 디지털 트윈 / CI/CD

- **Omniverse DSX (Sim)**: 물리적으로 정확한 AI Factory 디지털 트윈을 만들어 레이아웃·전력·열·운영정책·하드웨어/워크로드 변경을 **구축/배포 전에** 실시간 시뮬레이션. OpenUSD 기반 실제 릴리스 제품.
- **NVIDIA Air**: 하드웨어 배포 전 풀스택 시뮬레이션으로 day-zero 운영(네트워크 프로비저닝·자동화·보안정책 설계/테스트/검증) 수행.
- → "**apply 전에 모델에서 변경을 검증**" 하는 pillar. AIPub 의 dry-run/시뮬레이션 게이트로 개념 전이 가능.

---

## 4. 외부 검증 사례 — Oracle OCI 의 reconcile 루프

NVIDIA 자체 자료를 넘어, 클라우드 사업자가 이 패턴을 **프로덕션에서** 적용한 유일한 1차 근거:

- Oracle OCI 는 GB200/NVL72 배포에서 **지속적 모니터링·reconciliation 프로세스**를 돌려 "NVL72 컨트롤 플레인(NVLink 도메인, InfiniBand/RoCE)과 **고객의 의도(intention) 간 divergence 를 조기 감지**" 하고, 워크로드에 영향을 주기 전에 인스턴스·네트워크 상태의 불일치를 **자동 교정**한다.
- → 코퍼스에서 가장 명시적인 **observe → detect-divergence-from-intent → auto-correct** 루프. AIPub 이 빌려올 개념 패턴을 검증해 준다.

> ⚠️ **과장 금지**: "의도(intention)를 명시적 source-of-truth 로 규정한다" 와 "OCI 가 fabric 을 Terraform IaC 로 노출한다" 두 강한 주장은 검증에서 **refute(1-2) 됨**. 따라서 OCI 사례는 reconcile *패턴*을 뒷받침할 뿐, "완전한 선언적 IaC" 로 규정해선 안 된다.
