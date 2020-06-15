## About
This directory contains the artifacts and scripts needed to deploy all the frontend and backend (DNS-excluded) resources needed to support an instance of the Live Event app. This is done using the `deploy.js` script, which will perform a series of `npm`, `aws cli`, and `aws sam` commands to create the infrastructure, compile the static web assets, and host the app in your account. Instructions on using the the script and more details on exactly what the deploy script will do can be found in the `Instructions` and `What exactly will running deploy.js do?` sections.

The app can be deployed in 2 different types of deployments:

- CloudFront deployment
  - The app will be host behind the default CloudFront domain that is generated by the distribution. This will allow you to access the app at a url that looks something like `https://abcdefghi.cloudfront.net/appname`

- Domain Name deployment
  - The app will be hosted behind the custom domain name you specify. This will allow you to access the app at a url that looks something like `https://yourdomain.com/appname`
  - **This type requires that the domain to be registered and a related ACM certificate be provisioned and in an "Issued" status prior to deployment**


## Required deployment parameters
### CloudFront deployment
- Deployment Bucket Name
- Stack Name

For a CloudFront deployment, you only need to provide a name for the deployment bucket which will be used to facilitate the deployment and a name for the deployment stack that will be created. These are not references to existing AWS resources, they will be created at the time of deployment. Note: The bucket name must follow the AWS rules for name format and uniqueness. If the S3 bucket already exists, it will be reused for the deploy.


### Domain Name deployment
- All the parameters from the **CloudFront deployment** section
- Domain Name
- ACM Certificate Arn

A domain name can be registered through Route53 or any other domain name registrar. An ACM Certificate for the domain can created [using the AWS guide found here](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html).


## Preparation
- Ensure that your account has been used before. Start up an EC2 instance briefly (a few minutes running is sufficient) if this is a brand new account. This will help prevent failures to create resources later for newly-created accounts.
- Disable S3 `Block Public Access` at the account level. Having this enabled will prevent the Web assets from being accessed post-deploy, so it must be disabled. More details [can be found here](https://aws.amazon.com/blogs/aws/amazon-s3-block-public-access-another-layer-of-protection-for-your-accounts-and-buckets/).


### Private Broadcasting
If you want to restrict access to your broadcast output streams, you will need to generate a Cloudfront key pair to sign the credentials used to restrict access. This will require root access to your account. Instructions [can be found here](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-trusted-signers.html#private-content-creating-cloudfront-key-pairs). You must then store the private key portion of the key pair into AWS Secrets Manager in PEM format. Finally, you will pass the key ID and the ARN of the secret you installed in Secrets Manager into the `deploy.js` parameters as `--signing-key-id` and `--signing-key-arn`, respectively. You should also set the `--access-duration` parameter to specify how long you want viewers to be able to access the output stream to long enough to cover the entire event, plus some buffer. It defaults to one day, but you may want to restrict that to a shorter time period.


## Instructions
- Ensure you have your required parameters on hand.

    **Note**: The DomainName and ACM Certificate Arn are especially important for domain name deployments.
    They provide the CloudFront distribution with the alias and certificate information needed to allow the app to run behind your desired domain.
    Before deploying this stack, your domain registration should be complete and your ACM Certificate for that domain should be validated.
    More information on requesting and validating an ACM public cert [can be found here](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html)
- Ensure you have a Docker instance running. [Get Docker](https://docs.docker.com/get-docker/)
- Install the [AWS SAM CLI using the instructions for development host](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html).
- Configure the CLI to use your AWS account by running `aws configure` and configuring your Access and Secret Keys
- From the `serverless` directory, run the `deploy.js` script by running `node ./deploy.js` followed by the CLI arguments; the parameters are detailed below
- *(If creating a Domain Name deployment)* From the output of the CloudFormation stack, copy the `CloudFrontDomainName` and create an ALIAS A record for your domain name
- When you are ready to broadcast, you must start the Elemental MediaLive channel, as identified by the `MediaLiveChannelId` stack output value. If you specified `--start-channel` on deploy, this is unnecessary, but if not, you must start the MediaLive channel before it will accept, encode and pass the broadcast input stream(s).

### Updating
- To update the a deployment, simply update your project workspace with the newest changes and re-run the same exact deploy deploy script you used to deploy.


### Arguments

```
Usage: deploy.sh [opts]
  -r, --region                 Target region, default 'us-east-1'
  -b, --s3-bucket              S3 bucket for deployment, required
  -s, --stack-name             CloudFormation stack name, required
  -p, --event-prefix           Event prefix, default 'LiveEvent'
  -t, --talent-name            Talent full name, default 'John Smith'
  -d, --domain-name            Site domain name, default empty
  -c, --acm-cert-arn           ACM Certificate Arn , default empty
  -e, --event-bridge           Enable EventBridge integration, default is no integration
  -n, --non-strict-access-keys Use non strict access keys, default is strict
  -C, --input-codec            Codec for broadcast media input (AVC, HEVC, MPEG2; default is AVC)
  -R, --input-resolution       Resolution of broadcast media input (1080, 720, 540; default is 1080)
  -I, --input-cidr             CIDR range from which to allow broadcast input (default is 0.0.0.0/0)
  -S, --start-channel          Start MediaLive channel on stack creation (default is to not start)
  -K, --signing-key-id         Key ID of Cloudfront signing keypair (optional)
  -A, --signing-key-arn        ARN of Secrets Manager secret holding private key part of Cloudfront signing keypair, in PEM format (optional)
  -D, --access-duration        Number of seconds viewers should have access to broadcast (default is 3600)
  -h, --help                   Show help and exit
```

### Examples
#### CloudFront deployment
```
node ./deploy.js -r us-east-1 -b live-event-deployment-bucket -s live-event-deployment-stack
```

#### Domain Name deployment
```
node ./deploy.js -r us-east-1 -b live-event-deployment-bucket -s live-event-deployment-stack -d example.com -c "arn:aws:acm:us-east-1:123456789012:certificate/{uuid} -p LiveEvent -t John Smith"
```

#### CloudFront deployment with private broadcast
```
node ./deploy.js \
  -r us-east-1 \
  -b live-event-deployment-bucket \
  -s live-event-deployment-stack \
  -K APKAAAAAAAAAAAAAAAAA \
  -A "arn:aws:secretsmanager:us-east-1:999999999999:secret:cf-pkey-APKAAAAAAAAAAAAAAAAA-333333" \
  -D 7200
```
#### Transcoder Serverless stack
 Once you run ```deploy.js``` we will also execute deployment of Transcoder Serverless Application that contains resources for building an application that broadcast media from any URL of meeting sessions to a RTMP endpiont URL. Included is a Docker image and a serverless AWS CloudFormation template that you can deploy to your AWS account. The AWS CloudFormation template orchestrates resources (including Amazon Lambda and Amazon ECS) that run the broadcast application. When deployed, you can create Lambda test events to start/stop broadcast task in ECS. [checkout for more information or for custom deployment of transcoder app](../../../transcoding/README.md)

## Outputs
Upon stack creation, CloudFormation will output the CloudFront distribution name. This domain name should be aliased by the Domain Name passed to the deployment script in order for a Domain Name deployment to function correctly:

- `CloudFrontDomainName`

It will also output the app URLs for the Attendee, Moderator, and Talent web apps under the following keys respectively:

- `AttendeeURL`
- `ModeratorURL`
- `TalentURL`

Additionally, the URL for the backend API Gateway REST API and the URIs for the 2 API Gateway WebSockets. These can be used as inputs to other templates which might extend on the existing one.

- `ApiURL`
- `MessagingWebSocketURI`
- `HandRaiseWebSocketURI`

The stack outputs the channel ID of the Elemental MediaLive instance that will be used to consume broadcast stream input and encode it. This is so you can find it easily when you need to start it just prior to beginning your live event's start:

- `MediaLiveChannelId`

The stack also outputs the two MediaLive input endpoints that you will want to send your broadcast input streams towards. They accept RTMP format:

- `MediaLivePrimaryEndpoint`
- `MediaLiveSecondaryEndpoint`

The stack also outputs the output endpoints for consuming the broadcast stream, in HLS, MPEG-DASH, CMAF and Microsoft Smooth Streaming (MSS) transport formats, respectively:

- `CloudFrontHlsEnpoint`
- `CloudFrontDashEnpoint`
- `CloudFrontCmafEnpoint`
- `CloudFrontMssEnpoint`


## What exactly will running deploy.js do?
Following the order of operations, detailed below are highlights of what running the deploy script will do. All aws-related commands that are run will be in the context of the current aws configuration. To confirm your current configuration, run `aws configure --list`.

- Before running any transformational commands, the script will parse the arguments and ensure that required tools are installed: `npm`, `aws`, and `sam`
- The `npm install` and `npm run release` commands will run to install node dependencies and compile the front end assets. This will create a `/dist` directory containing the static web pages that will be server through the CloudFront distribution.
- A new deployment bucket, which will be used to hold the deployment artifacts, will be created in the configured aws account under the name passed to the script. (**If the bucket name does not me the criteria for name format and uniqueness, this will fail**)
- Through the `sam` command the local template will be packaged, uploaded to the S3 deployment bucket, and deployed. The deployment status and events can be followed in the CloudFormation console.
- The template will deploy the following resources to your aws account (**a full list can be found under the Resources tab of the CloudFormation template**):
    - A CloudFront distribution which will be used to host the app and serve broadcast media assets
        - All the CloudFront-related resources: an S3 bucket to hold the stack web pages and a bucket for the CloudFront logs
    - 3 API Gateways (1 REST api and 2 WebSocket apis) which will serve the app's backend
    - 22 Lambda functions, which will serve various purposes, including API Gateway endpoint integrations and authorizers, data import, static asset copying, and CloudFront Lambda@Edge
    - 7 DynamoDB tables to persist app data
    - 1 nested stack to deploy resources related to frontend asset copying
    - Elemental MediaLive and MediaPackage channels for broadcast encoding and packaging
- Executes deployment of [LiveEvents Transcoder stack](../../../transcoding)

## Additional security options
- Consider configuring the TLS settings of the deployed API to enforce specific versions. Related documentation [can be found here](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-custom-domain-tls-version.html)