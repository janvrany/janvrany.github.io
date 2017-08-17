---
title: "How to generate TSL/SSL certificate and get it signed"
created_at: 2017-08-17 14:39:46 +0100
kind: article
published: true
---

# Introduction

I'm not much of an expert in cryptography and [PKI][1] and these things are 
insanely complex. My use-case is quite simple: I run several [OpenVPN][2] servers
so I can access services behind that server from anywhere - safely. 

OpenVPN uses TSL/SSL certificates to authenticate clients, this means that
server accepts connection from any client that presents a valid certificate.
"Valid" means certificate that is signed by a [CA][3] the server is configured
to trust. So each client needs it's private key (`.key`) and certificate (`.crt`).

Now the question is: how does one get these? Suppose the guy I'd like to give an
access my VPN server is far away. Indeed I can generate all this, sign it and send
him these files over email, Jabber, Slack channel - you name it. But is this 
safe enough? I'd say all these channels are insecure so some bad guy can intercept
the message, stole key and certificate and use it for unauthorized access to 
the VPN server under a name of my friend. 

So the question is: how does one get these safely? The answer is: the one has
to generate her own key herself and then send the certificate to the CA for signing. 
This is safe enough - private key need not to leave ones computer and certificate
is by definition public so can be safely sent to anyone. 

Here's

## How to create a private key and certificate signing request

*Step 1: Create SSL configuration file*

As we're using `openssl`, you'd need a configuration file for it. Following 
would do (save it to some file, say `ssl.conf`):

    [ req ]
    default_bits	= 2048
    distinguished_name	= req_distinguished_name
    prompt		= yes
    default_md		= sha256

    [ req_distinguished_name ]
    
    [ client ]
    basicConstraints	= critical, CA:FALSE
    subjectKeyIdentifier	= hash
    authorityKeyIdentifier	= keyid:always,issuer:always
    keyUsage		= digitalSignature, keyEncipherment
    nsCertType		= client
    extendedKeyUsage	= critical, clientAuth

*Step 2: Generate a new key* 

    openssl genrsa -out client.key -des3 2048

This will generate a new 2048 bit RSA (private) key and save it to `client.key`. This is the
secret file no-one else should have access too. The `-des3` encrypts the key using
a passphrase - you'll be asked for it. Depending on your paranoia you may increase
key bitsize to say 4096. 

*Step 3: Generate certificate signing request*

Certificate signing request (`.csr`) is a file that it later processed by the CA
to actually issue a signed certificate. Essentially, `.csr` contains a public key
plus some other metadata such as who's certificate it's going be. This is a lengthy 
command:

    openssl req -config 'ssl.conf' -new -key 'client.key' -out 'client.csr' -extensions client \
        -subj '/C=UK/ST=United Kingdom/L=Dundee/O=Some Company/emailAddress=some.guy@somecompany.co.uk/CN=Guy Some/'

Replace values in `-subj` accordingly - this identifies who's certificate it is. 
This produdes a file `client.csr` which we will need shortly.

*Step 4: Send `.csr` to the CA to get it signed*

Now take the `client.csr` you've created earlier and send it to the CA (in the
case of OpenVPN run by me, send it to me). The CA will (eventyally) sign it and
send you back the signed certificate (`.crt`) which then can be used to 
authenticate to VPN server (or for other purposes). As there's nothing secret
in the `.csr` not `.crt` one may use email, Jabber, Slack channel, you name it, 
to send it back and forth. 

# Final word

Hope that helps, have fun. 


[1]: https://en.wikipedia.org/wiki/Public_key_infrastructure
[1]: https://openvpn.net/
[3]: https://en.wikipedia.org/wiki/Certificate_authority

