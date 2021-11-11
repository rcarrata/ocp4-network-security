# Egress Firewall with OpenShift SDN in a OpenShift project

You can create an egress firewall for a project that restricts egress traffic leaving your OpenShift Container Platform cluster.

You must have OpenShift SDN configured to use either the network policy or multitenant mode to configure an egress firewall.

If you use network policy mode, an egress firewall is compatible with only one policy per namespace and will not work with projects that share a network, such as global projects.


## Deploy example apps and initial tests

```sh
oc new-project egress-fw-test
```

```sh
oc -n egress-fw-test run --image=quay.io/openshifttest/hello-openshift:multiarch test-egress
pod/test-egress created
```

* Test ICMP / Ping to Google's DNS IP (8.8.8.8):

```sh
oc exec -ti test-egress -- ping -c2 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=52 time=9.09 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=52 time=8.29 ms
```

* Test ICMP / Ping to Cloudfare DNS IPs:

```sh
oc exec -ti test-egress -- ping -c2 1.1.1.1
```

* Test curl to the OpenShift docs webpage:

```sh
oc exec -ti test-egress -- curl https://docs.openshift.com -I | head -n1
HTTP/2 200
```

## Configure the Egress Firewall to Allow only Google's DNS IP

* Allow only Google DNS in the namespace of egress-test:

```sh
kind: EgressNetworkPolicy
apiVersion: network.openshift.io/v1
metadata:
  name: default
spec:
  egress:
  - type: Allow
    to:
      cidrSelector: 8.8.8.8/32
  - type: Deny
    to:
      cidrSelector: 0.0.0.0/0
```

as you can notice the egress.cidrSelector, allows the access only to the 8.8.8.8 IP and deny the rest of IPs (0.0.0.0/0) 

```sh
oc apply -f egress-fw/sdn/allow-google.yaml
```


* Test ICMP / Ping to Google's DNS IP (8.8.8.8):

```sh
oc exec -ti test-egress -- ping -c2 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=53 time=8.15 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=53 time=7.60 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 7.596/7.871/8.147/0.275 ms
```

this works like a charm because the IP 8.8.8.8 it's allowed in our EgressFirewall object definition, and the Egress Firewall allow this communication.

* Test ICMP / Ping to Cloudfare DNS IPs:

```sh
oc exec -ti test-egress -- ping -c2 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.

--- 1.1.1.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1017ms

command terminated with exit code 1
```

as expected this ping to 1.1.1.1 failed because it's denied by the EgressFirewall.

* Test curl to the OpenShift docs webpage:

```sh
oc exec -ti test-egress -- curl https://docs.openshift.com -I -m2
curl: (28) Failed to connect to docs.openshift.com port 443 after 1504 ms: Operation timed out
command terminated with exit code 28
```

### ISSUE 2: More than one EgressNetworkPolicy for namespace

NOTE: The [documentation](https://docs.openshift.com/container-platform/4.9/networking/openshift_sdn/configuring-egress-firewall.html#limitations-of-an-egress-firewall_openshift-sdn-egress-firewall) says in limitations (and in several places additionally):

```md
No project can have more than one EgressNetworkPolicy object.
```
If we copy as is the same egress firewall, with the same rules we can add the same policy with different name:

```sh
oc apply -f egress-fw/sdn/allow-google-pepe.yaml
egressnetworkpolicy.network.openshift.io/default unchanged
```

```sh
oc get egressnetworkpolicies.network.openshift.io
NAME      AGE
default   21m
pepe      115s
```


In the documentation says that you can't have more than one EgressNetworkPolicy, but here demonstrates that you can.

Even if I generate another policy with new rules, this will be allowed:

```sh
oc apply -f egress-fw/sdn/allow-google-john.yaml
egressnetworkpolicy.network.openshift.io/johndoe created
```

```sh
[root@dell-r730-003 ocp4-network-security]# oc get egressnetworkpolicies.network.openshift.io
NAME      AGE
default   32m
johndoe   17s
pepe      13m
```

```sh
[root@dell-r730-003 ocp4-network-security]# oc get egressnetworkpolicies.network.openshift.io johndoe -o yaml
apiVersion: network.openshift.io/v1
kind: EgressNetworkPolicy
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"network.openshift.io/v1","kind":"EgressNetworkPolicy","metadata":{"annotations":{},"name":"johndoe","namespace":"egress-fw-test"},"spec":{"egress":[{"to":{"cidrSelector":"1.1.1.1/32"},"type":"Allow"},{"to":{"cidrSelector":"0.0.0.0/0"},"type":"Deny"}]}}
  creationTimestamp: "2021-11-11T14:55:30Z"
  generation: 1
  name: johndoe
  namespace: egress-fw-test
  resourceVersion: "620425"
  uid: d3092e94-348a-4291-9dee-741baf624ad6
spec:
  egress:
  - to:
      cidrSelector: 1.1.1.1/32
    type: Allow
  - to:
      cidrSelector: 0.0.0.0/0
    type: Deny
```

But not processed by the SDN and would not be allowed: 

```
oc exec -ti test-egress -- ping -c2 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.

--- 1.1.1.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1057ms

command terminated with exit code 1
```

## Configure the Egress Firewall to one Allow IP CIDR and one DNS Name and deny the rest

```sh
kind: EgressNetworkPolicy
apiVersion: network.openshift.io/v1
metadata:
  name: default
spec:
  egress:
  - type: Allow
    to:
      dnsName: docs.openshift.com
  - type: Allow
    to:
      cidrSelector: 8.8.8.8/32
  - type: Deny
    to:
      cidrSelector: 0.0.0.0/0
```

This example allows Pods in the egress-fw-test namespace to connect to the host(s) that docs.openshift.com translates to and to any external host within the range 8.8.0.0 to 8.8.255.255.

The priority of a rule is determined by its placement in the egress array.

```sh
oc apply --namespace egress-fw-test -f egress-fw/sdn/allow-dns-and-ip.yaml
```

```sh
 oc get egressnetworkpolicies.network.openshift.io default -o yaml
apiVersion: network.openshift.io/v1
kind: EgressNetworkPolicy
metadata:
...
  generation: 1
  name: default
  namespace: egress-fw-test
  uid: a5a498f0-4b27-4895-9a78-f873f1fe7491
spec:
  egress:
  - to:
      dnsName: docs.openshift.com
    type: Allow
  - to:
      cidrSelector: 8.8.8.8/32
    type: Allow
  - to:
      cidrSelector: 0.0.0.0/0
    type: Deny
```

* Test ICMP / Ping to Google's DNS IP (8.8.8.8):

```sh
oc exec -ti test-egress -- ping -c2 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=53 time=8.22 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=53 time=7.57 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 7.573/7.895/8.217/0.322 ms
```

Again as the 8.8.8.8 IP is allowed this works like a charm.

* Test ICMP / Ping to Cloudfare DNS IPs:

```sh
oc exec -ti test-egress -- ping -c2 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.

--- 1.1.1.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1047ms

command terminated with exit code 1
```

As in the previous example, the rest of the IPs not allowed specifically will be denied by the Deny rule at the botton applying at all hosts.

* Test curl to the OpenShift docs webpage:

```sh
oc exec -ti test-egress -- curl https://docs.openshift.com -I -m2
curl: (28) Resolving timed out after 2000 milliseconds
command terminated with exit code 28
```

It works!

In this case the dnsName rule worked out of the box, and you don't need to add the Allow IP Cidr range of the Openshift-DNS pods, as we needed in the OVN Kubernetes part.
 