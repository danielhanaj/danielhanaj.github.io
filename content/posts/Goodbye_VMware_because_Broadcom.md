authorLink = "/about/"
lightgallery = true
title = "Goodbye VMware because Broadcom"
date = "2024-10-28"
description = "Goodbye VMware because Broadcom"
featured = true
draft = false
categories = ["VMware"]
tags = [
    "VMware"
]

+++
It's time for changes. In this blog post I will describe why I have decided to move away from the VMware stack and change my focus on different technologies. 
 <!--more-->
## A little bit of history
I've been working with the VMware products since Virtual Infrastructure 3.5. This was about 15 years ago and at that time, this was a revolutionary. I was growing up with their technology as an engineer exploring different products from their stack. Starting from  VI 3.5, Lab Manager, vSphere, VMware Horizon View, NSX-V, NSX-T and Cloud Director at the end. The company was growing and was growing with the company working with their products. It was a really great pleasure. This company gaved me a lot and I've never had a problem finding a good, well paid job. But like every jurney this has come to an end with recent Broadcom aquisition.

## Why? 
The obvious answer is because Broadcom has acquired VMware and implemented multiple changes starting from licensing, pricing., reducing community and impacting product quality.  
Many of you may know how Broadcom operates. They are focusing on optimizing the costs and maximizing the profits. There has been many blog posts and videos that actually describe their strategy pretty clear. 
In this blog post I will try to summarize why I've made such decision. In upcomming posts I will provide the alternatives for VMware and which one is the most promessing in my opinion.

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