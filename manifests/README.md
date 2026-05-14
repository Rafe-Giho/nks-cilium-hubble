# manifests

- 목적: Helm values, Kubernetes YAML, 샘플 설정의 저장 규칙을 관리합니다.
- 상태: active
- 마지막 갱신: 2026-05-14

실제 적용 자산을 관리합니다.

## 현재 자산

| 파일 | 용도 |
| --- | --- |
| `05-cni-configuration.yaml` | NKS Calico VXLAN etcd 기반 CNI 뒤에 `cilium-cni`를 붙이는 chaining ConfigMap |
| `05-cilium-chaining-values.yaml` | Cilium generic-veth chaining + Hubble Relay/UI Helm values |
| `06-cilium-hubble-metrics-values.yaml` | 05번 chaining 설정을 유지하면서 Cilium/Hubble metrics, ServiceMonitor, Grafana dashboard ConfigMap 활성화 |
| `06-cilium-values-hubble-exporter.yaml` | 06번 metrics values를 base로 Hubble exporter까지 추가한 values |
| `06-kube-prometheus-stack-values.yaml` | Cilium/Hubble ServiceMonitor와 dashboard ConfigMap을 수집하는 kube-prometheus-stack PoC values |
| `service-mesh/` | Cilium Gateway API/GAMMA 기반 Service Mesh PoC values와 샘플 manifest |

## 보존 자산

삭제한 Cilium full replacement 실험 자산은 `docs/91-full-replacement-assets-archive.md`에 요약 보존합니다.

## 저장 규칙

- Helm values, Kubernetes YAML, 샘플 설정만 둡니다.
- 비밀정보는 넣지 않습니다.
- 생성형 산출물은 가능하면 `artifacts/`에 두고, 재사용할 값만 이 디렉터리에 둡니다.
- 파일명은 목적이 드러나게 작성합니다.

## 후속 작성 대상

- Cilium chaining + Hubble 운영형 모니터링 values 보완
- Cilium Service Mesh primary CNI 환경용 샘플 values 보완
