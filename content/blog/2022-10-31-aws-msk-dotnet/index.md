---
title: Connecting .NET to AWS Kafka (MSK)
date: 2022-10-31
categories: dotnet kafka aws msk openssl
---

This guide looks at connecting a .NET Core client to an AWS Kafka (MSK) cluster using Mutual TLS. We're going to focus on the mutual TLS authentication, how to make certificates, a CA and certificate requests.

> WARNING: You will be charged by AWS.

Much of the instructions below will include a charge by AWS. That's my only warning.

## Pre-requisites

- `openssl` [Windows binaries here](https://wiki.openssl.org/index.php/Binaries)
- [AWS CLI access](https://aws.amazon.com/cli/)

## AWS API and Kafka Connections

The first aspect to wrap your head around is that there are two API. AWS MSK and Confluent Kafka. 

AWS MSK is for managing the cluster itself and needs IAM permissions. You use the [AWSSDK.Kafka nuget package](https://www.nuget.org/packages/AWSSDK.Kafka/) for that. That package will **not** let you connect to the Kafka brokers themselves. Only for cluster admin.

Confluent Kafka is for producing/consuming messages to/from the Kafka queue itself. You use the [Confluent.Kafka nuget package](https://www.nuget.org/packages/Confluent.Kafka/) for that.

If you're not going to be administering the AWS MSK cluster through code then you don't need the AWSSDK!

## Not covering generic Kafka use

There are lots of guides on how to use Confluent.Kafka to connect to a Kafka cluster from .NET. I liked [Developing .NET Core Applications for Apache Kafka and Amazon MSK](https://awswith.net/2020/11/01/developing-net-core-applications-for-apache-kafka-and-amazon-msk/) although it is very light on the detail on the MSK part.

[The official documentation](https://docs.confluent.io/kafka-clients/dotnet/current/overview.html) is also good but leans heavily on Confluent's own cloud.

## Create an AWS Private Certificate Authority

A [certificate authority](https://www.digicert.com/blog/what-is-a-certificate-authority) (CA) is usually a company or organisation that validates websites, emails and so on. A *private* certificate authority is where *you* are the organisation doing the validating (through AWS) for your own services inside AWS.

You can use certificates issued by [Verisign](https://www.verisign.com/en_US/website-presence/online/ssl-certificates/index.xhtml) or [LetsEncrypt](https://letsencrypt.org/) or some other issuer but in this case it's your AWS service and your client, so you can trust you. If any third party wants to use it, you can send the certificate to them. We'll come onto that later.

The MSK cluster is going to use that certificate authority in two ways:

1. It's going to authenticate clients using it.
1. It's going to let clients authenticate it.

(that's why it's [mutual transport layer security](https://www.cloudflare.com/en-gb/learning/access-management/what-is-mutual-tls/), both sides have to authenticate that they know who they are)

### Steps

To create a private CA, go to [AWS's Private Certificate Authority](https://eu-west-2.console.aws.amazon.com/acm-pca/home?region=eu-west-2#/firstrunscreen) (beware the link is to eu west 2 region).

Settings:

- Type: Root
- O,OU,C,CN, etc: set these sensibly for your company
- Key algorithm: RSA 2048 is fine at time of writing
- Certificate revocation options: none - they don't work with Kafka. You use Kafka Access Control Lists (ACLs) to control revocation instead.
- CA permissions options: Check
- Pricing: ⚠ Check (it's going to cost!)
- Click Create CA.

That's it. Click through to your certificate and make a note of the new private certificate authority Amazon Resource Name (ARN). Being the root of a certificate chain, the Certificate Authority has a special role but it's just a certificate like any other. It will look something like this:

```
-----BEGIN CERTIFICATE-----
MIIEKjCCAxKgAwIBAgIQaEuWKnMW5/K/++B9NulGxTANBgkqhkiG9w0BAQsFADCB
hTELMAkGA1UEBhMCR0IxHTAbBgNVBAoMFE15IENsaW5pY2FsIE91dGNvbWVzMRQw
HQOr3V2CW1z6Idh6RS4=
-----END CERTIFICATE-----
```

(Not a real certificate, just me banging the keyboard).

## Export, convert the Private CA

The .NET client needs to make sure that the Kafka server it is talking to is the right one. If you're using a Private CA then you will need to install it in the Windows Certificate store on every computer that needs a connection to Kafka (including giving it to 3rd parties if others are connecting to it).

 - Log into the AWS Console.
 - Find your Private CA, click Actions->Get CA Certificate
 - Click "Export certificate body to a file"
 - A file called `Certificate.pem` is downloaded.
 - Rename the file `kafka_private_ca.pem`

The .NET client needs this certificate installed into the "Trusted Root Store" of the Certificate Manager. But `.pem` files are not supported, we need to convert it using `openssl`:

 - Open Powershell to `kafka_private_ca.pem`
 - Use `openssl` to convert it to pkcs12 format:

```powershell
openssl pkcs12 -export -out kafka_private_ca.p12 -in kafka_private_ca.pem -nokeys
```

 - Add an export password and make a note of it, you will need it to install it

NB: We need to use `-nokeys` because there are no private keys in the PEM file, openssl doesn't know that unless we tell it.

 - In windows explorer, double click the `kafka_private_ca.p12` and then go through the Certificate Import Wizard.
 - Store Location: Local Machine
 - File to Import: path should be to kafka_private_ca.p12
 - Password: that you just set on `openssl pkcs12 export`
 - Automatically detect store
 - Next and Finish.

The AWS Private CA will now be installed on this computer. You will need to repeat the process for any computer you're using it on. If a 3rd party wants to connect to your client, they will need this certificate too.

## Create a Client Certificate

MSK needs to know that the client is authenticated to make calls. For that, the client needs it's own certificate. Here are the steps:

- Create a new certificate called a client certificate with a signing request
- Use the signing request to ask the Private CA on AWS to sign the client certificate
- Get the newly signed client certificate from AWS
- Convert the newly signed client certificate into a format that the .NET Kafka client can use

### Create a new client certificate and signing request

- In PowerShell
- Create new certificate with signing request:

```powershell
openssl req -newkey rsa:2048 -nodes -keyout kafka_test_client.key -out kafka_test_client.csr
```

That will generate the key first:

```
Generating a RSA private key
........................................................................................................................................................+++++
.......................+++++
writing new private key to 'kafka_test_client.key'
```

Then ask you to fill in details for the certificate request:

```
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:UK
State or Province Name (full name) [Some-State]:London
Locality Name (eg, city) []:London
Organization Name (eg, company) [Internet Widgits Pty Ltd]:My Company
Organizational Unit Name (eg, section) []:Development
Common Name (e.g. server FQDN or YOUR name) []:mycompany.com
Email Address []:me@mycompany.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []: APassword
An optional company name []: My Company
```
 - Put in answers for country name (AU), Province, City, Organisation,
 - FQDN is the Fully Qualified Domain name - you don't need one of these but if you have a corporate web domain, use that: mycompany.com
 - (Optional) It's recommended that set a password. It can be blank if you are going to protect your certificate in some other way. If you use a password then make sure you store that in a secure place away from your certificates.

At the end of that, two files are generated:

 - The certificate signing request: `kafka_test_client.csr` 
 - The certificate's private key: `kafka_test_client.key`

> Keep the .key file secret!

### Sign the client certificate

We're now going to take the certificate signing request and ask our AWS Private CA to sign it for us.

 - From AWS, get the ARN of your Private CA certificate, e.g. `arn:aws:acm-pca:eu-west-2:123456789:certificate-authority/11111111-1111-1111-1111-111111111111`
 - Open PowerShell
 - Sign the certificate:

```powershell 
aws acm-pca issue-certificate --certificate-authority-arn "arn:aws:acm-pca:eu-west-2:123456789:certificate-authority/11111111-1111-1111-1111-111111111111" --csr fileb://kafka_test_client.csr --signing-algorithm "SHA256WITHRSA" --validity Value=300,Type="DAYS"
```

 - The result will be the ARN of the signed certificate in a JSON snippet:

```json
{
    "CertificateArn": "arn:aws:acm-pca:eu-west-2:123456789:certificate-authority/11111111-1111-1111-1111-111111111111/certificate/11111111111111111111111"
}
```

> Note: that's just the id of the certificate, we haven't downloaded it yet!

 - Using the `CertificateArn` in the previous step, download the certificate:

 ```powershell
aws acm-pca get-certificate --certificate-authority-arn "arn:aws:acm-pca:eu-west-2:123456789:certificate-authority/11111111-1111-1111-1111-111111111111" --certificate-arn "arn:aws:acm-pca:eu-west-2:123456789:certificate-authority/11111111-1111-1111-1111-111111111111/certificate/11111111111111111111111" --output text
 ```

 - The result looks like (not real certificates, real ones are longer):

```
-----BEGIN CERTIFICATE-----
MIIEKjCCAxKgAwIBAgIQaEuWKnMW5/K/++B9NulGxTANBgkqhkiG9w0BAQsFADCB
hTELMAkGA1UEBhMCR0IxHTAbBgNVBAoMFE15IENsaW5pY2FsIE91dGNvbWVzMRQw
HQOr3V2CW1z6Idh6RS4=
-----END CERTIFICATE-----       -----BEGIN CERTIFICATE-----
MIID2DCCAsCgAwIBAgIQQJDUBmoYft8TvHvqvHN65DANBgkqhkiG9w0BAQsFADCB
hTELMAkGA1UEBhMCR0IxHTAbBgNVBAoMFE15IENsaW5pY2FsIE91dGNvbWVzMRQw
EgYDVQQLDAtEZXZlbG9wbWVudDEPMA0GA1UECAwGTG9uZG9uMR8wHQYDVQQDDBZt
```

- Copy the whole result into a new file `kafka_test_client.cer`, this is the client certificate.
- Replace all the characters between `-----END CERTIFICATE-----` and `-----BEGIN CERTIFICATE-----` with a return carriage so that it looks like this:

```
-----BEGIN CERTIFICATE-----
MIIEKjCCAxKgAwIBAgIQaEuWKnMW5/K/++B9NulGxTANBgkqhkiG9w0BAQsFADCB
hTELMAkGA1UEBhMCR0IxHTAbBgNVBAoMFE15IENsaW5pY2FsIE91dGNvbWVzMRQw
HQOr3V2CW1z6Idh6RS4=
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIID2DCCAsCgAwIBAgIQQJDUBmoYft8TvHvqvHN65DANBgkqhkiG9w0BAQsFADCB
hTELMAkGA1UEBhMCR0IxHTAbBgNVBAoMFE15IENsaW5pY2FsIE91dGNvbWVzMRQw
EgYDVQQLDAtEZXZlbG9wbWVudDEPMA0GA1UECAwGTG9uZG9uMR8wHQYDVQQDDBZt
```

The `kafka_test_client.cer` and `kafka_test_client.key` files are used by the .NET client to authenticate. You don't need the CSR again, if you're going through a new signing, it's best to start with a new key altogether.

## Now Build Your MSK Cluster

Out of the scope of this blog, but once you have the Private CA, you can [create the MSK Cluster](https://docs.aws.amazon.com/msk/latest/developerguide/msk-create-cluster.html) using Mutual TLS. The client configuration will need the list of brokers, which is called the "bootstrap servers".

## .NET Client Config

Create a new .NET core command line application to test your code and install the Confluent.Kafka nuget package.

We're going to get a list of topics from the Kafka cluster.

```csharp
// Get this from AWS MSK Console, select your cluster, click "View Client Information", public endpoint
var bootstrap = "b-1-public.nameofcluster.xxxxx.c2.kafka.eu-west-2.amazonaws.com:9194,b-2-public.nameofcluster.xxxxx.c2.kafka.eu-west-2.amazonaws.com:9194";

// We're using Admin Client here but this will work for produce and consumer
var adminClientConfig = new AdminClientConfig
{
    BootstrapServers = bootstrap,
    SecurityProtocol = SecurityProtocol.Ssl,
    SslCertificateLocation = "kafka_test_client.cer",
    SslKeyLocation = "kafka_test_client.key"
};

using var adminClient = new AdminClientBuilder(adminClientConfig).Build();

// Get the cluster's metadata and print out the topics to the console.
var metadata = adminClient.GetMetadata();
foreach (var topic in metadata.Topics)
{
    Console.WriteLine($"Topic - {topic.Topic}");
}
```

If you are struggling to get connected, add a `Debug`configuration item for more detail.

```csharp
var adminClientConfig = new AdminClientConfig
{
    Debug = "broker,protocol,metadata,topic",
```

### Note on CA Location

If you are running on Windows, you cannot use the client config `SslCaLocation` to reference a physical CA certificate. The .NET client only appears to obey the Windows Certificate Store. [View the docs for more info](https://docs.confluent.io/platform/current/clients/confluent-kafka-dotnet/_site/api/Confluent.Kafka.ClientConfig.html#Confluent_Kafka_ClientConfig_SslCaLocation).