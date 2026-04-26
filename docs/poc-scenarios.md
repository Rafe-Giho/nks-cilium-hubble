# PoC 시나리오

- 목적: 실제 업무 애플리케이션 없이 Cilium, Ingress, Hubble 가시성을 검증할 최소 시나리오를 정의합니다.
- 상태: draft
- 마지막 갱신: 2026-04-22

## 기본 원칙

- 실제 HAProxy, Tomcat, 서비스메쉬 연동은 이번 PoC 범위에서 제외합니다.
- 샘플 HTTP 워크로드와 테스트 Pod로 east-west, north-south, DNS 흐름을 관측합니다.
- Hubble UI/CLI에서 바로 확인 가능한 흐름을 우선 검증합니다.

## 권장 샘플 구성

- namespace `demo-a`: HTTP echo backend 1개
- namespace `demo-b`: HTTP echo backend 1개
- namespace `tools`: `curl` 또는 `netshoot` 계열 테스트 Pod
- Cilium Ingress: `demo-a` backend를 외부에 노출하는 단일 Ingress

## 최소 검증 시나리오

1. `tools` Pod에서 `demo-a` Service 호출
   기대 결과: Hubble에서 Pod-to-Service/Pod-to-Pod 흐름 확인
2. `tools` Pod에서 `demo-b` namespace backend 호출
   기대 결과: cross namespace 흐름 확인
3. `tools` Pod에서 DNS 질의 수행
   기대 결과: CoreDNS 관련 flow 확인
4. 외부에서 Cilium Ingress 경유로 `demo-a` 접근
   기대 결과: north-south 요청과 backend 연결 흐름 확인
5. 선택 사항으로 간단한 NetworkPolicy 추가
   기대 결과: 허용/차단 변화가 Hubble에 반영

## 성능/운영 관측 항목

- 샘플 HTTP 요청 p50/p95 지연
- 샘플 backend Pod CPU/메모리 사용량
- Cilium/Hubble/Ingress 관련 Pod CPU/메모리 사용량
- Node CPU/메모리 사용량
- HPA 적용 시 scale-out 이벤트와 Hubble flow 변화
- 설치/점검 단계 수와 장애 확인 지점 수
- 운영형 모니터링 선택 시 Hubble metrics scrape 가능 여부와 exporter 로그 생성 여부

## 후속 비교 시나리오

- 서비스메쉬 도입 전후 리소스 사용량 비교
- Cilium Ingress와 기존 Ingress Controller 비교
- Hubble 가시성과 Jaeger/OTel/Beyla 조합 비교
