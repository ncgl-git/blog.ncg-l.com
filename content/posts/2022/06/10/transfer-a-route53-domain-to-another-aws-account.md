---
title: "Route53: Transfer domain to another AWS account"
date: 2022-06-10T09:03:20-08:00
draft: false
categories: [developer guide]
---

In the source AWS account, verify that you have this domain:

```
    aws \
    --region <domain region> \
    --profile <source profile name> \
    route53domains list-domains
```

Initiate the domain transfer:

```
    aws \
    --region <domain region> \
    --profile <source profile name> \
    route53domains transfer-domain-to-another-aws-account \
    --domain-name <your domain name> \
    --account-id <target account id>
```

You will receive an operation id and password from this request, save both of these for later commands.

```
    aws \
    --region <domain region> \
    --profile <source profile name> \
    route53domains get-operation-detail \
    --operation-id=<operation id from previous command>
```

You will receive a payload that look similiar to this:

```
{
    "OperationId": <operation id>,
    "Status": "IN_PROGRESS",
    "DomainName": <domain name>,
    "Type": "INTERNAL_TRANSFER_OUT_DOMAIN",
    "SubmittedDate": 1653969049.349
}
```

Lastly, we'll change AWS profiles to the target AWS account and accept the transfer request:

```
    aws \
    --region <domain region> \
    --profile <target profile name> \
    route53domains accept-domain-transfer-from-another-aws-account \
    --domain-name=ncg-l.com \
    --password=<password from transfer initiation command>
```

And that's it.

P.S. - One thing to note is that the DNS entries on the source domain name are blown and will need to be recreated.