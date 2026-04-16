# nks-cilium-hubble

NHN Cloud NKS에서 기본 CNI인 Calico 대신 Cilium을 검토하고, Cilium Ingress와 Hubble 기반 네트워크 가시화를 설계·검증하기 위한 PoC 작업 공간입니다.

## 현재 상태

- 단일 NKS 클러스터를 대상으로 한 PoC 범위와 문서 골격이 정리된 상태입니다.
- 실제 인프라 생성과 명령 실행은 사용자가 수행하고, 이 저장소는 실행 방안, 검토, 검증 기준, 문서화를 담당합니다.
- 실제 업무 애플리케이션은 없으므로 샘플 워크로드 기반 가시성 검증을 전제로 합니다.

## 현재 PoC 범위

- 대상: 판교 리전 NKS 단일 클러스터
- 시작점: 클러스터 생성 직후 기본 Calico 상태
- 목표: Cilium CNI 전환, Cilium Ingress 활성화, Hubble Relay/UI/CLI 구성
- 목적: 서비스메쉬 기반 sidecar 오버헤드 대안으로서 네트워크 가시성과 운영 단순화 가능성 확인
- 제약: NKS 릴리스 노트 기준 CNI 변경 기능은 비지원 상태이므로, 이번 작업은 `unsupported PoC`로 취급합니다.

## 먼저 볼 문서

- `docs/project-scope.md`
- `docs/environment-inventory.md`
- `docs/execution-plan.md`
- `docs/poc-scenarios.md`
- `reference/official-findings.md`
- `reference/source-tracking.md`

## 디렉터리

- `docs/`: 범위, 계획, 결정, 검증 문서
- `reference/`: 외부 근거와 제약사항 정리
- `manifests/`: 실제 적용 YAML/values
- `scripts/`: 반복 작업 자동화
- `artifacts/`: 실행 결과와 캡처

## 다음 단계

1. 노드 수와 flavor를 확정합니다.
2. 공식 근거를 기준으로 설치/전환 초안을 구체화합니다.
3. 샘플 워크로드와 Hubble/Ingress 검증 절차를 manifests 수준으로 내립니다.
