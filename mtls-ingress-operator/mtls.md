# mTLS Ingress Operator

## Creating the CA certificate

First we would need a CA certificate which can sign both the client and server certificates. So let's create our directory structure to store the CA certificate and key.

```sh
mkdir /tmp/mtls

cd /tmp/mtls

mkdir certs private
```

Next create an index.txt and serial file to track the list of certificates signed by the CA certificate.

```
echo 01 > serial
touch index.txt
```

Download the openssl.cnf as example:

```
wget https://raw.githubusercontent.com/rcarrata/ocp4-network-security/main/mtls-ingress-operator/openssl.cnf
```

* Create **private key**. We would need a private key for the CA certificate:

```sh
openssl genrsa -out private/cakey.pem 4096

Generating RSA private key, 4096 bit long modulus (2 primes)
.................++++
...++++
e is 65537 (0x010001)
```

* Set the Common Name (CN) for the CA Cert:

```sh
SUBJ_CACERT="/CN=$(hostname)/ST=Madrid/C=ES/O=None/OU=None"

echo $SUBJ_CACERT
/CN=anubis.rober.lab/ST=Madrid/C=ES/O=None/OU=None
```

I've used for the CN the hostname of my lab host.

* Create **CA certificate**. We will use the generated private key from the step before to generate our CA certificate:

```sh
openssl req -new -subj ${SUBJ_CACERT} -x509 -days 3650 -config openssl.cnf -key private/cakey.pem -out certs/cacert.pem
```

* Convert certificate to PEM format:

```sh
openssl x509 -in certs/cacert.pem -out certs/cacert.pem -outform PEM
```

* Check the CA certificate generated with openssl:

```sh
openssl x509 -in certs/cacert.pem -text -noout

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            16:36:62:c3:82:7b:25:db:3d:13:0c:e5:aa:c3:ea:d8:13:da:7d:f2
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = anubis.rober.lab, ST = Madrid, C = ES, O = None, OU = None
        Validity
            Not Before: Nov 11 20:31:49 2021 GMT
            Not After : Nov  9 20:31:49 2031 GMT
        Subject: CN = anubis.rober.lab, ST = Madrid, C = ES, O = None, OU = None
```

* In the certificate we can identify the Basic Contraints extension:

```sh
openssl x509 -in certs/cacert.pem -text -noout | grep -B4 CA

            X509v3 Authority Key Identifier:
                keyid:9F:65:8B:55:A9:A5:95:D6:62:1D:93:60:5E:54:98:88:AC:DC:13:0A

            X509v3 Basic Constraints: critical
                CA:TRUE
```

Typically the application will contain an option to point to an extension section. If critical is present then the extension will be critical.

In this case the CA: TRUE identifies that this certificate is a CA Certificate.

## Create client certificate

Now we will create the client certificate which will be used by the client node.


* Create **private key** for the client certificate. We will again need a different private key for the client certificate.

```sh
openssl genrsa -out private/client.key.pem 4096

Generating RSA private key, 4096 bit long modulus (2 primes)
...................++++
.............................................................................................................................................................++++
e is 65537 (0x010001)
```

NOTE: It is important that any CSR you generate either for client or server certificates, should have matching Country Name, State and Organization Name with the CA certificate or else while signing the certificate

* Generate Certificate Signing Request (CSR). Next we need to generate the CSR for the client certificate. For our CA certificate we had given few details such as Country Name. State, Locality etc.

```sh
SUBJ_CLIENT="/CN=$(hostname)/ST=Madrid/C=ES/O=None/OU=None"

echo $SUBJ_CLIENT
/CN=anubis.rober.lab/ST=Madrid/C=ES/O=None/OU=None
```

```sh
openssl req -new -subj ${SUBJ_CLIENT} -key private/client.key.pem -out certs/client.csr
```

* To check the CSR generated in the step before:

```
openssl req -text -noout -verify -in certs/client.csr

verify OK
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: CN = anubis.rober.lab, ST = Madrid, C = ES, O = None, OU = None
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (4096 bit)
...
```

* Add **certificate extensions**. We will also add some certificate extensions to our client certificate. We will create an additional configuration file with the required extensions

```sh
cat <<EOF > client_ext.cnf
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection
EOF
```

* Next we will create our client certificate:

```sh
openssl ca -config openssl.cnf -extfile client_ext.cnf -days 3650 -notext -batch -in certs/client.csr -out certs/client.cert.pem

Using configuration from openssl.cnf
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 2 (0x2)
        Validity
            Not Before: Nov 11 22:57:09 2021 GMT
            Not After : Nov  9 22:57:09 2031 GMT
        Subject:
            countryName               = ES
            stateOrProvinceName       = Madrid
            organizationName          = None
            organizationalUnitName    = None
            commonName                = anubis.rober.lab
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            Netscape Cert Type:
                SSL Client, S/MIME
            Netscape Comment:
                OpenSSL Generated Client Certificate
            X509v3 Subject Key Identifier:
                81:38:58:D3:93:D0:8A:69:12:12:95:54:63:E9:02:7B:82:17:4A:FE
            X509v3 Authority Key Identifier:
                keyid:9F:65:8B:55:A9:A5:95:D6:62:1D:93:60:5E:54:98:88:AC:DC:13:0A

            X509v3 Key Usage: critical
                Digital Signature, Non Repudiation, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Client Authentication, E-mail Protection
Certificate is to be certified until Nov  9 22:57:09 2031 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated
```

The path of the CA certificate required to sign the certificate will be picked from the openssl.cnf file.

Set restricted permissions to the certs:

```sh
chmod 400 certs/client.cert.pem
```

We can check the client certificate generated:

```sh
openssl x509 -in certs/client.cert.pem -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 2 (0x2)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = anubis.rober.lab, ST = Madrid, C = ES, O = None, OU = None
        Validity
            Not Before: Nov 11 22:57:09 2021 GMT
            Not After : Nov  9 22:57:09 2031 GMT
        Subject: C = ES, ST = Madrid, O = None, OU = None, CN = anubis.rober.lab
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (4096 bit)
```

as we can see the certificate is signed by of our brand new CA cert as we can check in the Issuer.

Now we have generated this certificate, the index.txt will be updated reflecting the generated certificate:

```sh
cat index.txt
V       311109225709Z           02      unknown /C=ES/ST=Madrid/O=None/OU=None/CN=anubis.rober.lab
```

Let's do a bit of housekeeping deleting the csr once we generated the cert:

```sh
rm -rf certs/client.csr
```

## Create Server Certificate

In the mTLS, both server and client are verified, so we need to generate certs for client and server and signed with our CA generated in the first place.

In this PoC only we will be using the client certificate, but in the real environment you need to generate the certificate to the server and update them in the ingress controller. Let's review the generation of the server certificate for educational (and because sometimes I have bad memory and I need to check how to do it! :D). 

* Generate a **private key** for the server certificate:

```sh
openssl genrsa -out private/server.key.pem 4096
```

* Let's extract our domain for the apps in our OpenShift cluster:

```sh
APPS_DOMAIN=$(oc whoami --show-console | cut -f 2- -d '.')

echo $APPS_DOMAIN
apps.ocp2.rober.lab
```

* Define the SUBJ SERVER used during the generation of the CSR & Cert.

```sh
SERVER_NAME=*.$APPS_DOMAIN

SUBJ_SERVER="/CN=${SERVER_NAME}/ST=Madrid/C=ES/O=None/OU=None"

echo $SUBJ_SERVER
/CN=*.apps.ocp2.rober.lab/ST=Madrid/C=ES/O=None/OU=None
```



* Create **Certificate Signing Request** (CSR) for the server certificate

```sh
openssl req -new -subj ${SUBJ_SERVER} -key private/server.key.pem -out certs/server.csr
```

* Verify the CSR that we generated:

```sh
openssl req -text -noout -verify -in certs/server.csr
verify OK
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: CN = *.apps.ocp2.rober.lab, ST = Madrid, C = ES, O = None, OU = None
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (4096 bit)
```

Let's extract the IP of the LB of my Baremetal node where is virtualized with KVM + Libvirt the VMs where the OpenShift cluster runs, and that will be the entrypoint towards the *apps and the api of OpenShift cluster:

```sh
SERVER_IP=$(ip addr show eno1 | grep inet | head -n1 | awk '{ print $2 }' | cut -f 1 -d '/')

echo $SERVER_IP
10.1.8.13
```

* Add certificate extensions. Similar to client certificate, we will again add some extensions to our server certificate. Additionally we are adding Subject Alternative Name field also known as SAN. This is used to define multiple Common Name.


```sh
cat <<EOF > server_ext.cnf
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
IP.1 = ${SERVER_IP}
DNS.1 = ${SERVER_NAME}
EOF
```

as you can see the SERVER_IP is the IP of my BM node where is allocated the OpenShift cluster virtualized, and we will add also as another SAN.

* Create server certificate. Now we will use this extension file along with the private key and CSR to generate our server certificate:

```sh
openssl ca -config openssl.cnf -extfile server_ext.cnf -days 3650 -notext -batch -in certs/server.csr -out certs/server.cert.pem
Using configuration from openssl.cnf
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 3 (0x3)
        Validity
            Not Before: Nov 12 00:13:52 2021 GMT
            Not After : Nov 10 00:13:52 2031 GMT
        Subject:
            countryName               = ES
            stateOrProvinceName       = Madrid
            organizationName          = None
            organizationalUnitName    = None
            commonName                = *.apps.ocp2.rober.lab
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            Netscape Cert Type:
                SSL Server
            Netscape Comment:
                OpenSSL Generated Server Certificate
            X509v3 Subject Key Identifier:
                xxx
            X509v3 Authority Key Identifier:
                keyid:xxx
                DirName:/CN=anubis.rober.lab/ST=Madrid/C=ES/O=None/OU=None
                serial:xxx

            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Subject Alternative Name:
                IP Address:10.1.8.13, DNS:mtls.apps.ocp2.rober.lab
Certificate is to be certified until Nov 10 00:13:52 2031 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated
```

* Let's verify the server that we generated:

```sh
openssl x509 -in certs/server.cert.pem -text -noout

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 3 (0x3)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = anubis.rober.lab, ST = Madrid, C = ES, O = None, OU = None
        Validity
            Not Before: Nov 12 00:13:52 2021 GMT
            Not After : Nov 10 00:13:52 2031 GMT
        Subject: C = ES, ST = Madrid, O = None, OU = None, CN = *.apps.ocp2.rober.lab
...
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            Netscape Cert Type:
                SSL Server
            Netscape Comment:
                OpenSSL Generated Server Certificate
            X509v3 Subject Key Identifier:
                xxx
            X509v3 Authority Key Identifier:
                keyid:xx
                DirName:/CN=anubis.rober.lab/ST=Madrid/C=ES/O=None/OU=None
                serial:xxx

            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Subject Alternative Name:
                IP Address:10.1.8.13, DNS:mtls.apps.ocp2.rober.lab
    Signature Algorithm: sha256WithRSAEncryption
```

* Let's modify the permissions of the cert:

```sh
chmod 400 certs/server.cert.pem
```

* Let's do a bit of housekeeping:

```sh
rm -f certs/server.csr
rm -f serial*
rm -f index.*
```

## Test the OpenShift Ingress Controller without the mTLS enabled

Before to enable the mTLS we need to ensure that we can reach properly the OpenShift ingress controller, and we will use the console url because have a secure route with TLS (reencrypt/Redirect): 

```sh
curl -LIv https://console-openshift-console.apps.ocp2.rober.lab
* Rebuilt URL to: https://console-openshift-console.apps.ocp2.rober.lab/
*   Trying 192.168.126.51...
* TCP_NODELAY set
* Connected to console-openshift-console.apps.ocp2.rober.lab (192.168.126.51) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/pki/tls/certs/ca-bundle.crt
  CApath: none
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, [no content] (0):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server did not agree to a protocol
* Server certificate:
*  subject: CN=*.apps.ocp2.rober.lab
*  start date: Nov 10 15:00:03 2021 GMT
*  expire date: Nov 10 15:00:04 2023 GMT
*  subjectAltName: host "console-openshift-console.apps.ocp2.rober.lab" matched cert's "*.apps.ocp2.rober.lab"
*  issuer: CN=ingress-operator@1636556232
*  SSL certificate verify ok.
* TLSv1.3 (OUT), TLS app data, [no content] (0):
> HEAD / HTTP/1.1
> Host: console-openshift-console.apps.ocp2.rober.lab
> User-Agent: curl/7.61.1
> Accept: */*
>
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS app data, [no content] (0):
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
```

All seems ok, and the ingress-controller is presenting the default certificate with the wildcard *.apps.ocp2.rober that matches the FQDN of the console host url. 

## Create CA ConfigMap and enable the mTLS in the IngressController

We can configure the Ingress Controller to enable mutual TLS (mTLS) authentication by setting a spec.clientTLS value.
The clientTLS value configures the Ingress Controller to verify client certificates. This configuration includes setting a clientCA value, which is a reference to a config map.

The config map contains the PEM-encoded CA certificate bundle that is used to verify a client’s certificate.

* Let's create a config map that is in the openshift-config namespace

```sh
oc create configmap router-ca-certs-default --from-file=ca-bundle.pem=certs/cacert.pem -n opens
hift-config

configmap/router-ca-certs-default created
```

* Edit the IngressController resource in the openshift-ingress-operator project:

```sh
oc edit IngressController default -n openshift-ingress-operator
```

* Add the spec.clientTLS field and subfields to configure mutual TLS:

```sh
oc edit IngressController default -n openshift-ingress-operator

spec:
...
  clientTLS:
    clientCertificatePolicy: Required
    clientCA:
      name: router-ca-certs-default
    allowedSubjectPatterns:
    - anubis.rober.lab
```

