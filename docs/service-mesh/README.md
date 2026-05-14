# Cilium Service Mesh 문서

- 목적: 고객 요청으로 Cilium Service Mesh 도입이 필요할 때 사용할 계획, 실행 절차, 검증 기준을 관리합니다.
- 상태: draft
- 마지막 갱신: 2026-05-13

현재 클러스터는 NKS 기본 Calico VXLAN을 유지하고 Cilium `generic-veth` chaining을 적용한 상태입니다.
검증 결과 현재 NKS Calico chaining 환경에서는 Cilium Service Mesh를 운영 수준으로 구성하지 않습니다.
Service Mesh 표준 실행 가이드는 Cilium primary CNI가 가능한 별도 환경 기준으로 관리합니다.

## 문서 목록

| 파일 | 용도 |
| --- | --- |
| `00-build-plan.md` | NKS Calico chaining 환경의 검증 계획과 최종 판정 |
| `01-cilium-service-mesh-runbook.md` | Cilium primary 기반 Service Mesh/Hubble L7 표준 실행 절차 |
| `02-validation-checklist.md` | 적용 전후 검증 항목 |
| `03-cilium-service-mesh-concepts.md` | Cilium Service Mesh 개념, 구성요소, 현재 NKS 적용 의미 |
| `04-operational-weight-comparison.md` | Cilium chaining 관측 구성과 Istio Ambient + Kiali 운영 무게 비교 |
| `05-build-result-report.md` | 현재 NKS 구성에서 Cilium Service Mesh 실패 결과와 가능 범위 |

## 관련 manifest

- `manifests/service-mesh/`

## 기본 원칙

- 클러스터 변경은 사용자가 직접 수행합니다.
- 이 디렉터리는 실행 계획, 명령 초안, values, 샘플 manifest를 관리합니다.
- `kubeProxyReplacement=true`, Gateway API CRD 설치는 영향 범위가 큰 변경으로 분리 검토합니다.
- 실제 NKS 검증 결과와 운영 가능/불가 판정은 `05-build-result-report.md`에 분리해서 기록합니다.
- 현재 NKS Calico chaining 클러스터에서는 Cilium Service Mesh 재적용 절차를 권장하지 않습니다.
