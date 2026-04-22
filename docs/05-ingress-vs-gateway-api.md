# Cilium Ingress vs Gateway API

- 목적: Cilium Ingress와 Cilium Gateway API를 현재 PoC 관점에서 비교하고 선택 기준을 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

## 결론

현재 기본안은 `Cilium Ingress`입니다.

이유:

- 1차 PoC에서 가장 빨리 north-south 흐름을 만들 수 있습니다.
- 리소스 수가 적고 문제 위치를 좁히기 쉽습니다.
- Hubble UI/CLI에서 Ingress -> Service -> Pod 흐름을 확인하기 쉽습니다.

Gateway API는 장기 표준 후보입니다.

이유:

- Gateway와 Route가 분리되어 운영 모델이 명확합니다.
- HTTPRoute, GRPCRoute, TLSRoute 등 확장성이 좋습니다.
- 추후 GAMMA 또는 서비스메쉬 대체 방향과 연결성이 좋습니다.

## 차이 요약

| 항목 | Cilium Ingress | Cilium Gateway API |
| --- | --- | --- |
| 현재 우선순위 | 1순위 | 2순위 |
| 진입 리소스 | `Ingress` | `Gateway`, `HTTPRoute` |
| CRD 사전 설치 | 불필요 | 필요 |
| PoC 속도 | 빠름 | 중간 |
| 표현력 | 기본 HTTP/TLS 중심 | 더 넓음 |
| 운영 역할 분리 | 약함 | 강함 |
| 장기 표준성 | 보통 | 높음 |
| Hubble 연동 | 가능 | 가능 |

## 현재 프로젝트 기준 선택

Ingress를 선택할 조건:

- 앱이 아직 없음
- 샘플 HTTP만 보면 됨
- LoadBalancer 생성과 Hubble 흐름 확인이 우선
- 빠른 실패/성공 판단이 중요

Gateway API를 선택할 조건:

- 장기 구조를 Gateway API로 가져갈 계획
- gRPC/TLSRoute/traffic split 가능성을 빨리 보고 싶음
- 플랫폼 팀과 앱 팀 역할 분리를 고려함
- 서비스메쉬 대체 방향까지 이어서 볼 예정

## 실행 파일

- Ingress 기본안: `docs/10-track-a-full-replacement-ingress.md`
- Gateway API 대안: `docs/20-track-b-full-replacement-gateway-api.md`
- 전체 인덱스: `docs/00-install-runbook.md`

## 참고 출처

- https://docs.cilium.io/en/stable/network/servicemesh/ingress/
- https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/
- https://gateway-api.sigs.k8s.io/concepts/api-overview/
