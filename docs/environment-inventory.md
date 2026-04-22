# 환경 인벤토리

- 목적: 현재 NKS 클러스터와 네트워크 관련 전제 조건을 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

## 클러스터 기본 정보

| 항목 | 값 |
| --- | --- |
| 프로젝트/테넌트 | `kubectl` 조회만으로 확인 불가 |
| 리전 | 판교. kubeconfig API endpoint 기준 `nks-kr1`, 노드 zone 기준 `kr-pub-a`, `kr-pub-b` |
| 클러스터명 | `ta-sgh-cilium-hubble-cls` |
| kubeconfig context | `nks_ta-sgh-cilium-hubble-cls_7e449296-7304-4066-8307-824b4f71e37e` |
| Kubernetes 버전 | server `v1.33.4`, client `v1.34.1` |
| 노드 OS/이미지 | `Ubuntu 24.04.4 LTS` |
| 노드 커널 | `6.8.0-88-generic` |
| 컨테이너 런타임 | `containerd://1.6.21` |
| 노드풀 구성 | `default-worker` node group, worker 3대 |
| 노드 flavor | `c2.c2m2` |
| 노드 zone 분산 | node-0: `kr-pub-a`, node-1/2: `kr-pub-b` |
| 퍼블릭/프라이빗 여부 | 노드 External-IP는 없음. 노드 zone label은 `kr-pub-*`. 실제 subnet/public-private 정책은 NKS 콘솔 확인 필요 |

## 접속 및 운영 정보

| 항목 | 값 |
| --- | --- |
| `kubectl` 접속 경로 | WSL 기준 kubeconfig 연결 확인 |
| API server | `https://7e449296-nks-kr1.container.nhncloud.com:6443` |
| 접근 방식(직접/VPN/Bastion) | WSL에서 직접 `kubectl` 접근 가능. Windows PowerShell은 kubeconfig 권한 문제로 미사용 |
| 변경 가능 시간대 | PoC 기준 유연 |
| 운영 담당자 | 사용자 |
| 롤백 의사결정자 | 사용자 |

## 네트워크 및 애드온 현황

| 항목 | 값 |
| --- | --- |
| 현재 CNI 배포 형태 | Calico VXLAN. `calico_backend: vxlan` |
| Calico 버전 | `v3.30.2` (`calico/node`, `kube-controllers`, `typha`) |
| NetworkPolicy 사용 여부 | 현재 리소스 없음 |
| Ingress Controller | Cilium Ingress 도입 예정 |
| 현재 Ingress 리소스 | 없음 |
| 현재 LoadBalancer Service | 없음. `default/my-service`는 임시 생성 후 사용자 삭제 완료 |
| LoadBalancer 사용 방식 | NKS 기본 LoadBalancer 사용 가능 여부는 샘플 Ingress/Service로 재검증 필요 |
| 서비스 메시 사용 여부 | PoC 대상 클러스터에는 미구축 |
| 관측 스택(Prometheus/Grafana 등) | Hubble Relay/UI/CLI 우선, 추가 스택은 후속 검토 |
| 기본 Add-on | CoreDNS, metrics-server, OpenStack Cinder CSI |
| Pod CIDR | Node spec과 Calico IPPool CRD에서 직접 확인 불가. 관측된 Pod IP는 `10.100.184.0/24`, `10.100.188.0/24` 대역 |
| Service CIDR | API server flag로 직접 확인 불가. 관측된 Service IP는 `10.254.0.1`, `10.254.0.10`, `10.254.x.x` 대역 |

## 확인 필요 사항

- Pod CIDR과 Service CIDR의 공식 값: `kubectl` 조회만으로 확정 불가. NKS 콘솔 또는 생성 파라미터 확인 필요
- 보안그룹: Hubble Relay용 노드 간 `TCP 4244` 허용 여부 확인 필요
- LoadBalancer: Cilium Ingress 또는 샘플 `Service type LoadBalancer`로 실제 생성 가능 여부 확인 필요
- 커널 또는 eBPF 관련 제약: 현재 노드 커널은 `6.8.0-88-generic`, Calico는 `v3.30.2`
- 보안 정책 또는 승인 절차: PoC 기준 미정
- 멀티 클러스터 여부: 단일 클러스터
- 다운타임 허용 범위: PoC 기준 엄격 제약 없음

## 설계 가정

- worker node 수는 최소 3대를 권장합니다.
- 실제 업무 워크로드 대신 샘플 HTTP 워크로드와 테스트 Pod로 가시성을 검증합니다.
- Cilium Ingress는 NKS LoadBalancer 기반 진입을 우선 가정합니다.
- 실패 시 일반 백업 복구보다 PoC 클러스터 재생성을 기본 롤백 경로로 둡니다.

## 메모

- 현재 문서는 WSL `kubectl` read-only 조회 결과를 기준으로 작성되었습니다.
- 클러스터 직접 변경은 수행하지 않았습니다.
- 퍼블릭/프라이빗 여부, Pod/Service CIDR 공식 값, 보안그룹 세부 정책은 NKS 콘솔 기준 추가 확인이 필요합니다.
