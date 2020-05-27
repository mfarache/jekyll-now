---
layout: post
title: API Gateway custom domain names using Route53 and AWS Certs Manager
tags: [aws, route53, api-gateway, dns]
---

![_config.yml]({{ site.baseurl }}/images/APIGW-ROUTE53-MAIN.png)

Recently we had to setup new domains for API Gateway endpoints in the project I'm working at, so I try to keep track on decisions we took while trying to automate as much as possible the setup in AWS.

- Register a new domain per API Gateway (using Custom DNS vs Route53)
- Adding SSL certificates to your domain (bring your own vs create a new one)
- Create custom domains for API Gateway
- Automate everything (using Serverless vs CloudFormation)

# To Route53 or not To Route53

In case you are not familiar, Route53 is a highly available and scalable cloud Domain Name System (DNS) web service. So if you are familiar with the kind of tasks that can be done using your favourite DNS provider, Route53 allows you to do that and probably a few things more such us DNS HealthChecks, buy domains, etc.

If you do not know what a DNS is, you can see it as a database where mappings are stored and allow the resolution from domains to ips or other domains. The specification of record types is huge so we will see some concepts on the way.

Initially we planned to register fresh new domains using Route53, but eventually we used our own DNS hosting because we wanted to repurpose our corporative domain with subdomains semantically related with the nature of our API endpoints.

Regardless of such decision, I will describe some concepts in the context of Route53 that can be extended to other DNS providers, as is probably useful for getting understanding on how it works.

1. Route53 allows one to create a DNS hosted Zone. We can chose its custom domain name and whether is public or private VPC. It will create Record set, mainly one of type NS and other of type SOA. We are interesting on the NS record, each of the entries is a name server. If we plan to register our domain we need those values handy.

2. Route 53 also enables configuring new domains from scratch.
   Chose your name and type (.com, .uk, .es, etc) and after a shopping-cart-like process and verification on emails you will be getting your domain per one year.

3. Route53 acts as a proper DNS so you can create records to cover different needs. Several examples that come to mind:

- Hosting a webserver in a EC2 instance with a fix elastc ip adress. We would need to create a record that will route any request to domain to beforwarded to our ec2 instance. Other scenarios are
- Creation of A record from www.mydomain.com , using no Alias to be routed to our IPv4 Address
- Creation of A record from mydomain.com , using Alias so is routed to www.mydomain.com
- Creation of C-NAME records to other domain.
- Creation of URL records for redirections.

**We will revisit this one later, as this is how we will be able to map our API Gateways**

The changes are not immediate and they will take a while to propagate

# SSL Certificates

Once we have our domain, if we want to secure traffic from the customer to our API or website, we should use certificated.

If you bring your own would be ideal but in our case our top level certificate is not meant to be issued for internal subdomains or wildcard.

So our options were reduced to use something like https://letsencrypt.org/es/ or find if it's equivalent in the AWS ecosystem was good enough. In a serverless world using AWS Cert manager seemed the way to go.

We will describe the steps to get a new public certificate using AWS Certificate manager

AWS Cert manager enables to Provision certificate of two different types:

- Public - Accepted by all browsers and operative systems
- Private - From the organization's certificate authority

We will focus on the public one. It's is totally free.

We can associate one or multiple domains to the certificate.
If we want to have subdomains we can also set those specific ones or use wildcard domains

Once we have our domains we need to validate them. Certificate can be validated in two different ways

- We can chose email validation only if we fully own the domains
- We can chose DNS Validation - By setting a DNS record.

AWS will suggest some CNAMES record that need to be introduced (One per domain)

In case you were using Route53 to set up your DNS records, AWS enables you to validate and create directly
CNAME records for each of the domains for validation purposes.

If you manage DNS externally you will have to add CNAME records differently modifying the zone using your DNS provider.
The status of the certificate remains pending till ALL the subdomain are validated.
The next screen show the certificate status after moving from Pending to Issued.

![_config.yml]({{ site.baseurl }}/images/APIGW-AWS-CERT-VALIDATION.png)

# Creation of custom domains

If you create an API Gateway endpoint AWS will assign automatically some default base url.
The url cannot be tagged as friendly names easy to be remembered.

If you use Serverless tool or any IAC tool such CloudFormation or Terraform you might find by accident that a removal and redeploy of the stack associated would assign a new domain totally different to the original one. This is not a problem in a dev environment, but once your service is life you want to be 100% sure that those mistakes do not jeopardize your customer, so having a static domain would help you to reduce/mitigate that risk

Here is where the concept of custom domains for API Gateway comes handy

The definition of a custom domain will create a target endpoint which is associated to API stage.
We can configure this endpoint to be regional or server by a cloufront optimized edge

![_config.yml]({{ site.baseurl }}/images/APIGW-CUSTOM-DOMAIN.png)

The highlighted target endpoint is the entry we will to add as a CNAME record in our DNS:

```
www.myfriendlydomain.com CNAME <target-endpoint>
```

# Automation

If you followed the steps you have realized that both domain registration and issuing certificates require certain manual validations so they are not good candidates to be automated. However we can automate the creation of custom domains.

We use Serverless framework to deploy anything AWS related, so a natural choice is simply find for the existance of any plugin that can do easily the job. Amplify provides a promising plugin

https://github.com/amplify-education/serverless-domain-manager

The plugin looks awesome as will help us to create custom domain names, create optional Route53 records and link with the certificates.

However there is a little gotcha! This plugin works in conjunction with the enpoint API definition.
Let me explain the implications:

Serverless creates a CloudFormation stack on deployment.
If we remove the stack it will remove not only the API itself... it will also remove the custom domain's targent endpoint, obligating us to modify our DNS record to repoint again to the new target endpoint.
This totally defeats the purpose as we do not want to change DNS everytime the stack is recreated...

Are there better ways? Yes!!!!

Luckily we came with an hybrid approach, ruling out the usage of the plugin in favour of plain CloudFormation resources.

I will provide some context to understand our proyect layout.

Let's imagine two services (A and B) served via API Gateway endpoint.
Each service has its own deployment lifecycle, secured in different way and therefore we use a different repository per service.

We created a new proyect where we define a custom domain mapping for API A and other for API B
Let's call this custom-domains. We will create here Custom domain resources.

```
resources:
  Resources:
    ServiceADomain:
      Type: AWS::ApiGateway::DomainName
      Properties:
        RegionalCertificateArn: ${self:custom.myCertificateArn}
        DomainName: friendlynameA-${self:custom.stage}.${self:custom.myDomainName}
        EndpointConfiguration:
          Types:
            - REGIONAL
        SecurityPolicy: TLS_1_2
    ServiceBCustomDomain:
      Type: AWS::ApiGateway::DomainName
      Properties:
        RegionalCertificateArn: ${self:custom.myCertificateArn}
        DomainName: friendlynameB-${self:custom.stage}.${self:custom.myDomainName}
        EndpointConfiguration:
          Types:
            - REGIONAL
        SecurityPolicy: TLS_1_2

```

Then on project A we created the mapping of API A against the custom domain we created on custom-domains.
We did the same for proyect B.

```
DomainNameMapping:
      Type: AWS::ApiGatewayV2::ApiMapping
      Properties:
        ApiId: !Ref ApiGatewayRestApi
        DomainName: ${self:custom.domainName}
        Stage: ${self:custom.stage}
```

So now no matter how many times we redeploy or remove stack from service A or B, the only thing that would be redeploying a mapping resource which does not impact.
