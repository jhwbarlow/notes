# etcd With TLS on Ubuntu Server

Install etcd:

```
sudo apt install etcd
```

Ensure unit is running:

```
systemctl status etcd
```

Ensure the health is OK:

```
curl http://localhost:2379/health --> {"health": "true"}
```

Create a systemd drop-in override for the etcd service unit:

```
sudo systemctl edit etcd.service
```

```
[Service]
ExecStart=
ExecStart=/usr/bin/etcd \
  --cert-file '/var/lib/etcd/tls/client.crt.pem' \
  --key-file '/var/lib/etcd/tls/client.key.pem' \
  --listen-client-urls https://127.0.0.1:2379 \
  --advertise-client-urls https://127.0.0.1:2379

```

Note the empty `ExecStart` at the beginning of the drop-in. This unsets the `ExecStart` set by the main unit file stored in `/lib/systemd/system/etcd.service` and allows us to replace its value without running into duplicate errors.

The drop-in file is created in `/etc/systemd/system` on Ubuntu Server:

```
cat /etc/systemd/system/etcd.service.d/override.conf
```

Create the private key:

```
openssl genrsa -out etcd-server.key 2048
```

Create a OpenSSL config file to store SAN information for our CSR and the certificate:

```
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = ubuntu-server
DNS.2 = localhost
IP.1 = 127.0.0.1
IP.2 = 192.168.122.210
```

Create a CSR:

```
openssl req -new -key etcd-server.key -subj "/CN=etcd-server" -out etcd-server.csr -config etcd-server-cert.conf
```

Create a certificate using a CSR and the local self-signed CA:

```
openssl x509 -req -in etcd-server.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out etcd-server.crt -extensions v3_req -extfile etcd-server-cert.conf -days 1000
```

Create a directory to store the TLS material, copy the material over and ensure it is owned by `etcd:etcd`:

```
sudo mkdir /var/lib/etcd/tls
sudo cp etcd-server.key /var/lib/etcd/tls/client.key.pem
sudo cp etcd-server.crt /var/lib/etcd/tls/client.crt.pem
sudo chown -R etcd /var/lib/etcd/tls
sudo chgrp -R etcd /var/lib/etcd/tls
```

Restart etcd to enable TLS:

```
sudo systemctl restart etcd
```

Check that it is now accessible using HTTPS and not accessible using HTTP:

```
curl http://localhost:2379/health --> Client sent an HTTP request to an HTTPS server.
curl --cacert ca.crt https://localhost:2379/health --> {"health": "true"}
```
