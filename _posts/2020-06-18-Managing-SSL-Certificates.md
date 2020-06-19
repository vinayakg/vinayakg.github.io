---
title: Managing SSL Certificates
#categories: [tech, learnings, productivity]
tags: [ssl, tls, devops, monitor, aws]
layout: post
permalink: /manage-ssl-tls-certificates
---



This article is about a SSL/TLS certificate expiry issue we had with one of the web properties I was managing. Wanted to share the lessons learnt and few thoughts around how certificates can be managed better using the current tools and how we can also automate most of the parts.



## Introduction   

Every website today uses [SSL/TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security) certificate to secure the content that is delivered to end users. And these certificates have a certain fixed expiry, generally 3 months - 1 year max. And **Shorter validity periods** limit damage from key compromise and mis-issuance, stolen keys and mis-issued certificates are valid for a shorter period of time. Which means we need to renew or create new certificates post expiry so that your users will continue to have a secure experience. Since there is a periodic renewal and upgrading/updating of certificates needed for the web properties, we need to monitor them for expiry, update/renew/create certificates and finally apply them to our web properties. We will see what we can do for managing/maintaining the certificates.

The below steps will apply even for folks who are creating new TLS certificates, since they will need to keep their certificates up to date.

 If you need more details around securing the certificates and details of SSL, this is beyond the scope of this article. 

Lets start looking at various steps involved in monitoring, updating/creating and applying them to your web properties.

## Monitoring Certificates

In 2020, if you had thought that there is no need to monitor certificates for expiry and other stuff, thats not the case though - even when you infrastructure is hosted on cloud providers like AWS which also offers TLS certificates via [ACM](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)

However, if you are hosted on [GitHub pages](https://guides.github.com/features/pages/) or [Netlify](https://www.netlify.com/) for personal websites (then they might take care for you). They both use [LetsEncrypt](https://letsencrypt.org/) for TLS infrastructure. Similar could be the case if you are running your commerce on shopify with a custom domain (have not tried myself)

Lets look at various options for monitoring our website SSL certificates. We will look at online tools and cli tools as well which can be easily automated in your workflow



### Online Tools

You may use online tools like [site24x7](https://www.site24x7.com/help/admin/adding-a-monitor/ssl-certificate-monitor.html), [letsencrypt monitor](https://letsmonitor.org/), or even do this in [nagios](https://kifarunix.com/monitor-ssl-tls-certificates-expiry-with-nagios/) or [solarwinds](https://www.solarwinds.com/server-application-monitor/use-cases/ssl-certificate-monitor) or [ssllabs](https://www.ssllabs.com/ssltest/analyze.html?d=vinayakg.dev&latest) tools

### CLI tools - validity

#### AWS SDK 

Applicable only if you are hosted on AWS and you have certificates created with ACM.

You can use `aws acm` to find the certificate you need. If you don't have `aws cli` installed, head [here](https://aws.amazon.com/cli/) and get started

```bash
aws acm list-certificates --certificate-statuses "ISSUED" "PENDING_VALIDATION" | jq '.CertificateSummaryList[] | select(.DomainName == "vinayakg.dev") | .CertificateArn'
```

The above command looks for certificates that are issued or are pending_validation and we run it through `jq` and filter the ones with my domain **vinayakg.dev** and project only CertificateArn. `jq` is a command line tool that lets you process json as you would do with any programming languages. You may learn more [here](https://stedolan.github.io/jq/)

We can use the above CertificateArn and use it to check the validity of our certificate with the following command

````bash
aws acm describe-certificate --certificate-arn "certificatearn" | jq '.Certificate.NotAfter'
````



#### Linux

There are ready made tools cli on Linux to find validity of certificates

````bash
ssl-cert-check -s vinayakg.dev -p 443
````

If you don't want to install `ssl-cert-check`, you may choose more primitive tools like below to find the same with some work

````bash
 curl --insecure -v https://vinayakg.dev 2>&1 | awk 'BEGIN { cert=0 } /^\* SSL connection/ { cert=1 } /^\*/ { if (cert) print }'
````

You may compute/use the no of days from the above script and then generate alerts basis checks.

 

### CLI Tools - Vulnerability

`````bash
testssl.sh vinayakg.dev
`````

This tool  checks a server's service on any port for the support of TLS/SSL ciphers, protocols as well as cryptographic flaws and much more. Give it a spin and you will be overwhelmed with the output, I would also its good to be able to see so many details.



### AWS Config

[AWS config](https://ap-south-1.console.aws.amazon.com/config) is a tool that can be used to set rules like `acm-certificate-expiration-check` to check for certificate expiration. Here is the complete [list](https://docs.aws.amazon.com/config/latest/developerguide/managed-rules-by-aws-config.html) of what you can do with those rules. There are some interesting things you can do to improve visibility and monitoring of storage, vpc, networking, and ec2 - will keep the details for other post and not digress from that.



## Upgrade/Renewal of Certificates

Like I mentioned in the introduction, certificates only have a certain after they are invalid and you need to renew/upgrade them.



AWS does not have any options to renew public certificates using `aws cli`, you can renew only private certificates.

### Create Certificate on AWS

#### Method 1

##### Creation

You can use `aws sdk` to create a certificate for **vinayakg.dev** then you could use the below command 

````bash
aws acm request-certificate --domain-name vinayakg.dev --validation-method DNS --idempotency-token 91224
````

In the above command we are requesting a new certificate for vinayakg.dev (remember we cant renew public certificates on AWS ACM), our validation method is DNS (dns entries can be automated and hence the choice) and also idempotency to make sure even accidental repeat requests are considered as one. For more options, please check this [link](https://docs.aws.amazon.com/cli/latest/reference/acm/).

##### Validation

We can also automate addition of dns entries to Route 53, if the domain is hosted there. If your domain on hosted on non-AWS infrastructure, its gonna be manual.

````bash
certificatearn=$(aws acm list-certificates --certificate-statuses "ISSUED" "PENDING_VALIDATION" | jq '.CertificateSummaryList[] | select(.DomainName == "longweekend.co.in") | .CertificateArn')
````

We get the certificate arn from the above command which we use to find the dns validation records with the help of below command

```bash
aws acm describe-certificate --certificate-arn $certificatearn | jq '.Certificate.DomainValidationOptions[] | select(.DomainName == "vinayakg.dev") | .ResourceRecord | {Name: .Name, Type: .Type, Value: .Value} '
```

Now we take the Name, Type and Value and use them to create the dns records

```bash
(hosted_zone_id)=$(aws route53 list-hosted-zones | jq '.HostedZones[] | select(.Name == "vinayakg.dev.") | .Id')

aws route53 change-resource-record-sets --hosted-zone-id $(hosted_zone_id) --change-batch $(change_batch) | jq -r '.ChangeInfo.Id' | cut -d'/' -f3
function change_batch()
  jq -c -n "{\"Changes\": [{\"Action\": \"INSERT\", \"ResourceRecordSet\": {\"Name\": \"$record_name\", \"Type\": \"CNAME\", \"TTL\": 300, \"ResourceRecords\": [{\"Value\": \"$record_value\"} ] } } ] }"
}
```



In the above script you need to set $record_name and $record_value from the `describe-certificate` cmd. Reference script is attached in the references.



#### Method 2

You may also use cloud formation template to link the aws hosted zone and add the dns records readily. The script is [here](https://github.com/jthomerson/lastweekingoogle.com/blob/07ed0573cf8611e7b89f270ea402f4c00c2e5d77/infra/lastweekingoogle-site/serverless.yml#L192-L218)



### LetsEncrypt

LetsEncrypt has become the default TLS provider for most orgs as it costs nothing for the organizations and LetsEncrypt lets you automate all aspects of Certificate management - this was built right  



### References

https://realguess.net/2016/10/11/a-cli-method-to-check-ssl-certificate-expiration-date/

https://www.shellhacks.com/openssl-check-ssl-certificate-expiration-date/

https://gist.github.com/justinclayton/0a4df1c85e4aaf6dde52