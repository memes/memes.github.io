---
title: "Migrate OC GCP domain to Google Cloud Platform, with DNSSEC enabled"
date: 2017-11-10T12:13:15-08:00
categories: [ "how-to" ]
tags: [ "google", "gcp", "meetup", "ocgcp", "dns", "dnssec" ]
series: "Migrating to GCP"
subtitle: "Preparing for a dynamic site on GCP, part 1"
---
As the demos for [OC GCP Meetup](https://meetup.com/oc-gcp/) get
better and more interactive, I thought it was time to have a single
domain that can be used to access them and to host future demos. This
post, and the [others](/series/migrating-to-gcp) that follow, will
show how to migrate or setup a new domain to use GCP services exclusively.

First step in the process is to transfer DNS responsibilities to [Cloud
DNS](https://cloud.google.com/dns/); I'll be using ocgcp.com domain in
these examples, but nothing here is unique to our meetup. Since it's
2017, [DNSSEC](http://www.dnssec.net/) will be enabled to offer as
much protection to our domain as possible.
<!--more-->

## 1. Create a zone in Cloud DNS for the domain

The first step is to create a new zone for the domain and to enable
DNSSEC. This can be achieved simply by using `gcloud beta dns` command
to create the zone and turn on DNSSEC.

```bash
$ gcloud beta dns managed-zones create ocgcp-root \
	--dns-name=ocgcp.com \
	--dnssec-state on \
	--description="Root zone for ocgcp.com with DNSSEC enabled"
Created [https://www.googleapis.com/dns/v1beta2/projects/ocgcp-projects/managedZones/ocgcp-root].
```

## 2. Add record sets to the zone

Cloud DNS will process changes to record sets within a transaction,
resulting in an all or nothing change to the zone.

### 2.1 Start the transaction

```bash
$ gcloud dns record-sets transaction start \
	--zone=ocgcp-root
Transaction started [transaction.yaml].
```

### 2.2 Add Google Webmaster domain verification record

I add a CNAME record to make sure that Google's tools can verify that
the domain is owned by me. Usually this is a TXT record to avoid
polluting sub-domains, but since I use a TXT record for SPF (way more
important than this entry) I have to use CNAME option for
verification.

```bash
$ gcloud dns record-sets transaction add \
	--ttl=3600 \
	--type=CNAME \
	--zone=ocgcp-root
	--name=n55v23lwpel4.ocgcp.com. \
	gv-atfe4vdgdiwhdg.dv.googlehosted.com.
Record addition appended to transaction at [transaction.yaml].
```

### 2.3 Add email records

I use an alias domain for ocgcp.com, so MX records for gmail and a
simple SPF rule are needed. Note that all MX servers are added in a
single MX entry; spaces will be used as record markers, so the space
between priority and server must be escaped so that it is not
interpreted as a record demarcation.

```bash
$ gcloud dns record-sets transaction add \
	--name=ocgcp.com. \
	--ttl=3600 \
	--type=MX \
	--zone=ocgcp-root \
	1\ ASPMX.L.GOOGLE.COM. 5\ ALT1.ASPMX.L.GOOGLE.COM. 5\ ALT2.ASPMX.L.GOOGLE.COM. 10\ ALT3.ASPMX.L.GOOGLE.COM. 10\ ALT4.ASPMX.L.GOOGLE.COM.
Record addition appended to transaction at [transaction.yaml].
$ gcloud dns record-sets transaction add \
	--name=ocgcp.com. \
	--ttl=3600 \
	--type=TXT \
	--zone=ocgcp-root \
	"\"v=spf1 include:_spf.google.com ~all\""
Record addition appended to transaction at [transaction.yaml].
```

Finally, add the DKIM record; note that the value has to be broken
into pieces of 255 bytes or less surrounded by embedded
double-quotes.

```bash
$ gcloud dns record-sets transaction add \
	--name=ocgcp.com. \
	--ttl=3600 \
	--type=TXT \
	--zone=ocgcp-root \
	--name=google._domainkey.ocgcp.com \
	"\"v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAi00VSESgcIZhr7nnDqCZ4nZqe4CEM/lqqzKmkoWrzYeTPU7cmzqZ/THD44V6+ftNZexplaDaQN1K+qywWXaocyjaefDUv9BPZzPZ7yPz+udDGa0Q7xT0AcTJApJwjaUJ5Hg/\" \"Gju26lV96IZv/QzHAVbgleE5gO89stM/TFpgjFquXObLFgZZDT/nG++sYTL3K10rcloOBvjTncI50mgV0Wd1+5CVeRyAUZUS7LodgSqYUZswJwFidRz/0AYjTd62+kvvHwKerRU7CNlbr5SI23l5//OG+9cBy7LFCGZkRcyvK/oxL7LBb2be4HckJL6uYDzByGzVbfjLY+xyXwfHVwIDAQAB\""
Record addition appended to transaction at [transaction.yaml].
```

### 2.4 Add CNAME for hosting from GCS

We have a simple website ready to roll, and since it's fully static it
can be served from Google Cloud Storage bucket. Unfortunately, GCS
doesn't permit hosting the root domain from a bucket, but we can start
with a www sub-domain.


```bash
$ gcloud dns record-sets transaction add \
	--ttl=3600 \
	--type=CNAME \
	--zone=ocgcp-root
	--name=www.ocgcp.com. \
	c.storage.googleapis.com.
Record addition appended to transaction at [transaction.yaml].
```

### 2.5 Commit the transaction

Executing (committing) the transaction will apply all the record set
changes in one operation.

```bash
$ gcloud dns record-sets transaction execute --zone=ocgcp-root
Executed transaction [transaction.yaml] for managed-zone [ocgcp-root].
Created [https://www.googleapis.com/dns/v1/projects/ocgcp-projects/managedZones/ocgcp-root/changes/2].
ID  START_TIME                STATUS
2   2017-11-10T20:17:32.438Z  pending
```

Assuming there are no errors, the additional record sets added above
will be present.

## 3. Host the static website on GCS

In a later post, I'll walk through making a dynamic site with content
from Container or App Engine, but for now a simple static site will
demonstrate that the domain is up and running.

**Note:** The biggest issue with this approach in 2017 is that the
content can only be served via HTTP; GCS will not serve the content as
HTTPS. We'll address that once the site is dynamically hosted.

### 3.1 Create a new bucket for the site

```bash
$ gsutil mb gs://www.ocgcp.com/
Creating gs://www.ocgcp.com/...
$ gsutil defacl ch -u AllUsers:R gs://www.ocgcp.com
Updated default ACL on gs://www.ocgcp.com/
```

### 3.2 Transfer the site files to the bucket

```bash
$ gsutil -m rsync -R ./ gs://www.ocgcp.com/
Building synchronization state...
Starting synchronization
Copying file://./css/vendor.css [Content-Type=text/css]...
Copying file://./css/base.css [Content-Type=text/css]...
Copying file://./favicon.png [Content-Type=image/png]...
Copying file://./css/main.css [Content-Type=text/css]...
Copying file://./js/modernizr.js [Content-Type=application/javascript]...
Copying file://./images/oc-gcp-meetup-background.jpg [Content-Type=image/jpeg]...
Copying file://./images/ocgcp-logo.png [Content-Type=image/png]...
Copying file://./js/jquery-2.1.3.min.js [Content-Type=application/javascript]...
Copying file://./index.html [Content-Type=text/html]...
Copying file://./js/plugins.js [Content-Type=application/javascript]...
Copying file://./js/main.js [Content-Type=application/javascript]...
| [12/12 files][  8.7 MiB/  8.7 MiB] 100% Done
Operation completed over 12 objects/8.7 MiB.
```

### 3.3 Use `index.html` for all pages, including 404s

The example site only has a single HTML page, so let's set that as the
default to use for index and 404 pages.

```bash
$ gsutil web set -m index.html -e index.html gs://www.ocgcp.com
Setting website configuration on gs://www.ocgcp.com/...
```

## 4. Update Domain Registration settings

Follow the instructions provided by your domain registrar to update
the name servers for the domain. The new name servers to use are:-

* ns-cloud-e1.googledomains.com
* ns-cloud-e2.googledomains.com
* ns-cloud-e3.googledomains.com
* ns-cloud-e4.googledomains.com

Configuring DNSSEC will require that your domain registrar supports
adding the DNSSEC keys and configuration for your domain. Even though
Cloud DNS will be providing all lookups, DNSSEC requires that the keys
be registered at the original domain registrar.

### 4.1 Complete DNSSEC registration with original registrar
Assuming that your domain registrar does support DNSSEC, the next step
is to find the DNSSEC key and configuration. You can get this from
Network Services / Cloud DNS section of cloud console, or from the
trusty command line.

First, list the keys associated with the newly created zone.

```bash
$ gcloud gcloud beta dns dnskeys list --zone=ocgcp-root
ID  KEY_TAG  TYPE         IS_ACTIVE  DESCRIPTION
0   60071    keySigning   True
1   36546    zoneSigning  True
```

The *keySiging* entry is the one that is needed here; in the output it
is the key with ID **0**. Describing that key will provide the details
needed for domain registrar.

```bash
$ gcloud beta dns dnskeys describe 0 --zone=ocgcp-root
algorithm: rsasha256
creationTime: '2017-11-10T18:02:06.346Z'
digests:
- digest: 3109B7DB8ABC274BDD259928F0C81118D386E299090E70C0F2BCAA22A3240E5F
  type: sha256
dsRecord: 60071 8 2 3109B7DB8ABC274BDD259928F0C81118D386E299090E70C0F2BCAA22A3240E5F
id: '0'
isActive: true
keyLength: 2048
keyTag: 60071
kind: dns#dnsKey
publicKey: AwEAAYOdwvK//Y13cwUy7OYbDFC0XYRby96ZHZ+VxuRGj1S5EmkVbtCfB/KKDjVXlfqjXAYJJjxSi2r3hNE07mSmrrGGPRY8bJ3qOxFpjXnP+4aiuzoY4W1GHuLhI90/tlSf9lDjiAQ8c8nXNBNB0v57NO37d1du/1Mu80SzrcucuBHZORJDFMC/nnhL6bjUEEF1OQzSJ3qvtX+noKA4xfe9m60FJ1oYj1cH9sV7Mvj8BOhna272sEWQJXTPgsJWi61ASxxuDHiNHIJtx8hQU6jCBCXeAfJmg9+1hO7dUdOvtb+LLTmYS6gZP/EZ5cKgSh5HkDYCAqrsoiQxozuA+eiHAYk=
type: keySigning
```

Modify the DNSSEC records for your domain registrar, using the values
for *keyTag*, *algorithm*, *digest*, and *digest type*.

Once complete, you can verify DNSSEC is working by using Verisign's
[DNSSEC Analyzer][1]; it will
analyse your DNS records and confirm that DNSSEC is setup completely.

The image below shows the output from the [analyzer][1] for ocgcp.com.

{{< figure src="/images/gcpdnssec/dnssec_analyzer.png" title="DNSSEC Debugger output for ocgcp.com" >}}

## 5. Verify the site is running

Finally, open [www.ocgcp.com](http://www.ocgcp.com/) in a browser to
confirm the site is loading and that DNS is working correctly.

Success!

[1]: https://dnssec-debugger.verisignlabs.com
