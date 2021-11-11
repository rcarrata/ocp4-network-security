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
oc apply -f egress-fw/ovn/allow-google.yaml
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

```
oc apply -f egress-fw/ovn/allow-google-pepe.yaml

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
cat egress-fw/ovn/allow-dns-and-ip.yaml
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
      cidrSelector: 8.8.8.8/16
  - type: Deny
    to:
      cidrSelector: 0.0.0.0/0
```

This example allows Pods in the egress-fw-test namespace to connect to the host(s) that docs.openshift.com translates to and to any external host within the range 8.8.0.0 to 8.8.255.255.

The priority of a rule is determined by its placement in the egress array.

```sh
oc apply --namespace egress-fw-test -f egress-fw/ovn/allow-dns-and-ip.yaml
```

```sh
oc get egressfirewalls.k8s.ovn.org default -o yaml
apiVersion: k8s.ovn.org/v1
kind: EgressFirewall
metadata:
...
  name: default
  namespace: egress-fw-test
  resourceVersion: "1045242"
  uid: 25b98c67-fb36-47de-8447-8500eaee2c7e
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

Again as the 8.8.8.8 IP is allowed this works like a charm.

* Test ICMP / Ping to Cloudfare DNS IPs:

```sh
oc exec -ti test-egress -- ping -c2 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.

--- 1.1.1.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1010ms

command terminated with exit code 1
```

As in the previous example, the rest of the IPs not allowed specifically will be denied by the Deny rule at the botton applying at all hosts.

* Test curl to the OpenShift docs webpage:

```sh
oc exec -ti test-egress -- curl https://docs.openshift.com -I -m2
curl: (28) Resolving timed out after 2000 milliseconds
command terminated with exit code 28
```

What happened? The curl failed! But we allowed the dnsName in the EgressFirewall, why we can't reach the docs.openshift.com webpage from our pod?

Let's check the resolv.conf inside of our pods:

```
oc exec -ti test-egress -- cat /etc/resolv.conf
search egress-fw-test.svc.cluster.local svc.cluster.local cluster.local ocp.rober.lab
nameserver 172.30.0.10
options ndots:5
```

Seems ok, isn't? Why then the curl is not working properly?

Well, we in fact allowed the dnsName, but who actually resolves the dns resolution is the Openshift DNS based in the CoreDNS.

When a DNS request is performed inside of Kubernetes/OpenShift, CoreDNS handles this request and tries to resolved in the domains that the search describes, but because have not the proper answer, CoreDNS running within the Openshift-DNS pods, forward the query to the external DNS configured during the installation.  

Check the [Deep Dive in DNS in OpenShift for more information](https://rcarrata.com/openshift/dns-deep-dive-in-openshift/).

In our case the rules are allowing only the dnsName, but denying the rest of the IPs, including... the Openshift-DNS / CoreDNS ones!

Let's allow the IPs from the Openshift-DNS, but first we need to check which are these IPs.

```
oc get pod -n openshift-dns -o wide | grep dns
dns-default-6wx2g     2/2     Running   2          2d1h   10.128.0.4       ocp-8vr6j-master-2         <none>           <none>
dns-default-8hm8x     2/2     Running   2          2d1h   10.130.0.3       ocp-8vr6j-master-1         <none>           <none>
dns-default-bgxqh     2/2     Running   4          2d1h   10.128.2.4       ocp-8vr6j-worker-0-82t6f   <none>           <none>
dns-default-ft6w2     2/2     Running   2          2d1h   10.131.0.3       ocp-8vr6j-worker-0-kvxr9   <none>           <none>
dns-default-nfsm6     2/2     Running   2          2d1h   10.129.2.7       ocp-8vr6j-worker-0-sl79n   <none>           <none>
dns-default-nnlsf     2/2     Running   2          2d1h   10.129.0.3       ocp-8vr6j-master-0         <none>           <none>
```

So we need to add a specific range from the 10.128.0.0 to the 10.130.0.0, but we will add a range a bit larger only for PoC purposes. Let's add a rule to allow the 10.0.0.0/16 cidr:

```
  egress:
  - to:
      dnsName: docs.openshift.com
    type: Allow
  - to:
      cidrSelector: 10.0.0.0/16
    type: Allow
  - to:
      cidrSelector: 0.0.0.0/0
    type: Deny
```

Apply the new modified EgressFirewall object with the allow rule described before: 

```
oc apply -f egress-fw/ovn/allow-dns-and-ip-good.yaml
```

And try again the same curl to the docs.openshift.com:

```
oc exec -ti test-egress -- curl https://docs.openshift.com -I -m2 | head -n1
HTTP/2 200
```

It works!

Using the DNS feature assumes that the nodes and masters are located in a similar location as the DNS entries that are added to the ovn database are generated by the master.

NOTE: use Caution when using DNS names in deny rules. The DNS interceptor will never work flawlessly and could allow access to a denied host if the DNS resolution on the node is different then in the master.
