# pilot

## 요약!

각 pod마다 사이드카로 떠있는 엔보이는 pilot agent를 통해 초기 설정이 주입되고,

이후 실제 우리가 crd를 통해 설정하는 규칙들은 istiod(pilot)를 통해 동적으로 업데이트 된다.

---



## Pilot Agent

Pilot Agent는 Envoy config 파일을 생성하고 Envoy 프록시 실행을 담당한다.

대부분의 Envoy config 정보는 xDS를(ads) 통해 Pilot에서 dynamic하게 받아오는 구조이며

pilot agent는 엔보이를 초기화하는 데 필요한 일부 static config 정보만 추가 해준다.

## Envoy

Envoy는 Pilot agent에 의해 시작한다!

엔보이 시작 후  pilot agent가 생성 한 static config 파일을 읽고,  여기 설정된 Pilot 주소를 가져와 나머지 config를 동적으로 받아온다.

실제로 envoy config를 열어보면 listener, cluster 등의 정보를 ads로 받아오게 되어있으며,

이 ads에 연결된(구현한) xds-grpc 클러스터는 i~~stiod.istio-system를 업스트림으로 가리키고 있다.  (구버전)~~

![pilot%2089955bc5d33049af8709d34f7b5ee919/Untitled.png](pilot%2089955bc5d33049af8709d34f7b5ee919/Untitled.png)

실제로 istioctl proxy-config endpoint 명령어로 xds 엔드포인트를 확인

![pilot%2089955bc5d33049af8709d34f7b5ee919/Untitled%201.png](pilot%2089955bc5d33049af8709d34f7b5ee919/Untitled%201.png)

Envoy가 ADS로부터 받아온 Cluster, Listener, Endpoint는 설정 파일로부터는 확인할 수 없어서

istioctl proxy-config와 같은 명령어를 사용해서 확인하거나 Envoy config-dump를 통해 확인해야만 한다 

(하지만 config dump는 너무나 방대하다...)

Envoy config-dump는 Envoy Admin 페이지에서 가져올 수 있으며, istio에서는 기본적으로 15000 포트를 통해 접근할 수 있다.

---

# 예제로 살펴보는 Envoy Config 상세

![https://zhaohuabing.com/img/2019-12-05-istio-traffic-management-impl-intro/bookinfo.png](https://zhaohuabing.com/img/2019-12-05-istio-traffic-management-impl-intro/bookinfo.png)

## Envoy Init Container (istio-init)

사이드카 엔보이가 주입된 productpage pod의 yaml을 보면, 아래와 같은 init container도 추가된 걸 볼 수 있다.

istio-init 컨테이너는 istio-iptables 스크립트를 실행한다.  [https://github.com/istio/cni/blob/master/tools/packaging/common/istio-iptables.sh](https://github.com/istio/cni/blob/master/tools/packaging/common/istio-iptables.sh)

이 스크립트를 통해 iptables 명령어를 호출하여 트래픽을 가로 채기위한 iptables 규칙을 만든다.

- p 15001 : 아웃 바운드 트래픽이 iptable에 의해 Envoy 15001 포트로 리디렉션됨
- z 15006: 인바운드 트래픽이 iptable에 의해 Envoy 15006 포트로 리디렉션됨
- u 1337 : (사이드카 엔보이는 UID,GID 1337로 동작하도록 설정되어 있다) UID/GID가 1337로 동작하는 Envoy에서 보낸 패킷이 다시 Envoy로 리디렉션하여 무한 루프를 형성하는 것을 방지한다.

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

```
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

실제 iptable 룰을 확인해보자.

productpage pod이 떠있는 노드에 접속 → 사용자 앱 컨테이너의 pid를 얻고, 

얘가 있는 네임스페이스로 들어가서 iptables 명령어로 확인

![pilot%2089955bc5d33049af8709d34f7b5ee919/Untitled%202.png](pilot%2089955bc5d33049af8709d34f7b5ee919/Untitled%202.png)

## Envoy

istio-proxy 컨테이너 안에는 pilot agent & envoy process가 있다.

![pilot%2089955bc5d33049af8709d34f7b5ee919/Untitled%203.png](pilot%2089955bc5d33049af8709d34f7b5ee919/Untitled%203.png)

### **초기 static config**

- 초기 static config

    ```json
    {
      "node": {
        "id": "sidecar~10.240.2.38~productpage-v1-64794f5db4-6bsbj.default~default.svc.cluster.local",
        "cluster": "productpage.default",
        "locality": {
        },
        "metadata": {"APP_CONTAINERS":"productpage","CLUSTER_ID":"Kubernetes","EXCHANGE_KEYS":"NAME,NAMESPACE,INSTANCE_IPS,LABELS,OWNER,PLATFORM_METADATA,WORKLOAD_NAME,MESH_ID,SERVICE_ACCOUNT,CLUSTER_ID","INSTANCE_IPS":"10.240.2.38","INTERCEPTION_MODE":"REDIRECT","ISTIO_PROXY_SHA":"istio-proxy:aaff48d79dc578e4e6ef8525f96734b972a7670f","ISTIO_VERSION":"1.8.1","LABELS":{"app":"productpage","istio.io/rev":"default","pod-template-hash":"64794f5db4","security.istio.io/tlsMode":"istio","service.istio.io/canonical-name":"productpage","service.istio.io/canonical-revision":"v1","version":"v1"},"MESH_ID":"cluster.local","NAME":"productpage-v1-64794f5db4-6bsbj","NAMESPACE":"default","OWNER":"kubernetes://apis/apps/v1/namespaces/default/deployments/productpage-v1","POD_PORTS":"[{\"containerPort\":9080,\"protocol\":\"TCP\"}]","PROXY_CONFIG":{"binaryPath":"/usr/local/bin/envoy","concurrency":2,"configPath":"./etc/istio/proxy","controlPlaneAuthPolicy":"MUTUAL_TLS","discoveryAddress":"istiod.istio-system.svc:15012","drainDuration":"45s","envoyAccessLogService":{},"envoyMetricsService":{},"parentShutdownDuration":"60s","proxyAdminPort":15000,"proxyMetadata":{"DNS_AGENT":""},"serviceCluster":"productpage.default","statNameLength":189,"statusPort":15020,"terminationDrainDuration":"5s","tracing":{"zipkin":{"address":"zipkin.istio-system:9411"}}},"SERVICE_ACCOUNT":"bookinfo-productpage","WORKLOAD_NAME":"productpage-v1"}
      },
      "layered_runtime": {
          "layers": [
              {
                  "name": "deprecation",
                  "static_layer": {
                      "envoy.deprecated_features:envoy.config.listener.v3.Listener.hidden_envoy_deprecated_use_original_dst": true,
                      "envoy.reloadable_features.strict_1xx_and_204_response_headers": false,
                      "re2.max_program_size.error_level": 1024
                  }
              },
              {
                  "name": "admin",
                  "admin_layer": {}
              }
          ]
      },
      "stats_config": {
        "use_all_default_tags": false,
        "stats_tags": [
          {
            "tag_name": "cluster_name",
            "regex": "^cluster\\.((.+?(\\..+?\\.svc\\.cluster\\.local)?)\\.)"
          },
          {
            "tag_name": "tcp_prefix",
            "regex": "^tcp\\.((.*?)\\.)\\w+?$"
          },
          {
            "regex": "(response_code=\\.=(.+?);\\.;)|_rq(_(\\.d{3}))$",
            "tag_name": "response_code"
          },
          {
            "tag_name": "response_code_class",
            "regex": "_rq(_(\\dxx))$"
          },
          {
            "tag_name": "http_conn_manager_listener_prefix",
            "regex": "^listener(?=\\.).*?\\.http\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
          },
          {
            "tag_name": "http_conn_manager_prefix",
            "regex": "^http\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
          },
          {
            "tag_name": "listener_address",
            "regex": "^listener\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
          },
          {
            "tag_name": "mongo_prefix",
            "regex": "^mongo\\.(.+?)\\.(collection|cmd|cx_|op_|delays_|decoding_)(.*?)$"
          },
          {
            "regex": "(reporter=\\.=(.*?);\\.;)",
            "tag_name": "reporter"
          },
          {
            "regex": "(source_namespace=\\.=(.*?);\\.;)",
            "tag_name": "source_namespace"
          },
          {
            "regex": "(source_workload=\\.=(.*?);\\.;)",
            "tag_name": "source_workload"
          },
          {
            "regex": "(source_workload_namespace=\\.=(.*?);\\.;)",
            "tag_name": "source_workload_namespace"
          },
          {
            "regex": "(source_principal=\\.=(.*?);\\.;)",
            "tag_name": "source_principal"
          },
          {
            "regex": "(source_app=\\.=(.*?);\\.;)",
            "tag_name": "source_app"
          },
          {
            "regex": "(source_version=\\.=(.*?);\\.;)",
            "tag_name": "source_version"
          },
          {
            "regex": "(source_cluster=\\.=(.*?);\\.;)",
            "tag_name": "source_cluster"
          },
          {
            "regex": "(destination_namespace=\\.=(.*?);\\.;)",
            "tag_name": "destination_namespace"
          },
          {
            "regex": "(destination_workload=\\.=(.*?);\\.;)",
            "tag_name": "destination_workload"
          },
          {
            "regex": "(destination_workload_namespace=\\.=(.*?);\\.;)",
            "tag_name": "destination_workload_namespace"
          },
          {
            "regex": "(destination_principal=\\.=(.*?);\\.;)",
            "tag_name": "destination_principal"
          },
          {
            "regex": "(destination_app=\\.=(.*?);\\.;)",
            "tag_name": "destination_app"
          },
          {
            "regex": "(destination_version=\\.=(.*?);\\.;)",
            "tag_name": "destination_version"
          },
          {
            "regex": "(destination_service=\\.=(.*?);\\.;)",
            "tag_name": "destination_service"
          },
          {
            "regex": "(destination_service_name=\\.=(.*?);\\.;)",
            "tag_name": "destination_service_name"
          },
          {
            "regex": "(destination_service_namespace=\\.=(.*?);\\.;)",
            "tag_name": "destination_service_namespace"
          },
          {
            "regex": "(destination_port=\\.=(.*?);\\.;)",
            "tag_name": "destination_port"
          },
          {
            "regex": "(destination_cluster=\\.=(.*?);\\.;)",
            "tag_name": "destination_cluster"
          },
          {
            "regex": "(request_protocol=\\.=(.*?);\\.;)",
            "tag_name": "request_protocol"
          },
          {
            "regex": "(request_operation=\\.=(.*?);\\.;)",
            "tag_name": "request_operation"
          },
          {
            "regex": "(request_host=\\.=(.*?);\\.;)",
            "tag_name": "request_host"
          },
          {
            "regex": "(response_flags=\\.=(.*?);\\.;)",
            "tag_name": "response_flags"
          },
          {
            "regex": "(grpc_response_status=\\.=(.*?);\\.;)",
            "tag_name": "grpc_response_status"
          },
          {
            "regex": "(connection_security_policy=\\.=(.*?);\\.;)",
            "tag_name": "connection_security_policy"
          },
          {
            "regex": "(permissive_response_code=\\.=(.*?);\\.;)",
            "tag_name": "permissive_response_code"
          },
          {
            "regex": "(permissive_response_policyid=\\.=(.*?);\\.;)",
            "tag_name": "permissive_response_policyid"
          },
          {
            "regex": "(source_canonical_service=\\.=(.*?);\\.;)",
            "tag_name": "source_canonical_service"
          },
          {
            "regex": "(destination_canonical_service=\\.=(.*?);\\.;)",
            "tag_name": "destination_canonical_service"
          },
          {
            "regex": "(source_canonical_revision=\\.=(.*?);\\.;)",
            "tag_name": "source_canonical_revision"
          },
          {
            "regex": "(destination_canonical_revision=\\.=(.*?);\\.;)",
            "tag_name": "destination_canonical_revision"
          },
          {
            "regex": "(cache\\.(.+?)\\.)",
            "tag_name": "cache"
          },
          {
            "regex": "(component\\.(.+?)\\.)",
            "tag_name": "component"
          },
          {
            "regex": "(tag\\.(.+?);\\.)",
            "tag_name": "tag"
          },
          {
            "regex": "(wasm_filter\\.(.+?)\\.)",
            "tag_name": "wasm_filter"
          }
        ],
        "stats_matcher": {
          "inclusion_list": {
            "patterns": [
              {
              "prefix": "reporter="
              },
              {
              "prefix": "cluster_manager"
              },
              {
              "prefix": "listener_manager"
              },
              {
              "prefix": "server"
              },
              {
              "prefix": "cluster.xds-grpc"
              },
              {
              "prefix": "wasm"
              },
              {
              "prefix": "component"
              }
            ]
          }
        }
      },
      "admin": {
        "access_log_path": "/dev/null",
        "profile_path": "/var/lib/istio/data/envoy.prof",
        "address": {
          "socket_address": {
            "address": "127.0.0.1",
            "port_value": 15000
          }
        }
      },
      "dynamic_resources": {
        "lds_config": {
          "ads": {},
          "resource_api_version": "V3"
        },
        "cds_config": {
          "ads": {},
          "resource_api_version": "V3"
        },
        "ads_config": {
          "api_type": "GRPC",
          "transport_api_version": "V3",
          "grpc_services": [
            {

              "envoy_grpc": {
                "cluster_name": "xds-grpc"
              }

            }
          ]
        }
      },
      "static_resources": {
        "clusters": [
          {
            "name": "prometheus_stats",
            "type": "STATIC",
            "connect_timeout": "0.250s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": {
              "cluster_name": "prometheus_stats",
              "endpoints": [{
                "lb_endpoints": [{
                  "endpoint": {
                    "address":{
                      "socket_address": {
                        "protocol": "TCP",
                        "address": "127.0.0.1",
                        "port_value": 15000
                      }
                    }
                  }
                }]
              }]
            }
          },
          {
            "name": "agent",
            "type": "STATIC",
            "connect_timeout": "0.250s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": {
              "cluster_name": "prometheus_stats",
              "endpoints": [{
                "lb_endpoints": [{
                  "endpoint": {
                    "address":{
                      "socket_address": {
                        "protocol": "TCP",
                        "address": "127.0.0.1",
                        "port_value": 15020
                      }
                    }
                  }
                }]
              }]
            }
          },
          {
            "name": "sds-grpc",
            "type": "STATIC",
            "http2_protocol_options": {},
            "connect_timeout": "1s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": {
              "cluster_name": "sds-grpc",
              "endpoints": [{
                "lb_endpoints": [{
                  "endpoint": {
                    "address":{
                      "pipe": {
                        "path": "./etc/istio/proxy/SDS"
                      }
                    }
                  }
                }]
              }]
            }
          }
          // xds-grpc
           , {
            "name": "xds-grpc",
            "type" : "STATIC",
            "connect_timeout": "1s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": {
              "cluster_name": "xds-grpc",
              "endpoints": [{
                "lb_endpoints": [{
                  "endpoint": {
                    "address":{
                      "pipe": {
                        "path": "./etc/istio/proxy/XDS"
                      }
                    }
                  }
                }]
              }]
            },
            "circuit_breakers": {
              "thresholds": [
                {
                  "priority": "DEFAULT",
                  "max_connections": 100000,
                  "max_pending_requests": 100000,
                  "max_requests": 100000
                },
                {
                  "priority": "HIGH",
                  "max_connections": 100000,
                  "max_pending_requests": 100000,
                  "max_requests": 100000
                }
              ]
            },
            "upstream_connection_options": {
              "tcp_keepalive": {
                "keepalive_time": 300
              }
            },
            "max_requests_per_connection": 1,
            "http2_protocol_options": { }
          }

          ,
          {
            "name": "zipkin",
            "type": "STRICT_DNS",
            "respect_dns_ttl": true,
            "dns_lookup_family": "V4_ONLY",
            "connect_timeout": "1s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": {
              "cluster_name": "zipkin",
              "endpoints": [{
                "lb_endpoints": [{
                  "endpoint": {
                    "address":{
                      "socket_address": {"address": "zipkin.istio-system", "port_value": 9411}
                    }
                  }
                }]
              }]
            }
          }

        ],
        "listeners":[
          {
            "address": {
              "socket_address": {
                "protocol": "TCP",
                "address": "0.0.0.0",
                "port_value": 15090
              }
            },
            "filter_chains": [
              {
                "filters": [
                  {
                    "name": "envoy.filters.network.http_connection_manager",
                    "typed_config": {
                      "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                      "codec_type": "AUTO",
                      "stat_prefix": "stats",
                      "route_config": {
                        "virtual_hosts": [
                          {
                            "name": "backend",
                            "domains": [
                              "*"
                            ],
                            "routes": [
                              {
                                "match": {
                                  "prefix": "/stats/prometheus"
                                },
                                "route": {
                                  "cluster": "prometheus_stats"
                                }
                              }
                            ]
                          }
                        ]
                      },
                      "http_filters": [{
                        "name": "envoy.filters.http.router",
                        "typed_config": {
                          "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
                        }
                      }]
                    }
                  }
                ]
              }
            ]
          },
          {
            "address": {
              "socket_address": {
                "protocol": "TCP",
                "address": "0.0.0.0",
                "port_value": 15021
              }
            },
            "filter_chains": [
              {
                "filters": [
                  {
                    "name": "envoy.filters.network.http_connection_manager",
                    "typed_config": {
                      "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                      "codec_type": "AUTO",
                      "stat_prefix": "agent",
                      "route_config": {
                        "virtual_hosts": [
                          {
                            "name": "backend",
                            "domains": [
                              "*"
                            ],
                            "routes": [
                              {
                                "match": {
                                  "prefix": "/healthz/ready"
                                },
                                "route": {
                                  "cluster": "agent"
                                }
                              }
                            ]
                          }
                        ]
                      },
                      "http_filters": [{
                        "name": "envoy.filters.http.router",
                        "typed_config": {
                          "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
                        }
                      }]
                    }
                  }
                ]
              }
            ]
          }
        ]
      }
      ,
      "tracing": {
        "http": {
          "name": "envoy.tracers.zipkin",
          "typed_config": {
            "@type": "type.googleapis.com/envoy.config.trace.v3.ZipkinConfig",
            "collector_cluster": "zipkin",
            "collector_endpoint": "/api/v2/spans",
            "collector_endpoint_version": "HTTP_JSON",
            "trace_id_128bit": true,
            "shared_span_context": false
          }
        }
      }

    }
    ```

1. Pilot-agent는 Envoy의 초기 config 파일인 envoy-rev0.json을 생성 & Envoy 프로세스를 시작
2. 요 파일로 Envoy는 xds에 대한 주소 정보를 얻고(→ istiod랑 연결? 바라본다) 이 xds 인터페이스를 사용하여 istiod(pilot)에서 listener, cluster 등등등등의 dynamic config를 얻어온다.
3. Envoy는 이러한 config에 따라 라우팅해준다.

### **Clusters**

istio envoy에 설정된 주요 cluster들을 살펴보자.

(cluster == upstream host 정보)

**1) Outbound Cluster**


대부분의 서비스가 outbound clsuter이다. (엔보이가 있는 pod이 아닌 다른 서비스)

예를 들어, productpage의 envoy를 기준으로  하면 review, detail 서비스가 outbound cluster이다.

아래 명령어로  details 서비스의 envoy clsuter 설정을 조회하면,  클러스터 이름이 `"outbound|9080|v1|details.default.svc.cluster.local"` 이런식으로 된걸 볼 수 있다.

```bash
// productpage 서비스의 envoy의 config를 조회하는 것이므로 istioctl proxy-config cluster {productpagePod} 로 수행한다

istioctl proxy-config cluster productpage-v1-64794f5db4-6bsbj --fqdn reviews.default.svc.cluster.local -o json 
```

```json
{
     "version_info": "2021-01-26T09:21:45Z/76",
     "cluster": {
      "@type": "type.googleapis.com/envoy.config.cluster.v3.Cluster",
      "name": "outbound|9080||reviews.default.svc.cluster.local",  
      "type": "EDS",
      "eds_cluster_config": {
       "eds_config": {
        "ads": {},
        "resource_api_version": "V3"
       },
       "service_name": "outbound|9080||reviews.default.svc.cluster.local"
      },
      "connect_timeout": "10s",
      
       // ...
      "last_updated": "2021-01-26T09:21:58.538Z"
}
```

cluster metadata에 subset 정보가 있다.

- cluster 상세

    ```json
    {
      "node": {
        "id": "sidecar~10.240.2.38~productpage-v1-64794f5db4-6bsbj.default~default.svc.cluster.local",
        "cluster": "productpage.default",
        "locality": {
        },
        "metadata": {"APP_CONTAINERS":"productpage","CLUSTER_ID":"Kubernetes","EXCHANGE_KEYS":"NAME,NAMESPACE,INSTANCE_IPS,LABELS,OWNER,PLATFORM_METADATA,WORKLOAD_NAME,MESH_ID,SERVICE_ACCOUNT,CLUSTER_ID","INSTANCE_IPS":"10.240.2.38","INTERCEPTION_MODE":"REDIRECT","ISTIO_PROXY_SHA":"istio-proxy:aaff48d79dc578e4e6ef8525f96734b972a7670f","ISTIO_VERSION":"1.8.1","LABELS":{"app":"productpage","istio.io/rev":"default","pod-template-hash":"64794f5db4","security.istio.io/tlsMode":"istio","service.istio.io/canonical-name":"productpage","service.istio.io/canonical-revision":"v1","version":"v1"},"MESH_ID":"cluster.local","NAME":"productpage-v1-64794f5db4-6bsbj","NAMESPACE":"default","OWNER":"kubernetes://apis/apps/v1/namespaces/default/deployments/productpage-v1","POD_PORTS":"[{\"containerPort\":9080,\"protocol\":\"TCP\"}]","PROXY_CONFIG":{"binaryPath":"/usr/local/bin/envoy","concurrency":2,"configPath":"./etc/istio/proxy","controlPlaneAuthPolicy":"MUTUAL_TLS","discoveryAddress":"istiod.istio-system.svc:15012","drainDuration":"45s","envoyAccessLogService":{},"envoyMetricsService":{},"parentShutdownDuration":"60s","proxyAdminPort":15000,"proxyMetadata":{"DNS_AGENT":""},"serviceCluster":"productpage.default","statNameLength":189,"statusPort":15020,"terminationDrainDuration":"5s","tracing":{"zipkin":{"address":"zipkin.istio-system:9411"}}},"SERVICE_ACCOUNT":"bookinfo-productpage","WORKLOAD_NAME":"productpage-v1"}
      },
      "layered_runtime": {
          "layers": [
              {
                  "name": "deprecation",
                  "static_layer": {
                      "envoy.deprecated_features:envoy.config.listener.v3.Listener.hidden_envoy_deprecated_use_original_dst": true,
                      "envoy.reloadable_features.strict_1xx_and_204_response_headers": false,
                      "re2.max_program_size.error_level": 1024
                  }
              },
              {
                  "name": "admin",
                  "admin_layer": {}
              }
          ]
      },
      "stats_config": {
        "use_all_default_tags": false,
        "stats_tags": [
          {
            "tag_name": "cluster_name",
            "regex": "^cluster\\.((.+?(\\..+?\\.svc\\.cluster\\.local)?)\\.)"
          },
          {
            "tag_name": "tcp_prefix",
            "regex": "^tcp\\.((.*?)\\.)\\w+?$"
          },
          {
            "regex": "(response_code=\\.=(.+?);\\.;)|_rq(_(\\.d{3}))$",
            "tag_name": "response_code"
          },
          {
            "tag_name": "response_code_class",
            "regex": "_rq(_(\\dxx))$"
          },
          {
            "tag_name": "http_conn_manager_listener_prefix",
            "regex": "^listener(?=\\.).*?\\.http\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
          },
          {
            "tag_name": "http_conn_manager_prefix",
            "regex": "^http\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
          },
          {
            "tag_name": "listener_address",
            "regex": "^listener\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
          },
          {
            "tag_name": "mongo_prefix",
            "regex": "^mongo\\.(.+?)\\.(collection|cmd|cx_|op_|delays_|decoding_)(.*?)$"
          },
          {
            "regex": "(reporter=\\.=(.*?);\\.;)",
            "tag_name": "reporter"
          },
          {
            "regex": "(source_namespace=\\.=(.*?);\\.;)",
            "tag_name": "source_namespace"
          },
          {
            "regex": "(source_workload=\\.=(.*?);\\.;)",
            "tag_name": "source_workload"
          },
          {
            "regex": "(source_workload_namespace=\\.=(.*?);\\.;)",
            "tag_name": "source_workload_namespace"
          },
          {
            "regex": "(source_principal=\\.=(.*?);\\.;)",
            "tag_name": "source_principal"
          },
          {
            "regex": "(source_app=\\.=(.*?);\\.;)",
            "tag_name": "source_app"
          },
          {
            "regex": "(source_version=\\.=(.*?);\\.;)",
            "tag_name": "source_version"
          },
          {
            "regex": "(source_cluster=\\.=(.*?);\\.;)",
            "tag_name": "source_cluster"
          },
          {
            "regex": "(destination_namespace=\\.=(.*?);\\.;)",
            "tag_name": "destination_namespace"
          },
          {
            "regex": "(destination_workload=\\.=(.*?);\\.;)",
            "tag_name": "destination_workload"
          },
          {
            "regex": "(destination_workload_namespace=\\.=(.*?);\\.;)",
            "tag_name": "destination_workload_namespace"
          },
          {
            "regex": "(destination_principal=\\.=(.*?);\\.;)",
            "tag_name": "destination_principal"
          },
          {
            "regex": "(destination_app=\\.=(.*?);\\.;)",
            "tag_name": "destination_app"
          },
          {
            "regex": "(destination_version=\\.=(.*?);\\.;)",
            "tag_name": "destination_version"
          },
          {
            "regex": "(destination_service=\\.=(.*?);\\.;)",
            "tag_name": "destination_service"
          },
          {
            "regex": "(destination_service_name=\\.=(.*?);\\.;)",
            "tag_name": "destination_service_name"
          },
          {
            "regex": "(destination_service_namespace=\\.=(.*?);\\.;)",
            "tag_name": "destination_service_namespace"
          },
          {
            "regex": "(destination_port=\\.=(.*?);\\.;)",
            "tag_name": "destination_port"
          },
          {
            "regex": "(destination_cluster=\\.=(.*?);\\.;)",
            "tag_name": "destination_cluster"
          },
          {
            "regex": "(request_protocol=\\.=(.*?);\\.;)",
            "tag_name": "request_protocol"
          },
          {
            "regex": "(request_operation=\\.=(.*?);\\.;)",
            "tag_name": "request_operation"
          },
          {
            "regex": "(request_host=\\.=(.*?);\\.;)",
            "tag_name": "request_host"
          },
          {
            "regex": "(response_flags=\\.=(.*?);\\.;)",
            "tag_name": "response_flags"
          },
          {
            "regex": "(grpc_response_status=\\.=(.*?);\\.;)",
            "tag_name": "grpc_response_status"
          },
          {
            "regex": "(connection_security_policy=\\.=(.*?);\\.;)",
            "tag_name": "connection_security_policy"
          },
          {
            "regex": "(permissive_response_code=\\.=(.*?);\\.;)",
            "tag_name": "permissive_response_code"
          },
          {
            "regex": "(permissive_response_policyid=\\.=(.*?);\\.;)",
            "tag_name": "permissive_response_policyid"
          },
          {
            "regex": "(source_canonical_service=\\.=(.*?);\\.;)",
            "tag_name": "source_canonical_service"
          },
          {
            "regex": "(destination_canonical_service=\\.=(.*?);\\.;)",
            "tag_name": "destination_canonical_service"
          },
          {
            "regex": "(source_canonical_revision=\\.=(.*?);\\.;)",
            "tag_name": "source_canonical_revision"
          },
          {
            "regex": "(destination_canonical_revision=\\.=(.*?);\\.;)",
            "tag_name": "destination_canonical_revision"
          },
          {
            "regex": "(cache\\.(.+?)\\.)",
            "tag_name": "cache"
          },
          {
            "regex": "(component\\.(.+?)\\.)",
            "tag_name": "component"
          },
          {
            "regex": "(tag\\.(.+?);\\.)",
            "tag_name": "tag"
          },
          {
            "regex": "(wasm_filter\\.(.+?)\\.)",
            "tag_name": "wasm_filter"
          }
        ],
        "stats_matcher": {
          "inclusion_list": {
            "patterns": [
              {
              "prefix": "reporter="
              },
              {
              "prefix": "cluster_manager"
              },
              {
              "prefix": "listener_manager"
              },
              {
              "prefix": "server"
              },
              {
              "prefix": "cluster.xds-grpc"
              },
              {
              "prefix": "wasm"
              },
              {
              "prefix": "component"
              }
            ]
          }
        }
      },
      "admin": {
        "access_log_path": "/dev/null",
        "profile_path": "/var/lib/istio/data/envoy.prof",
        "address": {
          "socket_address": {
            "address": "127.0.0.1",
            "port_value": 15000
          }
        }
      },
      "dynamic_resources": {
        "lds_config": {
          "ads": {},
          "resource_api_version": "V3"
        },
        "cds_config": {
          "ads": {},
          "resource_api_version": "V3"
        },
        "ads_config": {
          "api_type": "GRPC",
          "transport_api_version": "V3",
          "grpc_services": [
            {

              "envoy_grpc": {
                "cluster_name": "xds-grpc"
              }

            }
          ]
        }
      },
      "static_resources": {
        "clusters": [
          {
            "name": "prometheus_stats",
            "type": "STATIC",
            "connect_timeout": "0.250s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": {
              "cluster_name": "prometheus_stats",
              "endpoints": [{
                "lb_endpoints": [{
                  "endpoint": {
                    "address":{
                      "socket_address": {
                        "protocol": "TCP",
                        "address": "127.0.0.1",
                        "port_value": 15000
                      }
                    }
                  }
                }]
              }]
            }
          },
          {
            "name": "agent",
            "type": "STATIC",
            "connect_timeout": "0.250s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": {
              "cluster_name": "prometheus_stats",
              "endpoints": [{
                "lb_endpoints": [{
                  "endpoint": {
                    "address":{
                      "socket_address": {
                        "protocol": "TCP",
                        "address": "127.0.0.1",
                        "port_value": 15020
                      }
                    }
                  }
                }]
              }]
            }
          },
          {
            "name": "sds-grpc",
            "type": "STATIC",
            "http2_protocol_options": {},
            "connect_timeout": "1s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": {
              "cluster_name": "sds-grpc",
              "endpoints": [{
                "lb_endpoints": [{
                  "endpoint": {
                    "address":{
                      "pipe": {
                        "path": "./etc/istio/proxy/SDS"
                      }
                    }
                  }
                }]
              }]
            }
          }
          // xds-grpc
           , {
            "name": "xds-grpc",
            "type" : "STATIC",
            "connect_timeout": "1s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": {
              "cluster_name": "xds-grpc",
              "endpoints": [{
                "lb_endpoints": [{
                  "endpoint": {
                    "address":{
                      "pipe": {
                        "path": "./etc/istio/proxy/XDS"
                      }
                    }
                  }
                }]
              }]
            },
            "circuit_breakers": {
              "thresholds": [
                {
                  "priority": "DEFAULT",
                  "max_connections": 100000,
                  "max_pending_requests": 100000,
                  "max_requests": 100000
                },
                {
                  "priority": "HIGH",
                  "max_connections": 100000,
                  "max_pending_requests": 100000,
                  "max_requests": 100000
                }
              ]
            },
            "upstream_connection_options": {
              "tcp_keepalive": {
                "keepalive_time": 300
              }
            },
            "max_requests_per_connection": 1,
            "http2_protocol_options": { }
          }

          ,
          {
            "name": "zipkin",
            "type": "STRICT_DNS",
            "respect_dns_ttl": true,
            "dns_lookup_family": "V4_ONLY",
            "connect_timeout": "1s",
            "lb_policy": "ROUND_ROBIN",
            "load_assignment": {
              "cluster_name": "zipkin",
              "endpoints": [{
                "lb_endpoints": [{
                  "endpoint": {
                    "address":{
                      "socket_address": {"address": "zipkin.istio-system", "port_value": 9411}
                    }
                  }
                }]
              }]
            }
          }

        ],
        "listeners":[
          {
            "address": {
              "socket_address": {
                "protocol": "TCP",
                "address": "0.0.0.0",
                "port_value": 15090
              }
            },
            "filter_chains": [
              {
                "filters": [
                  {
                    "name": "envoy.filters.network.http_connection_manager",
                    "typed_config": {
                      "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                      "codec_type": "AUTO",
                      "stat_prefix": "stats",
                      "route_config": {
                        "virtual_hosts": [
                          {
                            "name": "backend",
                            "domains": [
                              "*"
                            ],
                            "routes": [
                              {
                                "match": {
                                  "prefix": "/stats/prometheus"
                                },
                                "route": {
                                  "cluster": "prometheus_stats"
                                }
                              }
                            ]
                          }
                        ]
                      },
                      "http_filters": [{
                        "name": "envoy.filters.http.router",
                        "typed_config": {
                          "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
                        }
                      }]
                    }
                  }
                ]
              }
            ]
          },
          {
            "address": {
              "socket_address": {
                "protocol": "TCP",
                "address": "0.0.0.0",
                "port_value": 15021
              }
            },
            "filter_chains": [
              {
                "filters": [
                  {
                    "name": "envoy.filters.network.http_connection_manager",
                    "typed_config": {
                      "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                      "codec_type": "AUTO",
                      "stat_prefix": "agent",
                      "route_config": {
                        "virtual_hosts": [
                          {
                            "name": "backend",
                            "domains": [
                              "*"
                            ],
                            "routes": [
                              {
                                "match": {
                                  "prefix": "/healthz/ready"
                                },
                                "route": {
                                  "cluster": "agent"
                                }
                              }
                            ]
                          }
                        ]
                      },
                      "http_filters": [{
                        "name": "envoy.filters.http.router",
                        "typed_config": {
                          "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
                        }
                      }]
                    }
                  }
                ]
              }
            ]
          }
        ]
      }
      ,
      "tracing": {
        "http": {
          "name": "envoy.tracers.zipkin",
          "typed_config": {
            "@type": "type.googleapis.com/envoy.config.trace.v3.ZipkinConfig",
            "collector_cluster": "zipkin",
            "collector_endpoint": "/api/v2/spans",
            "collector_endpoint_version": "HTTP_JSON",
            "trace_id_128bit": true,
            "shared_span_context": false
          }
        }
      }

    }
    ```

아래 명령어로  "outbound|9080||reviews.default.svc.cluster.local"  클러스터에 연결된 endpoint를 조회하면, 실제 reviews pod 3개의 ip가 잘 나오는 걸 볼 수 있다.

```
$ istioctl proxy-config endpoint productpage-v1-64794f5db4-6bsbj --cluster "outbound|9080||reviews.default.svc.cluster.local"

ENDPOINT              STATUS      OUTLIER CHECK     CLUSTER
10.240.2.112:9080     HEALTHY     OK                outbound|9080||reviews.default.svc.cluster.local
10.240.2.40:9080      HEALTHY     OK                outbound|9080||reviews.default.svc.cluster.local
10.240.2.74:9080      HEALTHY     OK                outbound|9080||reviews.default.svc.cluster.local
```

![pilot%2089955bc5d33049af8709d34f7b5ee919/Untitled%204.png](pilot%2089955bc5d33049af8709d34f7b5ee919/Untitled%204.png)

**2) Inbound Cluster**

Enovy가 있는 Pod 안의 서비스를 말한다.

productpage 서비스의 envoy를 기준으로 하면 productpage 서비스에 연결된 클러스터가  inbound cluster이고, inbound cluster의 호스트는 127.0.0.1이다.

127.0.0.1은 위에서 본 iptables 룰에서 제외 되었으므로, inbound 요청은 envoy를 건너뛰고 해당 서비스(앱) 컨테이너로 직접 전송된다.

```json
{
    "name": "inbound|9080||",
    "type": "STATIC",
    "connectTimeout": "10s",
    "loadAssignment": {
      "clusterName": "inbound|9080||",
      "endpoints": [
        {
          "lbEndpoints": [
            {
              "endpoint": {
                "address": {
                  "socketAddress": {
                    "address": "127.0.0.1",
                    "portValue": 9080
                  }
                }
              }
            }
          ]
        }
      ]
    },
    "metadata": {
      "filterMetadata": {
        "istio": {
          "services": [
            {
              "host": "productpage.default.svc.cluster.local",
              "name": "productpage",
              "namespace": "default"
            }
          ]
        }
      }
    }
  }
```

**3) BlackHoleCluster**

[https://istio.io/latest/blog/2019/monitoring-external-service-traffic/](https://istio.io/latest/blog/2019/monitoring-external-service-traffic/)

요청을 끊어주는 애이다.

예를 들어, 라우팅 할 클러스터(업스트림)을 찾지 못한 경우 요청이 드랍되도록 설정하고 싶을 때 블랙홀 클러스터를 사용할 수 있다.

실제 이 클러스터에 연결된 업스트림 host 정보가 없다 → 이름이 블랙홀이 듯이 여기로 들어온 트래픽은 바로 드랍된다!

```json
{
    "name": "BlackHoleCluster",
    "type": "STATIC",
    "connectTimeout": "10s",
    "filters": [
      {
        "name": "istio.metadata_exchange",
        "typedConfig": {
          "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
          "typeUrl": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
          "value": {
            "protocol": "istio-peer-exchange"
          }
        }
      }
    ]
 }
```

블랙홀 클러스터의 라우팅 설정이다. ( 매칭 되는 애가 없어서 얘 한테로 오면 바로 502를 리턴)

```json
{
  "name": "block_all",
  "domains": [
    "*"
  ],
  "routes": [
    {
      "match": {
        "prefix": "/"
      },
      "directResponse": {
        "status": 502
      }
    }
  ]
}
```

**4) PassthroughCluster**

PassthroughCluster는 원래 요청이 들어온 곳으로 트래픽을 보내주는 애이다.  

(이름 그대로 그냥 한번 거쳐가는 애)

istio 아웃바운드 모드를 "pod의 아웃바운드 요청을 일단 모두 15001 포트로 보내 주고, 그 다음에 실제 요청된 포트에 맞춰서 요청을 보내주는 구조"로 설정할 수 있다.

이 때 "모든 트래픽을 받아서 원래 요청 들어온 곳으로 보내주는" 서비스(??? 무언가...)가  필요한데, 이럴 때 PassthroughCluster를 사용한다.

```json
 {
    "name": "PassthroughCluster",
    "type": "ORIGINAL_DST",
    "connectTimeout": "10s",
    "lbPolicy": "CLUSTER_PROVIDED",
    "protocolSelection": "USE_DOWNSTREAM_PROTOCOL",
    "filters": [
      {
        "name": "istio.metadata_exchange",
        "typedConfig": {
          "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
          "typeUrl": "type.googleapis.com/envoy.tcp.metadataexchange.config.MetadataExchange",
          "value": {
            "protocol": "istio-peer-exchange"
          }
        }
      }
    ]
  },
```

라우팅 설정은 아래와 같다.

```json
{
  "name": "allow_any",
  "domains": [
    "*"
  ],
  "routes": [
    {
      "match": {
        "prefix": "/"
      },
      "route": {
        "cluster": "PassthroughCluster"
      }
    }
  ]
}
```

### **Listener**

envoy listener는 요청을 수신하고 처리하는 부분이다.

istio에서는 우선 VirtualOutbound Listener를 통해 한 포트(15001)에서 모든 아웃바운드 요청을 수신 한 다음,  실제 요청된 port에 따라 적절한 리스너로 전달하여 개별 처리를 한다.

### **1) VirtualOutbound Listener**

위의 iptables 설정에서 확인한 것처럼, 엔보이는 모든 아웃바운드 요청을 15001 포트로 보낸다.

이 15001 포트를 수신하고 있는 애가 VirtualOutbound 리스너이다.

실제 요청이 들어왔던 port에 맞춰서 적절한 리스너로 요청을 전달해주는 것 빼고는 아무런 처리도 하지 않는다 (그래서 virtual 이라고 붙여졌다고 한다?)

- 엔보이 리스너 설정에서 [use_original_dest가](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/listener_filters/original_dst_filter) true인 경우, request의 원래 목적지 address와 연관된 리스너로 요청이 전달됨

Enovy에서 대상 port의 리스너를 찾지 못하고 요청한 경우 Istio의 `outboundTrafficPolicy` 옵션값에 따라 다르게 처리된다.

- ALLOW_ANY
    - 서비스가 Pilot의 서비스 레지스트리에 있는지 여부에 관계없이 모든 외부 서비스에 대한 요청을 허용 → VirtualOutbound 리스너에 클러스터로 **PassthroughCluster**를 추가한다 (정확히는 envoy.filters.network.tcp_proxy 필터로 등록한다)
    - 일치하는 port의 리스너를 찾을 수없는 요청은 **PassthroughCluster**를 통해 실제 요청 IP로 전송된다
- REGISTRY_ONLY
    - Pilot의 서비스 레지스트리에 있는 서비스에 대해서만 외부 요청을 허용한다. → VirtualOutbound 리스너에 클러스터로 BlackHoleCluster를 등록한다. (정확히는 envoy.filters.network.tcp_proxy 필터로 등록한다)
    - 일치하는 port의 리스너를 찾을 수없는 요청은 BlackHoleCluster를 통해 드랍된다.

예제 서비스의 경우 outboundTrafficPolicy가 ALLOW_ANY로 되어있기 때문에

아래 명령어로 15001번 포트의 listener를 조회하면, VirtualOutbound (0.0.0.0 15001 ALL)에 

destination가 `PassthroughCluster`로 설정된 걸 볼 수 있다.





```
$ istioctl proxy-config listener productpage-v1-64794f5db4-6bsbj --port 15001

ADDRESS PORT  MATCH         DESTINATION
0.0.0.0 15001 ALL           PassthroughCluster
0.0.0.0 15001 Addr: *:15001 Non-HTTP/Non-TCP
```
에레
**2) Outbound Listener**

비즈니스 로직을 처리하는데 필요한 리스너 + istio 자체 컴포넌트 간의 통신을 처리하는데 필요한 리스너가 생성된다.

- 0.0.0.0_9080 : 사용자 앱 (details, reviews, ratings)으로 보내는 요청을 처리
- 0.0.0.0_9411 : Zipkin으로 나가는 요청 처리
- 0.0.0.0_3000 : grafana로 나가는 요청 처리
- ...

사용자 앱 (details, reviews, ratings)으로 보내는 요청을 처리하는 9080 포트 리스너를 조회하고, 

해당 리스너에 라우트(Route: 9080) 설정을 확인하자.

```
$ istioctl proxy-config listener productpage-v1-64794f5db4-6bsbj --port 9080

ADDRESS PORT MATCH                        DESTINATION
0.0.0.0 9080 Trans: raw_buffer; App: HTTP Route: 9080
0.0.0.0 9080 ALL                          PassthroughCluster

$ istioctl proxy-config route productpage-v1-64794f5db4-6bsbj --name 9080

NAME     DOMAINS         MATCH     VIRTUAL SERVICE
9080     details         /*        details.default
9080     productpage     /*        productpage.default
9080     ratings         /*        ratings.default
9080     reviews         /*        reviews.default
```

review, details, ratings 서비스가 각각 도메인에 따라 라우팅 룰이 생성되어 있다.

실제 비즈니스 로직상 productpage가 ratings 서비스를 직접 호출하지 않지만, envoy는 이를 모르기 때문에 ratings 서비스에 대한 라우팅 설정도 추가된 걸 볼 수 있다.

```json
[
    {
        "name": "9080",
        "virtualHosts": [
            {
                "name": "allow_any",
                "domains": [
                    "*"
                ],
                "routes": [
                    {
                        "name": "allow_any",
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "PassthroughCluster",
                            "timeout": "0s",
                            "maxStreamDuration": {
                                "maxStreamDuration": "0s"
                            }
                        }
                    }
                ],
                "includeRequestAttemptCount": true
            },
            {
                "name": "details.default.svc.cluster.local:9080",
                "domains": [
                    "details.default.svc.cluster.local",
                    "details.default.svc.cluster.local:9080",
                    "details",
                    "details:9080",
                    "details.default.svc.cluster",
                    "details.default.svc.cluster:9080",
                    "details.default.svc",
                    "details.default.svc:9080",
                    "details.default",
                    "details.default:9080",
                    "10.231.37.72",
                    "10.231.37.72:9080"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "outbound|9080|v1|details.default.svc.cluster.local",
                            "timeout": "0s",
                            "retryPolicy": {
                                "retryOn": "connect-failure,refused-stream,unavailable,cancelled,retriable-status-codes",
                                "numRetries": 2,
                                "retryHostPredicate": [
                                    {
                                        "name": "envoy.retry_host_predicates.previous_hosts"
                                    }
                                ],
                                "hostSelectionRetryMaxAttempts": "5",
                                "retriableStatusCodes": [
                                    503
                                ]
                            },
                            "maxStreamDuration": {
                                "maxStreamDuration": "0s"
                            }
                        },
                        "metadata": {
                            "filterMetadata": {
                                "istio": {
                                    "config": "/apis/networking.istio.io/v1alpha3/namespaces/default/virtual-service/details"
                                }
                            }
                        },
                        "decorator": {
                            "operation": "details.default.svc.cluster.local:9080/*"
                        }
                    }
                ],
                "includeRequestAttemptCount": true
            },
            {
                "name": "productpage.default.svc.cluster.local:9080",
                "domains": [
                    "productpage.default.svc.cluster.local",
                    "productpage.default.svc.cluster.local:9080",
                    "productpage",
                    "productpage:9080",
                    "productpage.default.svc.cluster",
                    "productpage.default.svc.cluster:9080",
                    "productpage.default.svc",
                    "productpage.default.svc:9080",
                    "productpage.default",
                    "productpage.default:9080",
                    "10.231.37.139",
                    "10.231.37.139:9080"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "outbound|9080|v1|productpage.default.svc.cluster.local",
                            "timeout": "0s",
                            "retryPolicy": {
                                "retryOn": "connect-failure,refused-stream,unavailable,cancelled,retriable-status-codes",
                                "numRetries": 2,
                                "retryHostPredicate": [
                                    {
                                        "name": "envoy.retry_host_predicates.previous_hosts"
                                    }
                                ],
                                "hostSelectionRetryMaxAttempts": "5",
                                "retriableStatusCodes": [
                                    503
                                ]
                            },
                            "maxStreamDuration": {
                                "maxStreamDuration": "0s"
                            }
                        },
                        "metadata": {
                            "filterMetadata": {
                                "istio": {
                                    "config": "/apis/networking.istio.io/v1alpha3/namespaces/default/virtual-service/productpage"
                                }
                            }
                        },
                        "decorator": {
                            "operation": "productpage.default.svc.cluster.local:9080/*"
                        }
                    }
                ],
                "includeRequestAttemptCount": true
            },
            {
                "name": "ratings.default.svc.cluster.local:9080",
                "domains": [
                    "ratings.default.svc.cluster.local",
                    "ratings.default.svc.cluster.local:9080",
                    "ratings",
                    "ratings:9080",
                    "ratings.default.svc.cluster",
                    "ratings.default.svc.cluster:9080",
                    "ratings.default.svc",
                    "ratings.default.svc:9080",
                    "ratings.default",
                    "ratings.default:9080",
                    "10.231.20.222",
                    "10.231.20.222:9080"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "outbound|9080|v1|ratings.default.svc.cluster.local",
                            "timeout": "0s",
                            "retryPolicy": {
                                "retryOn": "connect-failure,refused-stream,unavailable,cancelled,retriable-status-codes",
                                "numRetries": 2,
                                "retryHostPredicate": [
                                    {
                                        "name": "envoy.retry_host_predicates.previous_hosts"
                                    }
                                ],
                                "hostSelectionRetryMaxAttempts": "5",
                                "retriableStatusCodes": [
                                    503
                                ]
                            },
                            "maxStreamDuration": {
                                "maxStreamDuration": "0s"
                            }
                        },
                        "metadata": {
                            "filterMetadata": {
                                "istio": {
                                    "config": "/apis/networking.istio.io/v1alpha3/namespaces/default/virtual-service/ratings"
                                }
                            }
                        },
                        "decorator": {
                            "operation": "ratings.default.svc.cluster.local:9080/*"
                        },
                        "typedPerFilterConfig": {
                            "envoy.filters.http.fault": {
                                "@type": "type.googleapis.com/envoy.extensions.filters.http.fault.v3.HTTPFault",
                                "delay": {
                                    "fixedDelay": "2s",
                                    "percentage": {
                                        "numerator": 100
                                    }
                                }
                            }
                        }
                    }
                ],
                "includeRequestAttemptCount": true
            },
            {
                "name": "reviews.default.svc.cluster.local:9080",
                "domains": [
                    "reviews.default.svc.cluster.local",
                    "reviews.default.svc.cluster.local:9080",
                    "reviews",
                    "reviews:9080",
                    "reviews.default.svc.cluster",
                    "reviews.default.svc.cluster:9080",
                    "reviews.default.svc",
                    "reviews.default.svc:9080",
                    "reviews.default",
                    "reviews.default:9080",
                    "10.231.20.47",
                    "10.231.20.47:9080"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "outbound|9080|v2|reviews.default.svc.cluster.local",
                            "timeout": "0.500s",
                            "retryPolicy": {
                                "retryOn": "connect-failure,refused-stream,unavailable,cancelled,retriable-status-codes",
                                "numRetries": 2,
                                "retryHostPredicate": [
                                    {
                                        "name": "envoy.retry_host_predicates.previous_hosts"
                                    }
                                ],
                                "hostSelectionRetryMaxAttempts": "5",
                                "retriableStatusCodes": [
                                    503
                                ]
                            },
                            "maxStreamDuration": {
                                "maxStreamDuration": "0.500s"
                            }
                        },
                        "metadata": {
                            "filterMetadata": {
                                "istio": {
                                    "config": "/apis/networking.istio.io/v1alpha3/namespaces/default/virtual-service/reviews"
                                }
                            }
                        },
                        "decorator": {
                            "operation": "reviews.default.svc.cluster.local:9080/*"
                        }
                    }
                ],
                "includeRequestAttemptCount": true
            }
        ],
        "validateClusters": false
    }
]

```

**3) VirtualInbound Listener & Inbound Listener**

예전 버전에서는 15001 포트에서 인바운드/아웃바운드 요청을 같이 처리하는 구조였다고 한다.

(무한 루프가 발생할 경우가 있어서 각각 포트를 따로 두는 걸로 수정했다고 하는데 자세히는 잘 모르겠따 )

15006 포트를 수신하는 리스터를 살펴보자.

VirtualInbound Listener로 들어온 요청은 필터 설정에 의해  `inbound|9080||` 클러스터로 연결되고

(DESTINATION = "inbound|9080||")

"inbound|9080||" 클러스터의 업스트립 호스트는 `127.0.0.1:9080`이다  

→ 즉 pod에 떠있는 사용자 컨테이너인 productpage로 요청이 바로 간다.

```
$ ist proxy-config listener $envoy --port 15006

ADDRESS PORT  MATCH                                           DESTINATION
0.0.0.0 15006 Addr: *:15006                                   Non-HTTP/Non-TCP
0.0.0.0 15006 Trans: tls; App: TCP TLS; Addr: 0.0.0.0/0       InboundPassthroughClusterIpv4
0.0.0.0 15006 Trans: raw_buffer; Addr: 0.0.0.0/0              InboundPassthroughClusterIpv4
0.0.0.0 15006 Trans: tls; App: HTTP TLS; Addr: 0.0.0.0/0      InboundPassthroughClusterIpv4
0.0.0.0 15006 Trans: raw_buffer; App: HTTP; Addr: 0.0.0.0/0   InboundPassthroughClusterIpv4
0.0.0.0 15006 Trans: tls; App: Istio HTTP Plain; Addr: *:9080 Cluster: inbound|9080||
0.0.0.0 15006 Trans: raw_buffer; Addr: *:9080                 Cluster: inbound|9080||
```

```json
[
    {
        "name": "inbound|9080||",
        "addedViaApi": true,
        "hostStatuses": [
            {
                "address": {
                    "socketAddress": {
                        "address": "127.0.0.1",
                        "portValue": 9080
                    }
                },
                "stats": [
                    {
                        "name": "cx_connect_fail"
                    },
                    {
                        "value": "251",
                        "name": "cx_total"
                    },
                    {
                        "name": "rq_error"
                    },
                    {
                        "value": "250",
                        "name": "rq_success"
                    },
                    {
                        "name": "rq_timeout"
                    },
                    {
                        "value": "250",
                        "name": "rq_total"
                    },
                    {
                        "type": "GAUGE",
                        "name": "cx_active"
                    },
                    {
                        "type": "GAUGE",
                        "name": "rq_active"
                    }
                ],
                "healthStatus": {
                    "edsHealthStatus": "HEALTHY"
                },
                "weight": 1,
                "locality": {}
            }
        ],
        "circuitBreakers": {
            "thresholds": [
                {
                    "maxConnections": 4294967295,
                    "maxPendingRequests": 4294967295,
                    "maxRequests": 4294967295,
                    "maxRetries": 4294967295
                },
                {
                    "priority": "HIGH",
                    "maxConnections": 1024,
                    "maxPendingRequests": 1024,
                    "maxRequests": 1024,
                    "maxRetries": 3
                }
            ]
        }
    }
]
```

## 실제 서비스 간의 트래픽 흐름

![https://zhaohuabing.com/img/2019-12-05-istio-traffic-management-impl-intro/envoy-traffic-route.png](https://zhaohuabing.com/img/2019-12-05-istio-traffic-management-impl-intro/envoy-traffic-route.png)

1. productpage에서 review service (v1) 호출 `[http://reviews:9080/reviews/](http://reviews:9080/reviews/0) ..`
2. 이 요청은 productpage pod의 iptable 룰에 의해 로컬 포트 **15001(outbound)로 리다이렉션**됨
3. (prodectpage pod의 엔보이가 기준이다) 포트 15001에서 수신하는 엔보이 **Virtual Outbound Listener**로 요청이 수신됨
4. 이후 요청은 **Virtual Outbound Listener의 route 설정**에 따라 원래 대상 ip 및 포트 (9080)를 기반으로 아웃 바운드 리스너 0.0.0.0_9080으로 전달됨
5. **0.0.0.0_9080 Listener의 route(name=9080) 설정**에서, 도메인 "reviews.default.svc.cluster.local"랑 맵핑되는 클러스터는 "outbound|9080||reviews.default.svc.cluster.local" 이다
6. **outbound|9080||reviews.default.svc.cluster.local** 클러스터의 endpoint를 조회(eds query)한다 (reviews v1,v2,v3 3개 Pod의 ip:9080이 나올 것이다)
7. 이 중 **subset=v1인 10.240.2.40:8090 엔드포인트**(review v1 pod)로 요청을 보내게 된다.
8. reviews v1 pod으로 들어온 요청은 iptable 룰에 의해 **15006(inbound) 포트로 리다이렉션** 된다.
9. (이제 reviews v1 pod의 엔보이가 기준이다) 포트 15006을 수신하는 엔보이 **Virtual Inbound Listener**로 요청이 수신됨
10. Virtual Inbound Listener에 연결된 필터 설정에 의해 **"inbound|9080||" 클러스터**의 호스트로 요청이 전달된다.
11. 이때, "inbound|9080||" 클러스터의 호스트는 **"127.0.0.1:9080"** 이다.
12. 실제 사용자 앱인 **reviews (v1)** 컨테이너로 요청이 전달된다.

---

**참고**

[https://ssup2.github.io/theory_analysis/Istio_Sidecar/](https://ssup2.github.io/theory_analysis/Istio_Sidecar/)

[https://jimmysong.io/en/blog/sidecar-injection-iptables-and-traffic-routing/](https://jimmysong.io/en/blog/sidecar-injection-iptables-and-traffic-routing/)

[https://zhaohuabing.com/post/2018-09-25-istio-traffic-management-impl-intro/#virtual-listener](https://zhaohuabing.com/post/2018-09-25-istio-traffic-management-impl-intro/#virtual-listener)
