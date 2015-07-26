#Let's Encrypt Without Sudo with Sudo

This is a step back and fork of great program [Let's Encrypt Without Sudo](https://github.com/diafygi/letsencrypt-nosudo/)
written by diafygi based on [Let's Encrypt](https://letsencrypt.org/). 

His idea was getting signed certificate without allowing program to run under
root permission. This is not the same reason why I prefer to use non-sudo
version over the original. I do not mind program to be run as root, but I do
not wish my server configuration to be changed. I would only like free TSL
certificate please...

Since I do not mind the program to be run as root, I can modify it to make it
faster and even easier to obtain the signed certificate then diafygi's version.

diafygi has done a great job on documentation of his program and since I have done
only tiny modifications I can leave his original manual unchanged below. Just keep
in mind not all step will be necessary as you follow them.


#Let's Encrypt Without Sudo

**WARNING: THE LET'S ENCRYPT CERTIFICATE AUTHORITY IS NOT YET READY! ANY
CERTIFICATES YOU HAVE SIGNED NOW WILL STILL RETURN BROWSER WARNINGS!**

The [Let's Encrypt](https://letsencrypt.org/) initiative is a fantastic program
that is going to offer **free** https certificates! However, the one catch is
that you need to use their command program to get a free certificate. You have
to run it on your your server as root, and it tries to edit your apache/nginx
config files.

I love the Let's Encrypt devs dearly, but there's no way I'm going to trust
their script to run on my server as root and be able to edit my server configs.
I'd just like the free ssl certificate, please.

So I made a script that does that. You generate your private key and certificate
signing request (CSR) like normal, then run `sign_csr.py` with your CSR to get
it signed. The script goes through the [ACME protocol](https://github.com/letsencrypt/acme-spec)
with the Let's Encrypt certificate authority and outputs the signed certificate
to stdout.

This script doesn't know or ask for your private key, and it doesn't need to be
run on your server. There are some parts of the ACME protocol that require your
private key and access to your server. For those parts, this script prints out
very minimal commands for you to run to complete the requirements. There is only
one command that needs to be run as root on your server and it is a very simple
python https server that you can inspect for yourself before you run it.

##Table of Contents

* [Donate](#donate)
* [Prerequisites](#prerequisites)
* [How to use the script](#how-to-use-the-script)
* [Example Use](#example-use)
* [How to use the signed https certificate](#how-to-use-the-signed-https-certificate)
* [Demo](#demo)
* [Feedback/Contributing](#feedbackcontributing)

##Donate

If this script is useful to you, please donate to the EFF. I don't work there,
but they do fantastic work.

[https://eff.org/donate/](https://eff.org/donate/)

##Prerequisites

* openssl
* python

##How to use the script

First, you need to generate an user account key for Let's Encrypt.
This is the key that you use to register with Let's Encrypt. If you
already have user account key with Let's Encrypt, you can skip this
step.

```sh
openssl genrsa 4096 > user.key
openssl rsa -in user.key -pubout > user.pub
```

Second, you need to generate the domain key and a certificate request.
This is the key that you will get signed for free for your domain (replace
"example.com" with the domain you own). If you already have a domain key
and CSR for your domain, you can skip this step.

```sh
#Create a CSR for example.com
openssl genrsa 4096 > domain.key
openssl req -new -sha256 -key domain.key -subj "/CN=example.com" > domain.csr

#Alternatively, if you want both example.com and www.example.com
openssl genrsa 4096 > domain.key
openssl req -new -sha256 -key domain.key -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:example.com,DNS:www.example.com")) > domain.csr
```

Third, you run the script using python and passing in the path to your user
account public key and the domain CSR. The paths can be relative or absolute.

```sh
python sign_csr.py user.pub domain.csr > signed.crt
```

When you run the script, it will ask you do do some manual commands. It has to
ask you to do these because it doesn't know your private key or have access to
your server. You can edit the manual commands to fit your situation (e.g. if
your sudo user is different or private key is in a different location).

NOTE: When the script asks you to run these manual commands, you need to run
them in a separate terminal window. You need to keep the script open while you
run them. They sign temporary test files that the script created, so if you exit
or continue the script before you run the commands, those test files will be
destroyed before they can be used correctly (and you'll have to run the script
again).

The `*.json` and `*.sig` files are temporary files automatically generated by
the script and will be destroyed when the script stops. They only contain the
protocol requests and signatures. They do NOT contain your private keys
because this script does not have access to your private keys.

###Help text
```
user@hostname:~$ python sign_csr.py --help
usage: sign_csr.py [-h] pubkey_path csr_path

Get a SSL certificate signed by a Let's Encrypt (ACME) certificate authority and
output that signed certificate. You do NOT need to run this script on your
server and this script does not ask for your private keys. It will print out
commands that you need to run with your private key or on your server as root,
which gives you a chance to review the commands instead of trusting this script.

NOTE: YOUR ACCOUNT KEY NEEDS TO BE DIFFERENT FROM YOUR DOMAIN KEY.

Prerequisites:
* openssl
* python

Example: Generate an account keypair, a domain key and csr, and have the domain csr signed.
--------------
$ openssl genrsa 4096 > user.key
$ openssl rsa -in user.key -pubout > user.pub
$ openssl genrsa 4096 > domain.key
$ openssl req -new -sha256 -key domain.key -subj "/CN=example.com" > domain.csr
$ python sign_csr.py user.pub domain.csr > signed.crt
--------------

positional arguments:
  pubkey_path  path to your account public key
  csr_path     path to your certificate signing request

optional arguments:
  -h, --help   show this help message and exit
user@hostname:~$
```

##Example Use

###Commands (what you do in your main terminal window)
```
user@hostname:~$ openssl genrsa 4096 > user.key
Generating RSA private key, 4096 bit long modulus
.............................................................................................................................................................................++
....................................................++
e is 65537 (0x10001)
user@hostname:~$ openssl rsa -in user.key -pubout > user.pub
writing RSA key
user@hostname:~$ openssl genrsa 4096 > domain.key
Generating RSA private key, 4096 bit long modulus
.................................................................................................................................................................................++
...........................................++
e is 65537 (0x10001)
user@hostname:~$ openssl req -new -sha256 -key domain.key -subj "/CN=letsencrypt.daylightpirates.org" > domain.csr
user@hostname:~$ python sign_csr.py user.pub domain.csr > signed.crt
Reading pubkey file...
Found public key!
Reading csr file...
Found domains letsencrypt.daylightpirates.org

STEP 1: You need to sign some files (replace 'user.key' with your user private key).

openssl dgst -sha256 -sign user.key -out register_TYtLJT.sig register_i3UGRo.json
openssl dgst -sha256 -sign user.key -out domain_ZdDFx2.sig domain_F5CAvm.json
openssl dgst -sha256 -sign user.key -out challenge_NF5S_I.sig challenge_ETkPkW.json

Press Enter when you've run the above commands in a new terminal window...
Registering webmaster@letsencrypt.daylightpirates.org...
Requesting challenges for letsencrypt.daylightpirates.org...

STEP 2: You need to run these two commands on letsencrypt.daylightpirates.org (don't stop the python command until the next step).

sudo python -c "import BaseHTTPServer; \
    h = BaseHTTPServer.BaseHTTPRequestHandler; \
    h.do_GET = lambda r: r.send_response(200) or r.end_headers() or r.wfile.write('b636mznlTFh4wNaY2R6Px1nsKykhyGzC7siaO_Mf7zA'); \
    s = BaseHTTPServer.HTTPServer(('0.0.0.0', 80), h); \
    s.serve_forever()"

Press Enter when you've got the python command running on your server...
Requesting verification for letsencrypt.daylightpirates.org...

FINAL STEP: You need to sign one more file (replace 'user.key' with your user private key).

openssl dgst -sha256 -sign user.key -out cert_NWCQzv.sig cert_QQJGmK.json

Press Enter when you've run the above command in a new terminal window...
Requesting signature...
Certificate signed!
You can stop running the python command on your server (Ctrl+C works).
user@hostname:~$ cat signed.crt
-----BEGIN CERTIFICATE-----
MIIFPDCCBCagAwIBAgIQEwAAAAAAASFvBeWRs661qDALBgkqhkiG9w0BAQswHzEd
MBsGA1UEAwwUaGFwcHkgaGFja2VyIGZha2UgQ0EwHhcNMTUwNjExMTkxMDAwWhcN
MTYwNjEwMTkxMDAwWjAqMSgwJgYDVQQDEx9sZXRzZW5jcnlwdC5kYXlsaWdodHBp
cmF0ZXMub3JnMIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAykwwRsfK
3U6BCBn6MhIvi5yp7wvM9p5dAjesWcek3aKmpDWolNhaXhwRpIuWrftpcL3co+1j
8Oq1c1cqOGK/9y4bLN1YedL4NBC3quF8b+aP5y9Wb0+IUmlKZfS8rzPqnprNx1tS
vabFKfiUq4uGaRGxbN9e7/Zz9tkpd1Dd2pL2AZx4eY91Mn06/f1LkzUQGndQC44R
12v7DgCuHFrrStT3qcms6DhvHu2s5BUXncgKIGak9tpiVQn5KK0wc3MDKfA5NOAt
BvoFKbIwCbPxqTCGYmNWXuKbQDC5iDnXFOES2jzCTY6ctuJMqVVDQjVVmBsdalj6
2Qly42ZOZ74QwygW7uPYIGQNnog/U3OjFAzr4vTGDG8Vmx2Q3UC54E1uAFoU6vPv
tnC9lF7BXaJv/brA8j+QnJLHAOu42DGaqyhc43PZXYQ9LFKiSDOzIz99BTwISLtV
TlWpacpIfvJRxzWgAbzE1tUDe6o4yUWfqJ+l7pLzMb5KdFTA4o1LYeb3OA5DZSGF
chLcDzotneoyB3/oMxcx7Izgt8A+ukI+YS39FUIZtaQEUvsrQa4A0z038a9M+0lP
4CiGoqChiRmtIfg+AnK2jePCw0RMkgMinLlVW0BoSDkjKlqpaNystfPy4Xng50VB
anCHccfBz7iePL1ZYtRD2s6Ic26AbfPoI/sCAwEAAaOCAWswggFnMA4GA1UdDwEB
/wQEAwIAoDATBgNVHSUEDDAKBggrBgEFBQcDATAMBgNVHRMBAf8EAjAAMB0GA1Ud
DgQWBBQnrwlHgS8Q3KeHDrvFgFToJn6h7jAfBgNVHSMEGDAWgBT7eE8S+WAVgyyf
F380GbMuNupBiTBuBggrBgEFBQcBAQRiMGAwLgYIKwYBBQUHMAGGImh0dHA6Ly9p
bnQteDEubGV0c2VuY3J5cHQub3JnL29jc3AwLgYIKwYBBQUHMAKGImh0dHA6Ly9p
bnQteDEubGV0c2VuY3J5cHQub3JnL2NlcnQwKgYDVR0RBCMwIYIfbGV0c2VuY3J5
cHQuZGF5bGlnaHRwaXJhdGVzLm9yZzAiBgNVHSAEGzAZMA0GCysGAQQBgt8TAQEB
MAgGBmeBDAECATAyBgNVHR8EKzApMCegJaAjhiFodHRwOi8vaW50LXgxLmxldHNl
bmNyeXB0Lm9yZy9jcmwwCwYJKoZIhvcNAQELA4IBAQBRvvFNlY1KkVmgGQvjRjDY
BlZ9nypDk3qRWfN/9PPZarShQr5tsfEFzB09AspBoOZO3hQDegjFllqBSeZ3T/No
SdgApizz0WnQT3PsgRZoObmr88RniRCY+2DOkMQCeuhRJt5f/nQCtPmQMwvO3G6V
Ruud8omn+coSoY1WGd5BYBaUoHznrfAuWFN7BwUW2yPa2z14iAMDfXh6RfWWS2dZ
TfS6UWrh6s631gS6ptbsAyD1n4eAtcg6dh1IlgqVG4oDGxTZYrx7uqB4Cwge+cOD
o/PkONqRzD++KGDfPHyjBdmAp99+MXD9EhxUQopQ884PaSYvcIhwcpFOxD9lzOun
-----END CERTIFICATE-----
user@hostname:~$
```

###Manual Commands (the stuff the script asked you to do in a 2nd terminal)
```
#first set of signed files
user@hostname:~$ openssl dgst -sha256 -sign user.key -out register_TYtLJT.sig register_i3UGRo.json
user@hostname:~$ openssl dgst -sha256 -sign user.key -out domain_ZdDFx2.sig domain_F5CAvm.json
user@hostname:~$ openssl dgst -sha256 -sign user.key -out challenge_NF5S_I.sig challenge_ETkPkW.json
user@hostname:~$

#second set of signed files
user@hostname:~$ openssl dgst -sha256 -sign user.key -out cert_NWCQzv.sig cert_QQJGmK.json
user@hostname:~$
```

###Server Commands (the stuff the script asked you to do on your server)
```
ubuntu@letsencrypt.daylightpirates.org:~sudo python -c "import BaseHTTPServer; \
>     h = BaseHTTPServer.BaseHTTPRequestHandler; \
>     h.do_GET = lambda r: r.send_response(200) or r.end_headers() or r.wfile.write('b636mznlTFh4wNaY2R6Px1nsKykhyGzC7siaO_Mf7zA'); \
>     s = BaseHTTPServer.HTTPServer(('0.0.0.0', 80), h); \
>     s.serve_forever()"
54.183.196.250 - - [11/Jun/2015 16:07:45] "GET /.well-known/acme-challenge/Abc46LNljZ5zjen6f-mcCA HTTP/1.1" 200 -
^CTraceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/usr/lib/python2.7/SocketServer.py", line 236, in serve_forever
    poll_interval)
  File "/usr/lib/python2.7/SocketServer.py", line 155, in _eintr_retry
    return func(*args)
KeyboardInterrupt
ubuntu@letsencrypt.daylightpirates.org:~$
```

##How to use the signed https certificate

The signed https certificate that is output by this script can be used along
with your private key to run an https server. You just security transfer (using
`scp` or similar) the private key and signed certificate to your server, then
include them in the https settings in your web server's configuration. Here's an
example on how to configure an nginx server:

```nginx
server {
    listen 443;
    server_name letsencrypt.daylightpirates.org;
    ssl on;
    ssl_certificate signed.crt;
    ssl_certificate_key domain.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers EECDH+aRSA+AES256:EDH+aRSA+AES256:EECDH+aRSA+AES128:EDH+aRSA+AES128;
    ssl_session_cache shared:SSL:50m;
    ssl_prefer_server_ciphers on;

    location / {
        return 200 'Let\'s Encrypt Example: https://github.com/diafygi/letsencrypt-nosudo';
        add_header Content-Type text/plain;
    }
}
```

##Demo

Here's a website that is using a certificate signed using `sign_csr.py`:

[https://letsencrypt.daylightpirates.org/](https://letsencrypt.daylightpirates.org/)

##Feedback/Contributing

I'd love to receive feedback, issues, and pull requests to make this script
better. The script itself, `sign_csr.py`, is less than 400 lines of code, so
feel free to read through it! I tried to comment things well and make it crystal
clear what it's doing.

For example, it currently can't do any ACME challenges besides SimpleHTTP. Maybe
someone could do a pull request to add more challenge compatibility? Also, it
currently can't revoke certificates, and I don't want to include that in the
`sign_csr.py` script. Perhaps there should also be a `revoke_crt.py` script?


