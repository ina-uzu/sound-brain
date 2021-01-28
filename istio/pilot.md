## 요약

각 pod마다 사이드카로 떠있는 엔보이는 pilot agent를 통해 초기 설정이 주입되고,

이후 실제 우리가 crd를 통해 설정하는 규칙들은 istiod(pilot)를 통해 동적으로 업데이트 된다. 



### Pilot Agent
Pilot Agent는 Envoy config 파일을 생성하고 Envoy 프록시 실행을 담당한다. 

대부분의 Envoy config 정보는 xDS를(ads) 통해 Pilot에서 dynamic하게 받아오는 구조이며

pilot agent는 엔보이를 초기화하는 데 필요한 일부 static config 정보만 추가 해준다.



### Envoy
Envoy는 Pilot agent에 의해 시작한다!

엔보이 시작 후  pilot agent가 생성 한 static config 파일을 읽고,  여기 설정된 Pilot 주소를 가져와 나머지 config를 동적으로 받아온다. 

실제로 envoy config를 열어보면 listener, cluster 등의 정보를 ads로 받아오게 되어있으며,

이 ads에 연결된(구현한) xds-grpc 클러스터는 istiod.istio-system를 업스트림으로 가리키고 있다.  (구버전)


Envoy가 ADS로부터 받아온 Cluster, Listener, Endpoint는 설정 파일로부터는 확인할 수 없어서

istioctl proxy-config와 같은 명령어를 사용해서 확인하거나 Envoy config-dump를 통해 확인해야만 한다 (하지만 config dump는 너무나 방대하다...)

Envoy config-dump는 Envoy Admin 페이지에서 가져올 수 있으며, istio에서는 기본적으로 15000 포트를 통해 접근할 수 있다. 



## 예제로 살펴보는 Pilot 동작 과정


### 1) Envoy Init Container (istio-init)
사이드카 엔보이가 주입된 productpage pod의 yaml을 보면, 아래와 같은 init container도 추가된 걸 볼 수 있다. 

istio-init 컨테이너는 istio-iptables 스크립트를 실행한다.  https://github.com/istio/cni/blob/master/tools/packaging/common/istio-iptables.sh

이 스크립트를 통해 iptables 명령어를 호출하여 트래픽을 가로 채기위한 iptables 규칙을 만든다. 

- p 15001 :  아웃 바운드 트래픽이 iptable에 의해 Envoy 15001 포트로 리디렉션됨

- z 15006: 인바운드 트래픽이 iptable에 의해 Envoy 15006 포트로 리디렉션됨

- u 1337 : (사이드카 엔보이는 UID,GID 1337로 동작하도록 설정되어 있다) UID/GID가 1337로 동작하는 Envoy에서 보낸 패킷이 다시 Envoy로 리디렉션하여 무한 루프를 형성하는 것을 방지한다. 



```yaml
istio-init
initContainers:
- args:
  - istio-iptables
  - -p
  - "15001"
  - -z
  - "15006"
  - -u
  - "1337"
  - -m
  - REDIRECT
  - -i
  - '*'
  - -x
  - ""
  - -b
  - '*'
  - -d
  - 15090,15021,15020
  env:
  - name: DNS_AGENT
  image: docker.io/istio/proxyv2:1.8.1
  imagePullPolicy: Always
  name: istio-init
  resources:
    limits:
      cpu: "2"
      memory: 1Gi
    requests:
      cpu: 10m
      memory: 40Mi
  securityContext:
    allowPrivilegeEscalation: false
    capabilities:
      add:
      - NET_ADMIN
      - NET_RAW
      drop:
      - ALL
    privileged: false
    readOnlyRootFilesystem: false
    runAsGroup: 0
    runAsNonRoot: false
    runAsUser: 0
  terminationMessagePath: /dev/termination-log
  terminationMessagePolicy: File
  volumeMounts:
  - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
    name: bookinfo-productpage-token-5r7qg
    readOnly: true
```

- 참고
```
Usage:
  istio-iptables [flags]
 
Flags:
  -n, --dry-run                                     Do not call any external dependencies like iptables
  -p, --envoy-port string                           Specify the envoy port to which redirect all TCP traffic (default $ENVOY_PORT = 15001)
  -h, --help                                        help for istio-iptables
  -z, --inbound-capture-port string                 Port to which all inbound TCP traffic to the pod/VM should be redirected to (default $INBOUND_CAPTURE_PORT = 15006)
      --iptables-probe-port string                  set listen port for failure detection (default "15002")
  -m, --istio-inbound-interception-mode string      The mode used to redirect inbound connections to Envoy, either "REDIRECT" or "TPROXY"
  -b, --istio-inbound-ports string                  Comma separated list of inbound ports for which traffic is to be redirected to Envoy (optional). The wildcard character "*" can be used to configure redirection for all ports. An empty list will disable
  -t, --istio-inbound-tproxy-mark string
  -r, --istio-inbound-tproxy-route-table string
  -d, --istio-local-exclude-ports string            Comma separated list of inbound ports to be excluded from redirection to Envoy (optional). Only applies  when all inbound traffic (i.e. "*") is being redirected (default to $ISTIO_LOCAL_EXCLUDE_PORTS)
  -o, --istio-local-outbound-ports-exclude string   Comma separated list of outbound ports to be excluded from redirection to Envoy
  -i, --istio-service-cidr string                   Comma separated list of IP ranges in CIDR form to redirect to envoy (optional). The wildcard character "*" can be used to redirect all outbound traffic. An empty list will disable all outbound
  -x, --istio-service-exclude-cidr string           Comma separated list of IP ranges in CIDR form to be excluded from redirection. Only applies when all  outbound traffic (i.e. "*") is being redirected (default to $ISTIO_SERVICE_EXCLUDE_CIDR)
  -k, --kube-virt-interfaces string                 Comma separated list of virtual interfaces whose inbound traffic (from VM) will be treated as outbound
      --probe-timeout duration                      failure detection timeout (default 5s)
  -g, --proxy-gid string                            Specify the GID of the user for which the redirection is not applied. (same default value as -u param)
  -u, --proxy-uid string                            Specify the UID of the user for which the redirection is not applied. Typically, this is the UID of the proxy container
  -f, --restore-format                              Print iptables rules in iptables-restore interpretable format (default true)
      --run-validation                              Validate iptables
      --skip-rule-apply                             Skip iptables apply
```

실제 iptable 룰을 확인해보자. 
productpage pod이 떠있는 노드에 접속 → 사용자 앱 컨테이너의 pid를 얻고, 얘가 있는 네임스페이스로 들어가서 iptables 명령어로 확인

```bash
nsenter -n --target {pid}
iptables -t nat -nvL
```





### 2) Envoy 
istio-proxy 컨테이너 안에는 pilot agent & envoy process가 있다. 



#### Envoy 초기 구성 파일
/etc/istio/proxy/envoy-rev0.json 원본 펼치기
Pilot-agent는 Envoy의 초기 config 파일인 envoy-rev0.json을 생성 &  Envoy 프로세스를 시작
요 파일로 Envoy는 xds에 대한 주소 정보를 얻고(→ istiod랑 연결? 바라본다) 이 xds 인터페이스를 사용하여 istiod(pilot)에서 listener, cluster 등등등등의 dynamic config를 얻어온다. 
Envoy는 이러한 config에 따라 proxy


Inbound Cluster
outbound Cluster


- productpage에서 review service 호출  http://reviews:9080/reviews/0..
- 이 요청은 productpage pod의 iptable 룰에 의해 로컬 포트 15001(outbound)로 리디렉션
- 포트 15001에서 수신하는 엔보이 Virtual Outbound Listener가 요청을 수신
- 이후 요청은 Virtual Outbound Listener에 있는 route 설정에 따라 원래 대상 ip및 포트 (9080)를 기반으로 아웃 바운드 리스너 0.0.0.0_9080으로 전달
- 0.0.0.0_9080 리스너의 http_connection_manager 필터 구성에 따라 9080경로를 사용하여 요청이 로드밸런싱 .....??????????/

TODO ...넘무 어렵다 ㅜㅜㅜ

### 참고 

https://ssup2.github.io/theory_analysis/Istio_Sidecar/

https://jimmysong.io/en/blog/sidecar-injection-iptables-and-traffic-routing/

https://zhaohuabing.com/post/2018-09-25-istio-traffic-management-impl-intro/#virtual-listener
