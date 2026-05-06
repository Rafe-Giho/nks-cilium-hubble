# docs

- 목적: 프로젝트에서 직접 작성하고 유지보수하는 문서 목록과 작성 규칙을 관리합니다.
- 상태: draft
- 마지막 갱신: 2026-05-06

직접 작성하고 유지보수하는 프로젝트 문서를 관리합니다.

## 표준 문서

- `project-scope.md`: 목표, 범위, 성공 기준, 제외 범위
- `environment-inventory.md`: 현재 NKS 환경과 제약사항
- `execution-plan.md`: 단계별 수행 계획과 게이트
- `00-install-runbook.md`: Cilium, Hubble, UI, Ingress 설치/접속 가이드 인덱스
- `05-ingress-vs-gateway-api.md`: Cilium Ingress와 Gateway API 비교
- `06-nks-security-group.md`: NKS 기본 보안그룹과 Cilium 필요 포트 확인
- `10-full-replacement-execution-record.md`: NKS Cilium full replacement 실행 기록 아카이브
- `11-full-replacement-assets-archive.md`: NKS Cilium full replacement manifest/script 자산 아카이브
- `15-hubble-network-monitoring.md`: Hubble UI/CLI, Grafana metrics, exporter 기반 모니터링 가이드
- `16-observability-architecture-concepts.md`: Calico chaining, Cilium, Hubble, Grafana, exporter 개념과 연계 구조
- `30-track-c-calico-chaining-hubble.md`: Calico 유지 + Cilium chaining + Hubble 실행 가이드
- `40-option-pixie.md`: Pixie 기반 sidecar-less 관측 가이드
- `50-option-grafana-beyla.md`: Grafana Beyla 기반 sidecar-less 관측 가이드
- `poc-scenarios.md`: 샘플 워크로드와 관측 시나리오
- `decision-log.md`: 주요 판단과 근거
- `validation-checklist.md`: 변경 전후 확인 항목

## 문서 작성 규칙

- 문서는 한국어 기준으로 작성합니다.
- 각 문서 첫머리에 `목적`, `상태`, `마지막 갱신`을 둡니다.
- 근거가 필요한 내용은 `reference/` 문서와 연결합니다.
- 실행 절차 문서는 명령 예시와 기대 결과를 함께 적습니다.
- 아직 모르는 값은 비워두지 말고 `[입력 필요]`로 표시합니다.

## 운영 규칙

- 설계 변경이나 방향 전환이 생기면 `decision-log.md`를 먼저 갱신합니다.
- 환경 사실이 바뀌면 `environment-inventory.md`를 우선 수정합니다.
- 실제 적용 절차를 만들기 전 `execution-plan.md`와 `validation-checklist.md`를 먼저 채웁니다.
- 번호형 실행 가이드가 추가/변경되면 `00-install-runbook.md`와 이 README를 함께 갱신합니다.
- Hubble 모니터링 범위가 바뀌면 `15-hubble-network-monitoring.md`, `validation-checklist.md`, `reference/source-tracking.md`를 함께 맞춥니다.
