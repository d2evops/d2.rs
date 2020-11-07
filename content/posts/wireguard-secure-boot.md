---
title: "wireguard secure boot"
date: 2020-11-07T18:22:34+01:00
draft: false
toc: false
images:
tags:
  - wireguard
  - kernel modules
  - secure boot
---

*how to enable wireguard, or any other custom kernel module when secure boot is enabled*

as `root` user create folder to store config and certificates
```bash
mkdir ~/signing-certs
```
enter `signing-certs` dir and set config (change `O`, `CN` and `emailAddress`)
```bash
cd ~/signing-certs
cat << EOF > configuration_file.config
[ req ]
default_bits = 4096
distinguished_name = req_distinguished_name
prompt = no
string_mask = utf8only
x509_extensions = myexts
[ req_distinguished_name ]
O = Firstname Lastname
CN = Firstname Lastname signing key
emailAddress = mail@example.com
[ myexts ]
basicConstraints=critical,CA:FALSE
keyUsage=digitalSignature
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid
EOF
```
generate public and private certificate
```bash
openssl req -new -nodes -utf8 -sha256 -days 36500 -batch -x509 -config configuration_file.config -outform DER -out kernel_key.der -keyout kernel_key.priv
```
find where `sign-file` script is located
```bash
dpkg -S sign-file
```
find where module is located (change module name if needed)
```bash
find /usr/lib/modules -name wireguard.ko
```
sign module (change module name if needed)
```bash
for module in $(find /usr/lib/modules -name wireguard.ko)
do
  $(dpkg -S sign-file | cut -d' ' -f2) sha256 kernel_key.priv kernel_key.der $module
done
```
set module to be loaded during boot (change module name if needed)
```bash
echo "wireguard" > /etc/modules-load.d/wireguard.conf
```
load public certificate into keystore (provide password when asked)
```bash
mokutil --import kernel_key.der
```
reboot and follow `mokutil` menu during boot to load cert (use same password from previous step when asked)
```bash
reboot
```