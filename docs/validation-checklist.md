# 검증 체크리스트

- 목적: 변경 전후 확인 항목을 일관되게 관리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

## 변경 전 확인

| 항목 | 기대 결과 | 상태 |
| --- | --- | --- |
| PoC 클러스터 재생성 기준 확인 | 실패 시 복구 경로 명확 | `[입력 필요]` |
| 클러스터가 생성 직후 기본 Calico 상태인지 확인 | 기준선 일치 | `[입력 필요]` |
| worker node 수와 flavor 확인 | PoC 최소 조건 충족 | `[입력 필요]` |
| NKS worker self TCP 전체 허용 확인 | `TCP 1-65535` from worker security group 유지 | 사용자 제공 기준 충족 |
| NKS worker self UDP 전체 허용 확인 | `UDP 1-65535` from worker security group 유지 | 사용자 제공 기준 충족 |
| Cilium VXLAN용 노드 간 8472/UDP 경로 검토 | Cilium tunnel 통신 가능 | 사용자 제공 기준 충족 예상 |
| Hubble Relay용 노드 간 4244/TCP 경로 검토 | Relay 통신 가능 | 사용자 제공 기준 충족 예상 |
| Cilium health용 4240/TCP와 ICMP 검토 | health 확인 가능 | TCP 충족 예상, ICMP 확인 필요 |
| 샘플 워크로드/Ingress 검증 자산 준비 | 테스트 가능 | `[입력 필요]` |
| 성능/운영 단순화 측정 항목 준비 | 성공 판단 가능 | `[입력 필요]` |

## Cilium 확인

| 항목 | 기대 결과 | 상태 |
| --- | --- | --- |
| Cilium Pod 정상 기동 | 핵심 Pod Ready | `[입력 필요]` |
| CNI 교체 후 Pod 네트워킹 정상 | 기본 통신 성공 | `[입력 필요]` |
| kube-system 주요 컴포넌트 통신 정상 | 제어면 영향 없음 | `[입력 필요]` |
| `cilium status` 정상 | 설치 상태 확인 | `[입력 필요]` |
| `cilium connectivity test` 또는 동등 샘플 테스트 통과 | 기본 네트워크 검증 | `[입력 필요]` |

## Hubble 확인

| 항목 | 기대 결과 | 상태 |
| --- | --- | --- |
| Hubble Relay 정상 기동 | Relay Ready | `[입력 필요]` |
| Hubble UI 정상 기동 | UI 접근 가능 | `[입력 필요]` |
| Hubble CLI 접근 가능 | Relay 연동 가능 | `[입력 필요]` |
| DNS, Pod-to-Pod, Service 흐름 조회 가능 | 요구 가시성 확보 | `[입력 필요]` |
| 네임스페이스 또는 라벨 기준 필터링 가능 | 운영 확인 가능 | `[입력 필요]` |
| Hubble metrics 활성화 여부 판단 | 운영형 모니터링 범위 결정 | `[입력 필요]` |
| Hubble exporter 사용 여부 판단 | 장기 flow 검색/감사 범위 결정 | `[입력 필요]` |

## Ingress 확인

| 항목 | 기대 결과 | 상태 |
| --- | --- | --- |
| Cilium Ingress Controller 정상 기동 | 핵심 컴포넌트 Ready | `[입력 필요]` |
| 샘플 HTTP backend에 외부 요청 전달 | north-south 경로 정상 | `[입력 필요]` |
| Ingress 관련 flow가 Hubble에서 식별 가능 | 가시성 확보 | `[입력 필요]` |
| 필요 시 `X-Forwarded-For`/source IP 동작 검토 | 후속 앱 연동 대비 | `[입력 필요]` |

## 샘플 워크로드 확인

| 항목 | 기대 결과 | 상태 |
| --- | --- | --- |
| same namespace 통신 | 정상 | `[입력 필요]` |
| cross namespace 통신 | 정상 | `[입력 필요]` |
| DNS 질의 흐름 | 정상 | `[입력 필요]` |
| NetworkPolicy 추가 시 허용/차단 변화 | Hubble에 반영 | `[입력 필요]` |

## 성능/운영성 확인

| 항목 | 기대 결과 | 상태 |
| --- | --- | --- |
| 샘플 HTTP 요청 p50/p95 지연 수집 | 후속 비교 기준 확보 | `[입력 필요]` |
| 샘플 backend Pod CPU/메모리 수집 | 워크로드 기준선 확보 | `[입력 필요]` |
| Cilium/Hubble/Ingress Pod CPU/메모리 수집 | 관측 구성 오버헤드 확인 | `[입력 필요]` |
| HPA scale-out 이벤트 관측 | 확장 시점 흐름 확인 | `[입력 필요]` |
| 설치/점검 단계 수 기록 | 운영 단순화 판단 근거 확보 | `[입력 필요]` |
| Hubble metrics 또는 exporter 선택 결과 기록 | PoC/운영 모니터링 경계 명확화 | `[입력 필요]` |

## 롤백 조건

- 핵심 워크로드 통신 장애 발생
- 제어면 또는 kube-system 비정상
- Cilium Ingress 또는 Hubble 구성 후 클러스터 기본 네트워킹 회복 불가
- Hubble 요구사항 미충족과 동시에 복구 불가

## 롤백 방식

- 1차 기본안은 PoC 클러스터 삭제 후 재생성입니다.
- 운영 워크로드가 없다는 전제에서 in-place 복구보다 재생성을 우선합니다.
- 재생성 전 `artifacts/`에 실패 시점의 Pod 상태, 이벤트, Cilium/Hubble 상태를 남깁니다.
