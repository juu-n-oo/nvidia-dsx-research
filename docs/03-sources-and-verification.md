# 03. 출처 · 검증 메타데이터

조사: `deep-research` 하네스 (2026-06).
파이프라인: 5 검색 각도 fan-out → 21 소스 fetch → 95 claim 추출 → 상위 25 claim 3표 적대적 검증 → 종합.
결과: **23 confirmed / 2 refuted**, 종합 후 11 findings.

## 검증 통계

| 항목 | 값 |
|---|---|
| 검색 각도 | 5 |
| Fetch 소스 | 21 |
| 추출 claim | 95 |
| 검증 claim | 25 (confirmed 23 / killed 2) |
| 종합 후 finding | 11 |
| 에이전트 호출 | 103 |

## 1차 출처 (primary)

### DSX / DSX OS
- NVIDIA DSX 제품 페이지 — https://www.nvidia.com/en-us/data-center/products/dsx/
- 보도자료: DSX gives infrastructure builders the playbook — https://nvidianews.nvidia.com/news/dsx-infrastructure-ai-factory
- 개발자 블로그: DSX OS delivers open modular software — https://developer.nvidia.com/blog/nvidia-dsx-os-delivers-open-modular-software-for-operating-ai-factories-at-scale/
- 보도자료: Vera Rubin DSX reference design + Omniverse DSX digital twin — https://nvidianews.nvidia.com/news/nvidia-releases-vera-rubin-dsx-ai-factory-reference-design-and-omniverse-dsx-digital-twin-blueprint-with-broad-industry-support

### 구성 관리 / fabric
- NMX-C 문서 — https://docs.nvidia.com/networking/display/nmx-controller-nmx-c-documentation-v1-3-0.0.pdf , https://networking-docs.nvidia.com/nmxcswum/11/nmx-controller
- NMX 소개 — https://networking-docs.nvidia.com/nmxcswum/11/nmx+introduction
- UFM — https://www.nvidia.com/en-us/networking/infiniband/ufm/ , https://www.nvidia.com/en-eu/networking/infiniband/ufm2/
- UFM Cable Validation Tool — https://docs.nvidia.com/networking/display/cablevalidationtool181/introduction
- NVUE (Cumulus Linux) — https://docs.nvidia.com/networking-ethernet-software/cumulus-linux-55/System-Configuration/NVIDIA-User-Experience-NVUE/NVUE-CLI/
- NetQ — https://www.nvidia.com/en-us/networking/ethernet-switching/netq/
- NetQ validation checks — https://docs.nvidia.com/networking-ethernet-software/cumulus-netq-51/Validate-Operations/Validation-Checks/
- Mission Control high-speed fabric management — https://docs.nvidia.com/mission-control/docs/systems-administration-guide/2.0.0/high-speed-fabric-management.html
- Spectrum-X telemetry — https://developer.nvidia.com/blog/next-generation-ai-factory-telemetry-with-nvidia-spectrum-x-ethernet/
- NVIDIA networking software — https://www.nvidia.com/en-us/networking/products/software/

### 외부 적용 사례
- Oracle OCI: behind the scenes, GB200 NVL72 deployments — https://blogs.oracle.com/cloud-infrastructure/behind-the-scenes-scale-nvidia-gb200-nvl72-deployments

### k8s 선언적 구성 관리 패턴 (blog/practitioner)
- Operator reconciliation loop — https://oneuptime.com/blog/post/2026-02-09-operator-reconciliation-loop/view
- Flux CD vs ArgoCD drift detection — https://oneuptime.com/blog/post/2026-03-13-flux-cd-vs-argocd-drift-detection/view
- Kubernetes configuration drift — https://komodor.com/learn/kubernetes-configuration-drift-causes-detection-and-prevention/
- Kubebuilder good practices — https://book.kubebuilder.io/reference/good-practices

## Refuted claims (검증 탈락, 2건)

1. **"OCI 가 고객의 의도(intention)를 명시적 source-of-truth 로 규정하고 물리 fabric 상태를 그에 reconcile 한다"** — vote 1-2 ✗. (reconcile 루프 존재는 confirmed 이나, "의도를 명시적 source-of-truth 로 규정" 까지는 근거 부족.)
2. **"OCI 가 GB200 NVL72 fabric 구성을 Terraform IaC 로 선언적 노출한다"** — vote 1-2 ✗.

→ 시사점: OCI 사례는 reconcile *패턴*을 뒷받침할 뿐, "완전한 선언적 IaC" 로 과장하지 말 것.

## Caveats (해석 시 주의)

1. **명칭**: 요청서의 "NVIDIA Config Manager (MVCM)" 는 검증 후 실제 제품으로 발견되지 않음. 실재 주체는 NMX-C / UFM / NVUE / NetQ / CVT. 모든 설계는 실명 기준으로.
2. **출처 편향**: DSX/DSX OS claim 거의 전부가 NVIDIA 자체 2026-06 런칭 자료(보도자료·제품페이지·개발자블로그) — 서술형 마케팅이며 모든 광고 기능의 성숙/제공 여부를 독립 검증한 3자 출처 없음. 공개 GitHub 레포명이 가장 강한 비-마케팅 신호.
3. **감지 vs 교정**: CVT·NetQ snapshot-compare·NVUE diff·UFM validation 은 **drift 감지/diff** 까지. 자동 교정 루프는 NMX-M→NMX-C 와 OCI 만 서술. "diff 기능" 과 "닫힌 reconcile 루프" 를 혼동 금지.
4. **시점 민감성**: DSX/Vera Rubin 은 2026-03~06 발표된 매우 빠르게 변하는 영역. 컴포넌트명·레포·기능이 바뀔 수 있음.
5. **AIPub 적용(02 문서)은 분석가 합성(medium confidence)** — 검증된 NVIDIA 패턴 + CLAUDE.md 의 AIPub 아키텍처에서 추론한 *제안*이지 sourced fact 아님.

## Open questions (후속 조사 거리)

1. DSX OS 자체가 선언적/k8s 네이티브 API(CRD)를 노출하는가? `NVIDIA/infra-controller-core`, `NVIDIA/doca-platform` 레포를 직접 fetch 해 API 모델 확인 필요.
2. 단일 DSX OS 배포에서 NMX(NVLink)/UFM(InfiniBand)/NVUE·NetQ(Ethernet) 의 분업과 관계 — **fabric 통합 단일 source-of-truth 인가, 3개 분리인가** (현재 근거상 분리로 보임).
3. AIPub `resource-group-controller` 가 이미 drift 감지 reconcile 을 구현하는지, 그리고 desired-vs-observed diff UI·dry-run·시뮬레이션 gap 이 실제로 어디인지 — **컨트롤러 코드 직접 확인 필요**.
4. NVIDIA 도구가 감지된 drift 를 자동 remediate 하는 문서화된 메커니즘이 NMX-M/OCI 외에 있는가, 아니면 human-in-the-loop 이 표준인가 — AIPub 의 auto-correct vs alert-and-approve 정책 결정에 영향.
