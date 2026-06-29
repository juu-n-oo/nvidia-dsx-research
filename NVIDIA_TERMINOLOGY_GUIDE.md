# NVIDIA DSX OS 핵심 용어 및 개념 가이드

이 문서는 `report.html`에서 다루는 핵심 기술 용어들을 쉽게 이해하도록 정리한 보충 설명서입니다.

---

## 1. Fabric (패브릭) — 네트워크 인프라 계층

### 정의
**Fabric은 물리 계층의 네트워크 인프라 스택을 뜻합니다.** 즉, GPU 클러스터를 연결하는 네트워크 프로토콜과 토폴로지를 의미합니다.

### 세 가지 주요 Fabric 유형

| Fabric 유형 | 프로토콜 | 주 용도 | 속도/지연성 | 관리 도구 |
|---|---|---|---|---|
| **Spectrum-X Ethernet** | RoCEv2 | 범용 네트워킹, East-West 트래픽 | 보통 | NVUE + NetQ + NICo |
| **InfiniBand (Quantum)** | IB | MPI/집합통신, 저지연 GPU 간 통신 | 빠름, 낮은 지연 | UFM + NICo |
| **NVLink (GB200)** | NVIDIA 독점 | GPU-to-GPU 고대역폭 메모리 공유 | 매우 빠름, 극저지연 | NMX + NICo |

### Fabric 진화 경로
```
Ethernet (범용) 
  ↓
InfiniBand (전문, 저지연)
  ↓
NVLink (GPU 특화, 초고대역폭)
```

**점점 GPU에 특화되고, 점점 더 빨라진다는 것이 맞습니다.**

- **Ethernet**: CPU 중심 네트워크 (모든 서버에 표준)
- **InfiniBand**: GPU 클러스터 전문 (고성능 컴퓨팅용)
- **NVLink**: GPU-to-GPU 직결 (메모리 공유, 극도로 빠름)

---

## 2. NICo (NVIDIA Infra Controller) — 인프라 제어기

### 정의
**NICo는 NVIDIA가 만든 bare-metal 라이프사이클 관리 + 멀티테넌트 네트워크 격리 시스템입니다.**

### 약어 풀이
- **NI**: NVIDIA Infra (NVIDIA 인프라)
- **Co**: Controller (제어기)

### 주 역할

1. **Bare-metal 라이프사이클 관리**
   - 물리 서버(호스트) 등록, 프로비저닝, 상태 추적
   - 파티션/네트워크 설정 자동화

2. **Hardware-enforced 멀티테넌트 격리**
   - DPU(BlueField) 기반 하드웨어 레벨 격리
   - 소프트웨어 정책이 아닌 물리적 분리

3. **Fabric별 Drift 감지 & 자동 교정**
   - InfiniBand: UFM과 연동해 파티션 설정 관리
   - NVLink: NVLink Manager와 연동해 파티션 관리
   - Ethernet: DPU Agent와 30초마다 상태 동기화

### 코드 위치
```
github.com/NVIDIA/infra-controller-core
  └─ Rust (49.5%) + Go (33.8%) 혼합
```

---

## 3. DPU (Data Processing Unit) — 데이터 처리 유닛

### 정의
**DPU는 NVIDIA BlueField라는 SmartNIC(지능형 네트워크 카드)입니다.**

CPU 옆에 장착되어 네트워킹·스토리지·보안 처리를 CPU에서 **offload**하는 전문 프로세서입니다.

### DPU의 특징

| 특징 | 설명 |
|---|---|
| **독립 OS** | DPU 위에는 자체 ARM CPU와 OS가 탑재됨 |
| **격리 실행** | 각 테넌트가 DPU 리소스를 독립적으로 사용 가능 |
| **네트워크 제어** | OVS(Open vSwitch) 기반 패킷 포워딩으로 L2/L3 제어 |
| **하드웨어 강제** | 소프트웨어 정책이 아닌 물리 하드웨어로 격리 강제 |

### NICo와 DPU의 관계

```
NICo (컨트롤러)
  ↓
  DPU Agent (BlueField 위에서 실행)
  ↓
  HBN (Host-Based Networking)
  ↓
  물리 네트워크 구성 변경
```

**NICo가 DPU를 활용해 네트워크를 제어합니다.** 직접적인 경쟁 관계가 아니라 상하위 관계입니다.

### "NICo와 DPU가 같은가?"

**아니요. 다릅니다.**

- **DPU**: 물리 하드웨어 (SmartNIC)
- **NICo**: 소프트웨어 제어기 (DPU를 관리하는 시스템)

비유: 자동차의 엔진(DPU)과 자동차 제어 시스템(NICo)의 관계와 같습니다.

---

## 4. Fabric 진화 순서 확인

### 당신의 이해가 맞습니다!

```
Ethernet → InfiniBand → NVLink
  (느림)  →  (중간)   →  (빠름)
  (범용)  → (전문)   → (GPU 특화)
```

**특성별 비교**

| 특성 | Ethernet | InfiniBand | NVLink |
|---|---|---|---|
| **지연시간** | 마이크로초 | 마이크로초 중간대 | 나노초 |
| **대역폭** | 400Gbps | 600Gbps | ~1.8Tbps |
| **GPU 최적화** | 간접 (RoCEv2) | 최적화 (MPI) | 완전 최적화 |
| **용도** | 일반 트래픽 | 집합통신 | GPU 메모리 공유 |

### Day-2 자동 교정의 차이

- **Ethernet**: DPU Agent가 30초마다 폴링 (자주)
- **InfiniBand**: IbFabricMonitor 주기 감지 (덜 자주)
- **NVLink**: NvlPartitionMonitor 주기 감지 (덜 자주)

Ethernet이 가장 자주 체크하는 이유: 소프트웨어 기반이라 변경이 빈번할 수 있기 때문입니다.

---

## 5. DPF (DOCA Platform Framework) — DOCA 플랫폼 프레임워크

### 정의
**DPF는 DPU를 Kubernetes 방식으로 프로비저닝하고 관리하는 오픈소스 프레임워크입니다.**

### 약어 풀이
- **D**: DOCA (NVIDIA 엣지 컴퓨팅 라이브러리)
- **P**: Platform
- **F**: Framework

### DPF의 역할

1. **DPU 프로비저닝**
   - BlueField DPU에 OS 이미지(BFB) 플래시
   - 네트워크 설정 자동 적용

2. **DPU 클러스터 관리**
   - Kamaji(경량 k8s 컨트롤 플레인)를 이용해 DPU 자신만의 k8s 클러스터 구성
   - Host Cluster와 DPU Cluster는 **완전히 분리됨**

3. **서비스 오케스트레이션**
   - DOCA 서비스(패킷 처리, 보안, 스토리지 등)를 CRD로 선언
   - ArgoCD와 Helm을 이용해 DPU 위에 자동 배포

### DPF 구조
```
┌─────────────────────────┐
│  HOST CLUSTER (k8s)     │
│  ┌─────────────────┐    │
│  │ DPF Operator    │    │
│  │ (프로비저닝)     │    │
│  │ (서비스 배포)    │    │
│  └─────────────────┘    │
└────────────┬────────────┘
             │
┌────────────┴────────────┐
│  DPU CLUSTER (k8s)      │
│  (Kamaji 기반)          │
│  DPU 노드들             │
│  DOCA 서비스 Pod        │
└─────────────────────────┘
```

### NICo vs DPF

| 구분 | NICo | DPF |
|---|---|---|
| **담당 레이어** | 물리 네트워크 인프라 | DPU 오케스트레이션 |
| **관리 대상** | 네트워크 구성 (partition, 포트) | DPU의 OS·서비스 |
| **제어 단위** | Fabric 리소스 | Kubernetes CR |
| **적용 방식** | API 호출 (UFM, NVLink Manager) | k8s-native (ArgoCD, Helm) |

**둘은 상호보완적입니다:**
- NICo는 네트워크를 제어
- DPF는 DPU를 k8s처럼 관리

---

## 요약 테이블

| 개념 | 역할 | 관계 |
|---|---|---|
| **Fabric** | 물리 네트워크 프로토콜/토폴로지 | 기반 인프라 |
| **NICo** | Fabric·멀티테넌트 네트워크 제어 | Fabric 관리자 |
| **DPU** | 네트워킹·보안 offload 하드웨어 | NICo가 제어하는 대상 |
| **DPF** | DPU의 k8s 오케스트레이션 | NICo와 직교 |

---

## AIPub 적용 시 유의점

당신의 AIPub 플랫폼 입장에서:

- **NICo/DPF는 물리 인프라 계층** → 건드리지 말 것
- **AIPub은 k8s 플랫폼 계층** → ResourceGroup/NetworkPolicy로 격리 구현
- **LayerBoundary**: 물리 fabric(NICo)과 k8s 플랫폼(AIPub)을 명확히 분리

report.html의 "AIPub 적용 방안" 섹션을 참고하세요.

---

**마지막 질문: HTML report에 이 설명을 "부가 설명" 섹션으로 추가하시겠습니까?**
