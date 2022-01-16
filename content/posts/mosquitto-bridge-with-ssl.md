---
title: "Mosquitto Bridge With Ssl"
date: 2022-01-16T21:34:43+01:00
draft: false
---

Recently one of my homeservers went haywire with a constant load of
+- 80 in 1/5/15.
Unrelated to this, buit it seems there is an issue with my local vaultwarden
instance.

But this got me thinking. What if my local mqtt server goes down..

I am running a local mosquitto server on my OpenWRT router
(A Mikrotik 750Gr3) to support my growing love for home
automation. Normally this should be quite stable, but I had issues with
OpenWRT in the past. Won't it be nice to have a bridge as backup, so in the
event that my mqtt server acts up, I can switch to that one, without losing any
data.

## Setting up an MQTT server

Using your favourite package manager, you can install mosquitto, an mqtt server.

Because this server will be running on a separate network, SSL is a must.
Setting this up with LetsEncrypt and DNS auth is quite easy.

### Certificates

I Already have a dns challenge setup for other domains, so reusing this. Maybe I'll document this in the future.

```sh
certbot certonly --preferred-challenges dns --authenticator dns-hetzner --dns-hetzner-credentials /etc/letsencrypt/hetzner.ini -d mqtt.server.domain
```

### Configuration

I like to keep my configurations quite clean. Luckly, Mosquitto supports splitting up the confgiration files by setting the configuration parameter `include_dir` and point it to a sensable path, like `/etc/mosquitto/conf.d/`.

Then, I created 3 files in that directory:

- 0-security.conf
- 10-listener-localhost.conf
- 20-listener-public-mqtt.server.domain-ssl.conf

Where `0-security.conf` contains the configured security and authentication
sections, disallowing anonymous access and specifing the password file:

```
# =================================================================
# Security
# =================================================================

# If set, only clients that have a matching prefix on their
# clientid will be allowed to connect to the broker. By default,
# all clients may connect.
# For example, setting "secure-" here would mean a client "secure-
# client" could connect but another with clientid "mqtt" couldn't.
#clientid_prefixes

# Boolean value that determines whether clients that connect
# without providing a username are allowed to connect. If set to
# false then a password file should be created (see the
# password_file option) to control authenticated client access.
#
# Defaults to true if no other security options are set. If `password_file` or
# `psk_file` is set, or if an authentication plugin is loaded which implements
# username/password or TLS-PSK checks, then `allow_anonymous` defaults to
# false.
#
#allow_anonymous true
allow_anonymous false

# -----------------------------------------------------------------
# Default authentication and topic access control
# -----------------------------------------------------------------

# Control access to the broker using a password file. This file can be
# generated using the mosquitto_passwd utility. If TLS support is not compiled
# into mosquitto (it is recommended that TLS support should be included) then
# plain text passwords are used, in which case the file should be a text file
# with lines in the format:
# username:password
# The password (and colon) may be omitted if desired, although this
# offers very little in the way of security.
#
# See the TLS client require_certificate and use_identity_as_username options
# for alternative authentication options. If an auth_plugin is used as well as
# password_file, the auth_plugin check will be made first.
#password_file
password_file /etc/mosquitto/passwd
```
`10-listener-localhost.conf` contains a localhost listener, handy for debugging:

```
listener 1883 localhost
```

and at last `20-listener-public-mqtt.server.domain-ssl.conf` containting the
public listener, with ssl configured:

```
listener 1883
protocol mqtt
certfile /etc/letsencrypt/live/mqtt.server.domain/cert.pem
cafile /etc/letsencrypt/live/mqtt.server.domain/chain.pem
keyfile /etc/letsencrypt/live/mqtt.server.domain/privkey.pem
```

So far, so good. 

Let's quickly configure the `renew_hook` for certbot to give mosquitto a kick when the cert is renewed:

```sh
echo "renew_hook = systemctl restart mosquitto" >> /etc/letsencrypt/renewal/mqtt.server.domain.conf
```

## Bridging

Now, we need to configure the local mosquitto on my router to bridge with the
remote one.

I'm using the LuCi configuration page here to setup the bridge which generates
the following configuration file:

```
# mosquitto.conf file generated from UCI config.
log_dest syslog
port 1883

listener 1888
protocol mqtt


# Bridge connection from UCI section
connection mqtt.subutux.be
address mqtt.server.domain:1883
notifications true
remote_password long-password-here
remote_username username
start_type automatic
topic # out 2
try_private true
bridge_capath /etc/ssl/certs # This one is important
```
First, I didn't know that **mosquitto only initiates an ssl connection if you specify `bridge_cafile` or `bridge_capath`** [man page](https://mosquitto.org/man/mosquitto-conf-5.html).

So i point `bridge_capath` to my router's global certificate store.

But, I kept receiving log messages signalling that there is still something wrong. My lets encrypt certificate doesn't seem to be validated. :/

Looking into `/etc/ssl/certs` it only contained one certificate.
After installing the package `ca-certificates` in OpenWRT, The connection became
active and and all my mqtt topics where (one-way) synchronized!

