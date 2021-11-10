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

## Configure the Egress Firewall to Allow only Google's DNS

* Allow only Google DNS in the namespace of egress-test:

```sh
apiVersion: k8s.ovn.org/v1
kind: EgressFirewall
metadata:
  name: allow-google-only
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