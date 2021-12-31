# irtnog-aws-static-website

This AWS CloudFormation stack hosts a static website using Amazon S3 and Amazon CloudFront.

## Theory of Operation

TODO

## Prerequisites

When deploying the stack with a web browser, whether interactively through [the CloudFormation console](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-using-console.html) or from the command line via [AWS CloudShell](https://aws.amazon.com/cloudshell/), you must log into the AWS Console with an IAM user or role that has [the developer power user job function](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_job-functions.html#jf_developer-power-user).

Alternatively, to deploy the stack from the command line on a PC or Mac:

- [Install the AWS CLI version 2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html).

- [Configure AWS API access](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html).
  Again, your IAM user account from which you derive your credentials must have the developer power user job function.

- The example shell commands require a contemporary version of [GNU Bash](https://www.gnu.org/software/bash/).
  On Windows, use the Git Bash app included with [Git](https://git-scm.com/download).
  On macOS, use the Terminal app.

- [Install jq](https://github.com/stedolan/jq/wiki/Installation).
  On Windows, download the 64-bit version of [jq 1.6](https://github.com/stedolan/jq/releases/download/jq-1.6/jq-win64.exe), rename `jq-win64.exe` to `jq.exe`, and copy it to a directory in the executable search path.
  On macOS, use [MacPorts](https://www.macports.org/) (preferred) or [HomeBrew](https://brew.sh/) to install jq.

- Optionally, set (and export) the environment variable **AWS_PROFILE** to the AWS CLI [named profile](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html) to use, e.g., `corpdev`.
  
- Optionally, set (and export) the environment variable **AWS_REGION** to the AWS data center cluster to which the CloudFormation stack will be deployed, e.g., `us-east-1`.

In the example commands shown below, these environment variables will be used as follows:

- **TEMPLATE_BUCKET** is the S3 bucket hosting the CloudFormation templates, e.g., `devops-library-${AWS_REGION}`, `cf-templates-1234567890abc-${AWS_REGION}`.
  While you may deploy the stack from the release bucket, `irtnog-aws-static-website`, we recommend deploying from an S3 bucket under your control.
  Refer to [Release Engineering](#release-engineering) for more information.

- **STACK_NAME** is a globally unique ID for the CloudFormation stack, e.g., `example-web-site`.

- **HOSTNAME** is the website's fully qualified domain name (FQDN), e.g., `www.example.com`.

## Certificate

[Amazon Certificate Manager (ACM)](https://docs.aws.amazon.com/acm/) must have a public certificate for your website [in the us-east-1 region](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html).
You may [import an existing certificate](https://docs.aws.amazon.com/acm/latest/userguide/import-certificate.html) or [request a new one](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html).
The first FQDN on the certificate must match the website's FQDN.
For example, commands similar to the following will request a new certificate from ACM and publish a domain control validation (DCV) record in the matching Amazon Route 53 hosted zone:

```bash
eval $(
    aws acm request-certificate \
        --domain-name "${HOSTNAME}" \
        --validation-method DNS \
        --output json \
    | jq -r '@sh "CERTARN=\(.CertificateArn)"'
)
CERTDCVRRNAME=""
until [ -n "${CERTDCVRRNAME}" ]
do
    sleep 5
    eval $(
        aws acm describe-certificate \
            --certificate-arn "${CERTARN}" \
            --output json \
        | jq -r '.Certificate.DomainValidationOptions[0].ResourceRecord
        | @sh "CERTDCVRRNAME=\(.Name); CERTDCVRRTYPE=\(.Type); CERTDCVRRVALUE=\(.Value)"'
    )
done
eval $(
    aws route53 list-hosted-zones-by-name --output json \
    | jq -r '.HostedZones[]|.Name as $name
    | select("'${CERTDCVRRNAME}'"|test($name))
    | @sh "ZONEID=\(.Id|split("/")[2])"'
)
aws route53 change-resource-record-sets \
    --hosted-zone-id "${ZONEID}" \
    --change-batch file:///dev/fd/0 <<EOF
{
    "Changes": [
        {
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "${CERTDCVRRNAME}",
                "Type": "${CERTDCVRRTYPE}",
                "TTL": 300,
                "ResourceRecords": [
                    {
                        "Value": "${CERTDCVRRVALUE}"
                    }
                ]
            }
        }
    ]
}
EOF
```

## Deployment

Launch the CloudFormation stack in your AWS account using one of the following links:

| AWS Region Code    | AWS Region Name                        | Launch                                                                                                                                                                                                                                                                                                                             |
|--------------------|----------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| us-east-1          | US East (N. Virginia)                  | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)          |
| us-east-2          | US East (Ohio)                         | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)          |
| us-west-1          | US West (N. California)                | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)          |
| us-west-2          | US West (Oregon)                       | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)          |
| ca-central-1       | Canada (Montreal)                      | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ca-central-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)       |
| eu-north-1         | EU (Stockholm)                         | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-north-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)         |
| eu-west-3          | EU (Paris)                             | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-3#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)          |
| eu-west-2          | EU (London)                            | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-2#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)          |
| eu-west-1          | EU (Ireland)                           | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)          |
| eu-central-1       | EU (Frankfurt)                         | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-central-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)       |
| eu-south-1         | EU (Milan)                             | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-south-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)         |
| ap-south-1         | Asia Pacific (Mumbai)                  | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-south-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)         |
| ap-northeast-1     | Asia Pacific (Tokyo)                   | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)     |
| ap-northeast-2     | Asia Pacific (Seoul)                   | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-2#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)     |
| ap-northeast-3     | Asia Pacific (Osaka-Local)             | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-3#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)     |
| ap-southeast-1     | Asia Pacific (Singapore)               | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-southeast-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)     |
| ap-southeast-2     | Asia Pacific (Sydney)                  | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-southeast-2#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)     |
| ap-southeast-3     | Asia Pacific (Jakarta)                 | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-southeast-3#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)     |
| ap-east-1          | Asia Pacific (Hong Kong) SAR           | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-east-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)          |
| sa-east-1          | South America (São Paulo)              | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=sa-east-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)          |
| cn-north-1         | China (Beijing)                        | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=cn-north-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)         |
| cn-northwest-1     | China (Ningxia)                        | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=cn-northwest-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)     |
| us-gov-east-1      | GovCloud (US-East)                     | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-gov-east-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)      |
| us-gov-west-1      | GovCloud (US-West)                     | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-gov-west-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)      |
| us-gov-secret-1    | AWS Secret Region (US-Secret)          | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-gov-secret-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)    |
| us-gov-topsecret-1 | AWS Top Secret-East Region (US-Secret) | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-gov-topsecret-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml) |
| us-gov-topsecret-2 | AWS Top Secret-West Region (US-Secret) | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-gov-topsecret-2#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml) |
| me-south-1         | Middle East (Bahrain)                  | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=me-south-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)         |
| af-south-1         | Africa (Cape Town)                     | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=af-south-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)         |
| eu-east-1          | EU (Spain)                             | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-east-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)          |
| eu-central-2       | EU (Zurich)                            | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-central-2#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)       |
| ap-south-2         | Asia Pacific (Hyderabad)               | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-south-2#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)         |
| ap-southeast-3     | Asia Pacific (Melbourne)               | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-southeast-3#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)     |
| me-south-2         | Middle East (United Arab Emirates)     | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=me-south-2#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)         |
| eu-north-1         | EU (Estonia)                           | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-north-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)         |
| eu-south-1         | EU (Cyprus)                            | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-south-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)         |
| me-west-1          | Middle East (Tel Aviv)                 | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=me-west-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)          |
| ru-central-1       | Russia (TBD)                           | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ru-central-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)       |
| ap-southeast-4     | Asia Pacific (Auckland)                | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-southeast-4#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)     |
| ca-west-1          | Canada (Calgary)                       | [![cloudformation-launch-stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ca-west-1#/stacks/new?stackName=StaticWebsite&templateURL=https://s3.amazonaws.com/irtnog-aws-static-website/irtnog-aws-static-website.yaml)          |

Alternatively, launch the stack using commands similar to the following:

```bash
eval $(
    aws acm list-certificates \
        --region us-east-1 \
        --output json \
    | jq -r '.CertificateSummaryList[]|select(.DomainName == "'${HOSTNAME}'")
    | @sh "CERTARN=\(.CertificateArn)"'
)
aws cloudformation create-stack \
    --stack-name ${STACK_NAME} \
    --template-url https://s3.amazonaws.com/${TEMPLATE_BUCKET}/irtnog-aws-static-website.yaml \
    --parameters \
        ParameterKey=TemplateBucket,ParameterValue=${TEMPLATE_BUCKET} \
        ParameterKey=FullyQualifiedDomainName,ParameterValue=${HOSTNAME} \
        ParameterKey=CertificateArn,ParameterValue=${CERTARN} \
;
```

To publish your website, create a [CNAME record](https://en.wikipedia.org/wiki/CNAME_record) for the website's FQDN that points to the stack's `DistributionFqdn` output value.  Note that certain restrictions apply to CNAME records.

However, if your domain is hosted in Route 53, create an [alias record](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html) instead.  For example, commands similar to the following will search Route 53 for a suitable hosted zone and create an alias record in it, overwriting any existing alias record:

```bash
eval $(
    aws route53 list-hosted-zones-by-name --output json \
    | jq -r '.HostedZones[]|.Name as $name|select("'${HOSTNAME}'."|test($name))
    | @sh "ZONEID=\(.Id|split("/")[2])"'
)
eval $(
    aws cloudformation describe-stacks \
        --stack-name ${STACK_NAME} \
        --output json \
    | jq -r '.Stacks[0].Outputs[]|select(.OutputKey == "DistributionFqdn")
    | @sh "DISTNAME=\(.OutputValue)"'
)
aws route53 change-resource-record-sets \
    --hosted-zone-id $ZONEID \
    --change-batch file:///dev/fd/0 <<EOF
{
    "Changes": [
        {
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "${HOSTNAME}.",
                "Type": "A",
                "AliasTarget": {
                    "DNSName": "${DISTNAME}",
                    "HostedZoneId": "Z2FDTNDATAQYW2",
                    "EvaluateTargetHealth": false
                }
            }
        }
    ]
}
EOF
```

## Maintenance

Upload the website content to the S3 bucket created by the stack.

## Removal

TODO

## Release Engineering

The [releng.yaml](releng.yaml) CloudFormation template creates resources used to deploy the stack.
Those resources must be created in the us-east-1 region due to current limitations in the construction of `TemplateURL` values.
The production release is hosted in the us-east-1 region in a bucket named `irtnog-aws-static-website`, with the relevant resources created using commands similar to the following:

```bash
aws cloudformation deploy \
    --template-file releng.yaml \
    --stack-name ${RELENG_STACK_NAME} \
    --region us-east-1
```

To publish a release to the S3 bucket created by the above:

```bash
eval $(
    aws cloudformation describe-stacks \
        --stack-name ${RELENG_STACK_NAME} \
        --region us-east-1 \
        --output json \
    | jq -r '.Stacks[0].Outputs[]|select(.OutputKey == "BucketUrl")
    | @sh "RELENG_BUCKET_URL=\(.OutputValue)"'
)
(cd templates; aws s3 sync . ${RELENG_BUCKET_URL}templates/ --region us-east-1)
aws s3 cp irtnog-aws-static-website.yaml ${RELENG_BUCKET_URL} --region us-east-1
```

## Contributing

TODO
