# 실행 계획

- 목적: 조사, 설계, 검증, 적용의 순서를 고정합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

## 단계

| 단계 | 목표 | 완료 조건 |
| --- | --- | --- |
| 1. 지원 범위 확인 | NKS에서 Cilium 적용 가능 여부와 비지원 리스크 확인 | 공식 근거 확보 |
| 2. 환경 인벤토리 확정 | 대상 단일 클러스터 상세 정보와 가정 정리 | `environment-inventory.md` 갱신 |
| 3. 구성 초안 설계 | Cilium, Ingress, Hubble Relay/UI/CLI 옵션 선정 | `00`~`50` 번호형 실행 가이드 정리 |
| 4. 샘플 시나리오 설계 | 실제 앱 없이 검증 가능한 워크로드/트래픽/측정 기준 정의 | `poc-scenarios.md`와 `validation-checklist.md` 갱신 |
| 5. PoC 수행 | 사용자가 클러스터 생성과 설치 수행 | 상태/흐름/Ingress 결과 확보 |
| 6. 결과 검토 | 기능 범위, 성능/운영 지표, 리스크, 후속 비교안 정리 | 검토 문서와 의사결정 기록 완료 |

## 현재 즉시 할 일

1. `docs/10-track-a-full-replacement-ingress.md`의 중단 조건을 확인합니다.
2. 새 Cilium Pod CIDR과 Hubble Relay `TCP 4244` 보안그룹을 확정합니다.
3. Track A 절차에 따라 Cilium secondary mode부터 node-by-node migration을 수행합니다.
4. 샘플 Ingress와 Hubble UI/CLI 검증 결과를 `artifacts/`에 남깁니다.

## 게이트 조건

- 1단계 근거가 없으면 설치안 확정으로 넘어가지 않습니다.
- 환경 인벤토리와 번호형 실행 가이드가 비어 있으면 PoC 수행으로 넘어가지 않습니다.
- 검증 기준과 롤백 포인트가 없으면 PoC 수행 단계로 넘어가지 않습니다.
- 성능/운영 단순화 측정 항목이 없으면 PoC 성공 판단으로 넘어가지 않습니다.
- 실제 앱이 없는 상태에서는 샘플 워크로드 검증을 최소 통과 조건으로 둡니다.
