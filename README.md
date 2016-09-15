# Cisco ASA plugin for Let's Encrypt client

This is a plugin for the [Certbot](https://github.com/certbot/certbot) client from [Let's Encrypt](https://letsencrypt.org).

It answers TLSSNI01 challenges using Cisco ASA boxes and installs the resulting certificates onto the ASAs. Both operations leverage the [Cisco ASA REST API](http://www.cisco.com/c/en/us/td/docs/security/asa/api/qsg-asa-api.html).

Neither the plugin nor the Certbot client run *on* the ASA. They run on a management host which speaks ACME to the Let's Encrypt service and talks to the REST API on the ASA boxes over HTTPS. The process looks something like this:

1. Certbot talks to Let's Encrypt, does it's usual account and key management stuff.
2. Certbot learns from the certbot-asa plugin that the plugin prefers TLSSNI01 challenges.
3. Certbot requests a certificate for the specified domain(s), receives challenge info from Let's Encrypt.
4. Certbot creates the certificates (local self-signed nonsense, one per domain) which are used to satisfy the challenge.
5. Certbot asks the plugin to install the challenge certificates.
6. The certbot-asa plugin converts the challenge certs/keys to PKCS12 format, imports into the ASA.
7. The certbot-asa plugin configures the ASA to use these certificates to satisfy challenges with: `ssl trust-point <something> domain <whatever>`
8. Certbot tells Let's Encrypt to please validate our control of the domain by checkign the installation of these new certificates.
9. The certbot-asa plugin removes the challenge certificates/keys from the ASA.
10. Certbot generates a CSR, submits it to Let's Encrypt.
11. Let's Encrypt delivers a new server certificate and chain to Certbot.
12. The certbot-asa plugin installs the new server certificate and any required chain elements onto the ASA.
13. The certbot-asa plugin configures the ASA to use the new certificate with one or more `ssl trust-point <something> domain <whatever>` lines.

## Requirements

* One or more Cisco ASA boxes with the REST API installed and configured. [Compatibility info](http://www.cisco.com/c/en/us/td/docs/security/asa/compatibility/asamatrx.html#pgfId-131643).
* A Linux host on which to run the Let's Encrypt client (certbot).
* ASA credentials for use by the Linux host.
* Accurate-ish clocks on both the Linux host and the ASA(s).
* A *management name* by which the Linux host knows the ASAs. This must be different than the name end users use. Something like **asa-mgmt.company.com** or even just **asa-mgmt** is probably appropriate here. It does not need to use the physical management interface on the ASA. We just need a name by which we'll refer to the ASA.
* A TLS certificate used for management access to the ASA(s). This certificate will be for the management name above, and must be trusted by the Linux host. It's how the Linux host ensures it's not sending credentials to an attacker. Options here include:
  * A self-signed certificate generated right on the ASA (that's what I do.)
  * A real certificate from a well known CA (kind of defeats the purpose of trying to use Let's Encrypt.)
  * A certificate signed by your internal CA.

## Setup

### Install / Configure CentOS 7

I used CentOS 7, so these examples will go smoothly if you do too. But you can use whatever. It'd probably run on Windows.

Freshen up and install some packages we'll need:

```sudo yum -y update
sudo yum -y install git openssl-perl```

By default, python doesn't validate TLS certificates. Madness! Probably not
necessary with the `requests` module, but I've still got some `urllib2` stuff
knocking around in there. Don't want to send credentials to a bad guy!

```sudo sed -i 's/^verify=.*$/verify=enable/' /etc/python/cert-verification.cfg```

Create pointers to the ASA management interfaces in `/etc/hosts` or use DNS.
These are the names we use for *management access*, so they must be different
from the names for which we're getting certificates from Let's Encrypt. We'll get
certificates for these names shortly.

```echo "192.168.100.11 asa-mgmt" | sudo tee -a /etc/hosts```

### Install / Enable the REST API

Not much to it:

* Download the API bundle from Cisco, copy it to the ASA
* Specify to the image with: `rest-api image disk0:/asa-restapi-etc-etc-etc`
* Enable it with `rest-api agent`
* The api takes a few minutes to become available. I like to watch it with `debug rest-api agent`
* If your ASA is configured with `aaa authorization command` check out [bug](https://bst.cloudapps.cisco.com/bugsearch/bug/CSCuv80223) and [bug](https://bst.cloudapps.cisco.com/bugsearch/bug/CSCuw60598). You can work around the issues by either:
  * Adding ENABLE_1 and ENABLE_15 to you AAA server (nobody needs to know the password - it's for command authorization only)
  * Temporarily removing `aaa authorization command` while the REST API starts up. Not great at reboot time.
* Allow API access from your Linux host with `http <address> <mask> <interface>` on the ASA(s).

### Test the REST API

Now we'll be putting some of those building blocks together. This `curl` command tests:

* Our credentials
* The hostname resolution
* Whether HTTPS access is allowed to the box
* The API configuration

<pre># <b>curl -ksu <username>:<password> https://asa-mgmt/api/monitoring/serialnumber | sed 'a\'</b>
{"kind":"object#QuerySerialNumber","serialNumber":"XXXXXXXXXX"}
</pre>

### Now enable TLS for the management connection

The best thing to do here is probably to have your internal CA issue a certificate for **asa-mgmt**, then:
* Load the certificate, keys and any intermediate certificates onto the ASA
* Configure the ASA to use the new certificate with `ssl trust-point <trustpoint-name> domain asa-mgmt`
I use a self-signed certificate generated on the ASA for this purpose. It's good for 10 years, which is handy

## Caveats
* Removal of keys
* TLSSNI01 may be deprecated
