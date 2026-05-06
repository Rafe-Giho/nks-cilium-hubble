# Repository AGENTS

- 목적: 이 저장소에서 Codex가 따라야 할 작업 원칙과 문서 관리 규칙을 정의합니다.
- 상태: active
- 마지막 갱신: 2026-05-06

이 저장소의 목적은 NHN Cloud NKS에서 sidecar-less 트래픽 관측 방안을 검토하고, Calico 유지 기반 Cilium chaining + Hubble 구성을 설계, 검증, 문서화하는 것입니다.

## 기본 원칙

- 응답과 문서는 기본적으로 한국어로 작성합니다.
- 먼저 관련 파일과 현재 구조를 확인한 뒤 작업합니다.
- 변경은 항상 최소 범위로 수행합니다.
- 관련 없는 리팩터링은 하지 않습니다.
- 설정값, 버전, 지원 범위, 제약사항은 추정하지 말고 `reference/`에 근거를 남깁니다.
- 클러스터 변경 전에 문서와 검증 기준부터 고정합니다.

## 역할 분담

- 사용자가 실제 클러스터 생성, `kubectl` 실행, 설치/변경 작업을 수행합니다.
- 이 저장소에서는 실행 방안, manifest 초안, 검토 의견, 검증 기준, 결과 문서화를 관리합니다.
- 문서에는 사용자가 직접 수행할 명령과 기대 결과를 명확히 남깁니다.
- 클러스터에 대한 직접 수행은 원칙적으로 하지 않고, 필요한 경우에도 read-only 정보 수집만 수행합니다.
- 클러스터 변경 명령은 사용자가 직접 실행하며, 이 저장소에서는 변경 명령 제안과 결과 검토만 담당합니다.

## 권장 작업 순서

1. `reference/`에 공식 근거와 버전 제약을 정리합니다.
2. `docs/`에 범위, 환경 정보, 실행 계획, 검증 기준을 정리합니다.
3. `docs/00-install-runbook.md`와 번호형 문서에 실행 순서를 정리합니다.
4. `manifests/`와 `scripts/`에 실제 적용 자산을 추가합니다.
5. 실행 결과와 캡처는 `artifacts/`에 남깁니다.

## 디렉터리 규칙

- `docs/`: 범위, 계획, 결정, 검증, 런북 등 팀이 관리하는 문서
- `reference/`: 공식 문서 요약, 버전 호환성, 제약사항, 외부 근거
- `manifests/`: Kubernetes YAML, Helm values, 샘플 설정
- `scripts/`: 반복 작업 자동화 스크립트
- `artifacts/`: 실행 로그, 캡처, 다이어그램, 임시 산출물

## 필수 문서

실제 클러스터 변경 전에 아래 문서는 초안 이상이어야 합니다.

- `docs/project-scope.md`
- `docs/03-environment-inventory.md`
- `docs/execution-plan.md`
- `docs/poc-scenarios.md`
- `docs/validation-checklist.md`
- `docs/00-install-runbook.md`
- `docs/01-observability-architecture-concepts.md`
- `docs/04-nks-security-group.md`
- `docs/05-calico-chaining-hubble-runbook.md`
- `docs/06-hubble-observability-runbook.md`

## 문서 규칙

- 새 문서를 추가하면 해당 디렉터리의 `README.md`에 목적과 위치를 반영합니다.
- 각 문서 첫머리에 `목적`, `상태`, `마지막 갱신`을 둡니다.
- 미확인 항목은 `[입력 필요]`로 남깁니다.
- `reference/` 문서에는 최소한 출처, 확인 날짜, 핵심 요약을 남깁니다.
- 주요 판단은 `docs/decision-log.md`에 기록합니다.

## 안전 규칙

- 실행 가능한 변경 전에 전제 조건, 영향 범위, 롤백 포인트를 먼저 정리합니다.
- 비밀정보, kubeconfig, 실제 토큰, 외부 접속 정보는 저장소에 넣지 않습니다.
- 검증은 가장 작은 관련 단위만 수행하고, 미검증이면 명확히 표시합니다.
- 지원 여부가 불명확한 상태에서 설치 명령이나 마이그레이션 절차를 확정하지 않습니다.

## 현재 우선순위

1. NKS 기본 Calico VXLAN 환경 제약과 Cilium chaining 적용 조건 정리
2. 단일 클러스터 PoC 환경 인벤토리 확정
3. `Calico 유지 + Cilium generic-veth chaining + Hubble` 실행 가이드 정합성 유지
4. Hubble UI/CLI, Grafana metrics, exporter 모니터링 선택 기준 정리
5. Pixie, Beyla, Calico-eBPF 등 후속 sidecar-less 관측 후보 비교
