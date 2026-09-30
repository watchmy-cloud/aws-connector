# aws-connector

One read-only IAM role. One permission. No access keys.

This is the CloudFormation template that connects your AWS account to [watchmy.cloud](https://watchmy.cloud). [`cross_account_role.yaml`](cross_account_role.yaml) is the exact file you deploy. Our CI publishes it here from the same commit it uploads to S3, so the two never differ:

```
https://watchmycloud-cfn-templates.s3.eu-central-1.amazonaws.com/cross_account_role.yaml
```

## What we can see

We see money, split by service and by hour. That is all.

We cannot see what is inside your buckets, databases or logs. We cannot launch, stop or change anything.

## The whole policy

Here is the whole permission policy. Nothing sits behind it.

```json
{
  "Effect": "Allow",
  "Action": "ce:GetCostAndUsage",
  "Resource": "*"
}
```

`ce:GetCostAndUsage` is the Cost Explorer call that returns your costs. `Resource: "*"` is required here. Cost Explorer has no per-resource permissions.

And here is who may use the role:

```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::181474919476:role/watchmycloud-ec2-role" },
  "Action": "sts:AssumeRole",
  "Condition": { "StringEquals": { "sts:ExternalId": "<your External ID>" } }
}
```

Only one role can use it: our server's role in AWS account `181474919476`. It must also present your External ID, a value only your stack and our database know. Our server asks STS, the AWS token service, for a session. Each session lasts 15 minutes. There are no access keys, so none can leak.

## What the stack creates

1. **The role**, `WatchMyCloud-CostReader-<id>`, with the policy above.
2. **A notification topic**, `WatchMyCloud-Callback-<id>`. SNS is AWS's notification relay. It tells us once that the role is ready, so the app can finish setup. The message carries the role ARN, the External ID and your account ID.

We store your cost data in AWS Frankfurt (eu-central-1).

## What it costs you

Cost Explorer charges your account, not ours, $0.01 per request. We read hourly data about once an hour, so that comes to about $7.50 a month. Hourly data in Cost Explorer is an extra AWS option. It costs $0.01 per 1,000 usage records a month.

## Connect from the terminal

The app shows this command with your values filled in. Take the External ID from the app. Do not make one up.

```bash
aws cloudformation create-stack \
  --region eu-central-1 \
  --stack-name WatchMyCloud-Integration-<first 8 characters of External ID> \
  --template-url https://watchmycloud-cfn-templates.s3.eu-central-1.amazonaws.com/cross_account_role.yaml \
  --parameters \
    ParameterKey=ExternalId,ParameterValue=<External ID from the app> \
    ParameterKey=CallbackUrl,ParameterValue=https://app.watchmy.cloud/api/aws-callback \
  --capabilities CAPABILITY_NAMED_IAM
```

AWS asks for `CAPABILITY_NAMED_IAM` because the template creates a role with a fixed name.

## Disconnect

Two steps. The second one is final.

1. Disconnect in the app. We stop reading.
2. Delete the stack in your AWS console. The role goes with it.

```bash
aws cloudformation delete-stack --region eu-central-1 --stack-name WatchMyCloud-Integration-<id>
```

You do not need us for step two.

## Older stacks

Stacks created before October 2026 carry four more read actions: `ce:GetCostForecast`, `ce:GetDimensionValues`, `ce:GetTags` and `cur:DescribeReportDefinitions`. We never called them. To drop them, update your stack with the current template.

Questions: pawel@watchmy.cloud.
