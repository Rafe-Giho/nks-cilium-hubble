# manifests

- 목적: Helm values, Kubernetes YAML, 샘플 설정의 저장 규칙을 관리합니다.
- 상태: active
- 마지막 갱신: 2026-05-06

실제 적용 자산을 관리합니다.

## 현재 자산

| 파일 | 용도 |
| --- | --- |
| `15-cilium-hubble-metrics-values.yaml` | Track C chaining 설정을 유지하면서 Cilium/Hubble metrics, ServiceMonitor, Grafana dashboard ConfigMap 활성화 |
| `15-kube-prometheus-stack-values.yaml` | Cilium/Hubble ServiceMonitor와 dashboard ConfigMap을 수집하는 kube-prometheus-stack PoC values |
| `30-cni-configuration.yaml` | NKS Calico VXLAN etcd 기반 CNI 뒤에 `cilium-cni`를 붙이는 chaining ConfigMap |
| `30-cilium-chaining-values.yaml` | Cilium generic-veth chaining + Hubble Relay/UI Helm values |

## 보존 자산

삭제한 Cilium full replacement 실험 자산은 `docs/11-full-replacement-assets-archive.md`에 요약 보존합니다.

## 저장 규칙

- Helm values, Kubernetes YAML, 샘플 설정만 둡니다.
- 비밀정보는 넣지 않습니다.
- 생성형 산출물은 가능하면 `artifacts/`에 두고, 재사용할 값만 이 디렉터리에 둡니다.
- 파일명은 목적이 드러나게 작성합니다.

## 후속 작성 대상

- Pixie 검토용 manifest 또는 설치 메모
- Grafana Beyla DaemonSet values
- Calico-eBPF 검토용 별도 values 또는 체크리스트
