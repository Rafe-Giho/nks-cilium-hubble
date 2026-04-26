# manifests

- 목적: Helm values, Kubernetes YAML, 샘플 설정의 저장 규칙을 관리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

실제 적용 자산을 관리합니다.

## 저장 규칙

- Helm values, Kubernetes YAML, 샘플 설정만 둡니다.
- 비밀정보는 넣지 않습니다.
- 생성형 산출물은 가능하면 `artifacts/`에 두고, 재사용할 값만 이 디렉터리에 둡니다.
- 파일명은 목적이 드러나게 작성합니다. 예: `cilium-values.yaml`

## 추가 시점

- 지원 범위와 환경 인벤토리가 확인된 뒤 추가합니다.
