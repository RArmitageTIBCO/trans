# Create CA keys

```
winpty openssl genrsa -out ca.key.pem 2048
MSYS_NO_PATHCONV=true openssl req -x509 -new -nodes -key ca.key.pem -subj '/CN=CARoot' -days 365 -out ca.cert.pem
```

# Create server key and sign

## Create server key and convert to pkcs8 format

```
winpty openssl genrsa -out broker.key.pem 2048
openssl pkcs8 -topk8 -inform PEM -outform PEM -in broker.key.pem -out broker.key-pk8.pem -nocrypt
```

## Create config file

vi broker.conf

```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=critical, digitalSignature, keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=@alt_names

[ dn ]
CN = broker

[ alt_names ]
DNS.1 = *.infra-pulsar.pod.cluster.local
DNS.2 = *.infra-pulsar.svc.cluster.local
DNS.3 = *.brokers.infra-pulsar.svc.cluster.local
```

## Generate certificate signing request

```
openssl req -new -config broker.conf -key broker.key.pem -out broker.csr.pem -sha256
```

## Sign broker certificate

```
openssl x509 -req -in broker.csr.pem -CA ca.cert.pem -CAkey ca.key.pem -CAcreateserial -out broker.cert.pem -days 365 -extensions v3_ext -extfile broker.conf -sha256
```

# Create client certificate

## Create client certificate and convert to pkcs8 format

```
winpty openssl genrsa -out client.key.pem 2048
openssl pkcs8 -topk8 -inform PEM -outform PEM -in client.key.pem -out client.key-pk8.pem -nocrypt
```

## Generate certificate signing request

```
MSYS_NO_PATHCONV=true openssl req -new -subj "/CN=client" -key client.key.pem -out client.csr.pem -sha256
```

## Sign client certificate

```
openssl x509 -req -in client.csr.pem -CA ca.cert.pem -CAkey ca.key.pem -CAcreateserial -out client.cert.pem -days 365 -sha256
```


# All commands

```
rm *.pem *.srl
openssl genrsa -out ca.key.pem 2048
openssl req -x509 -new -nodes -key ca.key.pem -subj '/CN=CARoot' -days 365 -out ca.cert.pem
openssl genrsa -out broker.key.pem 2048
openssl pkcs8 -topk8 -inform PEM -outform PEM -in broker.key.pem -out broker.key-pk8.pem -nocrypt
openssl req -new -config broker.conf -key broker.key.pem -out broker.csr.pem -sha256
openssl x509 -req -in broker.csr.pem -CA ca.cert.pem -CAkey ca.key.pem -CAcreateserial -out broker.cert.pem -days 365 -extensions v3_ext -extfile broker.conf -sha256
openssl genrsa -out client.key.pem 2048
openssl pkcs8 -topk8 -inform PEM -outform PEM -in client.key.pem -out client.key-pk8.pem -nocrypt
openssl req -new -subj "/CN=client" -key client.key.pem -out client.csr.pem -sha256
openssl x509 -req -in client.csr.pem -CA ca.cert.pem -CAkey ca.key.pem -CAcreateserial -out client.cert.pem -days 365 -sha256
```

curl https://brokers.infra-pulsar.svc.cluster.local:8443/admin/v2/persistent/public/functions/coordinate/stats -v -L --cacert /var/run/secrets/broker-certs/.ca.cert.pem


pulsar-client --authPlugin org.apache.pulsar.client.impl.auth.AuthenticationTls --authParams tlsCertFile:/var/run/secrets/client-certs/.client.cert.pem,tlsKeyFile:/var/run/secrets/client-certs/.client.key-pk8.pem
