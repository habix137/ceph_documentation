# Securing Ceph Dashboard with Custom HTTPS Certificates

This document provides step-by-step instructions on how to secure your Ceph Dashboard using custom certificates generated with OpenSSL.

## ✅ Generate Private Key

```bash
openssl genrsa -out private/ceph.key 2048
chmod 400 private/ceph.key
```

## ✅ Create Certificate Signing Request (CSR)

```bash
openssl req -config openssl.cnf \
    -new -key private/ceph.key \
    -sha256 \
    -reqexts v3_req_es_node \
    -out csr/ceph.csr
```

## ✅ Sign Certificate with Your Custom CA

```bash
openssl ca -config openssl.cnf \
    -extensions v3_req_es_node \
    -days 1095 \
    -notext -md sha256 \
    -in csr/ceph.csr \
    -out certs/ceph.crt
```

## ✅ Move Certificates to Ceph

```bash
sudo cp /home/user01/cert/ceph.crt /etc/ceph/certs/
sudo cp /home/user01/cert/ceph.key /etc/ceph/certs/
sudo chmod 600 /etc/ceph/certs/ceph.key
```

## ✅ Configure Ceph Dashboard

```bash
ceph dashboard set-ssl-certificate -i /etc/ceph/certs/ceph.crt
ceph dashboard set-ssl-certificate-key -i /etc/ceph/certs/ceph.key
ceph config set mgr mgr/dashboard/ssl true
```

## ✅ Restart Dashboard Module

```bash
ceph mgr module disable dashboard
ceph mgr module enable dashboard
```

## ✅ Example OpenSSL Configuration (openssl.cnf)

```ini
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = ./
certs             = $dir/certs
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
crldir            = $dir/crl
crl               = $crldir/crl.pem
private_key       = $dir/private/ca-key.pem
certificate       = $dir/ca-cert.pem
policy            = policy_strict
RANDFILE          = $dir/.rand

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 3650
default_crl_days  = 30
default_md        = sha256
preserve          = no
email_in_dn       = no
batch             = no
unique_subject    = no

[ policy_strict ]
countryName             = match
stateOrProvinceName     = optional
organizationName        = match
organizationalUnitName  = optional
commonName              = match
emailAddress            = optional

[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = CA:TRUE
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_req_es_node ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names_es_node

[ alt_names_es_node ]
DNS.1 = master
DNS.2 = worker1
DNS.3 = worker2
IP.1 = 213.233.184.217
IP.2 = 213.233.184.175
IP.3 = 213.233.184.176
DNS.4 = localhost
IP.4 = 127.0.0.1

[ req ]
default_bits        = 4096
default_md          = sha256
prompt              = yes
distinguished_name  = req_dn
x509_extensions     = v3_ca

[ req_dn ]
countryName                     = Country Name (2 letter code)
countryName_default             = IR
stateOrProvinceName             = State or Province Name
stateOrProvinceName_default     = Tehran
organizationName                = Organization Name
organizationName_default        = aip
commonName                      = Common Name
commonName_default             = els
emailAddress                    = Email Address
emailAddress_default            = en.habibi1377@gmail.com
```

## ✅ Access Dashboard

After completing these steps, your Ceph Dashboard should be accessible securely via HTTPS:

```
https://<mon_ip>:8443
```

## 💬 Notes

- Update the `openssl.cnf` to reflect your actual IP addresses and DNS names.
- Ensure you replace `<mon_ip>` with your monitor or manager node IP or hostname.

---

**Document prepared for internal and production use to improve Ceph Dashboard security.**

