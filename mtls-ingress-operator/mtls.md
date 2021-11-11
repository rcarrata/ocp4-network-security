# mTLS Ingress Operator

First we would need a CA certificate which can sign both the client and server certificates. So let's create our directory structure to store the CA certificate and key.

```sh
mkdir /root/mtls

cd /root/mtls/

mkdir certs private
```

Next create an index.txt and serial file to track the list of certificates signed by the CA certificate.

```
echo 01 > serial
touch index.txt
```

Download the openssl.cnf as example:

```

```
