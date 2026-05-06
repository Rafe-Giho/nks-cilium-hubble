# 실행 계획

- 목적: 조사, 설계, 검증, 적용의 순서를 고정합니다.
- 상태: draft
- 마지막 갱신: 2026-05-06

## 단계

| 단계 | 목표 | 완료 조건 |
| --- | --- | --- |
| 1. 지원 범위 확인 | NKS에서 Cilium 적용 가능 여부와 비지원 리스크 확인 | 공식 근거 확보 |
| 2. 환경 인벤토리 확정 | 대상 단일 클러스터 상세 정보와 가정 정리 | `03-environment-inventory.md` 갱신 |
| 3. 구성 초안 설계 | Calico chaining, Hubble Relay/UI/CLI, Hubble metrics/exporter 옵션 선정 | `00`~`50` 번호형 실행 가이드 정리 |
| 4. 샘플 시나리오 설계 | 실제 앱 없이 검증 가능한 워크로드/트래픽/측정 기준 정의 | `poc-scenarios.md`와 `validation-checklist.md` 갱신 |
| 5. PoC 수행 | 사용자가 클러스터 생성과 설치 수행 | 상태/흐름/Hubble 관측 결과 확보 |
| 6. 결과 검토 | 기능 범위, 성능/운영 지표, 리스크, 후속 비교안 정리 | 검토 문서와 의사결정 기록 완료 |

## 현재 즉시 할 일

1. `docs/05-calico-chaining-hubble-runbook.md` 기준으로 Cilium chaining + Hubble PoC를 수행합니다.
2. 설치 후 `docs/validation-checklist.md`에 결과를 기록합니다.
3. Hubble UI/CLI 확인 후 운영형 metrics/exporter 필요성을 `docs/06-hubble-observability-runbook.md` 기준으로 판단합니다.
4. Calico VXLAN 유지 + Pixie/Beyla 관측안을 비교합니다.
5. 신규 NKS 클러스터의 Calico-eBPF 기반 sidecar-less 관측안을 별도 설계합니다.

## 게이트 조건

- 1단계 근거가 없으면 설치안 확정으로 넘어가지 않습니다.
- 환경 인벤토리와 번호형 실행 가이드가 비어 있으면 PoC 수행으로 넘어가지 않습니다.
- 검증 기준과 롤백 포인트가 없으면 PoC 수행 단계로 넘어가지 않습니다.
- 성능/운영 단순화 측정 항목이 없으면 PoC 성공 판단으로 넘어가지 않습니다.
- Hubble 상시 모니터링을 목표에 포함하면 UI/CLI만으로 성공 처리하지 않고 metrics/exporter 선택 여부를 판단합니다.
- 실제 앱이 없는 상태에서는 샘플 워크로드 검증을 최소 통과 조건으로 둡니다.
