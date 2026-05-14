# 프로젝트 범위

- 목적: NKS에서 Cilium chaining + Hubble 관측과 Cilium Service Mesh 검증 범위, 성공 기준을 고정합니다.
- 상태: draft
- 마지막 갱신: 2026-05-14

## 목표

- NHN Cloud NKS에서 Calico 유지 기반 Cilium chaining + Hubble 적용 가능 여부를 검증합니다.
- Cilium Service Mesh를 현재 NKS Calico chaining 환경에서 구성할 수 있는지 검증하고, 실패 시 근거를 남깁니다.
- Cilium primary CNI 환경에서의 Service Mesh 표준 구성 가이드를 분리합니다.
- 적용 전후 검증 및 롤백 기준을 문서화합니다.

## 배경

- Cilium full replacement는 NKS에서 실패로 종결했습니다.
- 현재 검증 우선순위는 `Calico 유지 + Cilium chaining + Hubble`과 `Cilium Service Mesh 가능 범위`입니다.
- Hubble 모니터링은 PoC 단계의 UI/CLI와 운영형 metrics/exporter를 구분해서 검증합니다.
- Cilium Service Mesh는 현재 NKS Calico chaining 구성에서 운영 수준 불가로 판정했고, 표준 실행 절차는 Cilium primary 환경 기준으로 관리합니다.

## 포함 범위

- 현재 NKS 환경 제약 확인
- Calico 유지 + Cilium chaining 설치 방식 선정
- Hubble Relay/UI/CLI 구성 방식 선정
- Hubble metrics, Grafana, exporter 기반 운영형 모니터링 후보 정리
- Cilium Service Mesh 개념, 구축 조건, 실패 결과, 표준 primary CNI 가이드 정리
- 샘플 워크로드 기반 검증 항목과 롤백 포인트 정의

## 제외 범위

- 운영 전환 일정 확정
- 전 클러스터 일괄 확산
- HAProxy, Tomcat 등 실제 업무 애플리케이션 연동
- Cilium 외 관측 도구 비교
- Cilium 외 서비스메쉬 도입 확정
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
- 현재 NKS Calico chaining 환경에서 Cilium Service Mesh 가능/불가를 실제 검증 결과로 판정
- Cilium primary 기반 Service Mesh 표준 구축 절차와 현재 NKS 적용 불가 사유를 분리
- 샘플 워크로드 기준 CPU/메모리 사용량, 응답 지연, HPA scale-out 시 관측 가능한 병목을 수집
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
