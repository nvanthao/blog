---
title: "Envoy Incremental Xds"
date: 2023-11-29T21:38:36+11:00
draft: false
---

At work, I am facing a tricky bug that caused some Envoy Clusters to having missing Endpoints.

It's hard to debug the issue as communication between Envoy and Consul (Management Server) is over a bi-directional gRPC stream.

I have yet figured out the root cause of the issue but I found a neat trick. When debugging my local Consul server, I have these environment variables set in my VS Code debugger config.

```json
			"env": {
				"GRPC_GO_LOG_VERBOSITY_LEVEL": "99",
				"GRPC_GO_LOG_SEVERITY_LEVEL": "info"
			},
```

With this, my server log will show human-readable of the Protobuf messages (`DeltaDiscoveryRequest`, `DeltaDiscoveryResponse`) sent between Envoy and Consul.

This is neat as I can finally see the ACK/NACK response, which is a mechanism for Consul to only send the delta resources.

This is a sample Consul `TRACE` log for a downstream `frontend` proxy that has 2 upstreams of `backend` and `backend1`. I believe studying this log in detail will get me closer to the root cause of this bug.

```log
2023-11-29T13:54:25.608+1100 [TRACE] agent.envoy.xds: Incremental xDS v3: xdsVersion=v3 direction=request
  protobuf=
  | {
  |   "typeUrl":  "type.googleapis.com/envoy.config.cluster.v3.Cluster"
  | }
2023-11-29T13:54:25.608+1100 [TRACE] agent.envoy.xds: subscribing to type: xdsVersion=v3 typeUrl=type.googleapis.com/envoy.config.cluster.v3.Cluster
2023-11-29T13:54:25.608+1100 [TRACE] agent.proxycfg: A proxy config snapshot was requested: kind=connect-proxy proxy=frontend-sidecar-proxy service_id=frontend-sidecar-proxy
2023-11-29T13:54:25.609+1100 [TRACE] agent.envoy.xds: watching proxy, pending initial proxycfg snapshot for xDS: service_id=frontend-sidecar-proxy xdsVersion=v3
2023-11-29T13:54:25.609+1100 [DEBUG] agent.envoy.xds: generating cluster for: service_id=frontend-sidecar-proxy xdsVersion=v3 cluster=backend1.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul
2023-11-29T13:54:25.610+1100 [DEBUG] agent.envoy.xds: generating cluster for: service_id=frontend-sidecar-proxy xdsVersion=v3 cluster=backend.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul
2023-11-29T13:54:25.610+1100 [DEBUG] agent.envoy.xds: generating endpoints for: service_id=frontend-sidecar-proxy xdsVersion=v3 cluster=backend.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul
2023-11-29T13:54:25.610+1100 [DEBUG] agent.envoy.xds: generating endpoints for: service_id=frontend-sidecar-proxy xdsVersion=v3 cluster=backend1.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul
2023-11-29T13:54:25.610+1100 [TRACE] agent.envoy.xds: Got initial config snapshot: service_id=frontend-sidecar-proxy xdsVersion=v3
2023-11-29T13:54:25.610+1100 [TRACE] agent.envoy.xds: Invoking all xDS resource handlers and sending changed data if there are any: service_id=frontend-sidecar-proxy xdsVersion=v3
2023-11-29T13:54:25.620+1100 [TRACE] agent.envoy.xds: Incremental xDS v3: service_id=frontend-sidecar-proxy xdsVersion=v3 direction=response
  protobuf=
  | {
  |   "resources":  [
  |     {
  |       "name":  "backend1.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul",
  |       "version":  "0d83701cb4a68406b5050fab14bd55e2757f34a2552a86565c93a773c2ebe87e",
  |       "resource":  {
  |         "@type":  "type.googleapis.com/envoy.config.cluster.v3.Cluster",
  |         "name":  "backend1.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul",
  |         "altStatName":  "backend1.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul",
  |         "type":  "EDS",
  |         "edsClusterConfig":  {
  |           "edsConfig":  {
  |             "ads":  {},
  |             "resourceApiVersion":  "V3"
  |           }
  |         },
  |         "connectTimeout":  "5s",
  |         "circuitBreakers":  {},
  |         "outlierDetection":  {},
  |         "commonLbConfig":  {
  |           "healthyPanicThreshold":  {}
  |         },
  |         "transportSocket":  {
  |           "name":  "tls",
  |           "typedConfig":  {
  |             "@type":  "type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext",
  |             "commonTlsContext":  {
  |               "tlsParams":  {},
  |               "tlsCertificates":  [
  |                 {
  |                   "certificateChain":  {
  |                     "inlineString":  "-----BEGIN CERTIFICATE-----\nMIICHTCCAcKgAwIBAgIBCTAKBggqhkjOPQQDAjAxMS8wLQYDVQQDEyZwcmktMWVh\naGlpcGguY29uc3VsLmNhLjVlNmVjNGU3LmNvbnN1bDAeFw0yMzExMjkwMjE0NDJa\nFw0yMzEyMDIwMjE0NDJaMAAwWTATBgcqhkjOPQIBBggqhkjOPQMBBwNCAAThJSeW\nVLxPsQi/J4jeZm62nqi8EI0Reh8EpGhgJMQH1lYD5JbKFH5xOBLNXY1Chd2JdpiC\nxjPhV5nq902LPcafo4H7MIH4MA4GA1UdDwEB/wQEAwIDuDAdBgNVHSUEFjAUBggr\nBgEFBQcDAgYIKwYBBQUHAwEwDAYDVR0TAQH/BAIwADApBgNVHQ4EIgQgN7/guILt\nc/5rUgz8PiZBjXECUZqg52GgKGA2BvdWmdYwKwYDVR0jBCQwIoAg72PC4MZFxjv4\nzWFmwJFujHGQ7rx9N9yiYXJ+xMCJB8gwYQYDVR0RAQH/BFcwVYZTc3BpZmZlOi8v\nNWU2ZWM0ZTctODAxNy1lNzNkLTBhM2UtYjkzMzYyNDkwZDBhLmNvbnN1bC9ucy9k\nZWZhdWx0L2RjL2RjMS9zdmMvZnJvbnRlbmQwCgYIKoZIzj0EAwIDSQAwRgIhAOW2\nquRz3x9+ZI37yUqHhVnkQXvliLnCw5W1XB8Ee5n1AiEAhOFriAc432c83qPNS0P5\nXd6/9fMrbZH1ZekYqvZgMTc=\n-----END CERTIFICATE-----\n"
  |                   },
  |                   "privateKey":  {
  |                     "inlineString":  "-----BEGIN EC PRIVATE KEY-----\nMHcCAQEEIIKN7eSxTvNh+uMerJUVjdupB6yFfdyH6SYNGitQZCeWoAoGCCqGSM49\nAwEHoUQDQgAE4SUnllS8T7EIvyeI3mZutp6ovBCNEXofBKRoYCTEB9ZWA+SWyhR+\ncTgSzV2NQoXdiXaYgsYz4VeZ6vdNiz3Gnw==\n-----END EC PRIVATE KEY-----\n"
  |                   }
  |                 }
  |               ],
  |               "validationContext":  {
  |                 "trustedCa":  {
  |                   "inlineString":  "-----BEGIN CERTIFICATE-----\nMIICDzCCAbWgAwIBAgIBBzAKBggqhkjOPQQDAjAxMS8wLQYDVQQDEyZwcmktMWVh\naGlpcGguY29uc3VsLmNhLjVlNmVjNGU3LmNvbnN1bDAeFw0yMzExMjgyMzA3NDda\nFw0zMzExMjUyMzA3NDdaMDExLzAtBgNVBAMTJnByaS0xZWFoaWlwaC5jb25zdWwu\nY2EuNWU2ZWM0ZTcuY29uc3VsMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEwrfe\nxVAbI3rZ66IXA8ELH5tc58b6nOXwBalWL07ehTEiSf6eOMjcmUepu8kkgXw+Rnnh\n1zohEXP2kbLYrlp9uaOBvTCBujAOBgNVHQ8BAf8EBAMCAYYwDwYDVR0TAQH/BAUw\nAwEB/zApBgNVHQ4EIgQg72PC4MZFxjv4zWFmwJFujHGQ7rx9N9yiYXJ+xMCJB8gw\nKwYDVR0jBCQwIoAg72PC4MZFxjv4zWFmwJFujHGQ7rx9N9yiYXJ+xMCJB8gwPwYD\nVR0RBDgwNoY0c3BpZmZlOi8vNWU2ZWM0ZTctODAxNy1lNzNkLTBhM2UtYjkzMzYy\nNDkwZDBhLmNvbnN1bDAKBggqhkjOPQQDAgNIADBFAiAkSRjWMSnB0tFCtJjFMAre\n3w81pxP2eFjer31QhYnY9QIhAJQRValRsY9BCGJcZMwtjHfikuaY4y8zmRRzM/81\ndWX1\n-----END CERTIFICATE-----\n"
  |                 },
  |                 "matchSubjectAltNames":  [
  |                   {
  |                     "exact":  "spiffe://5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul/ns/default/dc/dc1/svc/backend1"
  |                   }
  |                 ]
  |               }
  |             },
  |             "sni":  "backend1.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul"
  |           }
  |         }
  |       }
  |     },
  |     {
  |       "name":  "backend.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul",
  |       "version":  "e392f0f2ca4b3ec99d75ed1bfbf4573ad6b67b882cfe23fb2bed19660e77c2a0",
  |       "resource":  {
  |         "@type":  "type.googleapis.com/envoy.config.cluster.v3.Cluster",
  |         "name":  "backend.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul",
  |         "altStatName":  "backend.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul",
  |         "type":  "EDS",
  |         "edsClusterConfig":  {
  |           "edsConfig":  {
  |             "ads":  {},
  |             "resourceApiVersion":  "V3"
  |           }
  |         },
  |         "connectTimeout":  "5s",
  |         "circuitBreakers":  {},
  |         "outlierDetection":  {},
  |         "commonLbConfig":  {
  |           "healthyPanicThreshold":  {}
  |         },
  |         "transportSocket":  {
  |           "name":  "tls",
  |           "typedConfig":  {
  |             "@type":  "type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext",
  |             "commonTlsContext":  {
  |               "tlsParams":  {},
  |               "tlsCertificates":  [
  |                 {
  |                   "certificateChain":  {
  |                     "inlineString":  "-----BEGIN CERTIFICATE-----\nMIICHTCCAcKgAwIBAgIBCTAKBggqhkjOPQQDAjAxMS8wLQYDVQQDEyZwcmktMWVh\naGlpcGguY29uc3VsLmNhLjVlNmVjNGU3LmNvbnN1bDAeFw0yMzExMjkwMjE0NDJa\nFw0yMzEyMDIwMjE0NDJaMAAwWTATBgcqhkjOPQIBBggqhkjOPQMBBwNCAAThJSeW\nVLxPsQi/J4jeZm62nqi8EI0Reh8EpGhgJMQH1lYD5JbKFH5xOBLNXY1Chd2JdpiC\nxjPhV5nq902LPcafo4H7MIH4MA4GA1UdDwEB/wQEAwIDuDAdBgNVHSUEFjAUBggr\nBgEFBQcDAgYIKwYBBQUHAwEwDAYDVR0TAQH/BAIwADApBgNVHQ4EIgQgN7/guILt\nc/5rUgz8PiZBjXECUZqg52GgKGA2BvdWmdYwKwYDVR0jBCQwIoAg72PC4MZFxjv4\nzWFmwJFujHGQ7rx9N9yiYXJ+xMCJB8gwYQYDVR0RAQH/BFcwVYZTc3BpZmZlOi8v\nNWU2ZWM0ZTctODAxNy1lNzNkLTBhM2UtYjkzMzYyNDkwZDBhLmNvbnN1bC9ucy9k\nZWZhdWx0L2RjL2RjMS9zdmMvZnJvbnRlbmQwCgYIKoZIzj0EAwIDSQAwRgIhAOW2\nquRz3x9+ZI37yUqHhVnkQXvliLnCw5W1XB8Ee5n1AiEAhOFriAc432c83qPNS0P5\nXd6/9fMrbZH1ZekYqvZgMTc=\n-----END CERTIFICATE-----\n"
  |                   },
  |                   "privateKey":  {
  |                     "inlineString":  "-----BEGIN EC PRIVATE KEY-----\nMHcCAQEEIIKN7eSxTvNh+uMerJUVjdupB6yFfdyH6SYNGitQZCeWoAoGCCqGSM49\nAwEHoUQDQgAE4SUnllS8T7EIvyeI3mZutp6ovBCNEXofBKRoYCTEB9ZWA+SWyhR+\ncTgSzV2NQoXdiXaYgsYz4VeZ6vdNiz3Gnw==\n-----END EC PRIVATE KEY-----\n"
  |                   }
  |                 }
  |               ],
  |               "validationContext":  {
  |                 "trustedCa":  {
  |                   "inlineString":  "-----BEGIN CERTIFICATE-----\nMIICDzCCAbWgAwIBAgIBBzAKBggqhkjOPQQDAjAxMS8wLQYDVQQDEyZwcmktMWVh\naGlpcGguY29uc3VsLmNhLjVlNmVjNGU3LmNvbnN1bDAeFw0yMzExMjgyMzA3NDda\nFw0zMzExMjUyMzA3NDdaMDExLzAtBgNVBAMTJnByaS0xZWFoaWlwaC5jb25zdWwu\nY2EuNWU2ZWM0ZTcuY29uc3VsMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEwrfe\nxVAbI3rZ66IXA8ELH5tc58b6nOXwBalWL07ehTEiSf6eOMjcmUepu8kkgXw+Rnnh\n1zohEXP2kbLYrlp9uaOBvTCBujAOBgNVHQ8BAf8EBAMCAYYwDwYDVR0TAQH/BAUw\nAwEB/zApBgNVHQ4EIgQg72PC4MZFxjv4zWFmwJFujHGQ7rx9N9yiYXJ+xMCJB8gw\nKwYDVR0jBCQwIoAg72PC4MZFxjv4zWFmwJFujHGQ7rx9N9yiYXJ+xMCJB8gwPwYD\nVR0RBDgwNoY0c3BpZmZlOi8vNWU2ZWM0ZTctODAxNy1lNzNkLTBhM2UtYjkzMzYy\nNDkwZDBhLmNvbnN1bDAKBggqhkjOPQQDAgNIADBFAiAkSRjWMSnB0tFCtJjFMAre\n3w81pxP2eFjer31QhYnY9QIhAJQRValRsY9BCGJcZMwtjHfikuaY4y8zmRRzM/81\ndWX1\n-----END CERTIFICATE-----\n"
  |                 },
  |                 "matchSubjectAltNames":  [
  |                   {
  |                     "exact":  "spiffe://5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul/ns/default/dc/dc1/svc/backend"
  |                   }
  |                 ]
  |               }
  |             },
  |             "sni":  "backend.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul"
  |           }
  |         }
  |       }
  |     },
  |     {
  |       "name":  "local_app",
  |       "version":  "c290294e3789282036b42a2f03198ac49549f863a945ab338e9c4ff5e8d16ff4",
  |       "resource":  {
  |         "@type":  "type.googleapis.com/envoy.config.cluster.v3.Cluster",
  |         "name":  "local_app",
  |         "type":  "STATIC",
  |         "connectTimeout":  "5s",
  |         "loadAssignment":  {
  |           "clusterName":  "local_app",
  |           "endpoints":  [
  |             {
  |               "lbEndpoints":  [
  |                 {
  |                   "endpoint":  {
  |                     "address":  {
  |                       "socketAddress":  {
  |                         "address":  "127.0.0.1",
  |                         "portValue":  9002
  |                       }
  |                     }
  |                   }
  |                 }
  |               ]
  |             }
  |           ]
  |         }
  |       }
  |     }
  |   ],
  |   "typeUrl":  "type.googleapis.com/envoy.config.cluster.v3.Cluster",
  |   "nonce":  "00000001"
  | }
  
2023-11-29T13:54:25.620+1100 [TRACE] agent.envoy.xds: sending response: service_id=frontend-sidecar-proxy typeUrl=type.googleapis.com/envoy.config.cluster.v3.Cluster xdsVersion=v3 nonce=00000001
2023-11-29T13:54:25.620+1100 [TRACE] agent.envoy.xds: sent response: service_id=frontend-sidecar-proxy typeUrl=type.googleapis.com/envoy.config.cluster.v3.Cluster xdsVersion=v3 nonce=00000001
2023-11-29T13:54:25.620+1100 [TRACE] agent.envoy.xds: Skipping delta computation for resource because there are dependent updates pending: service_id=frontend-sidecar-proxy xdsVersion=v3 typeUrl=type.googleapis.com/envoy.config.listener.v3.Listener dependent=type.googleapis.com/envoy.config.cluster.v3.Cluster
2023-11-29T13:54:25.755+1100 [TRACE] agent.envoy.xds: Incremental xDS v3: service_id=frontend-sidecar-proxy xdsVersion=v3 direction=request
  protobuf=
  | {
  |   "typeUrl":  "type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment",
  |   "resourceNamesSubscribe":  [
  |     "backend.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul",
  |     "backend1.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul"
  |   ]
  | }
  
2023-11-29T13:54:25.755+1100 [TRACE] agent.envoy.xds: subscribing resource for stream: service_id=frontend-sidecar-proxy typeUrl=type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment xdsVersion=v3 resource=backend.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul
2023-11-29T13:54:25.755+1100 [TRACE] agent.envoy.xds: subscribing resource for stream: service_id=frontend-sidecar-proxy typeUrl=type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment xdsVersion=v3 resource=backend1.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul
2023-11-29T13:54:25.755+1100 [TRACE] agent.envoy.xds: subscribing to type: service_id=frontend-sidecar-proxy xdsVersion=v3 typeUrl=type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment
2023-11-29T13:54:25.755+1100 [TRACE] agent.envoy.xds: Invoking all xDS resource handlers and sending changed data if there are any: service_id=frontend-sidecar-proxy xdsVersion=v3
2023-11-29T13:54:25.756+1100 [TRACE] agent.envoy.xds: Incremental xDS v3: service_id=frontend-sidecar-proxy xdsVersion=v3 direction=response
  protobuf=
  | {
  |   "resources":  [
  |     {
  |       "name":  "backend1.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul",
  |       "version":  "94fb540c19d56550305b3b5f97718c9ba3b2f165edb8f0c07ff161b25edd3671",
  |       "resource":  {
  |         "@type":  "type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment",
  |         "clusterName":  "backend1.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul",
  |         "endpoints":  [
  |           {
  |             "lbEndpoints":  [
  |               {
  |                 "endpoint":  {
  |                   "address":  {
  |                     "socketAddress":  {
  |                       "address":  "127.0.0.1",
  |                       "portValue":  21002
  |                     }
  |                   }
  |                 },
  |                 "healthStatus":  "HEALTHY",
  |                 "loadBalancingWeight":  1
  |               }
  |             ]
  |           }
  |         ]
  |       }
  |     },
  |     {
  |       "name":  "backend.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul",
  |       "version":  "b03681b8f4dbe8385fcb968aa4c92ebadbba5d390979ee97140b019d44b22c6b",
  |       "resource":  {
  |         "@type":  "type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment",
  |         "clusterName":  "backend.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul",
  |         "endpoints":  [
  |           {
  |             "lbEndpoints":  [
  |               {
  |                 "endpoint":  {
  |                   "address":  {
  |                     "socketAddress":  {
  |                       "address":  "127.0.0.1",
  |                       "portValue":  21001
  |                     }
  |                   }
  |                 },
  |                 "healthStatus":  "HEALTHY",
  |                 "loadBalancingWeight":  1
  |               }
  |             ]
  |           }
  |         ]
  |       }
  |     }
  |   ],
  |   "typeUrl":  "type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment",
  |   "nonce":  "00000002"
  | }
  
2023-11-29T13:54:25.756+1100 [TRACE] agent.envoy.xds: sending response: service_id=frontend-sidecar-proxy typeUrl=type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment xdsVersion=v3 nonce=00000002
2023-11-29T13:54:25.756+1100 [TRACE] agent.envoy.xds: sent response: service_id=frontend-sidecar-proxy typeUrl=type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment xdsVersion=v3 nonce=00000002
2023-11-29T13:54:25.756+1100 [TRACE] agent.envoy.xds: Skipping delta computation for resource because there are dependent updates pending: service_id=frontend-sidecar-proxy xdsVersion=v3 typeUrl=type.googleapis.com/envoy.config.listener.v3.Listener dependent=type.googleapis.com/envoy.config.cluster.v3.Cluster
2023-11-29T13:54:25.765+1100 [TRACE] agent.envoy.xds: Incremental xDS v3: service_id=frontend-sidecar-proxy xdsVersion=v3 direction=request
  protobuf=
  | {
  |   "typeUrl":  "type.googleapis.com/envoy.config.cluster.v3.Cluster",
  |   "responseNonce":  "00000001"
  | }
2023-11-29T13:54:25.765+1100 [TRACE] agent.envoy.xds: got ok response from envoy proxy: service_id=frontend-sidecar-proxy typeUrl=type.googleapis.com/envoy.config.cluster.v3.Cluster xdsVersion=v3 nonce=00000001
2023-11-29T13:54:25.765+1100 [TRACE] agent.envoy.xds: Invoking all xDS resource handlers and sending changed data if there are any: service_id=frontend-sidecar-proxy xdsVersion=v3
2023-11-29T13:54:25.765+1100 [TRACE] agent.envoy.xds: Skipping delta computation for resource because there are dependent updates pending: service_id=frontend-sidecar-proxy xdsVersion=v3 typeUrl=type.googleapis.com/envoy.config.listener.v3.Listener dependent=type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment
2023-11-29T13:54:25.766+1100 [TRACE] agent.envoy.xds: Incremental xDS v3: service_id=frontend-sidecar-proxy xdsVersion=v3 direction=request
  protobuf=
  | {
  |   "typeUrl":  "type.googleapis.com/envoy.config.listener.v3.Listener"
  | }
  
2023-11-29T13:54:25.766+1100 [TRACE] agent.envoy.xds: subscribing to type: service_id=frontend-sidecar-proxy xdsVersion=v3 typeUrl=type.googleapis.com/envoy.config.listener.v3.Listener
2023-11-29T13:54:25.766+1100 [TRACE] agent.envoy.xds: Invoking all xDS resource handlers and sending changed data if there are any: service_id=frontend-sidecar-proxy xdsVersion=v3
2023-11-29T13:54:25.767+1100 [TRACE] agent.envoy.xds: Skipping delta computation for resource because there are dependent updates pending: service_id=frontend-sidecar-proxy xdsVersion=v3 typeUrl=type.googleapis.com/envoy.config.listener.v3.Listener dependent=type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment
2023-11-29T13:54:25.768+1100 [TRACE] agent.envoy.xds: Incremental xDS v3: service_id=frontend-sidecar-proxy xdsVersion=v3 direction=request
  protobuf=
  | {
  |   "typeUrl":  "type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment",
  |   "responseNonce":  "00000002"
  | }
  
2023-11-29T13:54:25.768+1100 [TRACE] agent.envoy.xds: got ok response from envoy proxy: service_id=frontend-sidecar-proxy typeUrl=type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment xdsVersion=v3 nonce=00000002
2023-11-29T13:54:25.768+1100 [TRACE] agent.envoy.xds: Invoking all xDS resource handlers and sending changed data if there are any: service_id=frontend-sidecar-proxy xdsVersion=v3
2023-11-29T13:54:25.770+1100 [TRACE] agent.envoy.xds: Incremental xDS v3: service_id=frontend-sidecar-proxy xdsVersion=v3 direction=response
  protobuf=
  | {
  |   "resources":  [
  |     {
  |       "name":  "public_listener:127.0.0.1:21000",
  |       "version":  "11823f00da62985cff98cd415805c0fc42b2a7661164161ea93c24cd0a64a665",
  |       "resource":  {
  |         "@type":  "type.googleapis.com/envoy.config.listener.v3.Listener",
  |         "name":  "public_listener:127.0.0.1:21000",
  |         "address":  {
  |           "socketAddress":  {
  |             "address":  "127.0.0.1",
  |             "portValue":  21000
  |           }
  |         },
  |         "filterChains":  [
  |           {
  |             "filters":  [
  |               {
  |                 "name":  "envoy.filters.network.rbac",
  |                 "typedConfig":  {
  |                   "@type":  "type.googleapis.com/envoy.extensions.filters.network.rbac.v3.RBAC",
  |                   "rules":  {
  |                     "action":  "DENY"
  |                   },
  |                   "statPrefix":  "connect_authz"
  |                 }
  |               },
  |               {
  |                 "name":  "envoy.filters.network.tcp_proxy",
  |                 "typedConfig":  {
  |                   "@type":  "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy",
  |                   "statPrefix":  "public_listener",
  |                   "cluster":  "local_app"
  |                 }
  |               }
  |             ],
  |             "transportSocket":  {
  |               "name":  "tls",
  |               "typedConfig":  {
  |                 "@type":  "type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext",
  |                 "commonTlsContext":  {
  |                   "tlsParams":  {},
  |                   "tlsCertificates":  [
  |                     {
  |                       "certificateChain":  {
  |                         "inlineString":  "-----BEGIN CERTIFICATE-----\nMIICHTCCAcKgAwIBAgIBCTAKBggqhkjOPQQDAjAxMS8wLQYDVQQDEyZwcmktMWVh\naGlpcGguY29uc3VsLmNhLjVlNmVjNGU3LmNvbnN1bDAeFw0yMzExMjkwMjE0NDJa\nFw0yMzEyMDIwMjE0NDJaMAAwWTATBgcqhkjOPQIBBggqhkjOPQMBBwNCAAThJSeW\nVLxPsQi/J4jeZm62nqi8EI0Reh8EpGhgJMQH1lYD5JbKFH5xOBLNXY1Chd2JdpiC\nxjPhV5nq902LPcafo4H7MIH4MA4GA1UdDwEB/wQEAwIDuDAdBgNVHSUEFjAUBggr\nBgEFBQcDAgYIKwYBBQUHAwEwDAYDVR0TAQH/BAIwADApBgNVHQ4EIgQgN7/guILt\nc/5rUgz8PiZBjXECUZqg52GgKGA2BvdWmdYwKwYDVR0jBCQwIoAg72PC4MZFxjv4\nzWFmwJFujHGQ7rx9N9yiYXJ+xMCJB8gwYQYDVR0RAQH/BFcwVYZTc3BpZmZlOi8v\nNWU2ZWM0ZTctODAxNy1lNzNkLTBhM2UtYjkzMzYyNDkwZDBhLmNvbnN1bC9ucy9k\nZWZhdWx0L2RjL2RjMS9zdmMvZnJvbnRlbmQwCgYIKoZIzj0EAwIDSQAwRgIhAOW2\nquRz3x9+ZI37yUqHhVnkQXvliLnCw5W1XB8Ee5n1AiEAhOFriAc432c83qPNS0P5\nXd6/9fMrbZH1ZekYqvZgMTc=\n-----END CERTIFICATE-----\n"
  |                       },
  |                       "privateKey":  {
  |                         "inlineString":  "-----BEGIN EC PRIVATE KEY-----\nMHcCAQEEIIKN7eSxTvNh+uMerJUVjdupB6yFfdyH6SYNGitQZCeWoAoGCCqGSM49\nAwEHoUQDQgAE4SUnllS8T7EIvyeI3mZutp6ovBCNEXofBKRoYCTEB9ZWA+SWyhR+\ncTgSzV2NQoXdiXaYgsYz4VeZ6vdNiz3Gnw==\n-----END EC PRIVATE KEY-----\n"
  |                       }
  |                     }
  |                   ],
  |                   "validationContext":  {
  |                     "trustedCa":  {
  |                       "inlineString":  "-----BEGIN CERTIFICATE-----\nMIICDzCCAbWgAwIBAgIBBzAKBggqhkjOPQQDAjAxMS8wLQYDVQQDEyZwcmktMWVh\naGlpcGguY29uc3VsLmNhLjVlNmVjNGU3LmNvbnN1bDAeFw0yMzExMjgyMzA3NDda\nFw0zMzExMjUyMzA3NDdaMDExLzAtBgNVBAMTJnByaS0xZWFoaWlwaC5jb25zdWwu\nY2EuNWU2ZWM0ZTcuY29uc3VsMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEwrfe\nxVAbI3rZ66IXA8ELH5tc58b6nOXwBalWL07ehTEiSf6eOMjcmUepu8kkgXw+Rnnh\n1zohEXP2kbLYrlp9uaOBvTCBujAOBgNVHQ8BAf8EBAMCAYYwDwYDVR0TAQH/BAUw\nAwEB/zApBgNVHQ4EIgQg72PC4MZFxjv4zWFmwJFujHGQ7rx9N9yiYXJ+xMCJB8gw\nKwYDVR0jBCQwIoAg72PC4MZFxjv4zWFmwJFujHGQ7rx9N9yiYXJ+xMCJB8gwPwYD\nVR0RBDgwNoY0c3BpZmZlOi8vNWU2ZWM0ZTctODAxNy1lNzNkLTBhM2UtYjkzMzYy\nNDkwZDBhLmNvbnN1bDAKBggqhkjOPQQDAgNIADBFAiAkSRjWMSnB0tFCtJjFMAre\n3w81pxP2eFjer31QhYnY9QIhAJQRValRsY9BCGJcZMwtjHfikuaY4y8zmRRzM/81\ndWX1\n-----END CERTIFICATE-----\n"
  |                     }
  |                   }
  |                 },
  |                 "requireClientCertificate":  true
  |               }
  |             }
  |           }
  |         ],
  |         "trafficDirection":  "INBOUND"
  |       }
  |     },
  |     {
  |       "name":  "backend1:127.0.0.1:5001",
  |       "version":  "69f4d4fcb517dfb8018042c8ec9b7f3b5f4879c4c077bbe60ddec88dd0c75b39",
  |       "resource":  {
  |         "@type":  "type.googleapis.com/envoy.config.listener.v3.Listener",
  |         "name":  "backend1:127.0.0.1:5001",
  |         "address":  {
  |           "socketAddress":  {
  |             "address":  "127.0.0.1",
  |             "portValue":  5001
  |           }
  |         },
  |         "filterChains":  [
  |           {
  |             "filters":  [
  |               {
  |                 "name":  "envoy.filters.network.tcp_proxy",
  |                 "typedConfig":  {
  |                   "@type":  "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy",
  |                   "statPrefix":  "upstream.backend1.default.default.dc1",
  |                   "cluster":  "backend1.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul"
  |                 }
  |               }
  |             ]
  |           }
  |         ],
  |         "trafficDirection":  "OUTBOUND"
  |       }
  |     },
  |     {
  |       "name":  "backend:127.0.0.1:5000",
  |       "version":  "8030983f8abf8af026c49051446990581ad7f929bcd6feae16ac86cb5a8d370b",
  |       "resource":  {
  |         "@type":  "type.googleapis.com/envoy.config.listener.v3.Listener",
  |         "name":  "backend:127.0.0.1:5000",
  |         "address":  {
  |           "socketAddress":  {
  |             "address":  "127.0.0.1",
  |             "portValue":  5000
  |           }
  |         },
  |         "filterChains":  [
  |           {
  |             "filters":  [
  |               {
  |                 "name":  "envoy.filters.network.tcp_proxy",
  |                 "typedConfig":  {
  |                   "@type":  "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy",
  |                   "statPrefix":  "upstream.backend.default.default.dc1",
  |                   "cluster":  "backend.default.dc1.internal.5e6ec4e7-8017-e73d-0a3e-b93362490d0a.consul"
  |                 }
  |               }
  |             ]
  |           }
  |         ],
  |         "trafficDirection":  "OUTBOUND"
  |       }
  |     }
  |   ],
  |   "typeUrl":  "type.googleapis.com/envoy.config.listener.v3.Listener",
  |   "nonce":  "00000003"
  | }
  
2023-11-29T13:54:25.770+1100 [TRACE] agent.envoy.xds: sending response: service_id=frontend-sidecar-proxy typeUrl=type.googleapis.com/envoy.config.listener.v3.Listener xdsVersion=v3 nonce=00000003
2023-11-29T13:54:25.770+1100 [TRACE] agent.envoy.xds: sent response: service_id=frontend-sidecar-proxy typeUrl=type.googleapis.com/envoy.config.listener.v3.Listener xdsVersion=v3 nonce=00000003
2023-11-29T13:54:25.780+1100 [TRACE] agent.envoy.xds: Incremental xDS v3: service_id=frontend-sidecar-proxy xdsVersion=v3 direction=request
  protobuf=
  | {
  |   "typeUrl":  "type.googleapis.com/envoy.config.listener.v3.Listener",
  |   "responseNonce":  "00000003"
  | }
  
2023-11-29T13:54:25.780+1100 [TRACE] agent.envoy.xds: got ok response from envoy proxy: service_id=frontend-sidecar-proxy typeUrl=type.googleapis.com/envoy.config.listener.v3.Listener xdsVersion=v3 nonce=00000003
2023-11-29T13:54:25.780+1100 [TRACE] agent.envoy.xds: Invoking all xDS resource handlers and sending changed data if there are any: service_id=frontend-sidecar-proxy xdsVersion=v3
```