+++
author = "Daniel Hanaj"
authorLink = "/about/"
lightgallery = true
title = "SSL Handshake Failure after upgrading to Cloud Director 10.5"
date = "2023-10-07"
description = "SSL Handshake Failure after upgrading to Cloud Director 10.5"
featured = true
draft = false
categories = ["VMware"]
tags = [
    "VMware",
    "Cloud Director",
    "Linux"
]
thumbnail = "images/2023/10/07/lb.png"

featureImage = "/images/2023/10/07/lb.png"
+++
If you use NSX-T Load Balancer to load balance Cloud Director cell traffic you may see SSL Handshake Failure in Server Pools after upgrade to CD 10.5
 <!--more-->
 This will be a short post in case you have experienced SSL Handshake Failure on you NSX-T load balancer after upgrading to Cloud Director 10.5.

Couple of days ago I have upgraded Cloud Director from 10.4.2 to 10.5. The upgrade went smoothly, but NSX-T Load Balancer, that we use to load balance traffic to CLoud Director cells, started to report an error message. After checking the status in Server Pools, a pool members started to report SSL Handshake Failure and LB service went down. 

At that time I had to bypass LB to allow users to login into the environment.

## VMware Support case
Next day I have reported issue to VMware. a Cloud Director Support engineer contacted me and made some basic troubleshooting. 
After short check nothing was found on Cloud Director side. I was advised to report that to NSX-T Support Team.
Day after I had a call from NSX-T Support engineer where basic traffic capture was made.

## The solution
The issue was a lacking of allowed ciphers on Cloud Director cells after upgrading to 10.5.

To check the cipher status you need to go to fallowing directory
```console
 cd /opt/vmware/vcloud-director/bin/
```

To check allowed ciphers run this command:
```console
./cell-management-tool ciphers -l
```

In the output you should get 
```console
Allowed ciphers:
* TLS_AES_256_GCM_SHA384
* TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
* TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
```

Unfortunately this list is not enough for NSX-T to establish the session with cells.

The advice I got from VMware Support was to first add all allowed ciphers using following command:

```console
./cell-management-tool ciphers -d
```

Once done, allowed ciphers list looked like this:
```console
Allowed ciphers:
* TLS_AES_256_GCM_SHA384
* TLS_AES_128_GCM_SHA256
* TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
* TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
* TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
* TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
* TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA
* TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA
* TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384
* TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
* TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384
* TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
* TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA
* TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA
* TLS_RSA_WITH_AES_256_GCM_SHA384
* TLS_RSA_WITH_AES_128_GCM_SHA256
* TLS_RSA_WITH_AES_256_CBC_SHA256
* TLS_ECDH_RSA_WITH_AES_256_CBC_SHA
* TLS_RSA_WITH_AES_256_CBC_SHA
* TLS_RSA_WITH_AES_128_CBC_SHA256
* TLS_ECDH_ECDSA_WITH_AES_256_CBC_SHA
* TLS_ECDH_ECDSA_WITH_AES_128_CBC_SHA
* TLS_ECDH_RSA_WITH_AES_128_CBC_SHA
* TLS_RSA_WITH_AES_128_CBC_SHA
```  

This was not the end as now we had to shrink the list leaving only required ciphers. According VMware Support only following ciphers should be allowed:
```
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA
TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA
TLS_RSA_WITH_AES_128_GCM_SHA256
TLS_RSA_WITH_AES_128_CBC_SHA256
TLS_RSA_WITH_AES_128_CBC_SHA
TLS_EMPTY_RENEGOTIATION_INFO_SCSV
TLS_AES_256_GCM_SHA384
TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
``` 

To remove unnecessary ciphers following command was used:
```console
./cell-management-tool ciphers -d TLS_AES_128_GCM_SHA256, TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA, TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA, TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384, TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384, TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256, TLS_RSA_WITH_AES_256_GCM_SHA384, TLS_RSA_WITH_AES_256_CBC_SHA256, TLS_ECDH_RSA_WITH_AES_256_CBC_SHA, TLS_RSA_WITH_AES_256_CBC_SHA, TLS_ECDH_ECDSA_WITH_AES_256_CBC_SHA, TLS_ECDH_ECDSA_WITH_AES_128_CBC_SHA, TLS_ECDH_RSA_WITH_AES_128_CBC_SHA
```

With this single command we remove unwanted ciphers.

To validate the output we had to run list command again:
```console
./cell-management-tool ciphers -l
```

The output we got was:
```console
* TLS_AES_256_GCM_SHA384
* TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
* TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
* TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
* TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
* TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
* TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA
* TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA
* TLS_RSA_WITH_AES_128_GCM_SHA256
* TLS_RSA_WITH_AES_128_CBC_SHA256
* TLS_RSA_WITH_AES_128_CBC_SHA
```

Next we had to repeat this procedure on remaining cells.

Once completed you need to reboot all the cells to apply changes.

After that change the cell status changed to "Up" in NSX-T Load Balancer.

At the end that was a Cloud Director issue which was expected as I have upgraded only Cloud Dirctor cells, not NSX-T. 

Thats all. Thanks for reading.