# swan-demo-go
Secured Web Accountable Network (SWAN) - Demo in go of SWAN, SWIFT and OWID

The SWAN demo implements the concepts explained in 
[SWAN](https://github.com/51degrees/swan)

The deployment guide later on in this readme shows you how to set up example 
with multiple SWIFT nodes, publishers and marketers but every SWAN needs at 
least the following:

* 5x SWIFT Nodes
* 1x SWAN Node
* 1x Publisher
* 1x Marketer

An example of nodes and members with example domain names:
```
             +-----------+  +-----------+  +-----------+
             |           |  |           |  |           |
Swift Nodes  | 1.51da.uk |  | 2.51da.uk |  | 3.51da.uk |  ... etc
             |           |  |           |  |           |
             +-----------+  +-----------+  +-----------+

             +--------+
             |        |
Swan Nodes   | 51d.uk |
             |        |
             +--------+

             +--------------+  +-------------+
             |              |  |             |
Publisher    | swift-pub.uk |  | swan-pub.uk | ... etc
             |              |  |             |
             +--------------+  +-------------+

             +--------------+  +---------------+  +----------------+
             |              |  |               |  |                |
Marketer     | cool-cars.uk |  | cool-bikes.uk |  | cool-creams.uk | ... etc
             |              |  |               |  |                |
             +--------------+  +---------------+  +----------------+
```
# Deployment

### Get the code

This demo uses submodules, to clone the repository and the submodules at the 
same time, run:

```
git clone --recurse-submodules https://github.com/51degrees/swan-demo-go
```

## AWS Elastic Beanstalk - without docker

### Prerequisites

* Local Go version 1.15 or greater installation sufficient to run the Go command
line.

* Familiar with the concepts associated with 
[SWIFT](https://github.com/51degrees/swift) and 
[OWID](https://github.com/51Degrees/owid).

* AWS account with Elastic Beanstalk and DynamoDB administration privileges. 

* Setup domains to use with the demo in AWS using 
[Route 53](https://console.aws.amazon.com/route53). Domains are needed for the 
following roles.

    * SWAN access domain. For example ``51d.io``.

    * Each of the SWIFT nodes that will support SWAN. At least two domains are 
    needed. For the purposes of the demo they may be sub domains. For example
    ``51da.uk`` and ``51db.uk``.

    * At least two different domains for publishers. For example ``swan-pub.uk``
    and ``swift-pub.uk``.

    * At least two different domains for marketers. For example 
    ``cool-bikes.uk`` and ``cool-creams.uk``.

* Setup SSL certificates in AWS using 
[Certificate Manager](https://console.aws.amazon.com/acm/) 
for each of the domains.

### Steps

* Add the domains to the ``appsettings.json`` fields for ``demo-pubs``, 
``demo-mars``, and ``demo-swan``. ``Demo-swan`` is a single domain for the 
access point domain. ``Demo-pubs`` and ``demo-mars`` are a list of multiple 
domains for publishers and marketers respectively.

  ```
  {
      "demo-pubs": [
          "swan-pub.uk",
          "swift-pub.uk" 
      ],
      "demo-mars": [
          "cool-bikes.uk",
          "cool-creams.uk"
      ],
      "demo-swan": "51d.io"
  }
  ```

* You may need to support multiple SSL certificates if the demo deployment 
should respond to five or more domains. 
See AWS
[documentation](https://aws.amazon.com/premiumsupport/knowledge-center/elastic-beanstalk-ssl-load-balancer/).
Locate the ARN for each of the certificates that should be used with the demo.

* Add SSL certificate ARNs to a copy of ``.ebextensions/.config``. For example 
if you have the SSL ARNs A, B and C your .config file would contain the
following entries.

  ```
  option_settings:
    aws:elbv2:listener:443:
      Protocol: HTTPS
      SSLCertificateArns: "A"
  Resources:
    SSLCert2:
      Type: "AWS::ElasticLoadBalancingV2::ListenerCertificate"
      Properties:
        ListenerArn:
          Ref : "AWSEBV2LoadBalancerListener443"
        Certificates:
          - CertificateArn: "B"
    SSLCert3:
      Type: "AWS::ElasticLoadBalancingV2::ListenerCertificate"
      Properties:
        ListenerArn:
          Ref : "AWSEBV2LoadBalancerListener443"
        Certificates:
          - CertificateArn: "C"
  ```

* Run the build.bat (Windows) or build.sh (Linux) to create the 
``aws-eb-swan-demo.zip`` bundle. The bundle should contain the ``application`` 
executable compiled for Linux 64 bit, ``Procfile`` to tell Elastic Beanstalk how 
to start the application, and ``.ebextensions/.config`` to configure additional
HTTPS listeners for additional domains and SSL certificates.

* Create a new Elastic Beanstalk Application and Environment using the 
[AWS document](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/create_deploy_go.html).

* Give the Elastic Beanstalk Environment permissions to create and read DynamoDB
tables:
 
  * For the Elastic Beanstalk Environment role (default: `aws-elasticbeanstal-ec2-role`), attach the 
  [AmazonDynamoDBFullAccess](https://console.aws.amazon.com/iam/home#/policies/arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess$serviceLevelSummary) permission policy.

* Upload the ``aws-eb-swan-demo.zip`` bundle to the environment.

* Add an A record for each of the domains to direct traffic to the Elastic 
Beanstalk environment following the 
[AWS documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-beanstalk-environment.html).

* The SWAN access domain will be used to sign all the outgoing Open Web IDs and
also to capture people's preferences. Register this domain with the following
URL and entering any of the details requested. This will create a record in the 
``owidcreators`` table for the domain which will contain randomly generated 
public and private signing keys.

  ```
  https://swan-access-domain/owid/register
  ```

* For each of the storage nodes that will be used for the SWIFT component of the
demo register these using the following URL. Enter the network as "swan" (no 
quotes) to match the value provided in the ``appsettings.json`` in the 
``swanNetwork`` field. Leave the others as default.

  ```
  https://swift-node-domain/swift/register
  ```

* At least one SWIFT access node is required. Repeat the previous process but 
select the "Access Node" option rather than the default "Storage Node". The 
records from these steps will be visible in the ``swiftnodes`` and 
``swiftsecrets`` tables.

* Verify the demo is working by navigating to the publisher domain. The first 
request from a web browser will result in the progress circle as SWIFT nodes
are navigated between before the preference capture page from the SWAN domain
is displayed.

## Azure App Service

TODO - prerequisites and steps to set up the demo in an Azure environment

### Azure CosmosDB / Table Storage

If you are using Azure Storage Tables, this demo requires your storage account 
name and key to be securely stored in environment variables local to the machine 
running the demo:

If configuring an Azure App Service then see: 
[Configure an App Service app in the Azure portal](https://docs.microsoft.com/en-us/azure/app-service/configure-common#configure-app-settings)

## Local Installation

### Prerequisites 

* Local Go version 1.15 or greater installation sufficient to run the Go command
line.

* Specified a storage option, the demo supports:
  * Azure Storage Tables
  * AWS DynamoDB


* For AWS, specify region and credentials in either environment variables or in 
`~/.aws/credentials` and `~/.aws/config` files. See 
[Configuring AWS SDK for Go](https://docs.aws.amazon.com/sdk-for-go/v1/developer-guide/configuring-sdk.html) to set up your environment. Make sure to set the 
`AWS_REGION`, `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as a minimum.

* OR, for Azure Storage:
  
  Linux:
  ```bash
  export AZURE_STORAGE_ACCOUNT="<youraccountname>"
  export AZURE_STORAGE_ACCESS_KEY="<youraccountkey>"
  ```
  Windows:
  ```bat
  setx AZURE_STORAGE_ACCOUNT "<youraccountname>"
  setx AZURE_STORAGE_ACCESS_KEY "<youraccountkey>"
  ```

### Steps

* Having cloned the repository, configure the `appsettings.dev.json` file.

  ```
  cp appsettings.dev.json.rename appsettings.dev.json
  ```

* Configure your hosts file to point URLs to localhost, see 
(Environments)[#environments] section in this readme for platform specifics. 
The following host resolutions are used in the sample configuration:

  ```
  # publishers
  127.0.0.1	new-pork-limes.uk
  127.0.0.1	current-bun.uk
  127.0.0.1	swan-pub.uk
  127.0.0.1	swallow-pub.uk
  127.0.0.1	swift-pub.uk
  # marketers
  127.0.0.1	cool-bikes.uk
  127.0.0.1	cool-cars.uk
  127.0.0.1	cool-creams.uk
  # swan node
  127.0.0.1	51d.io
  # swift nodes
  127.0.0.1	1.51d.uk
  127.0.0.1	2.51d.uk
  127.0.0.1	3.51d.uk
  127.0.0.1	4.51d.uk
  127.0.0.1	5.51d.uk
  ```

* Run either the `./build.sh` file if you are on Linux or run the `./build.bat` file 
if you are on Windows.

* Run the server:

  ```
  ./src/server appsettings.dev.json
  ```

* The SWAN access domain will be used to sign all the outgoing Open Web IDs and
also to capture people's preferences. Register this domain with the following
URL and entering any of the details requested. This will create a record in the 
``owidcreators`` table for the domain which will contain randomly generated 
public and private signing keys.

  ```
  http://51d.io:5000/owid/register
  ```

* For each of the storage nodes that will be used for the SWIFT component of the
demo register these using the following URL. Enter the network as "swan" (no 
quotes) to match the value provided in the ``appsettings.json`` in the 
``swanNetwork`` field. Leave the others as default.

  ```
  http://1.51d.uk:5000/swift/register
  ```

* At least one SWIFT access node is required. Repeat the previous process but 
select the "Access Node" option rather than the default "Storage Node". The 
records from these steps will be visible in the ``swiftnodes`` and 
``swiftsecrets`` tables.

  ```
  http://5.51d.uk:5000/swift/register
  ```
* Now browse to one of the publisher URLs, you will be prompted to set your 
preferences:

  ```
  http://swan-pub.uk:5000
  ```

# Files

`Procfile` : needed by AWS Elastic Beanstalk to indicate the application 
executable for web services.

`build.bat` : builds AWS or Azure packages on Windows ready for manual deployment.

`build.sh` : builds AWS or Azure packages on Linux ready for manual deployment.

`appsettings.json.rename` : template application settings ready for Azure and AWS 
storage or DynameDB keys.

`appsettings.dev.json.rename` : development app settings template.

`.ebextensions/.config.rename` : AWS Elastic Beanstalk .config template ready for
additional SSL certificates.

Note: `.gitignore` will ignore `appsettings.json` and `appsettings.dev.json` to limit
the risk of commits containing access keys.

# Environments

This demo makes extensive use of multiple domains. For development purposes 
setup local domains to resolve to 127.0.0.1.

## Windows 

```
notepad C:\Windows\System32\drivers\etc\hosts
```

## Linux

```
vi /etc/hosts
```

## Visual Studio Code

Use the Command Palette (Ctrl + Shift + P) to running the `Go: Install/Update Tools` to install `gopkgs`, `dlv` and `gopls`.
