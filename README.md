# kube-service-mirror

Small app that watches for kubernetes services and endpoints in a target cluster
and mirrors them in a local namespace. Can be used in conjunction with coredns
in cases where pod networks are reachable between clusters and one needs to be
able to route virtual services for remote pods.

## Usage
```
Usage of ./kube-service-mirror:
  -kube-config string
        Path of a kube config file, if not provided the app will try to get in cluster config
  -log-level string
        log level, defaults to info (default "info")
  -mirror-ns string
        the namespace to create dummy mirror services in
  -resync-period duration
        Namespace watcher cache resync period (default 1h0m0s)
  -svc-prefix string
        a prefix to apply on all mirrored services names
  -target-kube-config string
        (required) Path of the target cluster kube config file to mirrot services from
```

## Coredns config example

To create a smoother experience when accessing a service coredns can be
configured using the `rewrite` functionality:
```
  Corefile: |
    cluster.<target> {
        errors
	health
        rewrite continue {
                name regex ([a-zA-Z0-9-_]*)\.([a-zv0-9-_]*)\.svc\.cluster\.<target> <target>-{1}-6d61657368-{2}.<namespace>.svc.cluster.local
                answer name <target>-([a-zA-Z0-9-_]*)-6d61657368-([a-zA-Z0-9-_]*)\.<namespace>\.svc\.cluster\.local {1}.{2}.svc.cluster.<target>
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          endpoint_pod_names
          upstream
          fallthrough in-addr.arpa ip6.arpa
        }
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          endpoint_pod_names
          upstream
          fallthrough in-addr.arpa ip6.arpa
        }

        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```
that way all queries for services under domain cluster.target will be rewritten
to match services on the local namespace that the services are mirrored.

* note that `<target>` and `<namespace>` should be replaced with a name for the
target cluster and the local namespace that contains the mirrored services.
* note that the above example assumes that you are running the mirroring service
with a prefix flag that matches the target cluster name.
