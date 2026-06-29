# nvidia-dsx-research

NVIDIA **DSX / DSX OS** 와 그 안의 **네트워크·AI Factory 구성 관리(configuration management)** 체계를 조사하고,
그 핵심 패턴(**기준 정보 기반 선언적 구성 관리 + drift 감지 + reconcile**)을 우리 **AIPub**(쿠버네티스 기반 ML 개발 플랫폼)에 어떻게 차용할 수 있을지 정리한 리서치 레포다.

> 조사 시점: 2026-06. DSX/Vera Rubin 은 2026-03~06 발표된 최신 영역이라 명칭·구성요소·기능이 빠르게 바뀔 수 있다.

## ⚠️ 가장 먼저 알아야 할 정정 사항

리서치 요청서의 가설 명칭이던 **"NVIDIA Config Manager (MVCM)" 는 실제 NVIDIA 제품명으로 확인되지 않았다.**
다중 소스 검증 결과, NVIDIA 의 실제 fabric/구성 관리 주체는 아래 5개다 (fabric 종류별로 분리):

| 영역 | 실제 제품/도구 | 역할 |
|---|---|---|
| NVLink fabric (GB200 등) | **NMX** (NMX-C/T/M/Oasis) | SDN 컨트롤 플레인 + 텔레메트리 + ML 분석 + 데이터레이크 |
| InfiniBand fabric | **UFM** (Unified Fabric Manager) | 프로비저닝·모니터링·검증(13 validation tests) |
| Ethernet (Spectrum-X / Cumulus) | **NVUE + NetQ** | 선언적 config 모델 + 운영/검증 툴셋 |
| 물리 케이블링 | **UFM Cable Validation Tool (CVT)** | 토폴로지 파일 기준 배선 drift 감지 |
| 배포 전 검증 | **Omniverse DSX (Sim)** | 디지털 트윈으로 변경 사전 시뮬레이션 |

따라서 본 리서치는 "MVCM" 이라는 가상의 단일 제품이 아니라, **위 실재 제품군이 공통으로 구현하는 "기준 정보(source-of-truth) → diff → drift 감지 → (부분적) 자동 교정" 패턴**에 초점을 맞춘다. 이 패턴이 AIPub 에 전이 가능한 진짜 자산이다.

## 핵심 결론 (TL;DR)

1. **DSX 는 단일 제품/OS 가 아니라 "AI Factory 를 설계·시뮬레이션·구축·운영" 하는 풀스택 프레임워크**다. `DSX OS` 는 그중 *운영(operations)* 담당 오픈소스 모듈 컴포넌트다. 모두 **물리/인프라 레이어**(칩·시스템·네트워크·전력·시설)를 다루며, 이는 **AIPub 이 위치한 ML 플랫폼 레이어보다 아래**다.
2. NVIDIA 의 구성 관리 도구들은 **선언적 desired-state + diff 기반 drift 감지** 를 일관되게 구현한다 (NVUE 4-state 모델, CVT 토폴로지 기준 배선 검증, NetQ snapshot-compare, UFM validation). 단 **대부분은 "감지(detection)" 까지**이고, **자동 교정(reconcile) 루프는 NMX-M→NMX-C 와 클라우드 사업자(예: Oracle OCI) 구현**에서만 확인된다.
3. **AIPub 적용 관점**: 이 "선언적 source-of-truth + 지속적 drift 감지 + reconcile" 모델은 사실 **쿠버네티스 오퍼레이터의 네이티브 패턴**이다. AIPub 은 이미 `resource-group-controller` 로 reconcile 기반이므로, NVIDIA 의 *물리 fabric 툴을 재구현하는 게 아니라* **그들이 보여준 UX/운영 패턴**(① desired-vs-observed diff 뷰, ② apply 전 dry-run/시뮬레이션 게이트, ③ 분류된 drift 리포트, ④ telemetry→분석→교정 루프)을 ML 플랫폼 레이어에 도입하는 것이 핵심이다.

## 문서 구성

| 문서 | 내용 |
|---|---|
| [`docs/01-nvidia-dsx-and-config-management.md`](docs/01-nvidia-dsx-and-config-management.md) | NVIDIA DSX/DSX OS 의 정체, 구성 관리 5종(NMX/UFM/NVUE/NetQ/CVT)의 desired-state·drift·reconcile 동작 — 검증된 사실 정리 |
| [`docs/02-aipub-application.md`](docs/02-aipub-application.md) | **(핵심)** AIPub 에 차용할 구체적 아이디어 — 레이어 경계, 매핑 표, 단계별 도입안, 안티패턴 |
| [`docs/03-sources-and-verification.md`](docs/03-sources-and-verification.md) | 출처 목록, 검증 메타데이터(confirmed/refuted), caveat, open questions |

## 방법론

`deep-research` 하네스(5 검색 각도 fan-out → 21 소스 fetch → 95 claim 추출 → 상위 25 claim 3표 적대적 검증)로 수행. 23 claim confirmed / 2 refuted. 거의 모든 DSX 1차 출처는 NVIDIA 의 2026-06 런칭 자료(보도자료·제품 페이지·개발자 블로그)라 **마케팅 프레이밍이 섞여 있음**에 유의. 자세한 검증 메타는 문서 03 참조.
