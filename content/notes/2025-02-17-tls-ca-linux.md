---
title: Setting up a trusted, self-signed SSL/TLS certificate authority in Linux
date: 2025-02-17
slug: /tls-ca-linux
---

With OpenSSL, it’s pretty easy to generate a simple [self-signed TLS certificate](https://en.wikipedia.org/wiki/Self-signed_certificate). Just run the following command:

```shell
openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes -keyout cert.key -out cert.crt
```

The files that this command generates, `cert.key` and `cert.crt`, could be passed to a web server, for example, and it will work fine; that is, all the connections made to that web server will be properly encrypted. The only problem with using this certificate, however, is that our browser doesn’t trust it, because it has not been signed with a [certificate authority](https://en.wikipedia.org/wiki/Certificate_authority) that the browser trusts.

When you visit a website that uses a self-signed TLS certificate, the browser will block your request and tell you that it’s not safe to proceed. You can, of course, ignore that warning and continue, and for your use-case that might be just fine. After all, self-signed certificates are only useful for private use, such as for testing. But even for private use, if the server is accessed via the open internet, as opposed to from within a trusted private network, it’ll be vulnerable to man-in-the-middle attacks and so it won’t be secure.

The solution to this problem is to create our own certificate authority (CA), sign our certificates using our CA, and add the CA to the operating system’s (or the browser’s) list of trusted certificate authorities.

(If you have absolutely no idea how SSL/TLS works, you might want to either read [this article](https://www.internetsociety.org/deploy360/tls/basics/) or [watch this video](https://www.youtube.com/watch?v=T4Df5_cojAs) or consult some other source before continuing. This post is only about the mechanics of getting this working on Linux, and so it doesn’t explain any of the concepts behind anything.)

First, we need to generate a private key and a certificate for our certificate authority:

```shell
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out rootca.key
openssl req -x509 -key rootca.key -out rootca.crt -subj "/CN=localhost-ca/O=localhost-ca"
```

Note that the certificate of our CA is self-signed, which is necessarily the case with root certificate authorities—and our CA is a root CA. And it’s issued by an organization named ‘localhost-ca’ (the name, of course, is completely arbitrary).

Next, we need to generate a private key for the signed certificate we’ll be creating. In this example, we’ll be issuing a certificate for `localhost`, so we’ll name the private key `localhost.key`:

```shell
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out localhost.key
```

Next, we need to create a [certificate signing request](https://en.wikipedia.org/wiki/Certificate_signing_request) file, which we’ll use to obtain a signed certificate from our CA. But before we do that, we need to create a config file which we’ll pass to the `openssl` commands that follow[^1]. We’ll name this file `csr.conf`:

```conf
[req]
distinguished_name = dn
prompt             = no
req_extensions = req_ext

[dn]
CN="localhost"

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.0 = localhost
```

The `CN=localhost` line specifies that we’re using this config file to obtain a certificate for the domain name `localhost`. Additional domain names, which the certificate will be valid for, could be specified in the `[alt_names]` section (with the first domain name named `DNS.0`, the second one named `DNS.1`, etc).

Now we can create the certificate signing request file:

```shell
openssl req -new -key localhost.key -out localhost.csr -config csr.conf
```

The final step is to obtain a signed certificate from our very own certificate authority:

```shell
openssl x509 -req -days 365 -extensions req_ext -extfile csr.conf -CA rootca.crt -CAkey rootca.key -in localhost.csr -out localhost.crt
```

The `localhost.crt` file that this command generates is our certificate for the `localhost` domain, and `localhost.key` is its private key. The certificate is now signed with the private key of our CA, so that anyone with our CA’s public certificate, `rootca.crt`, can verify that the certificate is signed by our CA.

All we have to do now is to get our operating-system or the browser to trust our CA. At this point, I would suggest spinning up an HTTPS server at `localhost`, with the certificate and the private key that we just created, to test whether our certificate is working as it should after we make our CA a trusted CA.

### Installing a root CA certificate system-wide on Ubuntu

On Ubuntu, root certificates can be installed with:

```shell
sudo cp rootca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

Firefox and Chromium, however, on Ubuntu, use their own certificate stores; so you’ll have to import the root certificate to these browsers individually. Programs like `curl` and `wget` should be able to access `https://localhost` without a problem now.

### Installing a root CA certificate system-wide on Arch Linux

On Arch Linux, root certificates can be installed using:

```shell
sudo trust anchor rootca.crt
```

There’s no need to add root certificates individually to Firefox or Chromium on Arch Linux as the browsers use the system-wide certificates.

### Installing a root CA certificate in Firefox

1. Go to Settings → Privacy & Security.
2. Scroll down to the ‘Certificates’ section and click ‘View certificates’.
3. Import the `rootca.crt` file in the ‘Authorities’ section.
4. Tick all the checkboxes and hit ‘OK’.
5. `https://localhost` should give you no warnings now (you might have to restart the browser).

### Installing a root CA certificate in Chromium

6. Go to Settings -> Privacy and Security -> Security -> Manage certificates.
7. Go to the ‘Authorities’ tab and import the `rootca.crt` file.
8. Tick all the checkboxes and hit ‘OK’.
9. `https://localhost` should give you no warnings now (you might have to restart the browser).

[^1]: Technically, it’s not necessary to create a config file. The information in the config file can be passed as either command-line arguments or via user prompting to `openssl` as well.
