# Grafana Beyla 가이드

- 목적: Cilium 외 대안으로서 Grafana Beyla를 사용해 sidecar-less 트래픽/애플리케이션 관측을 수행하는 절차를 정리합니다.
- 상태: draft
- 마지막 갱신: 2026-04-17

## 왜 후보인가

- Beyla는 eBPF 기반 auto-instrumentation 도구입니다.
- 애플리케이션 코드 수정 없이 HTTP/S, gRPC 서비스의 RED metrics와 기본 trace spans를 수집할 수 있습니다.
- DaemonSet 기반으로 sidecar-less 운영이 가능합니다.

## 이 방식의 위치

- 현재 프로젝트의 `추가 후보 2`입니다.
- 이유는 sidecar-less라는 목표에는 잘 맞지만, Cilium처럼 CNI/Ingress/Gateway를 대체하지는 않기 때문입니다.

## 핵심 특징

- eBPF 기반 애플리케이션 auto-instrumentation
- Kubernetes DaemonSet 방식 지원
- OTLP 또는 Prometheus 기반 export 지원
- Kubernetes metadata decoration 지원

## 주의점

- Beyla는 네트워크/애플리케이션 관측 도구이지 CNI replacement가 아닙니다.
- Cilium Ingress, Gateway API 같은 north-south 제어 기능은 없습니다.
- Hubble와 정확히 같은 네트워크 flow UX를 제공하는 도구도 아닙니다.
- 목적은 `RED metrics + basic traces + 앱 트래픽 관측`에 더 가깝습니다.

## Cilium과 함께 쓸 때 주의점

- Beyla와 Cilium은 둘 다 traffic control 계열 eBPF 프로그램을 사용합니다.
- 최신 커널에서 TCX를 쓰면 충돌 가능성이 낮습니다.
- TCX를 못 쓰는 환경에서는 Cilium의 `bpf-filter-priority` 조정이 필요할 수 있습니다.

## 기본 설치 방식

공식 문서 기준으로 Kubernetes에서는 `DaemonSet with Helm`이 가장 실용적입니다.

## 1. Helm 저장소 추가

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

## 2. 기본 배포

```bash
helm install beyla -n beyla --create-namespace grafana/beyla
```

공식 기본값은 아래 성격을 가집니다.

- `beyla` namespace에 DaemonSet 배포
- Prometheus metrics를 Pod `9090` 포트 `/metrics`로 노출
- 클러스터의 애플리케이션을 폭넓게 instrument 시도
- Kubernetes metadata decoration 활성

## 3. 범위를 줄인 커스텀 설정 예시

```yaml
config:
  data:
    discovery:
      instrument:
        - k8s_namespace: demo
    routes:
      unmatched: heuristic
```

적용:

```bash
helm upgrade beyla grafana/beyla -n beyla -f helm-beyla.yml
```

## 4. OTLP export 예시

```yaml
env:
  OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector.monitoring:4318"
```

만약 헤더가 필요하면 Secret로 전달합니다.

## 5. 상태 확인

```bash
kubectl get pods -n beyla
kubectl logs -n beyla -l app.kubernetes.io/name=beyla --tail=100
```

## 6. 수집 결과 확인

이후 결과는 연동 대상에 따라 다릅니다.

- Prometheus scrape
- Grafana dashboard
- OTLP collector
- trace backend

## 7. 성능/오버헤드 관점

공식 문서는 Beyla가 minimal impact를 목표로 한다고 설명하며, 기본 시나리오 기준 예시 수치를 제공합니다.
- 기본 계측 예시에서 메모리 약 `75MB`
- 기본 계측 예시에서 CPU 약 `0.5%`
- network 기능 활성 시 메모리와 CPU가 더 증가

이 수치는 공식 데모 기준 참고값이지, 현재 NKS 환경에서 그대로 보장되는 값은 아닙니다.

## 이 방식의 장점

- sidecar-less 애플리케이션 관측에 바로 맞습니다.
- 코드 변경 없이 RED metrics와 traces를 수집할 수 있습니다.
- Grafana/OTel 생태계와 연결이 쉽습니다.

## 이 방식의 한계

- Cilium/Hubble처럼 네트워크 중심 UX와는 다릅니다.
- CNI replacement나 ingress 제어를 제공하지 않습니다.
- Cilium과 함께 쓸 경우 eBPF attachment 방식 호환성을 봐야 합니다.

## 추천 사용 시점

- 애플리케이션 성능 관측이 우선일 때
- RED metrics와 traces가 필요할 때
- Grafana/OTel 파이프라인과 쉽게 붙이고 싶을 때

## 참고 출처

- https://grafana.com/docs/beyla/latest/
- https://grafana.com/docs/beyla/latest/setup/kubernetes-helm/
- https://grafana.com/docs/beyla/latest/setup/kubernetes/
- https://grafana.com/docs/beyla/latest/cilium-compatibility/
- https://grafana.com/docs/beyla/latest/performance/
