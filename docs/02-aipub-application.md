# 02. AIPub 적용 관점 (핵심)

> 본 문서는 검증된 NVIDIA 패턴(01 문서) + AIPub 아키텍처(workspace `CLAUDE.md` / `docs/aipub-domain.md`)에 근거한 **분석가 합성(synthesis)** 이다. NVIDIA 측 사실과 달리 *제안*이며, 신뢰도는 medium 으로 본다. 실제 도입 전 `resource-group-controller` 코드로 현 reconcile/gap 을 재확인할 것 (03 문서 open question).

## 0. 대전제 — 레이어 경계를 먼저 못 박는다

NVIDIA DSX/NMX/UFM/CVT/NVUE 는 **물리 AI Factory 인프라**(칩·NVLink/InfiniBand/Ethernet fabric·전력·시설)를 다룬다. **AIPub 은 그 위의 ML 플랫폼 레이어**다 (k8s 멀티테넌트 프로젝트/리소스 그룹, CR 워크로드 Workspace/Operation/Job, Harbor 레지스트리).

따라서 결론은 명확하다:

> **NVIDIA 의 fabric 툴(NMX/UFM/CVT/NVUE/Spectrum-X)을 재구현하지 않는다.**
> 대신 그들이 검증한 **제어 패턴과 운영 UX** 만 빌려, AIPub 레이어의 자원에 적용한다.
> 그리고 그 패턴은 사실 **쿠버네티스 오퍼레이터의 네이티브 모델**이므로, AIPub(특히 `resource-group-controller`)은 이미 절반쯤 와 있다.

AIPub 이 물리 인프라(ten1010.io 클러스터/네트워크)를 *연동/표시*할 수는 있어도 *소유/제어*하지 않는다는 점이 핵심 구분선이다.

## 1. NVIDIA 패턴 → AIPub 매핑

| NVIDIA 자산 (검증됨) | 핵심 아이디어 | AIPub 으로의 전이 |
|---|---|---|
| **NVUE** Pending/Applied/Operational/Startup 4-state + `nv config diff` | 의도(spec)와 실제(running)를 상태로 분리하고 둘을 diff | 리소스 그룹/쿼터/네트워크정책/워크로드 템플릿의 **desired(CR spec) vs observed(cluster status) diff 뷰** + **apply 전 미리보기(dry-run) 게이트** |
| **NMX-M → NMX-C** telemetry→분석→제어 폐루프, **OCI** 지속 reconcile | drift 를 지속 감지하고 자동 교정 | AIPub 오퍼레이터의 **reconcile 컨트롤러**가 선언된 리소스그룹/정책/쿼터 상태와 라이브 클러스터 상태의 drift 를 감지 → **자동 교정 또는 알림** |
| **CVT** 토폴로지 파일 기준 + Wrong-neighbor/Wrong-port/Unknown 분류 | source-of-truth 대비 **이산·actionable drift 분류** | AIPub **"expected vs actual" 검증 리포트** — 누락된 namespace, 쿼터 불일치, RBAC/정책 divergence, 누락 imagePullSecret 등을 **이산 카테고리**로 분류 |
| **Omniverse DSX Sim / NetQ+Air CI/CD** | 프로덕션 건드리기 전 모델에서 변경 검증 | 리소스그룹/쿼터/정책 변경을 적용 전 **dry-run·시뮬레이션**(영향 받는 namespace/워크로드/쿼터 초과 여부 사전 계산) |
| **UFM** 13 validation tests / NetQ preventive validation | 표준화된 health/validation 체크 집합 | 리소스 그룹 단위 **표준 validation 체크 묶음**(쿼터 정합성, RBAC 완전성, 레지스트리 시크릿 존재, 네임스페이스 라벨/오너레퍼런스 일관성) |

## 2. AIPub 현황과 gap

`resource-group-controller`(= project-controller)는 이미 **쿠버네티스 오퍼레이터**로서:
- `Project` / `AipubUser` / `NodeGroup` / `ImageHub` CR 을 watch → reconcile
- 파생 리소스(Namespace/RBAC/ResourceQuota/Secret/admission webhook) 를 생성·정합

즉 **"spec=desired → reconcile → 파생 리소스 정합"** 의 핵심 루프는 *이미 존재*한다. NVIDIA 가 보여준 것 중 AIPub 에 **빠져 있을 가능성이 높은 gap**(코드 재확인 필요):

1. **사용자 노출 desired-vs-observed diff 가 없음** — reconcile 은 백그라운드로 돌지만, 운영자가 "지금 이 리소스 그룹의 선언 상태와 실제 클러스터 상태가 어디가 다른가" 를 보는 UI/리포트가 없음 (NVUE `nv config diff` / CVT 리포트에 해당).
2. **apply 전 dry-run/시뮬레이션 게이트가 없음** — 쿼터 축소·정책 변경이 어떤 워크로드를 깨뜨릴지 사전 계산해 보여주는 단계 (DSX Sim / NetQ preventive validation 에 해당).
3. **drift 의 명시적 분류·리포팅이 없음** — drift 가 "감지되어 조용히 교정" 될 뿐, CVT 처럼 *분류된 actionable 리포트*로 남지 않음 (감사/디버깅 가치 큼).
4. **교정 vs 알림 정책 선택지가 불명확** — 자동 교정할지(self-heal) 사람 승인 후 적용할지(alert-and-approve)의 정책 레이어.

## 3. 단계별 도입 제안

NVIDIA 도구 대부분이 "**감지(detection) 먼저, 자동 교정은 신중하게**" 인 점을 그대로 따른다 (CVT/NetQ/NVUE/UFM 은 감지·diff 까지, 자동 교정은 NMX-M·OCI 뿐). 위험 낮은 것부터:

### Phase 1 — Drift 가시화 (감지·읽기 전용, 위험 0)
- 각 리소스 그룹(`Project`)에 대해 **desired(spec) vs observed(cluster) diff** 를 계산해 CR `status` 와 React 프론트에 노출.
- NVUE 4-state 차용: `spec(의도)` / `applied(마지막 reconcile 결과)` / `observed(현재 클러스터)` 를 구분 표시.
- drift 를 **이산 카테고리**로 분류(CVT 차용): `MissingNamespace`, `QuotaMismatch`, `RbacDivergence`, `MissingRegistrySecret`, `OrphanedResource`, `LabelOwnerRefDrift` 등.
- 산출물: 운영자용 "리소스 그룹 정합성 리포트". **로그/status 메시지는 영어**(workspace 규칙 `logs-and-status-messages-english`).

### Phase 2 — Validation-before-apply (dry-run 게이트)
- 리소스 그룹/쿼터/정책 변경 시 **적용 전 영향 시뮬레이션**: 영향 namespace, 쿼터 초과로 evict 될 워크로드, 끊길 RBAC 등을 사전 계산해 미리보기.
- k8s 네이티브로 충분: **server-side apply `--dry-run`**, admission/validation webhook(이미 `UserAuthorityReview` 등 webhook 존재), `kubectl diff` 류 로직.
- NVIDIA 의 디지털 트윈처럼 별도 시뮬레이터를 만들 필요 없음 — *API dry-run 으로 80% 달성*.

### Phase 3 — 정책 기반 reconcile (자동 교정 vs 승인)
- 리소스 그룹별로 drift 처리 정책 선언: `autoHeal`(즉시 교정) vs `alertOnly`(리포트만) vs `approvalRequired`(승인 후 교정).
- NMX-M/OCI 의 폐루프를 차용하되, **물리 인프라가 아닌 AIPub 자원 한정**. controller-runtime 의 주기적 resync 로 지속 감지.

### Phase 4 (선택) — Source-of-truth 외부화 / GitOps
- 리소스 그룹 정의를 Git 에 두고 ArgoCD/Flux 식 drift 감지(코퍼스의 k8s 모범사례)와 연결.
- AIPub 자체 CRD 가 이미 source-of-truth 이므로 **필수 아님** — 멀티클러스터/감사 요구가 생길 때만.

## 4. 안티패턴 (하지 말 것)

- ❌ **NVIDIA fabric 툴 재구현** — NMX/UFM/CVT/NVUE/Spectrum-X telemetry 를 AIPub 안에 다시 만들기. 레이어가 다르고 물리 장비 접근권도 없음.
- ❌ **"MVCM" 이라는 이름에 의존** — 실존 제품 아님(03 문서). 설계 문서/티켓에 가상 제품명을 적지 말 것.
- ❌ **k8s 가 이미 주는 것을 평행 구현** — reconcile/diff/dry-run/webhook 은 controller-runtime·server-side apply·admission webhook 으로 충분. 별도 구성관리 엔진을 새로 만들지 말 것 (코퍼스 k8s 모범사례 일관된 권고).
- ❌ **감지를 곧장 자동 교정으로** — NVIDIA 도구 대부분이 감지 우선인 이유를 존중. Phase 1(가시화)→2(검증)→3(정책 교정) 순서 유지.
- ❌ **물리 인프라 제어 주장** — AIPub 은 ten1010.io 클러스터/네트워크를 *연동/표시*할 뿐 소유/제어하지 않는다.

## 5. 한 문단 요약

NVIDIA DSX 가 AI Factory **물리 인프라**에서 보여준 "**선언적 기준정보 → 지속적 drift 감지 → (정책적) 자동 교정 + 적용 전 시뮬레이션**" 패턴은, 사실 쿠버네티스 오퍼레이터의 네이티브 모델이다. AIPub 은 `resource-group-controller` 로 이미 reconcile 루프를 갖고 있으므로, 새 엔진을 만들 게 아니라 **NVUE 식 desired-vs-observed diff 가시화 → dry-run 검증 게이트 → 분류된 drift 리포트 → 정책 기반 교정** 을 ML 플랫폼 레이어 자원(리소스 그룹/쿼터/RBAC/정책/레지스트리)에 점진 도입하면 된다. NVIDIA 의 fabric 도구 자체는 빌려오지 않는다 — 빌려오는 것은 **운영 패턴과 UX** 다.
