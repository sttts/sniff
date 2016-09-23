sniff
=====

SNIff is a small server that will accept incoming TLS connections, and parse
TLS Client Hello messages for the SNI Extension. If one is found, we'll go
ahead and forward that connection to a remote (or local!) host.

It has built-in [Kubernetes ingress](http://kubernetes.io/docs/user-guide/ingress/) support. This allows full end-to-end TLS support for Kubernetes pods.

sniff config
------------

```json
{
    "bind": {
        "host": "localhost",
        "port": 8443
    },
    "servers": [
        {
            "default": false,
            "regexp": false,
            "host": "97.107.130.79",
            "names": [
                "pault.ag",
                "www.pault.ag"
            ],
            "port": 443
        }
    ]
}
```

The following config will listen on port `8443`, and connect any requests
to `pault.ag` or `www.pault.ag` to port `443` on host `97.107.130.79`. If
nothing matches this, the socket will be closed.

Changing default to true would send any unmatched hosts (or TLS / SSL connections
without SNI) to that host.

By default, the requested domain name is compared literally with the strings
inside `names`. If `regexp` is true, then the names are interpreted as regular
expressions. Each server and name will be checked in the order they appear in
the file, stopping with the first match. If there is no match, then the
request is sent to the first server with `default` set.

Kubernetes ingress config
-------------------------

SNIff will watch [Kubernetes ingress](http://kubernetes.io/docs/user-guide/ingress/) in all namespaces which are annotated with
```yaml
kubernetes.io/ingress.class: sniff
```

A minimal example configuration looks like this:

```json
{
    "bind": {
        "host": "localhost",
        "port": 8443
    },
    "kubernetes": {}
}
```

In addition the following configuration options exist:

```
{
    "bind": {
        "host": "localhost",
        "port": 8443
    },
    "kubernetes": {
        "kubeconfig": "/run/secrets/kubernetes.io/serviceaccount",
        "ingressclass": "custom-ingress-class"
    }
}
```

By default the kubeconfig path `~/.kube/config`. The default ingress class is `sniff`. In the upper example, a Kubernetes service account is used which is nothing else than a kubeconfig which is available in a pod.

An example ingress looks like this:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: tcp
  annotations:
    kubernetes.io/ingress.class: sniff
spec:
  backend:
    serviceName: bar
    servicePort: 443
  rules:
  - host: foo
    http:
      paths:
      - backend:
          serviceName: foo
          servicePort: 443
  - host: bar
    http:
      paths:
      - backend:
          serviceName: bar
          servicePort: 443
```

**Note**: backends with a path not equal to `/` or empty are ignored.

**Note**: it is completely normal to run multiple instances of sniff and the same ingress class on different hosts for load-balancing, e.g. using a daemon-set.

using the parser
----------------

```
import (
    "fmt"

    "pault.ag/go/sniff/parser"
)

func main() {
    listener, err := net.Listen("tcp", "localhost:2222")
    if err != nil {
        return err
    }
}
```
