# Cilium Ingress vs Gateway API

- 목적: Cilium Ingress와 Cilium Gateway API를 현재 PoC 관점에서 비교하고 선택 기준을 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-17

## 현재 저장소 현황

- 지금까지 저장소에는 `Ingress vs Gateway API` 전용 비교 문서가 없었습니다.
- 기존 문서와 가이드는 `Cilium Ingress` 중심으로 작성되어 있었습니다.
- 이번 문서는 두 방식의 차이와 선택 기준을 정리하기 위해 추가합니다.

## 한 줄 요약

- 빠르게 north-south 경로와 Hubble 가시성을 보는 1차 PoC라면 `Cilium Ingress`가 더 단순합니다.
- 장기적으로 라우팅 표현력, 멀티팀 분리, gRPC/TLSRoute, 향후 서비스메쉬 확장까지 본다면 `Cilium Gateway API`가 더 유리합니다.

## 공통점

- 둘 다 Cilium의 네트워킹 스택과 밀접하게 결합되어 동작합니다.
- 둘 다 per-node Envoy와 eBPF datapath를 사용합니다.
- 둘 다 `kubeProxyReplacement=true`와 `l7Proxy=true` 전제가 사실상 중요합니다.
- 둘 다 기본적으로 `LoadBalancer` 또는 `NodePort` 또는 `hostNetwork` 기반 노출이 가능합니다.
- 둘 다 Hubble에서 north-south 흐름을 관찰할 수 있습니다.
- 둘 다 source IP visibility와 `X-Forwarded-For` 동작을 신경 써야 합니다.

## 차이점 요약

| 항목 | Cilium Ingress | Cilium Gateway API |
| --- | --- | --- |
| 진입 API | `Ingress` | `Gateway`, `HTTPRoute`, `GRPCRoute`, `TLSRoute` 등 |
| 학습 난이도 | 낮음 | 중간 |
| 초기 리소스 수 | 적음 | 많음 |
| CRD 추가 필요 | 없음 | 있음 |
| 라우팅 표현력 | 기본 HTTP/TLS 중심 | 더 구조적이고 확장성 높음 |
| 역할 분리 | 약함 | 강함 |
| 멀티팀 운영 적합성 | 보통 | 좋음 |
| gRPC/TLSRoute | 제한적 | 더 자연스러움 |
| 트래픽 분할/헤더 조작 | 제한적이거나 구현 의존 | HTTPRoute 중심으로 더 자연스러움 |
| PoC 진입 속도 | 빠름 | 보통 |
| 장기 아키텍처 적합성 | 보통 | 높음 |

## 상세 비교

### 1. API 모델

- Ingress는 리소스 하나에 진입점, TLS, 라우팅 규칙이 함께 들어갑니다.
- Gateway API는 `GatewayClass`, `Gateway`, `HTTPRoute` 등으로 역할이 분리됩니다.
- 따라서 Ingress는 단순하고 빠르지만, Gateway API는 구조가 더 명확합니다.

### 2. 표현력과 기능 범위

- Ingress는 host/path 기반 HTTP 라우팅과 TLS termination에 적합합니다.
- Gateway API는 HTTPRoute 외에도 `GRPCRoute`, `TLSRoute`, `ReferenceGrant`를 포함해 더 넓은 시나리오를 다룹니다.
- 추후 canary, header match, request/response 조작, cross-namespace 참조를 명확히 다루려면 Gateway API가 유리합니다.

### 3. 운영 모델

- Ingress는 보통 앱 팀이 바로 만지기 쉬운 구조입니다.
- Gateway API는 `Gateway`와 `Route`를 나누기 때문에 플랫폼 팀과 앱 팀의 역할 분리가 더 자연스럽습니다.
- 현재 PoC는 단일 사용자가 직접 수행하므로 이 장점이 즉시 필요하진 않지만, 장기 구조에는 의미가 있습니다.

### 4. Cilium 관점의 차이

- Ingress와 Gateway API 모두 Cilium에서는 Envoy와 eBPF를 통해 처리되므로, 일반적인 별도 Ingress Controller와 동작 방식이 다릅니다.
- 둘 다 Cilium Network Policy와 `ingress` identity를 기준으로 정책 적용을 받을 수 있습니다.
- 즉, Cilium 관점에서는 두 기능 모두 단순 L7 proxy가 아니라 네트워킹 스택 일부입니다.

### 5. 현재 프로젝트 적합성

현재 프로젝트는 아래 조건을 가집니다.

- 실제 업무 앱이 없음
- 우선 Hubble 가시성과 north-south 동작만 확인하면 됨
- 사용자가 직접 설치하고 저장소는 가이드/검토 중심

이 조건에서는 1차 검증 기준으로 `Cilium Ingress`가 더 빠릅니다.

이유는 아래와 같습니다.

- CRD 사전 설치가 필요 없습니다.
- 리소스 수가 적어 실수 여지가 낮습니다.
- `Ingress -> Service -> Pod` 흐름이 직관적입니다.
- Hubble UI에서 보기 위한 최소 트래픽 생성이 쉽습니다.

반대로 아래를 빨리 보고 싶으면 `Gateway API`가 더 낫습니다.

- 이후 구조를 Gateway 기반으로 가져갈 계획
- gRPC/TLSRoute/traffic split 같은 고급 라우팅
- 팀 간 역할 분리 구조
- 향후 GAMMA 또는 서비스메쉬 방향과의 연결성

## 추천 결론

### 단기 추천

- 1차 PoC: `Cilium Ingress`
- 2차 비교: `Cilium Gateway API`

### 중장기 추천

- 북남향 트래픽 제어를 장기 표준으로 가져갈 생각이면 `Gateway API`를 우선 검토하는 편이 낫습니다.
- 다만 지금처럼 앱이 없고 구축 절차 검증이 먼저인 단계에서는 `Ingress`로 빠르게 기준선을 만든 뒤 `Gateway API`로 확장하는 순서가 더 현실적입니다.

## 선택 기준

아래 질문에 `예`가 많으면 Gateway API 쪽이 유리합니다.

- 라우팅을 더 구조적으로 나누고 싶은가
- gRPC나 TLSRoute를 고려하는가
- 팀 간 역할 분리가 필요한가
- 향후 서비스메쉬/GAMMA까지 염두에 두는가

아래 질문에 `예`가 많으면 Ingress가 유리합니다.

- 빠르게 첫 동작을 보고 싶은가
- 앱/리소스가 아직 거의 없는가
- north-south HTTP 경로만 먼저 확인하면 되는가
- 운영 모델보다 구축 단순성이 더 중요한가

## 참고 가이드

- `docs/cilium-ingress-guide.md`
- `docs/cilium-gateway-api-guide.md`
- `docs/install-runbook.md`

## 참고 출처

- Cilium Ingress: https://docs.cilium.io/en/stable/network/servicemesh/ingress/
- Cilium Gateway API: https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/
- Gateway API 개요: https://gateway-api.sigs.k8s.io/concepts/api-overview/
- Ingress -> Gateway API 마이그레이션 개요: https://gateway-api.sigs.k8s.io/guides/getting-started/migrating-from-ingress
