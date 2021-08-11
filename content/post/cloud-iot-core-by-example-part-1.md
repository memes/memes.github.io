---
title: "Cloud Iot Core by Example Part 1"
date: 2017-11-01T17:17:22-07:00
draft: true
categories: [ "meetup", "work" ]
tags: [ "google", "gcp", "meetup", "ocgcp", "iot", "go", "angular4" ]
series: "Cloud IoT Core by Example"
subtitle: "Emulating a consumer device, registering with IoT Core, and publishing telemetry events"
---

Recently, the team at [Neudesic](https://www.neudesic.com/) have been
working on a consumer device that will use [Cloud IoT
Core](https://cloud.google.com/iot-core) as the connector between
physical device and the analytics processes that will consume the
data. Seems like a good topic for a meetup!

> Unfortunately I messed up the YouTube live stream this month, so
> there isn't a corresponding video to accompany this
> example, but the presentation that outlines some of this content can
> be found [here](https://goo.gl/LZF1h7). Sometime in the future I
> will record a new version, that may even go into part 2 of the
> series - data analysis!

This write-up will go into the details of emulating a consumer device
that is initially untrusted, through issuing a CA-signed certificate
and registering the device with Cloud IoT Core. At that point the
'device' will start streaming motion and location events to the
project via REST interface.
<!--more-->

# What is Cloud IoT Core?

IoT Core is Google's IoT connectivity offering; a fully managed,
global, scalable, data ingestion and distribution hub. It combines
secure MQTT and HTTP endpoints for incoming data and a device
registry that gives programmatic control of individual devices that

{{< figure src="/images/gcpiotcore/iotcore_hla.png" title="Figure 1: Cloud IoT Core architecture" >}}


## 1. Create a project to host the demo

The demo project will use billable resources, so must be linked to a
billing account; substitute your own billing account!

```bash
$ gcloud projects create ocgcp-iot-core
Create in progress for [https://cloudresourcemanager.googleapis.com/v1/projects/ocgcp-iot-core].
Waiting for [operations/pc.0000000000000000000] to finish...done.
$ gcloud beta billing projects link ocgcp-iot-core --billing-account 000000-000000-000000
billingAccountName: billingAccounts/000000-000000-000000
billingEnabled: true
name: projects/ocgcp-iot-core/billingInfo
projectId: ocgcp-iot-core

```

Since the project needs to use Cloud Pub/Sub and IoT Core, it is
important to ensure that these APIs are enabled, and that the service
account that IoT Core is using has access to Pub/Sub APIs. Just to be
certain, enable the APIs and grant access to IoT default service
account.

```bash
$ gcloud services enable pubsub.googleapis.com cloudiot.googleapis.com \
    --project ocgcp-iot-core
Waiting for async operation operations/tmo-acf.66ad1748-5160-486d-b92c-0ab976fbb6f4 to complete...
Operation finished successfully. The following command can describe the Operation details:
 gcloud services operations describe operations/tmo-acf.66ad1748-5160-486d-b92c-0ab976fbb6f4
Waiting for async operation operations/tmo-acf.618f3bf6-9652-4fd4-8746-67e734258baf to complete...
Operation finished successfully. The following command can describe the Operation details:
 gcloud services operations describe operations/tmo-acf.618f3bf6-9652-4fd4-8746-67e734258baf
$ gcloud projects add-iam-policy-binding ocgcp-iot-core \
    --member=serviceAccount:cloud-iot@system.gserviceaccount.com \
    --role=roles/pubsub.publisher
bindings:
- members:
  - user:matthew.emes@neudesic.com
  role: roles/owner
- members:
  - serviceAccount:cloud-iot@system.gserviceaccount.com
  role: roles/pubsub.publisher
etag: BwViwbFpv6c=
version: 1
```

## 2. Create Cloud Pub/Sub topic for the telemetry data

Telemetry data that devices send to Iot Core will be published to a
Pub/Sub topic that upstream services can process. Likewise, state
change data from devices can also be sent to another Pub/Sub
topic. Create the topics and corresponding subscriptions.

```bash
$ gcloud beta pubsub topics create shake-telemetry \
    --project ocgcp-iot-core
Created topic [projects/ocgcp-iot-core/topics/shake-telemetry].
$ gcloud beta pubsub topics create shake-state \
    --project ocgcp-iot-core
Created topic [projects/ocgcp-iot-core/topics/shake-state].
$ gcloud beta pubsub subscriptions create shake-telemetry-sub \
    --topic shake-telemetry \
    --project ocgcp-iot-core
Created subscription [projects/ocgcp-iot-core/subscriptions/shake-telemetry-sub].
$ gcloud beta pubsub subscriptions create shake-state-sub \
    --topic shake-state \
    --project ocgcp-iot-core
Created subscription [projects/ocgcp-iot-core/subscriptions/shake-state-sub].
```

## 3. Create the registry

All IoT projects use a _Registry_ to manage the set of devices that
can connect to the project. The next step is to create an empty
repository that can be used for the demonstration, and enable it to
use the Pub/Sub topics from prior step for telemetry and state data.

```bash
$ gcloud beta iot registries create memes-registry
--project=memes-sandbox --region=us-central1
--event-pubsub-topic=projects/memes-sandbox/topics/test

### Create a CA cert
$ make ca.pem

### Upload CA cert
gcloud beta iot registries credentials create --path ca.pem --registry
memes-registry --region us-central1

## Register a device

### Create a client certificate for authentication
$ make device-0.pem

### Register device
$ gcloud beta iot devices create device-0 --project memes-sandbox
--region us-central1 --registry memes-registry --public-key
path=device-0.pem,type=rs256

### Verify
$ gcloud beta iot devices list --registry memes-registry --region
us-central1
ID        NUM_ID            BLOCKED
device-0  2717837161731597


# Demonstration

For the [OC GCP](https://meetup.com/oc-gcp/) demo I used an Angular
web application that we ran on mobile devices brought to the
meeting.

## 1. Prepare the demonstration

Clone the GitHub repository to a local machine, or to a Cloud Shell
session running in an existing GCP project.

```bash
$ git clone https://github.com/NeudesicGCP/iot-core-demo.git
Cloning into 'iot-core-demo'...
remote: Counting objects: 77, done.
remote: Total 77 (delta 0), reused 0 (delta 0), pack-reused 77
Unpacking objects: 100% (77/77), done.
```

Cloud IoT Core requires a CA certificate that is used to authenticate
device connections. You can use any tool you like to create a
compatible CA certificate; the cloned source code contains a way

### 1.1 Create CA certificate using CFSSL

Edit ``Makefile`` and ``ca-csr.json`` files to change the DN settings
used in the certificate if you want. The files as-is will create a
certificate with distinguished name _CN=IoT Core Demo
CA/OU=GCP/O=Neudesic/L=Irvine/ST=California/C=US_.

```bash
$ cd certs
$ make ca.pem
$ make ca.pem
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
2017/11/12 11:58:25 [INFO] generating a new CA key and certificate from CSR
2017/11/12 11:58:25 [INFO] generate received request
2017/11/12 11:58:25 [INFO] received CSR
2017/11/12 11:58:25 [INFO] generating key: rsa-2048
2017/11/12 11:58:25 [INFO] encoded CSR
2017/11/12 11:58:25 [INFO] signed certificate with serial number 972520802569705297206480254400964207609756209759
```



##
diff --git a/registry/app.yaml b/registry/app.yaml
index d4ab7aa..6a9a8a3 100644
--- a/registry/app.yaml
+++ b/registry/app.yaml
@@ -13,11 +13,11 @@ env_variables:
   # Registry settings:-
      #
	     # Project ID
		 -  #REGISTRATION_SERVICE_PROJECTID: 'memes-sandbox'
		 +  REGISTRATION_SERVICE_PROJECTID: 'ocgcp-iot-demo'
		    # Location of registry
			   #REGISTRATION_SERVICE_LOCATIONID: 'us-central1'
			      # Name of registry
				  -  #REGISTRATION_SERVICE_REGISTRYID:
                     'memes-registry'
					 +  REGISTRATION_SERVICE_REGISTRYID:
                        'ocgcp-iot-demo'
						   # Device registration key expiration, in
                        format recognised by time.ParseDuration
						   #REGISTRATION_SERVICE_REGISTRY_EXPIRATION:
                        '720h'
						   #
						   diff --git
                        a/thingy/src/environments/environment.prod.ts
                        b/thingy/src/environments/environment.prod.ts
						index 92d6a80..3764e41 100644
						---
                        a/thingy/src/environments/environment.prod.ts
						+++
                        b/thingy/src/environments/environment.prod.ts
						@@ -2,5 +2,5 @@ export const environment = {
						   production: true,
						      debug: true,
							     registrationAuthToken: 'ocgcpDemo',
								 -  registrationURL:
                                    `https://registration-dot-memes-sandbox.appspot.com/`
									+  registrationURL:
                                       `https://registration-dot-ocgcp-iot-demo.appspot.com/`
									    };
