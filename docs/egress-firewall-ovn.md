# Egress Firewall with OVN Kubernetes in a OpenShift project

you can use an egress firewall to limit the external hosts that some or all pods can access from within the cluster. An egress firewall supports the following scenarios:

* A pod can only connect to internal hosts and cannot initiate connections to the public internet.
* A pod can only connect to the public internet and cannot initiate connections to internal hosts that are outside the OpenShift Container Platform cluster.
* A pod cannot reach specified internal subnets or hosts outside the OpenShift Container Platform cluster.
* A pod can connect to only specific external hosts.

For example, you can allow one project access to a specified IP range but deny the same access to a different project. Or you can restrict application developers from updating from Python pip mirrors, and force updates to come only from approved sources.

You configure an egress firewall policy by creating an EgressFirewall custom resource (CR) object. The egress firewall matches network traffic that meets any of the following criteria:

* An IP address range in CIDR format
* A DNS name that resolves to an IP address
* A port number
* A protocol that is one of the following protocols: TCP, UDP, and SCTP

	
## Deploy example apps and initial tests

```sh
oc new-project egress-fw-test
```

```sh
oc run --image=quay.io/openshifttest/hello-openshift:multiarch test-egress
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

```sh
dig +short docs.openshift.com
elb.apps.openshift-web.p0s5.p1.openshiftapps.com.
34.231.104.136
54.204.56.156
```

```sh
nslookup docs.openshift.com

Non-authoritative answer:
docs.openshift.com      canonical name = elb.apps.openshift-web.p0s5.p1.openshiftapps.com.
Name:   elb.apps.openshift-web.p0s5.p1.openshiftapps.com
Address: 34.231.104.136
Name:   elb.apps.openshift-web.p0s5.p1.openshiftapps.com
Address: 54.204.56.156
```

## Configure the Egress Firewall to Allow only Google's DNS IP

* Allow only Google DNS in the namespace of egress-test:

```sh
apiVersion: k8s.ovn.org/v1
kind: EgressFirewall
metadata:
  name: default-google
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
oc apply -f egress-fw/allow-google.yaml
```


* Test ICMP / Ping to Google's DNS IP (8.8.8.8):

```sh
oc exec -ti test-egress -- ping -c2 8.8.8.8

PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=52 time=8.97 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=52 time=8.01 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 8.010/8.489/8.969/0.479 ms
```

Worked because 

* Test ICMP / Ping to Cloudfare DNS IPs:

```sh
oc exec -ti test-egress -- ping -c2 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.

--- 1.1.1.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1017ms

command terminated with exit code 1
```

as expected this ping to 1.1.1.1 failed because it's denied by the EgressFirewall

* Test curl to the OpenShift docs webpage:

```sh
oc exec -ti test-egress -- curl https://docs.openshift.com -I -m2
curl: (28) Failed to connect to docs.openshift.com port 443 after 1504 ms: Operation timed out
command terminated with exit code 28
```

### ISSUE 1: Not allowed to use different name than default in the EgressPolicy

NOTE: you can't define a name of the EgressFirewall different of **default** because the EgressFirewall won't allow as we can see:

```
oc apply -f egress-fw/allow-google.yaml
The EgressFirewall "pepe" is invalid: metadata.name: Invalid value: "pepe": metadata.name in body should match '^default$'
```

NOTE2: The [Egress Firewall documentation](https://docs.openshift.com/container-platform/4.9/networking/openshift_sdn/configuring-egress-firewall.html#nw-egressnetworkpolicy-object_openshift-sdn-egress-firewall) doesn't reflect this, because into the docs there is a reference that you can use whenever name you want:

```
apiVersion: network.openshift.io/v1
kind: EgressNetworkPolicy
metadata:
  name: <name> 
spec:
  egress: 
    ...
```

The CRD definition for the EgressFirewall is available in this [link](https://github.com/ovn-org/ovn-kubernetes/blob/master/go-controller/pkg/crd/egressfirewall/v1/types.go) and the pkg library code is in this [link](https://github.com/ovn-org/ovn-kubernetes/blob/master/go-controller/pkg/ovn/egressfirewall.go)

The tests are also done using always the "default" name in [all the test cases](https://github.com/ovn-org/ovn-kubernetes/blob/master/go-controller/pkg/ovn/egressfirewall_test.go#L764).

## Configure the Egress Firewall to one Allow IP CIDR and one DNS Name and deny the rest

```sh
cat egress-fw/allow-dns-and-ip.yaml
kind: EgressFirewall
apiVersion: k8s.ovn.org/v1
metadata:
  name: default
spec:
  egress:
  - type: Allow
    to:
      dnsName: docs.openshift.com
  - type: Allow
    to:
      cidrSelector: 8.8.0.0/16
  - type: Deny
    to:
      cidrSelector: 0.0.0.0/0
```

This example allows Pods in the egress-gw-test namespace to connect to the host(s) that docs.openshift.com translates to and to any external host within the range 8.8.0.0 to 8.8.255.255.

The priority of a rule is determined by its placement in the egress array.

Using the DNS feature assumes that the nodes and masters are located in a similar location as the DNS entries that are added to the ovn database are generated by the master.

NOTE: use Caution when using DNS names in deny rules. The DNS interceptor will never work flawlessly and could allow access to a denied host if the DNS resolution on the node is different then in the master.

```sh
oc apply --namespace egress-fw-test -f egress-fw/al
low-dns-and-ip.yaml
```

```sh
oc get egressfirewalls.k8s.ovn.org default -o yaml
apiVersion: k8s.ovn.org/v1
kind: EgressFirewall
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"k8s.ovn.org/v1","kind":"EgressFirewall","metadata":{"annotations":{},"name":"default","namespace":"egress-fw-test"},"spec":{"egress":[{"to":{"cidrSelector":"8.8.8.8/32"},"type":"Allow"},{"to":{"cidrSelector":"0.0.0.0/0"},"type":"Deny"}]}}
  creationTimestamp: "2021-11-10T17:30:16Z"
  generation: 2
  name: default
  namespace: egress-fw-test
  resourceVersion: "1045242"
  uid: 25b98c67-fb36-47de-8447-8500eaee2c7e
spec:
  egress:
  - to:
      cidrSelector: 8.8.8.8/32
    type: Allow
  - to:
      cidrSelector: 0.0.0.0/0
    type: Deny
status:
  status: EgressFirewall Rules applied
```

* Test ICMP / Ping to Google's DNS IP (8.8.8.8):

```sh
oc exec -ti test-egress -- ping -c2 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=52 time=8.98 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=52 time=8.50 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 8.501/8.742/8.983/0.241 ms
```

* Test ICMP / Ping to Cloudfare DNS IPs:

```sh
oc exec -ti test-egress -- ping -c2 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.

--- 1.1.1.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1010ms

command terminated with exit code 1
```

* Test ICMP / Ping to Cloudfare DNS IPs:

```sh
oc exec -ti test-egress -- ping -c2 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.

--- 1.1.1.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1028ms

command terminated with exit code 1
```