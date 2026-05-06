# nks-cilium-hubble

- 목적: NHN Cloud NKS Cilium/Hubble PoC 문서와 실행 자산의 진입점을 제공합니다.
- 상태: draft
- 마지막 갱신: 2026-05-06

NHN Cloud NKS에서 sidecar-less 트래픽 관측 방안을 검토하고, Calico 유지 기반 Cilium chaining + Hubble 구성을 설계·검증하기 위한 PoC 작업 공간입니다.

## 현재 상태

- `Cilium full replacement`는 실패로 종결했고, 현재 실행 경로는 `Calico 유지 + Cilium generic-veth chaining + Hubble`입니다.
- 단일 NKS 클러스터를 대상으로 한 PoC 범위와 번호형 실행 가이드가 정리된 상태입니다.
- 실제 인프라 생성과 명령 실행은 사용자가 수행하고, 이 저장소는 실행 방안, 검토, 검증 기준, 문서화를 담당합니다.
- 실제 업무 애플리케이션은 없으므로 샘플 워크로드 기반 가시성 검증을 전제로 합니다.

## 현재 PoC 범위

- 대상: 판교 리전 NKS 단일 클러스터
- 시작점: 클러스터 생성 직후 기본 Calico 상태
- 목표: Calico 유지 + Cilium chaining + Hubble Relay/UI/CLI, Grafana metrics, exporter 후보 검증
- 목적: 서비스메쉬 기반 sidecar 오버헤드 대안으로서 네트워크 가시성과 운영 단순화 가능성 확인
- 제약: NKS 릴리스 노트 기준 CNI 변경 기능은 비지원 상태이므로, full replacement는 운영 후보에서 제외합니다.

## 먼저 볼 문서

- `docs/00-install-runbook.md`
- `docs/01-observability-architecture-concepts.md`
- `docs/project-scope.md`
- `docs/03-environment-inventory.md`
- `docs/execution-plan.md`
- `docs/04-nks-security-group.md`
- `docs/05-calico-chaining-hubble-runbook.md`
- `docs/06-hubble-observability-runbook.md`
- `docs/validation-checklist.md`
- `reference/official-findings.md`
- `reference/source-tracking.md`

## 디렉터리

- `docs/`: 범위, 계획, 결정, 검증 문서
- `reference/`: 외부 근거와 제약사항 정리
- `manifests/`: 실제 적용 YAML/values
- `scripts/`: 반복 작업 자동화
- `artifacts/`: 실행 결과와 캡처

## 다음 단계

1. `docs/01-observability-architecture-concepts.md`에서 현재 구조와 도구 역할을 확인합니다.
2. `docs/03-environment-inventory.md`와 `docs/04-nks-security-group.md`에서 기준선을 확인합니다.
3. `docs/05-calico-chaining-hubble-runbook.md` 기준으로 Cilium chaining + Hubble을 검증합니다.
4. Hubble 운영형 모니터링은 `docs/06-hubble-observability-runbook.md`에서 별도 선택합니다.
5. 실행 결과와 실패 로그는 `artifacts/`에 남깁니다.
