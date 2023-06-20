# BASH Simple CURL

```bash
/usr/bin/curl -v -s -w '%{http_code}' \ --insecure \ --user-agent lsccli/0.1.0 \ -H 'Authorization: Basic bW9uaXRvcjpNb0dlbjAx' \ -H 'Cache-control: no-cache' \ -H 'Accept: application/json' \ -H 'Content-Type: application/yang-data+json' \ --request GET 'http://127.0.0.1:8181/rests/data/network-topology:network-topology/topology=topology-netconf/node=ericsson_trafficnode_13322/yang-ext:mount/core-model-1-4:control-construct?content=nonconfig&fields=logical-termination-point(uuid;layer-protocol(local-id;layer-protocol-name))'
```
