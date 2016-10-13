# deploy-pcf-aws
===
### Supported PCF version:
* 1.6
* 1.7
* 1.8

===
## Overview

This Concourse pipeline is used to quickly spin up a PCF environment using the [p-runtime gem](https://github.com/pivotal-cf/p-runtime). The purpose of this pipeline is to quickly provider test/development PCF environments on AWS.

* The pipeline will deploy a single-az PCF environment on AWS with a self-signed cert.

* The pipeline of interest is named: **deploy-pcf-aws.yml**

* This pipeline is separated into 3 pipeline groups:

  * bootstrap-aws
  * deploy-pcf
  * destroy-pcf-aws


## deploy-pcf-aws.yml

#### Prerequisites:

* Github key that has access to pivotal-cf/p-runtime
* Pivnet API token
* Github repo for storing credentials and deployment manifest
* an ssh keypair for AWS to use when deploying instances must be added to this repo.
 **NOTE**: make sure your key-pair name (format: id_rsa_\<ENV_NAME\>)

Create a new keypair with the command below; Do not use the AWS Console or AWS CLI to create the keypair.
```
$ ssh-keygen -t rsa -f $environment_yml_git_repo/$environment_yml_folder/$aws_private_key_file_name

#Example:
$ ssh-keygen -t rsa -f /Users/pivotal/workspace/deployments-toolsmiths/aws/environments/<ENV_NAME>/id_rsa_<ENV_NAME>

```

#### Caveats:

You will need to ensure the wildcard cname and A record for the Ops Manager and PCF ELB does not exist before running the script.

#### Usage:

Edit pcfdeploycreds.yml and configure all the following values listed in the file:

```

# == pivnet ==
# download pivnet token from network.pivotal.io/users/dashboard/edit-profile
pivnet-token:

#Select Ops Manager Release version from network.pivotal.io/products/ops-manager; e.g. 1.8.4
ops-manager-version:

#Select PCF Elastic Runtime compatible with Ops Manager from network.pivotal.io/products/elastic-runtime; e.g. 1.8.5
elastic-runtime-version:

# == aws config ==
aws-access-key-id:
aws-secret-access-key:

# select any name for your environment
aws-environment-name:

# should begin with id_rsa_<ENVIRONMENT_NAME>
aws-key-pair-name:

# should begin with ssh-rsa ...
aws-public-key:

# this private key is expected to be inside your environment folder inside the git repo
aws-private-key-file-name:

# == environment config ==
#create a new private github repo in your account
private-github-repo: git@github.com:<YOUR-ORG>/<YOUR-REPO>

# create this path with your environment name within your git repo
environment-yml-folder: aws/environments/<ENVIRONMENT-NAME>

# pick any subdomin name for your system apps followed by your primary domain name; e.g. system.pcfplatform.com
aws-system-domain:

# pick secure username and password for your Relational Database Service (RDS)
aws-rds-username:
aws-rds-password:

# This varies per region: http://docs.aws.amazon.com/general/latest/gr/rande.html#s3_region
aws-s3-endpoint: 'https://s3.amazonaws.com'
aws-region:
aws-route-53-hosted-zone-id:

# == ops manager config ==
ops-manager-fqdn:
ops-manager-password:

# == concourse ==
github-user:
github-email:

# make sure to indent your github key below; note, only github keys without passphrase are compatible;
# reference: https://help.github.com/articles/checking-for-existing-ssh-keys/
github-key: |
  -----BEGIN RSA PRIVATE KEY-----
  xxxxxxxxxxxx......
  -----END RSA PRIVATE KEY-----

```

Optional: Edit deploy-pcf-aws.yml to setup up email notifications.
```
# OPTIONAL: This is for setting up E-mail notifications. If you do not want to set this up, delete the values "<...>"
smtp_from: &smtp_from <SMTP-FROM-ADDRESS>
smtp_address: &smtp_address <SMTP-ADDRESS>
smtp_port: &smtp_port <SMTP-PORT>
smtp_user: &smtp_user <SMTP-USER>
smtp_password: &smtp_password <SMTP-PASSWORD>
```

Concourse command to setup pipeline
```
fly -t <target> set-pipeline --pipeline pcf --config deploy-pcf-aws.yml --load-vars-from pcfdeploycreds.yml
fly -t <target> unpause-pipeline --pipeline pcf
```

### bootstrap-aws

The pipeline group will do the following:

* Generate a self-signed certificate and upload to AWS IAM using the awscli
* Upload your private key to your AWS account
* Download the corresponding cloudformation.json from Pivotal Network
* Run the cloudformation template using your defined aws credentials, regions, azs

### deploy-pcf

This pipeline group will do the following:

* Verify or generate your <ENVIRONMENT>.yml inside your environment repo
* Create your ops manager vm inside the region you specified
* Configure your Ops Manager director
* Download and upload the specified ERT tile from Pivotal Network and upload it to Ops Manager
* Configure the Elastic Runtime tile to use the self-signed cert, s3 buckets, rds instance and etc. all generated from the first bootstrap-aws pipeline group
* The pipeline will then trigger your install and you will have a working PCF deployment!

### destroy-pcf-aws

This pipeline group has two manual triggers:

* **destroy-pcf-aws:** This step will purge your AWS account of everything in your environment. It will delete your PCF deployment as well as tear down all the subnets, RDS instances, VPCs and etc generated by your cloudformation script

* **delete-all-bootstrap-resources:** This step is used if you made a mistake in your bootstrapping process. This step will delete the key uploaded to your AWS account as well as delete the cloudformation stack.
