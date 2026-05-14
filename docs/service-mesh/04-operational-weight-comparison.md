# Cilium Service Mesh와 Istio Ambient 운영 무게 비교

- 목적: Cilium Service Mesh와 Istio Ambient만을 대상으로 실제 사용량 근거, 공개 성능 자료, 운영 부담을 비교합니다.
- 상태: draft
- 마지막 갱신: 2026-05-14

## 결론

`resources.requests`는 실제 사용량이 아닙니다.
따라서 이 문서의 주 비교 기준은 request가 아니라 **실제 runtime 사용량을 확인할 수 있는 공개 자료와 실측 가능성**입니다.

현재 결론은 아래와 같습니다.

| 항목 | Cilium Service Mesh | Istio Ambient |
| --- | --- | --- |
| 현재 NKS에서 실측 가능 여부 | 불가. Calico chaining에서 Cilium Service Mesh datapath 실패 | 별도 PoC 전에는 불가 |
| 공식 runtime 사용량 자료 | Cilium agent/eBPF scalability 자료는 있음. Service Mesh/Envoy 전용 수치는 제한적 | ztunnel/waypoint 공식 runtime 수치 있음 |
| 독립 벤치마크 실제 사용량 | Cilium과 Istio Ambient 모두 mTLS benchmark 수치 있음. 단 실험 조건 한정 | 동일 |
| 운영 판단 | CNI/datapath 결합이 강하고 Cilium primary 전제 | 기존 CNI 위에 점진 도입 가능, mesh sizing 근거가 더 명확 |

정확한 해석은 이렇습니다.

- Cilium chart의 request가 비어 있다고 해서 사용량이 0인 것은 아닙니다.
- Cilium Service Mesh의 실제 사용량은 Cilium agent, Cilium Envoy, policy 수, Gateway/GAMMA route 수, L7 트래픽량에 따라 발생합니다.
- 현재 NKS 클러스터는 Cilium Service Mesh가 실패했기 때문에, 현재 Cilium/Hubble 사용량을 Cilium Service Mesh 사용량으로 쓰면 안 됩니다.
- Istio Ambient는 공식 문서가 ztunnel/waypoint runtime 사용량을 제시하므로 capacity planning 근거가 더 직접적입니다.
- Cilium Service Mesh의 운영 무게는 “request가 0이라 가볍다”가 아니라 “per-pod sidecar가 없고 per-node Cilium/Envoy로 처리하지만, CNI와 강하게 결합한다”로 설명해야 합니다.

## 비교 범위

| 후보 | 기본 포함 범위 | 제외 범위 |
| --- | --- | --- |
| Cilium Service Mesh | Cilium primary CNI, kube-proxy replacement, Cilium Envoy, Gateway API/GAMMA/L7 Policy | 현재 NKS chaining 실측값, 별도 관측 add-on, 외부 Prometheus/Grafana/trace backend |
| Istio Ambient | Istio base, istiod, istio-cni, ztunnel, 필요 시 waypoint/ingress gateway | 별도 UI add-on, 외부 Prometheus/Grafana/trace backend |

## 실제 사용량 근거

### Cilium 공식 자료

Cilium 공식 scalability report는 Service Mesh 전용 부하 테스트가 아니라 Cilium agent/eBPF datapath 대규모 테스트입니다.
그래도 Cilium runtime 사용량을 설명하는 공식 근거로는 사용할 수 있습니다.

| 항목 | 공식 수치 |
| --- | ---: |
| 테스트 worker | 1000대, worker당 `2 vCPU / 4GB` |
| 125 pods 생성 시 agent CPU peak | `6.8%` on 2 vCPU |
| 1000 node 준비 완료 후 agent CPU 평균 | `0.15%` on 2 vCPU |
| 50,000 pods 상태 agent CPU 평균 | `3.38%` on 2 vCPU |
| 50,000 pods 상태 agent CPU 순간 peak | `27.94%` on 2 vCPU |
| 50,000 pods agent memory average | `438MiB` |
| 50,000 pods agent memory max | `573MiB` |
| 50,000 pods eBPF memory max | `462.7MiB` |
| 250 CNP 생성 시 한 agent CPU | `100%` 15초 |
| 250 CNP 생성 시 평균 CPU peak | `40%` 90초 |

한계:

- Cilium Envoy/Gateway API/GAMMA/L7 Policy 전용 사용량이 아닙니다.
- Cilium Service Mesh 실제 비용은 Envoy 경유 HTTP/gRPC 트래픽량과 route/policy 수에 따라 별도 측정해야 합니다.
- 현재 NKS chaining 클러스터의 Cilium/Hubble 사용량은 Cilium Service Mesh 성공 환경이 아니므로 이 비교에 사용할 수 없습니다.

### Istio 공식 자료

Istio 공식 performance 문서는 data plane proxy 단위 runtime 사용량을 제공합니다.

| 항목 | 공식 조건 |
| --- | --- |
| version | Istio `1.24` performance summary |
| 요청 | HTTP `1000 rps` |
| payload | `1KB` |
| proxy worker | 2 |
| latency test | 5-node bare-metal, Flannel primary CNI, mTLS enabled |

| 구성요소 | CPU | Memory | 의미 |
| --- | ---: | ---: | --- |
| ztunnel 1개 | `0.06 vCPU` | `12MB` | ambient L4 secure overlay |
| waypoint 1개 | `0.25 vCPU` | `60MB` | ambient L7 처리 |
| sidecar proxy 1개 | `0.20 vCPU` | `60MB` | 참고용, 이 문서의 비교 대상 아님 |

해석:

- 2-node에서 ztunnel만 보면 공식 부하 조건 기준 runtime 사용량은 대략 `0.12 vCPU / 24MB`입니다.
- L7 waypoint 1개를 추가하면 data plane runtime 기준 `0.37 vCPU / 84MB` 수준입니다.
- 이 값은 runtime 사용량이며 chart request와 다릅니다.

### 독립 벤치마크

arXiv technical report는 Istio, Istio Ambient, Linkerd, Cilium을 mTLS 조건에서 비교했습니다.
공식 벤더 자료는 아니므로 절대값으로 운영 sizing을 확정하면 안 되지만, 실제 CPU/Memory 증가량 비교 근거로는 유용합니다.

테스트 조건:

| 항목 | 값 |
| --- | --- |
| 비교 대상 | Istio, Istio Ambient, Linkerd, Cilium |
| 부하 | `3200 RPS`, `1600 connections` |
| 통신 | intra-node, inter-node |
| 값 의미 | baseline 대비 증가량 |

intra-node 결과:

| 후보 | P99 latency 증가 | CPU 증가 | Memory 증가 |
| --- | ---: | ---: | ---: |
| Istio Ambient | `+0.02s` | client `+0.23`, server `+0.23` cores | client `+26MiB`, server `+26MiB` |
| Cilium | `+0.22s` | client `+0.12`, server `+0.08` cores | client `+95MiB`, server `+95MiB` |

inter-node 결과:

| 후보 | P99 latency 증가 | CPU 증가 | Memory 증가 |
| --- | ---: | ---: | ---: |
| Istio Ambient | `+0.03s` | client `+0.23`, server `+0.24` cores | client `+86MiB`, server `+97MiB` |
| Cilium | `+0.22s` | client `+0.10`, server `+0.03` cores | client `+93MiB`, server `+94MiB` |

이 벤치마크 기준 해석:

- Cilium은 CPU 증가량이 Istio Ambient보다 낮았습니다.
- Istio Ambient는 memory 증가량과 latency 증가량이 Cilium보다 낮았습니다.
- 이 결과는 특정 mTLS 테스트 조건의 결과이며, Cilium Gateway API/GAMMA 운영량 전체를 대표하지 않습니다.

## request 값의 위치

Helm chart request는 실제 사용량이 아니라 scheduler reservation입니다.
비교의 주 근거가 아니라 보조 정보로만 둡니다.

Cilium chart 기본 request가 비어 있다는 의미:

- 사용량이 0이라는 뜻이 아닙니다.
- chart가 기본 예약량을 강제하지 않는다는 뜻입니다.
- 운영에서는 실제 부하 테스트, 노드 여유율, route/policy 수, proxy 처리량을 기준으로 request/limit를 정해야 합니다.

Istio Ambient chart request가 크다는 의미:

- 실제로 항상 그만큼 사용한다는 뜻은 아닙니다.
- 그만큼 capacity를 예약한다는 뜻입니다.
- small cluster에서는 예약량 부담이 먼저 보이고, 큰 cluster에서는 waypoint/ztunnel 확장 모델을 따로 봐야 합니다.

따라서 이 문서는 CNI나 Service Mesh 컴포넌트를 request 합계로 직접 sizing하지 않습니다.
실제 sizing은 운영 후보 클러스터에서 `kubectl top`, Prometheus `container_cpu_usage_seconds_total`, `container_memory_working_set_bytes`, proxy별 처리량 지표로 측정합니다.

## 공개 사례와 근거 수준

### Cilium Service Mesh

| 사례 | 확인 내용 | 비교 의미 |
| --- | --- | --- |
| AWS EKS Open Source Blog | EKS에서 Cilium Service Mesh와 Ingress 구성을 데모로 설명. production code가 아니라 예시임을 명시 | 아키텍처 설명 근거. 운영 resource sizing 근거로는 제한적 |
| CNCF Zynga case study | AWS VPC CNI, kube-proxy, Istio service mesh, 별도 관측/보안 도구를 Cilium 기반 통합 스택으로 대체. Service Mesh 확장 계획 언급 | Cilium 통합 운영 사례. 공개 리소스 수치는 없음 |
| CNCF Ascend case study | Cilium을 production CNI로 사용하고 Gateway API/Cilium Service Mesh 검토 언급 | Cilium 운영 사례. Service Mesh 본운영 수치로는 부족 |
| CNCF ClickHouse case study | Cilium을 multi-cloud Kubernetes network/security 기반으로 사용 | Cilium 대규모 운영 근거. Service Mesh 성능 비교 근거는 아님 |

정리하면 Cilium은 production CNI와 통합 스택 사례는 충분하지만, **Cilium Service Mesh 자체의 공개 정량 리소스 사례는 제한적**입니다.

### Istio Ambient

| 사례 | 확인 내용 | 비교 의미 |
| --- | --- | --- |
| Istio Ambient GA announcement | Istio 1.24에서 ambient mode GA, ztunnel/waypoint/API Stable, broad production usage ready 명시 | 운영 도입 가능 상태 근거 |
| Istio Ambient GA user quotes | Quotech, EISST, Harri, Blip.pt 등 공개 사용자 의견과 production 사용 언급 | 실제 사용 사례 근거. 단, 상세 리소스 원자료는 제한적 |
| Istio performance docs | ztunnel/waypoint의 CPU/Memory 공식 runtime 수치 제공 | 리소스 산정 근거로 직접 사용 가능 |
| Istio Helm docs | ambient 설치 구성요소를 base, istiod, istio-cni, ztunnel, 선택 ingress gateway로 분리 | 운영 구성요소와 upgrade 범위 판단 근거 |

정리하면 Istio Ambient는 Cilium보다 mesh 전용 공개 runtime 수치가 더 명확합니다.
반면 기본 예약량이 크고, L7 사용 시 waypoint 설계와 용량 산정이 추가됩니다.

## 운영 무게 비교

점수는 `1`이 가볍고 `5`가 무겁습니다.
request 예약량이 아니라 실제 측정 가능성, 운영 결합도, 장애 분석 난이도 기준입니다.

| 항목 | Cilium Service Mesh | Istio Ambient | 판단 |
| --- | ---: | ---: | --- |
| 실제 사용량 근거 명확성 | 4 | 2 | Cilium은 Service Mesh/Envoy 전용 공개 수치가 부족하고, Istio는 ztunnel/waypoint 수치가 있음 |
| CNI 결합도 | 5 | 2 | Cilium Service Mesh는 Cilium primary/KPR 의존, Istio Ambient는 기존 CNI 위에 배치 가능 |
| L7 확장 운영 | 3 | 4 | Cilium은 per-node Envoy 중심, Istio는 waypoint 배치/스케일 정책 필요 |
| 장애 분석 난이도 | 4 | 4 | Cilium은 eBPF+Envoy datapath, Istio는 ztunnel+waypoint+xDS 경로를 봐야 함 |
| 기본 예약량 부담 | 2 | 4 | Cilium은 chart 기본 예약 없음, Istio는 2-node 기준 request가 큼 |
| 운영 기능 성숙도 설명 | 4 | 2 | Istio Ambient는 mesh 기능과 운영 문서가 더 직접적 |
| 기존 NKS Calico와 적합성 | 5 | 2 | 현재 검증상 Cilium chaining Service Mesh 실패, Istio Ambient는 별도 PoC 대상 |

총평:

| 후보 | 운영 무게 요약 |
| --- | --- |
| Cilium Service Mesh | per-pod sidecar 비용은 없지만, CNI/datapath 결합이 강하고 Cilium primary가 전제입니다. 공개된 Service Mesh 전용 실제 사용량 수치가 부족해 동일 조건 실측이 필요합니다. |
| Istio Ambient | ztunnel/waypoint 구조로 실제 사용량 근거가 더 명확합니다. 다만 기본 예약량과 waypoint 운영 설계 부담이 있고, L7을 넓게 켤수록 운영 요소가 늘어납니다. |

## 선택 기준

| 조건 | 권장 후보 |
| --- | --- |
| Cilium primary CNI를 표준으로 쓸 수 있고, 네트워크/보안/L7 라우팅을 Cilium 스택으로 통합하려는 경우 | Cilium Service Mesh |
| 기존 CNI를 유지해야 하고, sidecar 없이 mesh를 단계적으로 도입하려는 경우 | Istio Ambient |
| L4 mTLS와 L4 authorization부터 시작하고, L7은 필요한 namespace/service만 켜려는 경우 | Istio Ambient |
| 실제 runtime resource 근거를 고객에게 먼저 제시해야 하는 경우 | Istio Ambient가 유리 |
| CNI 교체나 kube-proxy replacement가 불가능한 관리형 Kubernetes 환경 | Istio Ambient가 유리 |

## 현재 프로젝트 적용 해석

현재 NKS 클러스터에서는 아래 결론을 분리해서 유지합니다.

| 항목 | 판정 |
| --- | --- |
| 현재 NKS 관측 트랙 | 가능 |
| 현재 NKS에서 Cilium Service Mesh | 실패 |
| 현재 NKS 실측 리소스를 Cilium Service Mesh 대표값으로 사용 | 불가 |
| Cilium Service Mesh 비교 | Cilium primary 가능한 별도 클러스터 기준 |
| Istio Ambient 비교 | 별도 PoC에서 실측 가능 |

즉 이 문서는 현재 클러스터 결과 보고서가 아니라, **후속 Service Mesh 후보 선정용 비교 문서**입니다.

## 실측 비교 계획

정량 비교를 확정하려면 아래 조건을 맞춰 별도 측정해야 합니다.

| 단계 | Cilium Service Mesh | Istio Ambient |
| --- | --- | --- |
| 클러스터 | Cilium primary CNI 가능 클러스터 | 동일 스펙의 기존 CNI 클러스터 |
| 기본 구성 | Cilium KPR, Envoy, Gateway API/GAMMA | istiod, istio-cni, ztunnel |
| L7 구성 | GAMMA 또는 L7 Policy | waypoint |
| north-south | Cilium Gateway API | Istio ingress gateway/Gateway API |
| 샘플 앱 | 동일 web/was/backend | 동일 web/was/backend |
| 부하 도구 | `fortio`, `k6`, `wrk` 중 하나로 통일 | 동일 |
| 측정값 | 실제 CPU, 실제 memory, p50/p95/p99, RPS, error rate, proxy metric | 동일 |
| 관측 backend | Prometheus/Grafana/trace backend 동일 조건 | 동일 |

## 참고 근거

- Cilium Service Mesh: <https://docs.cilium.io/en/stable/network/servicemesh/>
- Cilium Gateway API: <https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/>
- Cilium GAMMA: <https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gamma/>
- Cilium L7 Traffic Management: <https://docs.cilium.io/en/stable/network/servicemesh/l7-traffic-management/>
- Cilium scalability report: <https://docs.cilium.io/en/stable/operations/performance/scalability/report/>
- AWS Open Source Blog - Cilium Service Mesh on EKS: <https://aws.amazon.com/blogs/opensource/getting-started-with-cilium-service-mesh-on-amazon-eks/>
- CNCF Zynga case study: <https://www.cncf.io/case-studies/zynga/>
- CNCF Ascend case study: <https://www.cncf.io/case-studies/ascend/>
- CNCF ClickHouse case study: <https://www.cncf.io/case-studies/clickhouse/>
- Istio Ambient overview: <https://istio.io/latest/docs/ambient/overview/>
- Istio Ambient Helm install: <https://istio.io/latest/docs/ambient/install/helm/>
- Istio performance and scalability: <https://istio.io/latest/docs/ops/deployment/performance-and-scalability/>
- Istio Ambient GA: <https://istio.io/latest/blog/2024/ambient-reaches-ga/>
- Service Mesh mTLS technical report: <https://arxiv.org/abs/2411.02267>
