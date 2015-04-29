kube-http-proxy
===============
[![Docker Build Status](http://hubstatus.container42.com/noonien/kube-http-proxy)](https://registry.hub.docker.com/u/noonien/kube-http-proxy)
[![License: MIT](http://img.shields.io/badge/license-MIT-blue.svg?style=flat-square)](https://github.com/noonien/kube-http-proxy/blob/master/LICENSE)

This is a HTTP reverse proxy for [Kubernetes](https://github.com/GoogleCloudPlatform/kubernetes)
based on [Nginx](http://nginx.org/) and [confd](https://github.com/kelseyhightower/confd).

To run this container, you need Kubernetes v0.15.0 or later and access to the etcd
cluster on which Kubernetes operates.


Usage
-----

    docker run -e CONFD_ETCD_NODE=<etcd-address>:<etcd-port> -p 80:80 -p 443:443 noonien/kube-http-proxy

The reverse proxy is configured using Kubernetes [annotations](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/annotations.md)
on services.


The configuration is stored as serialized JSON strings. Two annotations are supported:

  - http-proxy-servers - To configure the servers and a root path for the service
  - http-proxy-paths - To configure additional paths for specified servers


Examples
--------

**http-proxy-servers:**
```json
[
    {
        "host": "some.example.com",
        "port": "8080",
        "targetPort": "3000",
    },
    {
        "host": "another.example.com",
        "port": "8000",
        "ssl": true,
        "sslPort": "8443",
        "path": "/somewhere/",
        "webSocket": true
    },
]
```

Field:
  - host - Server hostname. Required.
  - port - Port on which to listen for connections. Defaults to 80.
  - ssl - If enabled, ceritificate and key has to be available at /etc/nginx/ssl named <host>.crt and <host.key>. If enabled all http requests are redirected to https. Defaults to false.
  - sslPort - Port on which to listen to ssl connections. Defaults to 443.
  - targetPort - Port on which the service listens for connections. Defaults to 80.
  - path - Path on which this service is exposed. Defaults to "/".
  - webSocket - Enable if the service requires upgrading the HTTP conenction to a WebSocket. Defaults to false.


**http-proxy-paths:**
```json
{
    "some.example.com": [
        {
            "path": "/somewhere/",
            "targetPort": "3000",
            "webSocket": true
        },
        {
            "path": "/somewhere/else/",
            "targetPort": "3000",
        }
    ],
    "another.example.com": [
        {
            "path": "/api/",
            "targetPort": "8080",
        }
    ]
}
```

Fields:
  - path - Path on which this service is exposed. Required.
  - targetPort - Port on which the service listens for connections. Defaults to 80.
  - webSocket - Enable if the service requires upgrading the HTTP conenction to a WebSocket. Defaults to false.


Example service.yaml
```yaml
apiVersion: v1beta3
kind: List
items:
  - kind: Service
    apiVersion: v1beta3
    metadata:
      name: test-proxy
      annotations:
        http-proxy-servers: '[{"host": "some.example.com"}]'
    spec:
      selector:
        name: test-pod
      ports:
        - port: 80
          targetPort: http
  - kind: Service
    apiVersion: v1beta3
    metadata:
      name: test-proxy-api
      annotations:
        http-proxy-paths: '{"some.example.com": [{"path": "/api/", "targetPort": 8080}]}'
    spec:
      selector:
        name: test-pod-api
      ports:
        - port: 8080
          targetPort: api
```


Fleet
-----

This container should be ran outside of kubernetes because due to kube-proxy,
[client IPs are obscured](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/services.md#shortcomings).
Here's a service file that can be used to launch this container:

```
[Unit]
Description=Kubernetes HTTP Reverse Proxy
Documentation=https://github.com/noonien/kube-http-proxy
Requires=docker.service
Requires=etcd2.service
After=docker.service
After=etcd2.service

[Service]
EnvironmentFile=/etc/network-environment
ExecStartPre=-/usr/bin/docker rm -f kube-http-proxy
ExecStart=/usr/bin/docker run -e CONFD_ETCD_NODE=${DEFAULT_IPV4}:4001 -p 443:443 -p 80:80 --name kube-http-proxy noonien/kube-http-proxy
Restart=always
RestartSec=10
```

This example uses [setup-network-environment](https://github.com/kelseyhightower/setup-network-environment)
to get the correct IP address of the etcd server.
