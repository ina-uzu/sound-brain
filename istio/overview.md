## Istio

- 서비스간의 트래픽이나 API 호출을 컨트롤
- 통신사이의 트래픽을 암호화하고 인증과 권한제어가 가능함.
- 트래픽 정책이나 자원 제어
- 서비스들에 대한 tracing, monitoring, logging을 자동으로 수행
- CRD를 사용해 사이드카 패턴으로 서비스(앱)마다 Envoy Proxy를 같이 띄운다

---

## Architecture

실제 트래픽이 처리되는 Data Plane 부분과, 얘를 컨트롤 해주는 Control Plane으로 나눌 수 있다

### Data plane (Envoy proxy)

- 프록시들로 이루어져 실제로 트래픽을 받아 처리 해주는 부분
- control plane에 의해 컨트롤됨
- collect and report telemetry on all mesh traffic

**Envoy**

- Dynamic service discovery
- Load balancing
- TLS termination
- HTTP/2 and gRPC proxies
- Circuit breakers
- Health checks
- Staged rollouts with %-based traffic split
- Fault injection
- Rich metrics

**Why sidecar?**

This sidecar deployment allows Istio to enforce policy decisions and extract rich telemetry which can be sent to monitoring systems to provide information about the behavior of the entire mesh.

The sidecar proxy model also allows you to add Istio capabilities to an existing deployment without requiring you to rearchitect or rewrite code.

### Control Plane (istiod)

- 프록시들에 설정값 전달하고 관리해주는 부분 → CRD를 통해 관리한다
- Pilot, Citadel, Galley, Mixer 등으로 구성되어 있다.
- 1.5 버전 이전에는 각각의 컴포넌트로 떠있었는데, 이제 istiod에 통합되어서 뜬다.

**Pilot**

- envoy 설정 관리 & 서비스 디스커버리 기능 담당
- 서비스 트래픽 retry
- Circuit Breaker
- Timeout
1. 새로운 서비스가 시작되고 pilot의 `platform adapter`에게 그 사실을 알림
2. `platform adapter`는 서비스 인스턴스를 `Abstract model`에 등록
3. **Pilot**은 트래픽 규칙과 구성을 `Envoy Proxy`에 배포

**Mixer**

- 트래픽 정책 통제 (예를 들어, Header값의 User-Agent를 보고 트래픽을 v1으로 보낼지, v2로 보낼지 결정한다거나, [v1 으로는 90% 트래픽을 보내고 v2로는 10%의 트래픽을 보낸다거나..](https://www.notion.so/c11c784c6918496ea4426deeb6011bb3))
- 모니터링 지표 수집
- Service Mesh 전체에서 액세스 제어 및 정책 관리
- 각종 모니터링 지표 수집
- 플랫폼 독립적 → Istio가 다양한 호스트환경 & 백엔드와 인터페이스할 수 있는 이유

**Citadel**

- 보안 기능 담당 (tls)

**Galley**

- Istio Configuration 체크
