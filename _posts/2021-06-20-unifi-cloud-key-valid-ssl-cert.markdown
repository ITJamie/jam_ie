---
layout: post
title:  "Setting up a Unifi Cloud Key with a valid SSL cert"
date:   2021-06-20 21:00:00 +0100
categories: unifi debian cloudflare certbot ssl
---
So I recently moved from a self-hosted unifi controller to a unifi cloud key gen2. https://eu.store.ui.com/collections/unifi-accessories-cloud-key/products/unifi-cloud-key-gen2
I wanted to be able to access it locally though over https with a valid cert. 
It turns out the cloudkey just runs a basic debian strech install with standard apt commands! So heres how I made that happpen.

Pre-requisites: 
  * unifi cloud key
  * A domain with cloudflare DNS setup & API keys setup

First steps. 
  - Make sure the Cloud Key is upto date.
  - Enable SSH on the cloud key.
  - A subdomain setup with an A record of your local cloud key IP
  - Open an SSH session to the cloudkey. "root" is the username plus the password you setup in the cloudkey gui

Install certbot with cloudflare dns extension
{% highlight bash %}
apt update
apt install python3-certbot-dns-cloudflare
{% endhighlight %}

Create secrets dirs
{% highlight bash %}
mkdir ~/.secrets/
mkdir ~/.secrets/certbot
{% endhighlight %}

Create secrets file with your cloudflare api keys
{% highlight bash %}
vi ~/.secrets/certbot/cloudflare.ini
{% endhighlight %}
Example Content
```
# Cloudflare API credentials used by Certbot
dns_cloudflare_email = you@example.com
dns_cloudflare_api_key = wen_lambo?wen_moon?
```
Secure permissions on cloudflare creds (stops warnings in certbot output)
{% highlight bash %}
chmod 600 ~/.secrets/certbot/cloudflare.ini
{% endhighlight %}

Time to request the cert!
{% highlight bash %}
certbot certonly   --dns-cloudflare   --dns-cloudflare-credentials ~/.secrets/certbot/cloudflare.ini -d fqdn_of_your_cloudkey.example.com
{% endhighlight %}

Assuming that went well (read the output and debug if needed) move ahead to configure the cloudkey to use the key. Note the path of the cert and key from certbots output
{% highlight bash %}
cd /data/unifi-core/config/
cp unifi-core.crt unifi-core.crt.bk
cp unifi-core.key unifi-core.key.bk
rm unifi-core.crt
rm unifi-core.key
ln -s /etc/letsencrypt/live/fqdn_of_your_cloudkey.example.com/cert.pem unifi-core.crt # Your paths will be slightly different depending on your domain
ln -s /etc/letsencrypt/live/fqdn_of_your_cloudkey.example.com/privkey.pem unifi-core.key # Your paths will be slightly different depending on your domain
service unifi-core restart
{% endhighlight %}

Add the following to your root user crontab file (use `crontab -e`)
```
41 2 * * * certbot renew --post-hook "service unifi-core restart"
```

Now you should be able to access your cloudkey locally via `fqdn_of_your_cloudkey.example.com` with a valid cert!