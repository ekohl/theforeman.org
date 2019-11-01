---
layout: post
title: Foreman SSL explained
date: 2019-11-01 12:00:00
author: Ewoud Kohl van Wijngaarden
tags:
- foreman
- ssl
---

By default the foreman-installer installs using the [Puppet CA]. While options are provided to replace these, it's not obvious. In this blog we'll see how Foreman uses certificates and how to to replace them.

<!--more-->

While the Puppet CA is very convenient for a server admin using Puppet, it's inconvenient for clients connecting to the UI because often it's not in the certificate store. For those not using Puppet, the CA service is quite heavy. Others may need to use different certificates for compliance reasons. Then there are those who do use Puppet CA but want to understand the application. Whatever the reason, we'll go step by step in building a deployment.

## Starting small

This guide is going to use CentOS 7 and the latest Foreman. At the time of writing this is 1.23 but is applicable to many versions and all supported operating systems.

To install the host `foreman.example.com` we want [EPEL], the latest Foreman release and the latest Puppet release. These are needed to install `foreman-installer`.

```bash
yum -y install epel-release https://yum.theforeman.org/releases/latest/el7/x86_64/foreman-release.rpm https://yum.puppet.com/puppet-release-el-7.noarch.rpm
yum -y install foreman-installer
```

Just running `foreman-installer` without arguments will install various components, but that's not what we want right now. First we want some certificates. You can use your own certificates but for this example I'll use [ownca](https://github.com/iNecas/ownca). A very set of wrapper scripts for openssl.

```bash
yum -y install wget openssl
wget https://github.com/iNecas/ownca/archive/master.tar.gz
tar xaf master.tar.gz
cd ownca-master
```

This set of scripts first needs to generate a Certificate Authority (CA). The exact answers don't matter for our example. In this blog I'm using defaults except for the Common Name (CN). The CA will get `Foreman SSL Blog Example`. Later We'll see this again.

```
$ ./generate-ca.sh
Generating a RSA private key
.........................................................+++++
........................+++++
writing new private key to 'private/cakey.crt'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Organization Name (company) [My Company]:
Organizational Unit Name (department, division) []:
Email Address []:
Locality Name (city, district) [My Town]:
State or Province Name (full name) [State or Providence]:
Country Name (2 letter code) [US]:
Common Name (hostname, IP, or your name) []:Foreman SSL Blog Example
```

We can inspect the certificate as well to see it was created correctly:

```
$ openssl x509 -in cacert.crt -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            7b:d7:cb:83:9a:a5:57:e5:cf:34:a8:ac:51:a2:01:2b:e1:32:a3:0c
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: O = My Company, L = My Town, ST = State or Providence, C = US, CN = Foreman SSL Blog Example
        Validity
            Not Before: Nov  1 11:21:22 2019 GMT
            Not After : Oct 29 11:21:22 2029 GMT
        Subject: O = My Company, L = My Town, ST = State or Providence, C = US, CN = Foreman SSL Blog Example
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:c6:5f:5d:82:db:4b:2d:c2:41:20:65:3e:3a:b0:
                    d1:c1:03:0a:a2:93:3d:85:3d:eb:a5:2a:90:f4:16:
                    1b:b1:da:f6:ec:5d:bd:17:3f:b6:46:0d:fc:1b:ef:
                    5f:6d:0b:9b:10:37:2e:ba:14:12:a5:91:64:60:8d:
                    0b:a2:e4:c0:cf:6d:33:64:64:e6:48:78:03:d1:98:
                    34:f6:a3:76:fd:81:e8:44:02:3e:67:ed:36:77:2a:
                    7a:08:93:49:e4:57:b3:08:bd:84:7e:48:81:fe:55:
                    78:fb:8b:a4:f6:c8:e0:bf:82:bf:cb:70:e5:bf:a9:
                    80:24:08:73:f8:0f:43:d5:7a:2a:e8:c1:62:24:37:
                    4a:41:05:2f:9f:b1:23:1c:04:c0:3f:3d:bb:4c:92:
                    ed:e4:89:72:f7:23:3d:a9:8e:04:07:41:0b:6f:01:
                    98:08:18:21:b1:d8:2d:f4:23:e2:c0:b7:2d:ca:2d:
                    be:91:f1:2e:57:1b:15:b3:3a:a4:26:28:93:2d:e0:
                    67:93:56:3b:f4:96:2b:e4:33:4c:2e:1e:b8:da:bd:
                    3d:a0:db:77:1c:bb:f9:da:3b:6a:23:af:f7:3c:11:
                    e4:59:bb:14:7e:6e:61:5a:ad:85:de:e1:a2:24:32:
                    32:ac:d4:cc:29:be:b4:3f:48:1a:12:20:2d:8d:12:
                    76:b1
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints: 
                CA:TRUE
            X509v3 Subject Key Identifier: 
                F4:A7:B0:32:2F:2A:F3:E6:D7:9C:7F:74:D0:E6:37:0E:5D:3C:E5:3C
            X509v3 Authority Key Identifier: 
                keyid:F4:A7:B0:32:2F:2A:F3:E6:D7:9C:7F:74:D0:E6:37:0E:5D:3C:E5:3C
                DirName:/O=My Company/L=My Town/ST=State or Providence/C=US/CN=Foreman SSL Blog Example
                serial:7B:D7:CB:83:9A:A5:57:E5:CF:34:A8:AC:51:A2:01:2B:E1:32:A3:0C

    Signature Algorithm: sha256WithRSAEncryption
         94:70:29:d6:ca:20:4a:af:a8:83:cf:1b:43:2b:8c:5b:9a:1a:
         b5:61:e3:6e:8a:d0:cf:03:38:a7:5d:f1:9f:5d:e4:0e:cd:75:
         0f:58:30:01:04:3c:1d:a5:b0:50:a4:08:76:7d:c5:9e:6d:1f:
         eb:8b:eb:44:78:ff:25:09:c8:cd:e9:1b:e2:dc:0a:97:b0:31:
         3a:d3:08:5a:5d:32:2b:42:59:41:b1:28:ac:76:5b:6d:d4:b0:
         1f:e9:6b:4b:32:f0:30:fd:56:c5:f5:d3:b9:9b:85:0d:46:c5:
         7e:08:5f:f2:0a:dd:d4:53:5f:57:7a:e2:2d:15:1d:63:86:ae:
         97:ad:78:99:6f:a1:49:84:25:02:01:ea:c3:eb:6f:d0:47:ec:
         03:27:35:ce:f9:f9:8b:94:80:29:ec:60:22:24:18:f7:64:45:
         38:05:f7:9d:0d:7d:ac:2d:52:b1:e0:ba:62:c1:30:cc:28:9b:
         3c:92:21:f7:b8:18:db:97:57:23:44:4f:8d:e3:63:35:fd:c9:
         41:94:b6:d5:96:6c:fa:d3:5b:e8:1f:b9:72:7a:e1:62:5f:ae:
         11:4d:2c:15:dd:f7:0d:b1:00:03:d4:2a:0a:96:5c:6a:11:8d:
         71:0e:93:09:6d:11:a9:97:56:4b:2e:4e:9b:66:63:ba:d9:37:
         56:41:f0:92
```

Now we're going to generate a certificate for the Foreman instance running on `foreman.example.com`:

```
$ ./generate-crt.sh foreman.example.com
Generating a RSA private key
...............................+++++
............+++++
writing new private key to './foreman.example.com/foreman.example.com.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Organization Name (company) [My Company]:Organizational Unit Name (department, division) []:Email Address []:Locality Name (city, district) [My Town]:State or Province Name (full name) [State or Providence]:Country Name (2 letter code) [US]:Common Name (hostname, IP, or your name) []:Using configuration from ./openssl.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
organizationName      :PRINTABLE:'My Company'
localityName          :PRINTABLE:'My Town'
stateOrProvinceName   :PRINTABLE:'State or Providence'
countryName           :PRINTABLE:'US'
commonName            :PRINTABLE:'foreman.example.com'
Certificate is to be certified until Oct 31 11:24:49 2020 GMT (365 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

This certificate can also be inspected:

```
$ openssl x509 -in foreman.example.com/foreman.example.com.crt -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1048577 (0x100001)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: O = My Company, L = My Town, ST = State or Providence, C = US, CN = Foreman SSL Blog Example
        Validity
            Not Before: Nov  1 11:24:49 2019 GMT
            Not After : Oct 31 11:24:49 2020 GMT
        Subject: C = US, ST = State or Providence, O = My Company, CN = foreman.example.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:d7:38:66:18:35:42:c6:36:e8:86:13:37:e7:6b:
                    e3:98:61:3a:f6:4a:25:08:d5:77:1c:57:91:e2:89:
                    30:9e:3b:62:5a:a4:27:d8:73:0e:d6:a8:72:87:8d:
                    41:ea:cf:01:f9:80:03:c0:4b:d9:a6:4a:14:ae:0f:
                    4d:80:27:2e:8f:f6:66:76:05:7d:6a:da:18:30:65:
                    30:0f:8e:df:61:25:77:39:fb:99:49:25:6d:35:5b:
                    37:fa:0c:12:ce:8c:79:86:f5:73:5a:7a:77:75:20:
                    74:3a:e9:8b:d8:34:e2:72:26:47:34:93:3a:eb:a9:
                    7f:f0:4c:72:95:20:e0:87:62:18:3a:fc:de:c8:a8:
                    1b:63:3c:31:e8:73:c7:d3:68:17:64:ad:e7:35:30:
                    12:69:6a:a3:81:02:7b:9e:a9:a1:bd:83:cd:9e:bc:
                    bd:42:22:db:34:8a:cb:09:cb:96:eb:5f:6e:6a:59:
                    77:d6:58:ac:d0:a2:34:10:63:bd:e0:9f:4c:07:a2:
                    8b:f9:4a:88:18:03:df:bf:67:d3:7a:28:54:d5:dd:
                    09:ba:b5:1e:26:5e:60:3f:58:0c:74:c8:ba:25:23:
                    d7:67:a6:56:13:e1:09:c9:51:c0:ce:60:47:5f:27:
                    cf:ef:83:3b:eb:0e:5f:c5:57:c2:bd:e2:75:73:d2:
                    85:7d
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints: 
                CA:FALSE
            X509v3 Subject Key Identifier: 
                74:70:68:69:71:FD:4E:9F:6D:E1:43:16:4E:69:E2:F4:B9:55:87:1C
            X509v3 Key Usage: 
                Digital Signature, Non Repudiation, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication, Code Signing, E-mail Protection
            X509v3 Subject Alternative Name: 
                DNS:foreman.example.com
    Signature Algorithm: sha256WithRSAEncryption
         7b:68:6e:03:46:50:09:80:85:b2:15:20:33:c5:36:95:2c:81:
         9a:b6:f6:66:71:56:17:f8:fd:f6:78:7d:76:e4:54:ae:04:ee:
         ad:0e:b1:28:27:e2:2e:5b:bf:70:91:8b:4f:20:5f:da:a6:16:
         35:d2:97:0d:75:b4:32:1e:bb:c2:f3:ec:a3:d6:f9:20:21:4c:
         64:7f:e1:33:58:96:1f:97:23:9a:a5:e2:25:58:26:ef:18:c5:
         9b:92:89:4a:83:e0:f6:c9:d8:15:b6:31:3e:73:cd:e8:c1:9f:
         0f:f9:1b:e4:f8:a4:9e:94:1e:a5:37:12:55:e6:6a:bb:b6:ff:
         28:8c:ac:09:3f:66:35:4e:e0:66:87:f5:80:5e:01:af:32:03:
         5b:7b:36:77:58:db:b6:44:76:8d:69:7b:be:86:16:71:40:23:
         42:b2:4d:b4:6f:a1:eb:69:0a:ed:dd:01:9e:ba:38:8c:0e:a6:
         18:27:a7:dc:25:40:cf:cc:cf:92:50:3a:d9:dc:cb:57:75:3c:
         ea:52:63:b5:4d:ec:1d:ba:a8:57:55:43:37:48:e1:f0:d6:ab:
         95:aa:40:ec:80:e8:65:92:34:31:ae:1c:cf:f9:f3:2f:94:f0:
         21:00:1f:3d:5e:c4:69:37:a5:1d:0a:60:10:3a:27:35:f3:45:
         d2:ad:87:a1
```

This output is cryptic to most. We can ignore most, but look for the `Issuer`. This is the CA we just generated and we can recognize the CN we specified. Also imporant is the `Subject` which is describing this certificate. The CN is imporant too though nowadays many tools are using the X509v3 Subject Alternative Name extension which allows multiple values.

## Installing Foreman

Let's start small:

```bash
foreman-installer \
    --no-enable-foreman-cli \
    --no-enable-foreman-proxy \
    --no-enable-puppet \
    --foreman-server-ssl-ca /path/to/ownca-master/cacert.crt \
    --foreman-server-ssl-chain /path/to/ownca-master/cacert.crt \
    --foreman-server-ssl-cert /path/to/ownca-master/foreman.example.com/foreman.example.com.crt \
    --foreman-server-ssl-key /path/to/ownca-master/foreman.example.com/foreman.example.com.key \
    --foreman-server-ssl-crl "" \
    --foreman-websockets-ssl-cert /path/to/ownca-master/foreman.example.com/foreman.example.com.crt \
    --foreman-websockets-ssl-key /path/to/ownca-master/foreman.example.com/foreman.example.com.key \
    --foreman-client-ssl-ca /path/to/ownca-master/cacert.crt \
    --foreman-client-ssl-cert /path/to/ownca-master/foreman.example.com/foreman.example.com.crt \
    --foreman-client-ssl-key /path/to/ownca-master/foreman.example.com/foreman.example.com.key
```

Now that looks like a lot of duplication so we'll go over all options. We can recognize three groups: server, websockets and client. The server is what will be used by Apache and the most visible. Websockets might surprise people, but there's a bundled [websockify] that FOreman will spawn for VNC consoles to VMs. It's serving its own certificates and these options control this. Since websockify runs as the Foreman user so it needs to read these files. Generally the server and websocket should use the same certificates.

There's also client certificates which are used to connect to the Foreman Proxy (also known as Smart Proxy). In this case the certificate and key are used as **client** certificates rather than server. This means they're used for identification. Later we'll see how that's verified.

Note we're using the same files as before, but this isn't required. Foreman Proxies can use a different CA. This can be useful when using a public commercial Certificate Authority for the Foreman UI but a custom CA (like Puppet CA) for internal communication. In that case `--foreman-server-ssl-ca` must point to the public facing CA and `--foreman-server-ssl-chain` the internal CA. The `--foreman-client-ssl-*` options must belong to the internal CA.

## Adding a Foreman Proxy

In this case there is no Foreman Proxy installed. While perfectly valid, this blog splits it off. Since Foreman can have many Foreman Proxies, this is also a common scenario even when a proxy exists also on the Foreman host.

First a certificate is needed for our host `foreman-proxy.example.com`. We run this on `foreman.example.com` in the ownca directory.

```bash
foreman.example.com $ ./generate-crt.sh foreman-proxy.example.com
Generating a RSA private key
..........................................................................................................+++++
.....................................................................................................+++++
writing new private key to './foreman-proxy.example.com/foreman-proxy.example.com.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Organization Name (company) [My Company]:Organizational Unit Name (department, division) []:Email Address []:Locality Name (city, district) [My Town]:State or Province Name (full name) [State or Providence]:Country Name (2 letter code) [US]:Common Name (hostname, IP, or your name) []:Using configuration from ./openssl.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
organizationName      :PRINTABLE:'My Company'
localityName          :PRINTABLE:'My Town'
stateOrProvinceName   :PRINTABLE:'State or Providence'
countryName           :PRINTABLE:'US'
commonName            :PRINTABLE:'foreman-proxy.example.com'
Certificate is to be certified until Oct 31 12:58:10 2020 GMT (365 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

Of course this can also be inspected for for brevity this is left as a user excercise. Note the Issuer is still the same CA while the Subject now has the new hostname.

The installer can register a new Foreman Proxy and for this OAuth authentication is used. This is because there is no trusted channel yet. If the Foreman would blindly accept all new Foreman Proxies presenting a certificate signed by the same CA, it would be an attack vector when using a third party CA. By default the installer generates a random key and secret when installing Foreman but they can also be explicitly specified. To retrieve the values, you can go to the Foreman settings page. In this case we'll use the command line on `foreman.example.com`:

```bash
foreman.example.com $ foreman-rake -- config -k oauth_consumer_key
supersecret
foreman.example.com $ foreman-rake -- config -k oauth_consumer_secret
alsosecret
```

Copy the files `cacert.crt`, `foreman-proxy.example.com.crt` and `foreman-proxy.example.com.key` to that host. Make sure the foreman-proxy user can read these.

Also follow the same installation steps to get `foreman-installer` and run the installer on `foreman-proxy.example.com`:

```bash
foreman-installer \
    --no-enable-foreman \
    --no-enable-foreman-cli \
    --no-enable-puppet \
    --foreman-proxy-foreman-base-url https://foreman.example.com \
    --foreman-proxy-oauth-consumer-key supersecret \
    --foreman-proxy-oauth-consumer-secret alsosecret \
    --foreman-proxy-ssl-ca /path/to/cacert.crt \
    --foreman-proxy-ssl-cert /path/to/foreman-proxy.example.com.crt \
    --foreman-proxy-ssl-key /path/to/foreman-proxy.example.com.key
```

Here we specify the server certs, but don't explicitly configure the client certificates. That's because the same certificates as the server are used automatically. When using a different CA for Foreman, `--foreman-proxy-foreman-ssl-ca` must be specified as well.
