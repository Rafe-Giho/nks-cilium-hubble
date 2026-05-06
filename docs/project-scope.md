# 프로젝트 범위

- 목적: NKS에서 sidecar-less 트래픽 관측을 구현하기 위한 범위와 성공 기준을 고정합니다.
- 상태: draft
- 마지막 갱신: 2026-05-06

## 목표

- sidecar-less 트래픽 모니터링의 유효한 구현 경로를 우선순위 기반으로 검토합니다.
- NHN Cloud NKS에서 Calico 유지 기반 Cilium chaining + Hubble 적용 가능 여부를 검증합니다.
- Pixie, Grafana Beyla, Calico-eBPF 등 Cilium 외 sidecar-less 관측 후보를 비교합니다.
- 서비스메쉬 sidecar 오버헤드 대안으로서 운영 단순화 가능성을 확인합니다.
- 적용 전후 검증 및 롤백 기준을 문서화합니다.

## 배경

- 기존 운영 경험상 서비스메쉬와 Jaeger 기반 구조는 결합도와 운영 복잡도가 높습니다.
- sidecar 구조는 애플리케이션 로직과 직접 결합하지 않더라도 HPA 시점 자원 경합과 성능 저하 요인이 됩니다.
- Cilium full replacement는 NKS에서 실패로 종결했습니다.
- 후속 비교 우선순위는 `Calico 유지 + Cilium chaining + Hubble`, `Calico VXLAN 유지 + Pixie/Beyla`, `Calico-eBPF 기반 sidecar-less 관측`으로 재설계합니다.
- Hubble 모니터링은 PoC 단계의 UI/CLI와 운영형 metrics/exporter를 구분해서 검증합니다.

## 포함 범위

- 현재 NKS 환경 제약 확인
- Calico 유지 + Cilium chaining 설치 방식 선정
- Hubble Relay/UI/CLI 구성 방식 선정
- Hubble metrics, Grafana, exporter 기반 운영형 모니터링 후보 정리
- Pixie, Grafana Beyla, Calico-eBPF 후보 비교
- 샘플 워크로드 기반 검증 항목과 롤백 포인트 정의

## 제외 범위

- 운영 전환 일정 확정
- 전 클러스터 일괄 확산
- HAProxy, Tomcat 등 실제 업무 애플리케이션 연동
- 서비스메쉬 전체 대체 아키텍처 확정
- 장기 운영 조직/프로세스 설계

## 예상 산출물

- 환경 인벤토리 문서
- 실행 계획 문서
- 샘플 워크로드 및 관측 시나리오 문서
- 검증 체크리스트
- 필요 시 Helm values 또는 Kubernetes manifest

## 성공 기준

- 대상 클러스터에서 Cilium 적용 검토에 필요한 공식 근거를 확보
- 단일 NKS 클러스터에서 Calico 유지 + Cilium chaining + Hubble Relay/UI/CLI가 정상 기동
- 샘플 워크로드 기준 east-west, DNS, Service 흐름을 Hubble에서 확인
- 샘플 워크로드 기준 CPU/메모리 사용량, 응답 지연, HPA scale-out 시 관측 가능한 병목을 수집
- sidecar 기반 구조 대비 운영 단순화 판단에 필요한 구성 요소 수, 설치 단계 수, 장애 지점을 정리
- 변경 실패 시 롤백 경로와 중단 기준이 명확화

## 측정 기준

1차 PoC는 실제 운영 서비스메쉬와 직접 성능 비교하지 않습니다. 대신 같은 샘플 워크로드를 기준으로 아래 항목을 수집해 후속 비교 기준선을 만듭니다.

- 기능: Calico 유지 + Cilium chaining + Hubble Relay/UI/CLI 정상 기동 여부
- 가시성: DNS, Pod-to-Pod, Service 흐름 식별 가능 여부
- 모니터링: Hubble UI/CLI 즉시 진단과 Hubble metrics/exporter 기반 운영형 확장 가능성
- 성능: 샘플 HTTP 요청의 p50/p95 지연, Pod/Node CPU 및 메모리 사용량
- 확장성: 샘플 HPA 적용 시 scale-out 이벤트와 Hubble 관측 가능 여부
- 운영 단순성: sidecar 없는 구성에서 필요한 컴포넌트 수, 설치 단계, 장애 확인 지점

## 오픈 이슈

- NKS 릴리스 노트 기준 CNI 변경 기능이 비지원 상태이므로 full replacement는 제외하고 chaining/관측 도구 중심으로 검증합니다.
- Pod CIDR과 Service CIDR의 공식 값은 NKS 콘솔 또는 생성 파라미터 확인이 필요합니다.
- Hubble Relay용 노드 간 `TCP 4244`, NKS LoadBalancer 동작은 실제 보안그룹/샘플 리소스로 검증해야 합니다.
- 기존 Pod 재시작 범위와 chaining 적용 범위는 PoC 결과 기준으로 별도 판단해야 합니다.
