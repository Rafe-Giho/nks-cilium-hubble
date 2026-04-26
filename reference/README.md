# reference

- 목적: 공식 문서 근거, 제약사항, 버전 호환성 메모의 위치와 기록 규칙을 관리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

외부 자료를 기반으로 한 요약, 제약사항, 버전 호환성 메모를 관리합니다.

## 우선 관리 문서

- `source-tracking.md`: 어떤 출처를 확인했는지 추적
- `official-findings.md`: 현재 PoC에 직접 쓰는 공식 근거 요약
- 향후 주제별 문서 예시:
  - `nks-cni-support.md`
  - `cilium-version-matrix.md`
  - `hubble-requirements.md`
  - `migration-caveats.md`

## 기록 규칙

- 각 문서에 출처 URL 또는 문서명, 확인 날짜를 남깁니다.
- 원문 복붙보다 프로젝트 관점의 요약을 우선합니다.
- 변경 가능성이 큰 정보는 버전과 확인 시점을 분리해서 적습니다.
- 의사결정에 직접 쓰인 근거는 별도 섹션으로 표시합니다.
- 지원 여부가 불명확하면 단정하지 말고 `미확인`으로 표시합니다.

## 우선 수집 항목

- NKS에서 기본 CNI 대체 가능 여부와 지원 범위
- NKS Kubernetes 버전과 Cilium 지원 매트릭스
- Hubble Relay/UI/CLI 구성 요구사항
- Hubble metrics, Grafana, exporter 기반 운영형 모니터링 요구사항
- Cilium/Hubble/Calico node 간 필요 포트와 NKS worker 보안그룹 충족 여부
- NetworkPolicy, LoadBalancer, egress 관련 제약
- Calico 제거 또는 공존 시 주의사항
