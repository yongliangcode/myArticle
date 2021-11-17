

```
title: Mesh4# Envoy调试
categories: Mesh
tags: Mesh
date: 2021-10-15 11:55:01
```



# 引言



### 检查Envoy是否接受配置



使用proxy-status命令，下面是解释：即获取每个Envoy的同步状态，正确状态为SYNCED

```
proxy-status   Retrieves the synchronization status of each Envoy in the mesh [kube only]
```



```
istioctl proxy-status
NAME                                                   CDS        LDS        EDS        RDS        ISTIOD                      VERSION
details-v1-79f774bdb9-bkrbp.default                    SYNCED     SYNCED     SYNCED     SYNCED     istiod-5c4b9cb6b5-6n68m     1.11.0
dubbo-sample-consumer-5778c77995-5kvmf.default         SYNCED     SYNCED     SYNCED     SYNCED     istiod-5c4b9cb6b5-6n68m     1.10.4
dubbo-sample-consumer-5778c77995-dw99q.dubbo           SYNCED     SYNCED     SYNCED     SYNCED     istiod-5c4b9cb6b5-6n68m     1.10.4
dubbo-sample-provider-v1-d5668f85c-fcpm5.default       SYNCED     SYNCED     SYNCED     SYNCED     istiod-5c4b9cb6b5-6n68m     1.10.4
dubbo-sample-provider-v1-d5668f85c-h7985.dubbo         SYNCED     SYNCED     SYNCED     SYNCED     istiod-5c4b9cb6b5-6n68m     1.10.4
dubbo-sample-provider-v2-65995864f6-dzkqz.dubbo        SYNCED     SYNCED     SYNCED     SYNCED     istiod-5c4b9cb6b5-6n68m     1.10.4
dubbo-sample-provider-v2-65995864f6-z77fg.default      SYNCED     SYNCED     SYNCED     SYNCED     istiod-5c4b9cb6b5-6n68m     1.10.4
dubbo2istio-7488477bd4-vpfn8.dubbo                     SYNCED     SYNCED     SYNCED     SYNCED     istiod-5c4b9cb6b5-6n68m     1.10.4
istio-ingressgateway-7bd4d65f89-9wbsz.istio-system     SYNCED     SYNCED     SYNCED     SYNCED     istiod-5c4b9cb6b5-6n68m     1.10.4
productpage-v1-6b746f74dc-2c55l.default                SYNCED     SYNCED     SYNCED     SYNCED     istiod-5c4b9cb6b5-6n68m     1.11.0
ratings-v1-b6994bb9-7nvs2.default                      SYNCED     SYNCED     SYNCED     SYNCED     istiod-5c4b9cb6b5-6n68m     1.11.0
reviews-v1-545db77b95-mffvg.default                    SYNCED     SYNCED     SYNCED     SYNCED     istiod-5c4b9cb6b5-6n68m     1.11.0
reviews-v2-7bf8c9648f-pmqw8.default                    SYNCED     SYNCED     SYNCED     SYNCED     istiod-5c4b9cb6b5-6n68m     1.11.0
reviews-v3-84779c7bbc-sztp8.default                    SYNCED     SYNCED     SYNCED     SYNCED     istiod-5c4b9cb6b5-6n68m     1.11.0
zookeeper-5dbc74b5db-bj8bz.dubbo                       SYNCED     SYNCED     SYNCED     SYNCED     istiod-5c4b9cb6b5-6n68m     1.10.4
```



### Envoy检测不同的配置信息

**1.检测集群配置** 

```
istioctl proxy-config cluster productpage-v1-6b746f74dc-2c55l -n default

SERVICE FQDN                                                        PORT      SUBSET     DIRECTION     TYPE             DESTINATION RULE
                                                                    9080      -          inbound       ORIGINAL_DST     
BlackHoleCluster                                                    -         -          -             STATIC           
InboundPassthroughClusterIpv4                                       -         -          -             ORIGINAL_DST     
PassthroughCluster                                                  -         -          -             ORIGINAL_DST     
agent                                                               -         -          -             STATIC           
details.default.svc.cluster.local                                   9080      -          outbound      EDS              
grafana.istio-system.svc.cluster.local                              3000      -          outbound      EDS              
istio-ingressgateway.istio-system.svc.cluster.local                 80        -          outbound      EDS              
istio-ingressgateway.istio-system.svc.cluster.local                 443       -          outbound      EDS              
istio-ingressgateway.istio-system.svc.cluster.local                 15021     -          outbound      EDS              
istio-operator.istio-operator.svc.cluster.local                     8383      -          outbound      EDS              
istiod.istio-system.svc.cluster.local                               443       -          outbound      EDS              
istiod.istio-system.svc.cluster.local                               15010     -          outbound      EDS              
istiod.istio-system.svc.cluster.local                               15012     -          outbound      EDS              
istiod.istio-system.svc.cluster.local                               15014     -          outbound      EDS              
jaeger-collector.istio-system.svc.cluster.local                     14250     -          outbound      EDS              
jaeger-collector.istio-system.svc.cluster.local                     14268     -          outbound      EDS              
kiali.istio-system.svc.cluster.local                                9090      -          outbound      EDS              
kiali.istio-system.svc.cluster.local                                20001     -          outbound      EDS              
kube-dns.kube-system.svc.cluster.local                              53        -          outbound      EDS              
kube-dns.kube-system.svc.cluster.local                              9153      -          outbound      EDS              
kubernetes.default.svc.cluster.local                                443       -          outbound      EDS              
org.apache.dubbo.samples.basic.api.complexservice-product-2-0-0     20880     -          outbound      EDS              
org.apache.dubbo.samples.basic.api.complexservice-test-1-0-0        20880     -          outbound      EDS              
org.apache.dubbo.samples.basic.api.demoservice                      20880     -          outbound      EDS              
org.apache.dubbo.samples.basic.api.testservice                      20880     -          outbound      EDS              
productpage.default.svc.cluster.local                               9080      -          outbound      EDS              
prometheus.istio-system.svc.cluster.local                           9090      -          outbound      EDS              
prometheus_stats                                                    -         -          -             STATIC           
ratings.default.svc.cluster.local                                   9080      -          outbound      EDS              
reviews.default.svc.cluster.local                                   9080      -          outbound      EDS              
sds-grpc                                                            -         -          -             STATIC           
tracing.istio-system.svc.cluster.local                              80        -          outbound      EDS              
xds-grpc                                                            -         -          -             STATIC           
zipkin                                                              -         -          -             STRICT_DNS       
zipkin.istio-system.svc.cluster.local                               9411      -          outbound      EDS              
zookeeper.dubbo.svc.cluster.local                                   2181      -          outbound      EDS
```



**2.检索bootstrap配置**

```
istioctl proxy-config bootstrap productpage-v1-6b746f74dc-2c55l -n default

{
    "bootstrap": {
        "node": {
            "id": "sidecar~172.17.0.17~productpage-v1-6b746f74dc-2c55l.default~default.svc.cluster.local",
            "cluster": "productpage.default",
            "metadata": {
                    "ANNOTATIONS": {
                                "kubectl.kubernetes.io/default-container": "productpage",
                                "kubectl.kubernetes.io/default-logs-container": "productpage",
                                "kubernetes.io/config.seen": "2021-10-09T06:09:15.665633130Z",
                                "kubernetes.io/config.source": "api",
                                "prometheus.io/path": "/stats/prometheus",
                                "prometheus.io/port": "15020",
                                "prometheus.io/scrape": "true",
                                "sidecar.istio.io/status": "{\"initContainers\":[\"istio-init\"],\"containers\":[\"istio-proxy\"],\"volumes\":[\"istio-envoy\",\"istio-data\",\"istio-podinfo\",\"istio-token\",\"istiod-ca-cert\"],\"imagePullSecrets\":null,\"revision\":\"default\"}"
                            },
                    "APP_CONTAINERS": "productpage",
                    "CLUSTER_ID": "Kubernetes",
                    "ENVOY_PROMETHEUS_PORT": 15090,
                    "ENVOY_STATUS_PORT": 15021,
                    "INSTANCE_IPS": "172.17.0.17",
                    "INTERCEPTION_MODE": "REDIRECT",
                    "ISTIO_PROXY_SHA": "istio-proxy:494a674e70543a319ad4865482c125581f5746bf",
                    "ISTIO_VERSION": "1.11.0",
                    "LABELS": {
                                "app": "productpage",
                                "pod-template-hash": "6b746f74dc",
                                "security.istio.io/tlsMode": "istio",
                                "service.istio.io/canonical-name": "productpage",
                                "service.istio.io/canonical-revision": "v1",
                                "version": "v1"
                            },
                    "MESH_ID": "cluster.local",
                    "NAME": "productpage-v1-6b746f74dc-2c55l",
                    "NAMESPACE": "default",
                    "OWNER": "kubernetes://apis/apps/v1/namespaces/default/deployments/productpage-v1",
                    "PILOT_SAN": [
                                "istiod.istio-system.svc"
                            ],
                    "POD_PORTS": "[{\"containerPort\":9080,\"protocol\":\"TCP\"}]",
                    "PROV_CERT": "var/run/secrets/istio/root-cert.pem",
                    "PROXY_CONFIG": {
                                "binaryPath": "/usr/local/bin/envoy",
                                "concurrency": 2,
                                "configPath": "./etc/istio/proxy",
                                "controlPlaneAuthPolicy": "MUTUAL_TLS",
                                "discoveryAddress": "istiod.istio-system.svc:15012",
                                "drainDuration": "45s",
                                "parentShutdownDuration": "60s",
                                "proxyAdminPort": 15000,
                                "serviceCluster": "istio-proxy",
                                "statNameLength": 189,
                                "statusPort": 15020,
                                "terminationDrainDuration": "5s",
                                "tracing": {
                                            "zipkin": {
                                                        "address": "zipkin.istio-system:9411"
                                                    }
                                        }
                            },
                    "PROXY_VIA_AGENT": true,
                    "SERVICE_ACCOUNT": "bookinfo-productpage",
                    "WORKLOAD_NAME": "productpage-v1"
                },
            "locality": {

            },
            "userAgentName": "envoy",
            "userAgentBuildVersion": {
                "version": {
                    "majorNumber": 1,
                    "minorNumber": 19
                },
                "metadata": {
                        "build.type": "RELEASE",
                        "revision.sha": "494a674e70543a319ad4865482c125581f5746bf",
                        "revision.status": "Clean",
                        "ssl.version": "BoringSSL"
                    }
            },
            "extensions": [
                {
                    "name": "envoy.extensions.upstreams.http.v3.HttpProtocolOptions",
                    "category": "envoy.upstream_options"
                },
                {
                    "name": "envoy.upstreams.http.http_protocol_options",
                    "category": "envoy.upstream_options"
                },
                {
                    "name": "envoy.filters.thrift.rate_limit",
                    "category": "envoy.thrift_proxy.filters"
                },
                {
                    "name": "envoy.filters.thrift.router",
                    "category": "envoy.thrift_proxy.filters"
                },
                {
                    "name": "envoy.wasm.runtime.null",
                    "category": "envoy.wasm.runtime"
                },
                {
                    "name": "envoy.wasm.runtime.v8",
                    "category": "envoy.wasm.runtime"
                },
                {
                    "name": "envoy.tls.cert_validator.default",
                    "category": "envoy.tls.cert_validator"
                },
                {
                    "name": "envoy.tls.cert_validator.spiffe",
                    "category": "envoy.tls.cert_validator"
                },
                {
                    "name": "dubbo.hessian2",
                    "category": "envoy.dubbo_proxy.serializers"
                },
                {
                    "name": "envoy.filters.listener.http_inspector",
                    "category": "envoy.filters.listener"
                },
                {
                    "name": "envoy.filters.listener.original_dst",
                    "category": "envoy.filters.listener"
                },
                {
                    "name": "envoy.filters.listener.original_src",
                    "category": "envoy.filters.listener"
                },
                {
                    "name": "envoy.filters.listener.proxy_protocol",
                    "category": "envoy.filters.listener"
                },
                {
                    "name": "envoy.filters.listener.tls_inspector",
                    "category": "envoy.filters.listener"
                },
                {
                    "name": "envoy.listener.http_inspector",
                    "category": "envoy.filters.listener"
                },
                {
                    "name": "envoy.listener.original_dst",
                    "category": "envoy.filters.listener"
                },
                {
                    "name": "envoy.listener.original_src",
                    "category": "envoy.filters.listener"
                },
                {
                    "name": "envoy.listener.proxy_protocol",
                    "category": "envoy.filters.listener"
                },
                {
                    "name": "envoy.listener.tls_inspector",
                    "category": "envoy.filters.listener"
                },
                {
                    "name": "envoy.quic.crypto_stream.server.quiche",
                    "category": "envoy.quic.server.crypto_stream"
                },
                {
                    "name": "composite-action",
                    "category": "envoy.matching.action"
                },
                {
                    "name": "skip",
                    "category": "envoy.matching.action"
                },
                {
                    "name": "envoy.filters.dubbo.router",
                    "category": "envoy.dubbo_proxy.filters"
                },
                {
                    "name": "request-headers",
                    "category": "envoy.matching.http.input"
                },
                {
                    "name": "request-trailers",
                    "category": "envoy.matching.http.input"
                },
                {
                    "name": "response-headers",
                    "category": "envoy.matching.http.input"
                },
                {
                    "name": "response-trailers",
                    "category": "envoy.matching.http.input"
                },
                {
                    "name": "envoy.formatter.req_without_query",
                    "category": "envoy.formatter"
                },
                {
                    "name": "envoy.request_id.uuid",
                    "category": "envoy.request_id"
                },
                {
                    "name": "envoy.quic.proof_source.filter_chain",
                    "category": "envoy.quic.proof_source"
                },
                {
                    "name": "envoy.watchdog.abort_action",
                    "category": "envoy.guarddog_actions"
                },
                {
                    "name": "envoy.watchdog.profile_action",
                    "category": "envoy.guarddog_actions"
                },
                {
                    "name": "envoy.filters.network.upstream.metadata_exchange",
                    "category": "envoy.filters.upstream_network"
                },
                {
                    "name": "envoy.bootstrap.wasm",
                    "category": "envoy.bootstrap"
                },
                {
                    "name": "envoy.extensions.network.socket_interface.default_socket_interface",
                    "category": "envoy.bootstrap"
                },
                {
                    "name": "envoy.rate_limit_descriptors.expr",
                    "category": "envoy.rate_limit_descriptors"
                },
                {
                    "name": "envoy.transport_sockets.alts",
                    "category": "envoy.transport_sockets.downstream"
                },
                {
                    "name": "envoy.transport_sockets.quic",
                    "category": "envoy.transport_sockets.downstream"
                },
                {
                    "name": "envoy.transport_sockets.raw_buffer",
                    "category": "envoy.transport_sockets.downstream"
                },
                {
                    "name": "envoy.transport_sockets.starttls",
                    "category": "envoy.transport_sockets.downstream"
                },
                {
                    "name": "envoy.transport_sockets.tap",
                    "category": "envoy.transport_sockets.downstream"
                },
                {
                    "name": "envoy.transport_sockets.tls",
                    "category": "envoy.transport_sockets.downstream"
                },
                {
                    "name": "raw_buffer",
                    "category": "envoy.transport_sockets.downstream"
                },
                {
                    "name": "starttls",
                    "category": "envoy.transport_sockets.downstream"
                },
                {
                    "name": "tls",
                    "category": "envoy.transport_sockets.downstream"
                },
                {
                    "name": "envoy.resource_monitors.fixed_heap",
                    "category": "envoy.resource_monitors"
                },
                {
                    "name": "envoy.resource_monitors.injected_resource",
                    "category": "envoy.resource_monitors"
                },
                {
                    "name": "envoy.matching.matchers.consistent_hashing",
                    "category": "envoy.matching.input_matchers"
                },
                {
                    "name": "envoy.matching.matchers.ip",
                    "category": "envoy.matching.input_matchers"
                },
                {
                    "name": "envoy.filters.udp.dns_filter",
                    "category": "envoy.filters.udp_listener"
                },
                {
                    "name": "envoy.filters.udp_listener.udp_proxy",
                    "category": "envoy.filters.udp_listener"
                },
                {
                    "name": "envoy.compression.brotli.decompressor",
                    "category": "envoy.compression.decompressor"
                },
                {
                    "name": "envoy.compression.gzip.decompressor",
                    "category": "envoy.compression.decompressor"
                },
                {
                    "name": "envoy.access_loggers.file",
                    "category": "envoy.access_loggers"
                },
                {
                    "name": "envoy.access_loggers.http_grpc",
                    "category": "envoy.access_loggers"
                },
                {
                    "name": "envoy.access_loggers.open_telemetry",
                    "category": "envoy.access_loggers"
                },
                {
                    "name": "envoy.access_loggers.stderr",
                    "category": "envoy.access_loggers"
                },
                {
                    "name": "envoy.access_loggers.stdout",
                    "category": "envoy.access_loggers"
                },
                {
                    "name": "envoy.access_loggers.tcp_grpc",
                    "category": "envoy.access_loggers"
                },
                {
                    "name": "envoy.access_loggers.wasm",
                    "category": "envoy.access_loggers"
                },
                {
                    "name": "envoy.file_access_log",
                    "category": "envoy.access_loggers"
                },
                {
                    "name": "envoy.http_grpc_access_log",
                    "category": "envoy.access_loggers"
                },
                {
                    "name": "envoy.open_telemetry_access_log",
                    "category": "envoy.access_loggers"
                },
                {
                    "name": "envoy.stderr_access_log",
                    "category": "envoy.access_loggers"
                },
                {
                    "name": "envoy.stdout_access_log",
                    "category": "envoy.access_loggers"
                },
                {
                    "name": "envoy.tcp_grpc_access_log",
                    "category": "envoy.access_loggers"
                },
                {
                    "name": "envoy.wasm_access_log",
                    "category": "envoy.access_loggers"
                },
                {
                    "name": "envoy.grpc_credentials.aws_iam",
                    "category": "envoy.grpc_credentials"
                },
                {
                    "name": "envoy.grpc_credentials.default",
                    "category": "envoy.grpc_credentials"
                },
                {
                    "name": "envoy.grpc_credentials.file_based_metadata",
                    "category": "envoy.grpc_credentials"
                },
                {
                    "name": "envoy.compression.brotli.compressor",
                    "category": "envoy.compression.compressor"
                },
                {
                    "name": "envoy.compression.gzip.compressor",
                    "category": "envoy.compression.compressor"
                },
                {
                    "name": "envoy.retry_priorities.previous_priorities",
                    "category": "envoy.retry_priorities"
                },
                {
                    "name": "envoy.internal_redirect_predicates.allow_listed_routes",
                    "category": "envoy.internal_redirect_predicates"
                },
                {
                    "name": "envoy.internal_redirect_predicates.previous_routes",
                    "category": "envoy.internal_redirect_predicates"
                },
                {
                    "name": "envoy.internal_redirect_predicates.safe_cross_scheme",
                    "category": "envoy.internal_redirect_predicates"
                },
                {
                    "name": "envoy.bandwidth_limit",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.buffer",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.cors",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.csrf",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.ext_authz",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.ext_proc",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.fault",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.adaptive_concurrency",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.admission_control",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.alternate_protocols_cache",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.aws_lambda",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.aws_request_signing",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.bandwidth_limit",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.buffer",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.cache",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.cdn_loop",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.composite",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.compressor",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.cors",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.csrf",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.decompressor",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.dynamic_forward_proxy",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.dynamo",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.ext_authz",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.ext_proc",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.fault",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.grpc_http1_bridge",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.grpc_http1_reverse_bridge",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.grpc_json_transcoder",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.grpc_stats",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.grpc_web",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.header_to_metadata",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.health_check",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.ip_tagging",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.jwt_authn",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.local_ratelimit",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.lua",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.oauth2",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.on_demand",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.original_src",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.ratelimit",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.rbac",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.router",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.set_metadata",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.squash",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.tap",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.filters.http.wasm",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.grpc_http1_bridge",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.grpc_json_transcoder",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.grpc_web",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.health_check",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.http_dynamo_filter",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.ip_tagging",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.local_rate_limit",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.lua",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.rate_limit",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.router",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "envoy.squash",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "istio.alpn",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "istio_authn",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "match-wrapper",
                    "category": "envoy.filters.http"
                },
                {
                    "name": "preserve_case",
                    "category": "envoy.http.stateful_header_formatters"
                },
                {
                    "name": "auto",
                    "category": "envoy.thrift_proxy.protocols"
                },
                {
                    "name": "binary",
                    "category": "envoy.thrift_proxy.protocols"
                },
                {
                    "name": "binary/non-strict",
                    "category": "envoy.thrift_proxy.protocols"
                },
                {
                    "name": "compact",
                    "category": "envoy.thrift_proxy.protocols"
                },
                {
                    "name": "twitter",
                    "category": "envoy.thrift_proxy.protocols"
                },
                {
                    "name": "envoy.matching.common_inputs.environment_variable",
                    "category": "envoy.matching.common_inputs"
                },
                {
                    "name": "envoy.retry_host_predicates.omit_canary_hosts",
                    "category": "envoy.retry_host_predicates"
                },
                {
                    "name": "envoy.retry_host_predicates.omit_host_metadata",
                    "category": "envoy.retry_host_predicates"
                },
                {
                    "name": "envoy.retry_host_predicates.previous_hosts",
                    "category": "envoy.retry_host_predicates"
                },
                {
                    "name": "envoy.client_ssl_auth",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.echo",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.ext_authz",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.client_ssl_auth",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.connection_limit",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.direct_response",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.dubbo_proxy",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.echo",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.ext_authz",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.http_connection_manager",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.kafka_broker",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.local_ratelimit",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.metadata_exchange",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.mongo_proxy",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.mysql_proxy",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.postgres_proxy",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.ratelimit",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.rbac",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.redis_proxy",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.rocketmq_proxy",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.sni_cluster",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.sni_dynamic_forward_proxy",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.tcp_cluster_rewrite",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.tcp_proxy",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.thrift_proxy",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.wasm",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.filters.network.zookeeper_proxy",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.http_connection_manager",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.mongo_proxy",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.ratelimit",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.redis_proxy",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.tcp_proxy",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "forward_downstream_sni",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "sni_verifier",
                    "category": "envoy.filters.network"
                },
                {
                    "name": "envoy.cluster.eds",
                    "category": "envoy.clusters"
                },
                {
                    "name": "envoy.cluster.logical_dns",
                    "category": "envoy.clusters"
                },
                {
                    "name": "envoy.cluster.original_dst",
                    "category": "envoy.clusters"
                },
                {
                    "name": "envoy.cluster.static",
                    "category": "envoy.clusters"
                },
                {
                    "name": "envoy.cluster.strict_dns",
                    "category": "envoy.clusters"
                },
                {
                    "name": "envoy.clusters.aggregate",
                    "category": "envoy.clusters"
                },
                {
                    "name": "envoy.clusters.dynamic_forward_proxy",
                    "category": "envoy.clusters"
                },
                {
                    "name": "envoy.clusters.redis",
                    "category": "envoy.clusters"
                },
                {
                    "name": "default",
                    "category": "envoy.dubbo_proxy.route_matchers"
                },
                {
                    "name": "envoy.http.original_ip_detection.custom_header",
                    "category": "envoy.http.original_ip_detection"
                },
                {
                    "name": "envoy.http.original_ip_detection.xff",
                    "category": "envoy.http.original_ip_detection"
                },
                {
                    "name": "envoy.transport_sockets.alts",
                    "category": "envoy.transport_sockets.upstream"
                },
                {
                    "name": "envoy.transport_sockets.quic",
                    "category": "envoy.transport_sockets.upstream"
                },
                {
                    "name": "envoy.transport_sockets.raw_buffer",
                    "category": "envoy.transport_sockets.upstream"
                },
                {
                    "name": "envoy.transport_sockets.starttls",
                    "category": "envoy.transport_sockets.upstream"
                },
                {
                    "name": "envoy.transport_sockets.tap",
                    "category": "envoy.transport_sockets.upstream"
                },
                {
                    "name": "envoy.transport_sockets.tls",
                    "category": "envoy.transport_sockets.upstream"
                },
                {
                    "name": "envoy.transport_sockets.upstream_proxy_protocol",
                    "category": "envoy.transport_sockets.upstream"
                },
                {
                    "name": "raw_buffer",
                    "category": "envoy.transport_sockets.upstream"
                },
                {
                    "name": "starttls",
                    "category": "envoy.transport_sockets.upstream"
                },
                {
                    "name": "tls",
                    "category": "envoy.transport_sockets.upstream"
                },
                {
                    "name": "envoy.extensions.http.cache.simple",
                    "category": "envoy.http.cache"
                },
                {
                    "name": "dubbo",
                    "category": "envoy.dubbo_proxy.protocols"
                },
                {
                    "name": "envoy.filters.connection_pools.tcp.generic",
                    "category": "envoy.upstreams"
                },
                {
                    "name": "envoy.dynamic.ot",
                    "category": "envoy.tracers"
                },
                {
                    "name": "envoy.lightstep",
                    "category": "envoy.tracers"
                },
                {
                    "name": "envoy.tracers.datadog",
                    "category": "envoy.tracers"
                },
                {
                    "name": "envoy.tracers.dynamic_ot",
                    "category": "envoy.tracers"
                },
                {
                    "name": "envoy.tracers.lightstep",
                    "category": "envoy.tracers"
                },
                {
                    "name": "envoy.tracers.opencensus",
                    "category": "envoy.tracers"
                },
                {
                    "name": "envoy.tracers.skywalking",
                    "category": "envoy.tracers"
                },
                {
                    "name": "envoy.tracers.xray",
                    "category": "envoy.tracers"
                },
                {
                    "name": "envoy.tracers.zipkin",
                    "category": "envoy.tracers"
                },
                {
                    "name": "envoy.zipkin",
                    "category": "envoy.tracers"
                },
                {
                    "name": "envoy.dog_statsd",
                    "category": "envoy.stats_sinks"
                },
                {
                    "name": "envoy.graphite_statsd",
                    "category": "envoy.stats_sinks"
                },
                {
                    "name": "envoy.metrics_service",
                    "category": "envoy.stats_sinks"
                },
                {
                    "name": "envoy.stat_sinks.dog_statsd",
                    "category": "envoy.stats_sinks"
                },
                {
                    "name": "envoy.stat_sinks.graphite_statsd",
                    "category": "envoy.stats_sinks"
                },
                {
                    "name": "envoy.stat_sinks.hystrix",
                    "category": "envoy.stats_sinks"
                },
                {
                    "name": "envoy.stat_sinks.metrics_service",
                    "category": "envoy.stats_sinks"
                },
                {
                    "name": "envoy.stat_sinks.statsd",
                    "category": "envoy.stats_sinks"
                },
                {
                    "name": "envoy.stat_sinks.wasm",
                    "category": "envoy.stats_sinks"
                },
                {
                    "name": "envoy.statsd",
                    "category": "envoy.stats_sinks"
                },
                {
                    "name": "auto",
                    "category": "envoy.thrift_proxy.transports"
                },
                {
                    "name": "framed",
                    "category": "envoy.thrift_proxy.transports"
                },
                {
                    "name": "header",
                    "category": "envoy.thrift_proxy.transports"
                },
                {
                    "name": "unframed",
                    "category": "envoy.thrift_proxy.transports"
                },
                {
                    "name": "envoy.ip",
                    "category": "envoy.resolvers"
                },
                {
                    "name": "envoy.health_checkers.redis",
                    "category": "envoy.health_checkers"
                }
            ],
            "hiddenEnvoyDeprecatedBuildVersion": "494a674e70543a319ad4865482c125581f5746bf/1.19.0/Clean/RELEASE/BoringSSL"
        },
        "staticResources": {
            "listeners": [
                {
                    "address": {
                        "socketAddress": {
                            "address": "0.0.0.0",
                            "portValue": 15090
                        }
                    },
                    "filterChains": [
                        {
                            "filters": [
                                {
                                    "name": "envoy.filters.network.http_connection_manager",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                                        "statPrefix": "stats",
                                        "routeConfig": {
                                            "virtualHosts": [
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
                                        "httpFilters": [
                                            {
                                                "name": "envoy.filters.http.router",
                                                "typedConfig": {
                                                    "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
                                                }
                                            }
                                        ]
                                    }
                                }
                            ]
                        }
                    ]
                },
                {
                    "address": {
                        "socketAddress": {
                            "address": "0.0.0.0",
                            "portValue": 15021
                        }
                    },
                    "filterChains": [
                        {
                            "filters": [
                                {
                                    "name": "envoy.filters.network.http_connection_manager",
                                    "typedConfig": {
                                        "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                                        "statPrefix": "agent",
                                        "routeConfig": {
                                            "virtualHosts": [
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
                                        "httpFilters": [
                                            {
                                                "name": "envoy.filters.http.router",
                                                "typedConfig": {
                                                    "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
                                                }
                                            }
                                        ]
                                    }
                                }
                            ]
                        }
                    ]
                }
            ],
            "clusters": [
                {
                    "name": "prometheus_stats",
                    "type": "STATIC",
                    "connectTimeout": "0.250s",
                    "loadAssignment": {
                        "clusterName": "prometheus_stats",
                        "endpoints": [
                            {
                                "lbEndpoints": [
                                    {
                                        "endpoint": {
                                            "address": {
                                                "socketAddress": {
                                                    "address": "127.0.0.1",
                                                    "portValue": 15000
                                                }
                                            }
                                        }
                                    }
                                ]
                            }
                        ]
                    }
                },
                {
                    "name": "agent",
                    "type": "STATIC",
                    "connectTimeout": "0.250s",
                    "loadAssignment": {
                        "clusterName": "agent",
                        "endpoints": [
                            {
                                "lbEndpoints": [
                                    {
                                        "endpoint": {
                                            "address": {
                                                "socketAddress": {
                                                    "address": "127.0.0.1",
                                                    "portValue": 15020
                                                }
                                            }
                                        }
                                    }
                                ]
                            }
                        ]
                    }
                },
                {
                    "name": "sds-grpc",
                    "type": "STATIC",
                    "connectTimeout": "1s",
                    "loadAssignment": {
                        "clusterName": "sds-grpc",
                        "endpoints": [
                            {
                                "lbEndpoints": [
                                    {
                                        "endpoint": {
                                            "address": {
                                                "pipe": {
                                                    "path": "./etc/istio/proxy/SDS"
                                                }
                                            }
                                        }
                                    }
                                ]
                            }
                        ]
                    },
                    "typedExtensionProtocolOptions": {
                        "envoy.extensions.upstreams.http.v3.HttpProtocolOptions": {
                            "@type": "type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions",
                            "explicitHttpConfig": {
                                "http2ProtocolOptions": {

                                }
                            }
                        }
                    }
                },
                {
                    "name": "xds-grpc",
                    "type": "STATIC",
                    "connectTimeout": "1s",
                    "loadAssignment": {
                        "clusterName": "xds-grpc",
                        "endpoints": [
                            {
                                "lbEndpoints": [
                                    {
                                        "endpoint": {
                                            "address": {
                                                "pipe": {
                                                    "path": "./etc/istio/proxy/XDS"
                                                }
                                            }
                                        }
                                    }
                                ]
                            }
                        ]
                    },
                    "maxRequestsPerConnection": 1,
                    "circuitBreakers": {
                        "thresholds": [
                            {
                                "maxConnections": 100000,
                                "maxPendingRequests": 100000,
                                "maxRequests": 100000
                            },
                            {
                                "priority": "HIGH",
                                "maxConnections": 100000,
                                "maxPendingRequests": 100000,
                                "maxRequests": 100000
                            }
                        ]
                    },
                    "typedExtensionProtocolOptions": {
                        "envoy.extensions.upstreams.http.v3.HttpProtocolOptions": {
                            "@type": "type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions",
                            "explicitHttpConfig": {
                                "http2ProtocolOptions": {

                                }
                            }
                        }
                    },
                    "upstreamConnectionOptions": {
                        "tcpKeepalive": {
                            "keepaliveTime": 300
                        }
                    }
                },
                {
                    "name": "zipkin",
                    "type": "STRICT_DNS",
                    "connectTimeout": "1s",
                    "loadAssignment": {
                        "clusterName": "zipkin",
                        "endpoints": [
                            {
                                "lbEndpoints": [
                                    {
                                        "endpoint": {
                                            "address": {
                                                "socketAddress": {
                                                    "address": "zipkin.istio-system",
                                                    "portValue": 9411
                                                }
                                            }
                                        }
                                    }
                                ]
                            }
                        ]
                    },
                    "dnsRefreshRate": "30s",
                    "respectDnsTtl": true,
                    "dnsLookupFamily": "V4_ONLY"
                }
            ]
        },
        "dynamicResources": {
            "ldsConfig": {
                "ads": {

                },
                "initialFetchTimeout": "0s",
                "resourceApiVersion": "V3"
            },
            "cdsConfig": {
                "ads": {

                },
                "initialFetchTimeout": "0s",
                "resourceApiVersion": "V3"
            },
            "adsConfig": {
                "apiType": "GRPC",
                "transportApiVersion": "V3",
                "grpcServices": [
                    {
                        "envoyGrpc": {
                            "clusterName": "xds-grpc"
                        }
                    }
                ],
                "setNodeOnFirstMessageOnly": true
            }
        },
        "statsConfig": {
            "statsTags": [
                {
                    "tagName": "cluster_name",
                    "regex": "^cluster\\.((.+?(\\..+?\\.svc\\.cluster\\.local)?)\\.)"
                },
                {
                    "tagName": "tcp_prefix",
                    "regex": "^tcp\\.((.*?)\\.)\\w+?$"
                },
                {
                    "tagName": "response_code",
                    "regex": "(response_code=\\.=(.+?);\\.;)|_rq(_(\\.d{3}))$"
                },
                {
                    "tagName": "response_code_class",
                    "regex": "_rq(_(\\dxx))$"
                },
                {
                    "tagName": "http_conn_manager_listener_prefix",
                    "regex": "^listener(?=\\.).*?\\.http\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
                },
                {
                    "tagName": "http_conn_manager_prefix",
                    "regex": "^http\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
                },
                {
                    "tagName": "listener_address",
                    "regex": "^listener\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
                },
                {
                    "tagName": "mongo_prefix",
                    "regex": "^mongo\\.(.+?)\\.(collection|cmd|cx_|op_|delays_|decoding_)(.*?)$"
                },
                {
                    "tagName": "reporter",
                    "regex": "(reporter=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "source_namespace",
                    "regex": "(source_namespace=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "source_workload",
                    "regex": "(source_workload=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "source_workload_namespace",
                    "regex": "(source_workload_namespace=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "source_principal",
                    "regex": "(source_principal=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "source_app",
                    "regex": "(source_app=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "source_version",
                    "regex": "(source_version=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "source_cluster",
                    "regex": "(source_cluster=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "destination_namespace",
                    "regex": "(destination_namespace=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "destination_workload",
                    "regex": "(destination_workload=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "destination_workload_namespace",
                    "regex": "(destination_workload_namespace=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "destination_principal",
                    "regex": "(destination_principal=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "destination_app",
                    "regex": "(destination_app=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "destination_version",
                    "regex": "(destination_version=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "destination_service",
                    "regex": "(destination_service=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "destination_service_name",
                    "regex": "(destination_service_name=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "destination_service_namespace",
                    "regex": "(destination_service_namespace=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "destination_port",
                    "regex": "(destination_port=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "destination_cluster",
                    "regex": "(destination_cluster=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "request_protocol",
                    "regex": "(request_protocol=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "request_operation",
                    "regex": "(request_operation=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "request_host",
                    "regex": "(request_host=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "response_flags",
                    "regex": "(response_flags=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "grpc_response_status",
                    "regex": "(grpc_response_status=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "connection_security_policy",
                    "regex": "(connection_security_policy=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "source_canonical_service",
                    "regex": "(source_canonical_service=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "destination_canonical_service",
                    "regex": "(destination_canonical_service=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "source_canonical_revision",
                    "regex": "(source_canonical_revision=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "destination_canonical_revision",
                    "regex": "(destination_canonical_revision=\\.=(.*?);\\.;)"
                },
                {
                    "tagName": "cache",
                    "regex": "(cache\\.(.+?)\\.)"
                },
                {
                    "tagName": "component",
                    "regex": "(component\\.(.+?)\\.)"
                },
                {
                    "tagName": "tag",
                    "regex": "(tag\\.(.+?);\\.)"
                },
                {
                    "tagName": "wasm_filter",
                    "regex": "(wasm_filter\\.(.+?)\\.)"
                },
                {
                    "tagName": "authz_enforce_result",
                    "regex": "rbac(\\.(allowed|denied))"
                },
                {
                    "tagName": "authz_dry_run_action",
                    "regex": "(\\.istio_dry_run_(allow|deny)_)"
                },
                {
                    "tagName": "authz_dry_run_result",
                    "regex": "(\\.shadow_(allowed|denied))"
                }
            ],
            "useAllDefaultTags": false,
            "statsMatcher": {
                "inclusionList": {
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
                            "suffix": "rbac.allowed"
                        },
                        {
                            "suffix": "rbac.denied"
                        },
                        {
                            "suffix": "shadow_allowed"
                        },
                        {
                            "suffix": "shadow_denied"
                        },
                        {
                            "prefix": "component"
                        }
                    ]
                }
            }
        },
        "tracing": {
            "http": {
                "name": "envoy.tracers.zipkin",
                "typedConfig": {
                    "@type": "type.googleapis.com/envoy.config.trace.v3.ZipkinConfig",
                    "collectorCluster": "zipkin",
                    "collectorEndpoint": "/api/v2/spans",
                    "traceId128bit": true,
                    "sharedSpanContext": false,
                    "collectorEndpointVersion": "HTTP_JSON"
                }
            }
        },
        "layeredRuntime": {
            "layers": [
                {
                    "name": "deprecation",
                    "staticLayer": {
                            "envoy.deprecated_features:envoy.config.listener.v3.Listener.hidden_envoy_deprecated_use_original_dst": true,
                            "envoy.reloadable_features.new_tcp_connection_pool": false,
                            "envoy.reloadable_features.require_strict_1xx_and_204_response_headers": false,
                            "envoy.reloadable_features.treat_host_like_authority": false,
                            "re2.max_program_size.error_level": 1024
                        }
                },
                {
                    "name": "global config",
                    "staticLayer": {
                            "overload.global_downstream_max_connections": 2147483647
                        }
                },
                {
                    "name": "admin",
                    "adminLayer": {

                    }
                }
            ]
        },
        "admin": {
            "accessLogPath": "/dev/null",
            "profilePath": "/var/lib/istio/data/envoy.prof",
            "address": {
                "socketAddress": {
                    "address": "127.0.0.1",
                    "portValue": 15000
                }
            }
        }
    },
    "lastUpdated": "2021-10-09T06:09:34.762Z"
}

```



**3.检索listener配置** 

```
istioctl proxy-config listener productpage-v1-6b746f74dc-2c55l -n default

ADDRESS        PORT  MATCH                                                                                           DESTINATION
10.96.0.10     53    ALL                                                                                             Cluster: outbound|53||kube-dns.kube-system.svc.cluster.local
0.0.0.0        80    Trans: raw_buffer; App: HTTP                                                                    Route: 80
0.0.0.0        80    ALL                                                                                             PassthroughCluster
10.107.37.16   443   ALL                                                                                             Cluster: outbound|443||istio-ingressgateway.istio-system.svc.cluster.local
10.109.197.2   443   ALL                                                                                             Cluster: outbound|443||istiod.istio-system.svc.cluster.local
10.96.0.1      443   ALL                                                                                             Cluster: outbound|443||kubernetes.default.svc.cluster.local
10.102.177.173 2181  ALL                                                                                             Cluster: outbound|2181||zookeeper.dubbo.svc.cluster.local
10.108.45.67   3000  Trans: raw_buffer; App: HTTP                                                                    Route: grafana.istio-system.svc.cluster.local:3000
10.108.45.67   3000  ALL                                                                                             Cluster: outbound|3000||grafana.istio-system.svc.cluster.local
0.0.0.0        8383  Trans: raw_buffer; App: HTTP                                                                    Route: 8383
0.0.0.0        8383  ALL                                                                                             PassthroughCluster
0.0.0.0        9080  Trans: raw_buffer; App: HTTP                                                                    Route: 9080
0.0.0.0        9080  ALL                                                                                             PassthroughCluster
0.0.0.0        9090  Trans: raw_buffer; App: HTTP                                                                    Route: 9090
0.0.0.0        9090  ALL                                                                                             PassthroughCluster
10.96.0.10     9153  Trans: raw_buffer; App: HTTP                                                                    Route: kube-dns.kube-system.svc.cluster.local:9153
10.96.0.10     9153  ALL                                                                                             Cluster: outbound|9153||kube-dns.kube-system.svc.cluster.local
0.0.0.0        9411  Trans: raw_buffer; App: HTTP                                                                    Route: 9411
0.0.0.0        9411  ALL                                                                                             PassthroughCluster
10.102.158.31  14250 Trans: raw_buffer; App: HTTP                                                                    Route: jaeger-collector.istio-system.svc.cluster.local:14250
10.102.158.31  14250 ALL                                                                                             Cluster: outbound|14250||jaeger-collector.istio-system.svc.cluster.local
10.102.158.31  14268 Trans: raw_buffer; App: HTTP                                                                    Route: jaeger-collector.istio-system.svc.cluster.local:14268
10.102.158.31  14268 ALL                                                                                             Cluster: outbound|14268||jaeger-collector.istio-system.svc.cluster.local
0.0.0.0        15001 ALL                                                                                             PassthroughCluster
0.0.0.0        15001 Addr: *:15001                                                                                   Non-HTTP/Non-TCP
0.0.0.0        15006 Addr: *:15006                                                                                   Non-HTTP/Non-TCP
0.0.0.0        15006 Trans: tls; App: istio-http/1.0,istio-http/1.1,istio-h2; Addr: 0.0.0.0/0                        InboundPassthroughClusterIpv4
0.0.0.0        15006 Trans: raw_buffer; App: HTTP; Addr: 0.0.0.0/0                                                   InboundPassthroughClusterIpv4
0.0.0.0        15006 Trans: tls; App: TCP TLS; Addr: 0.0.0.0/0                                                       InboundPassthroughClusterIpv4
0.0.0.0        15006 Trans: raw_buffer; Addr: 0.0.0.0/0                                                              InboundPassthroughClusterIpv4
0.0.0.0        15006 Trans: tls; Addr: 0.0.0.0/0                                                                     InboundPassthroughClusterIpv4
0.0.0.0        15006 Trans: tls; App: istio,istio-peer-exchange,istio-http/1.0,istio-http/1.1,istio-h2; Addr: *:9080 Cluster: inbound|9080||
0.0.0.0        15006 Trans: raw_buffer; Addr: *:9080                                                                 Cluster: inbound|9080||
0.0.0.0        15010 Trans: raw_buffer; App: HTTP                                                                    Route: 15010
0.0.0.0        15010 ALL                                                                                             PassthroughCluster
10.109.197.2   15012 ALL                                                                                             Cluster: outbound|15012||istiod.istio-system.svc.cluster.local
0.0.0.0        15014 Trans: raw_buffer; App: HTTP                                                                    Route: 15014
0.0.0.0        15014 ALL                                                                                             PassthroughCluster
0.0.0.0        15021 ALL                                                                                             Inline Route: /healthz/ready*
10.107.37.16   15021 Trans: raw_buffer; App: HTTP                                                                    Route: istio-ingressgateway.istio-system.svc.cluster.local:15021
10.107.37.16   15021 ALL                                                                                             Cluster: outbound|15021||istio-ingressgateway.istio-system.svc.cluster.local
0.0.0.0        15090 ALL                                                                                             Inline Route: /stats/prometheus*
0.0.0.0        20001 Trans: raw_buffer; App: HTTP                                                                    Route: 20001
0.0.0.0        20001 ALL                                                                                             PassthroughCluster
0.0.0.0        20880 ALL                                                                                             Cluster: outbound|20880||org.apache.dubbo.samples.basic.api.complexservice-product-2-0-0
```



**4.检索路由配置** 

```
istioctl proxy-config route productpage-v1-6b746f74dc-2c55l -n default

NAME                                                          DOMAINS                               MATCH                  VIRTUAL SERVICE
                                                              *                                     /stats/prometheus*     
inbound|9080||                                                *                                     /*                     
jaeger-collector.istio-system.svc.cluster.local:14268         jaeger-collector.istio-system         /*                     
InboundPassthroughClusterIpv4                                 *                                     /*                     
                                                              *                                     /healthz/ready*        
istio-ingressgateway.istio-system.svc.cluster.local:15021     istio-ingressgateway.istio-system     /*                     
inbound|9080||                                                *                                     /*                     
grafana.istio-system.svc.cluster.local:3000                   grafana.istio-system                  /*                     
jaeger-collector.istio-system.svc.cluster.local:14250         jaeger-collector.istio-system         /*                     
kube-dns.kube-system.svc.cluster.local:9153                   kube-dns.kube-system                  /*                     
80                                                            istio-ingressgateway.istio-system     /*                     
80                                                            tracing.istio-system                  /*                     
8383                                                          istio-operator.istio-operator         /*                     
9080                                                          details                               /*                     
9080                                                          productpage                           /*                     
9080                                                          ratings                               /*                     
9080                                                          reviews                               /*                     
9090                                                          kiali.istio-system                    /*                     
9090                                                          prometheus.istio-system               /*                     
9411                                                          zipkin.istio-system                   /*                     
15010                                                         istiod.istio-system                   /*                     
15014                                                         istiod.istio-system                   /*                     
20001                                                         kiali.istio-system                    /*                     
InboundPassthroughClusterIpv4                                 *                                     /*         
```



**5.检查endpoint配置**

```
istioctl proxy-config endpoints productpage-v1-6b746f74dc-2c55l -n default

ENDPOINT                         STATUS      OUTLIER CHECK     CLUSTER
10.108.241.149:9411              HEALTHY     OK                zipkin
127.0.0.1:15000                  HEALTHY     OK                prometheus_stats
127.0.0.1:15020                  HEALTHY     OK                agent
172.17.0.10:9411                 HEALTHY     OK                outbound|9411||zipkin.istio-system.svc.cluster.local
172.17.0.10:14250                HEALTHY     OK                outbound|14250||jaeger-collector.istio-system.svc.cluster.local
172.17.0.10:14268                HEALTHY     OK                outbound|14268||jaeger-collector.istio-system.svc.cluster.local
172.17.0.10:16686                HEALTHY     OK                outbound|80||tracing.istio-system.svc.cluster.local
172.17.0.11:9080                 HEALTHY     OK                outbound|9080||reviews.default.svc.cluster.local
172.17.0.12:9080                 HEALTHY     OK                outbound|9080||ratings.default.svc.cluster.local
172.17.0.13:8080                 HEALTHY     OK                outbound|80||istio-ingressgateway.istio-system.svc.cluster.local
172.17.0.13:8443                 HEALTHY     OK                outbound|443||istio-ingressgateway.istio-system.svc.cluster.local
172.17.0.13:15021                HEALTHY     OK                outbound|15021||istio-ingressgateway.istio-system.svc.cluster.local
172.17.0.14:2181                 HEALTHY     OK                outbound|2181||zookeeper.dubbo.svc.cluster.local
172.17.0.17:9080                 HEALTHY     OK                outbound|9080||productpage.default.svc.cluster.local
172.17.0.19:9080                 HEALTHY     OK                outbound|9080||reviews.default.svc.cluster.local
172.17.0.2:53                    HEALTHY     OK                outbound|53||kube-dns.kube-system.svc.cluster.local
172.17.0.2:9153                  HEALTHY     OK                outbound|9153||kube-dns.kube-system.svc.cluster.local
172.17.0.20:20880                HEALTHY     OK                outbound|20880||org.apache.dubbo.samples.basic.api.complexservice-product-2-0-0
172.17.0.20:20880                HEALTHY     OK                outbound|20880||org.apache.dubbo.samples.basic.api.complexservice-test-1-0-0
172.17.0.20:20880                HEALTHY     OK                outbound|20880||org.apache.dubbo.samples.basic.api.demoservice
172.17.0.20:20880                HEALTHY     OK                outbound|20880||org.apache.dubbo.samples.basic.api.testservice
172.17.0.22:9080                 HEALTHY     OK                outbound|9080||reviews.default.svc.cluster.local
172.17.0.23:15010                HEALTHY     OK                outbound|15010||istiod.istio-system.svc.cluster.local
172.17.0.23:15012                HEALTHY     OK                outbound|15012||istiod.istio-system.svc.cluster.local
172.17.0.23:15014                HEALTHY     OK                outbound|15014||istiod.istio-system.svc.cluster.local
172.17.0.23:15017                HEALTHY     OK                outbound|443||istiod.istio-system.svc.cluster.local
172.17.0.3:8383                  HEALTHY     OK                outbound|8383||istio-operator.istio-operator.svc.cluster.local
172.17.0.4:3000                  HEALTHY     OK                outbound|3000||grafana.istio-system.svc.cluster.local
172.17.0.6:20880                 HEALTHY     OK                outbound|20880||org.apache.dubbo.samples.basic.api.complexservice-product-2-0-0
172.17.0.6:20880                 HEALTHY     OK                outbound|20880||org.apache.dubbo.samples.basic.api.complexservice-test-1-0-0
172.17.0.6:20880                 HEALTHY     OK                outbound|20880||org.apache.dubbo.samples.basic.api.demoservice
172.17.0.6:20880                 HEALTHY     OK                outbound|20880||org.apache.dubbo.samples.basic.api.testservice
172.17.0.7:9090                  HEALTHY     OK                outbound|9090||prometheus.istio-system.svc.cluster.local
172.17.0.8:9080                  HEALTHY     OK                outbound|9080||details.default.svc.cluster.local
172.17.0.9:9090                  HEALTHY     OK                outbound|9090||kiali.istio-system.svc.cluster.local
172.17.0.9:20001                 HEALTHY     OK                outbound|20001||kiali.istio-system.svc.cluster.local
192.168.49.2:8443                HEALTHY     OK                outbound|443||kubernetes.default.svc.cluster.local
unix://./etc/istio/proxy/SDS     HEALTHY     OK                sds-grpc
unix://./etc/istio/proxy/XDS     HEALTHY     OK                xds-grpc
```



### Envoy日志检查

```
kubectl get pods -n default
NAME                                        READY   STATUS             RESTARTS   AGE
details-v1-79f774bdb9-bkrbp                 2/2     Running            34         55d
dubbo-sample-consumer-5778c77995-5kvmf      2/2     Running            4          15d
dubbo-sample-provider-v1-d5668f85c-fcpm5    1/2     CrashLoopBackOff   421        15d
dubbo-sample-provider-v2-65995864f6-z77fg   1/2     CrashLoopBackOff   421        15d
productpage-v1-6b746f74dc-2c55l             2/2     Running            34         55d
ratings-v1-b6994bb9-7nvs2                   2/2     Running            34         55d
reviews-v1-545db77b95-mffvg                 2/2     Running            34         55d
reviews-v2-7bf8c9648f-pmqw8                 2/2     Running            34         55d
reviews-v3-84779c7bbc-sztp8                 2/2     Running            34         55d

```



```
kubectl logs productpage-v1-6b746f74dc-2c55l -c istio-proxy -n default

2021-10-09T06:09:34.612148Z	info	FLAG: --concurrency="2"
2021-10-09T06:09:34.613816Z	info	FLAG: --domain="default.svc.cluster.local"
2021-10-09T06:09:34.615144Z	info	FLAG: --help="false"
2021-10-09T06:09:34.616450Z	info	FLAG: --log_as_json="false"
2021-10-09T06:09:34.617996Z	info	FLAG: --log_caller=""
2021-10-09T06:09:34.618264Z	info	FLAG: --log_output_level="default:info"
2021-10-09T06:09:34.618973Z	info	FLAG: --log_rotate=""
2021-10-09T06:09:34.619071Z	info	FLAG: --log_rotate_max_age="30"
2021-10-09T06:09:34.619139Z	info	FLAG: --log_rotate_max_backups="1000"
2021-10-09T06:09:34.619158Z	info	FLAG: --log_rotate_max_size="104857600"
2021-10-09T06:09:34.619167Z	info	FLAG: --log_stacktrace_level="default:none"
2021-10-09T06:09:34.619186Z	info	FLAG: --log_target="[stdout]"
2021-10-09T06:09:34.619195Z	info	FLAG: --meshConfig="./etc/istio/config/mesh"
2021-10-09T06:09:34.619232Z	info	FLAG: --outlierLogPath=""
2021-10-09T06:09:34.619640Z	info	FLAG: --proxyComponentLogLevel="misc:error"
2021-10-09T06:09:34.619708Z	info	FLAG: --proxyLogLevel="warning"
2021-10-09T06:09:34.619729Z	info	FLAG: --serviceCluster="istio-proxy"
2021-10-09T06:09:34.619746Z	info	FLAG: --stsPort="0"
2021-10-09T06:09:34.619755Z	info	FLAG: --templateFile=""
2021-10-09T06:09:34.619763Z	info	FLAG: --tokenManagerPlugin="GoogleTokenExchange"
2021-10-09T06:09:34.619773Z	info	Version 1.11.0-57d639a4fd19ee8c3559b9a4032f91e4d23c6f14-Clean
2021-10-09T06:09:34.620004Z	info	Proxy role	ips=[172.17.0.17] type=sidecar id=productpage-v1-6b746f74dc-2c55l.default domain=default.svc.cluster.local
2021-10-09T06:09:34.620239Z	info	Apply proxy config from env {}

2021-10-09T06:09:34.621212Z	info	Effective config: binaryPath: /usr/local/bin/envoy
concurrency: 2
configPath: ./etc/istio/proxy
controlPlaneAuthPolicy: MUTUAL_TLS
discoveryAddress: istiod.istio-system.svc:15012
drainDuration: 45s
parentShutdownDuration: 60s
proxyAdminPort: 15000
serviceCluster: istio-proxy
statNameLength: 189
statusPort: 15020
terminationDrainDuration: 5s
tracing:
  zipkin:
    address: zipkin.istio-system:9411

2021-10-09T06:09:34.621817Z	info	JWT policy is third-party-jwt
2021-10-09T06:09:34.629497Z	info	CA Endpoint istiod.istio-system.svc:15012, provider Citadel
2021-10-09T06:09:34.629785Z	info	Using CA istiod.istio-system.svc:15012 cert with certs: var/run/secrets/istio/root-cert.pem
2021-10-09T06:09:34.630148Z	info	citadelclient	Citadel client using custom root cert: istiod.istio-system.svc:15012
2021-10-09T06:09:34.641187Z	info	Opening status port 15020
2021-10-09T06:09:34.701635Z	info	ads	All caches have been synced up in 278.87908ms, marking server ready
2021-10-09T06:09:34.705260Z	info	sds	SDS server for workload certificates started, listening on "etc/istio/proxy/SDS"
2021-10-09T06:09:34.705307Z	info	xdsproxy	Initializing with upstream address "istiod.istio-system.svc:15012" and cluster "Kubernetes"
2021-10-09T06:09:34.705901Z	info	Pilot SAN: [istiod.istio-system.svc]
2021-10-09T06:09:34.708317Z	info	starting Http service at 127.0.0.1:15004
2021-10-09T06:09:34.706961Z	info	sds	Starting SDS grpc server
2021-10-09T06:09:34.713863Z	info	Starting proxy agent
2021-10-09T06:09:34.713987Z	info	Epoch 0 starting
2021-10-09T06:09:34.714025Z	info	Envoy command: [-c etc/istio/proxy/envoy-rev0.json --restart-epoch 0 --drain-time-s 45 --drain-strategy immediate --parent-shutdown-time-s 60 --local-address-ip-version v4 --bootstrap-version 3 --file-flush-interval-msec 1000 --disable-hot-restart --log-format %Y-%m-%dT%T.%fZ	%l	envoy %n	%v -l warning --component-log-level misc:error --concurrency 2]
2021-10-09T06:09:49.651231Z	warn	ca	ca request failed, starting attempt 1 in 102.093205ms
2021-10-09T06:09:49.653265Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, connection error: desc = "transport: Error while dialing dial tcp: lookup istiod.istio-system.svc on 10.96.0.10:53: read udp 172.17.0.17:54308->10.96.0.10:53: read: connection refused"
2021-10-09T06:09:49.754697Z	warn	ca	ca request failed, starting attempt 2 in 217.620363ms
2021-10-09T06:09:49.973542Z	warn	ca	ca request failed, starting attempt 3 in 413.164804ms
2021-10-09T06:09:50.387224Z	warn	ca	ca request failed, starting attempt 4 in 790.034269ms
2021-10-09T06:09:54.732881Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T06:09:54.737140Z	info	cache	generated new workload certificate	latency=20.030197686s ttl=23h59m59.262871625s
2021-10-09T06:09:54.737189Z	info	cache	Root cert has changed, start rotating root cert
2021-10-09T06:09:54.737241Z	info	ads	XDS: Incremental Pushing:0 ConnectedEndpoints:0 Version:
2021-10-09T06:09:54.737290Z	info	cache	returned workload trust anchor from cache	ttl=23h59m59.262712873s
2021-10-09T06:10:00.260046Z	error	failed scraping envoy metrics: error scraping http://localhost:15090/stats/prometheus: Get "http://localhost:15090/stats/prometheus": dial tcp 127.0.0.1:15090: connect: connection refused
2021-10-09T06:10:09.822192Z	info	ads	ADS: new connection for node:productpage-v1-6b746f74dc-2c55l.default-1
2021-10-09T06:10:09.822349Z	info	cache	returned workload certificate from cache	ttl=23h59m44.177659137s
2021-10-09T06:10:09.822972Z	info	ads	SDS: PUSH for node:productpage-v1-6b746f74dc-2c55l.default resources:1 size:4.0kB resource:default
2021-10-09T06:10:09.824289Z	info	ads	ADS: new connection for node:productpage-v1-6b746f74dc-2c55l.default-2
2021-10-09T06:10:09.824412Z	info	cache	returned workload trust anchor from cache	ttl=23h59m44.175593445s
2021-10-09T06:10:09.824719Z	info	ads	SDS: PUSH for node:productpage-v1-6b746f74dc-2c55l.default resources:1 size:1.1kB resource:ROOTCA
2021-10-09T06:10:11.647403Z	info	Initialization took 37.137433315s
2021-10-09T06:10:11.647445Z	info	Envoy proxy is ready
2021-10-09T06:39:18.977224Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T06:39:19.115985Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T07:06:30.692276Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T07:06:30.708189Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T07:39:25.039910Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T07:39:25.453044Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T08:11:49.642424Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T08:11:49.662181Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T08:42:34.382434Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T08:42:34.888463Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T09:12:39.047044Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T09:12:39.264687Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T09:45:15.283934Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T09:45:15.624594Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T10:40:18.355648Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T10:40:19.445560Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T11:40:49.672142Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T11:40:49.750819Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T12:13:35.599017Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T12:13:35.613901Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T12:45:45.635004Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T12:45:45.863447Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T13:13:50.955289Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T13:13:51.357666Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T13:44:22.533385Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T13:44:22.608760Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T14:12:41.401729Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T14:12:41.479046Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T20:16:27.058652Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T20:16:27.115880Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T20:46:33.482070Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T20:46:33.717380Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-09T21:14:32.021185Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-09T21:14:32.147648Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-10T03:15:52.877232Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-10T03:15:53.228722Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-10T08:39:35.002795Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-10T08:39:35.065172Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-10T09:07:44.524430Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-10T09:07:44.859206Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-10T10:23:29.308030Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-10T10:23:29.345193Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-10T14:46:38.698099Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-10T14:46:38.982620Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T01:35:15.863417Z	info	ads	XDS: Incremental Pushing:0 ConnectedEndpoints:2 Version:
2021-10-11T01:35:16.775167Z	info	cache	generated new workload certificate	latency=875.9824ms ttl=23h59m59.2467445s
2021-10-11T01:35:16.775668Z	info	ads	SDS: PUSH for node:productpage-v1-6b746f74dc-2c55l.default resources:1 size:4.0kB resource:default
2021-10-11T01:39:50.240964Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T01:39:50.370981Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T02:07:22.152721Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T02:07:22.527678Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T02:37:36.552947Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T02:37:36.856751Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T03:07:39.209405Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T03:07:39.517859Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T03:40:10.466618Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T03:40:10.598101Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T04:13:16.379457Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T04:13:16.746549Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T04:41:02.506798Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T04:41:02.535715Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T05:08:33.426869Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T05:08:33.831736Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T05:39:58.498246Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T05:39:58.843118Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T06:10:46.026735Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T06:10:46.409047Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T06:39:33.874987Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T06:39:34.058491Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T07:10:25.205318Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T07:10:25.312517Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T07:38:19.248554Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T07:38:19.622943Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T08:08:55.249385Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T08:08:55.432749Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T08:39:31.143465Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T08:39:31.188042Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T09:11:26.393498Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T09:11:26.759407Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T09:43:32.959630Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T09:43:33.371005Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T10:14:57.177150Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T10:14:57.553715Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T10:45:36.372113Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T10:45:36.435046Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T11:17:46.255332Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T11:17:46.753532Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-11T11:50:29.509255Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-11T11:50:29.576997Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-12T01:38:05.160426Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-12T01:38:05.287482Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-12T02:08:46.877301Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-12T02:08:47.263647Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-12T02:36:42.277846Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-12T02:36:42.461016Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-12T02:52:14.520064Z	info	ads	XDS: Incremental Pushing:0 ConnectedEndpoints:2 Version:
2021-10-12T02:52:15.191168Z	info	cache	generated new workload certificate	latency=669.7031ms ttl=23h59m59.8088817s
2021-10-12T02:52:15.192069Z	info	ads	SDS: PUSH for node:productpage-v1-6b746f74dc-2c55l.default resources:1 size:4.0kB resource:default
2021-10-12T03:04:47.456832Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-12T03:04:47.876532Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-12T03:34:19.214548Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-12T03:34:19.699587Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-12T04:04:57.097292Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-12T04:04:57.264378Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-12T04:34:58.197878Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-12T04:34:58.530884Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2021-10-12T05:07:01.126353Z	warning	envoy config	StreamAggregatedResources gRPC config stream closed: 14, closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: 
2021-10-12T05:07:01.192366Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012

```







```













