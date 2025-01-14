# Runtime Context<a name="context"></a>

The AWS CDK uses *context* to retrieve information such as the Availability Zones in your account or Amazon Machine Image \(AMI\) IDs used to start your instances\. Context entries are key\-value pairs\.

To avoid unexpected changes to your deployments when, for example, a new Amazon Linux AMI is released, thus changing your Auto Scaling group, the AWS CDK stores context values in the `cdk.context.json` file within your project\. This ensures that the AWS CDK uses the same context values the next time it synthesizes your app\. Don't forget to put this file under version control\.

## Construct Context<a name="context_construct"></a>

Context values are made available to your AWS CDK app in five different ways:
+ Automatically from the current AWS account\.
+ Through the \-\-context option to the cdk command\.
+ In the `context` key of the project's `cdk.json` file\.
+ In the `context` key of a `~/cdk.json` file\.
+ In code using the `construct.node.setContext` method\.

Context values are scoped to the construct that created them; they are visible to child constructs, but not to siblings\. Context values set by the AWS CDK Toolkit \(the cdk command\), either automatically or from the \-\-context option, are implicitly set on the `App` construct, and so are visible to every construct in the app\.

You can get a context value using the `construct.node.tryGetContext` method\. If the requested entry is not found on the current construct or any of its parents, the result is `undefined` \(or your language's equivalent, such as `None` in Python\)\.

## Context Methods<a name="context_methods"></a>

The AWS CDK supports several context methods that enable AWS CDK apps to get contextual information\. For example, you can get a list of Availability Zones that are available in a given AWS account and AWS Region, using the [stack\.availabilityZones](https://docs.aws.amazon.com/cdk/api/latest/docs/@aws-cdk_core.Stack.html#availabilityzones) method\.

The following are the context methods:

[HostedZone\.fromLookup](https://docs.aws.amazon.com/cdk/api/latest/docs/@aws-cdk_aws-route53.HostedZone.html#static-from-wbr-lookupscope-id-query)  
Gets the hosted zones in your account\.

[stack\.availabilityZones](https://docs.aws.amazon.com/cdk/api/latest/docs/@aws-cdk_core.Stack.html#availabilityzones)  
Gets the supported Availability Zones\.

[StringParameter\.valueFromLookup](https://docs.aws.amazon.com/cdk/api/latest/docs/@aws-cdk_aws-ssm.StringParameter.html#static-value-wbr-from-wbr-lookupscope-parametername)  
Gets a value from the current Region's AWS Systems Manager Parameter Store\.

[Vpc\.fromLookup](https://docs.aws.amazon.com/cdk/api/latest/docs/@aws-cdk_aws-ec2.Vpc.html#static-from-wbr-lookupscope-id-options)  
Gets the existing VPCs in your accounts\.

If a given context information isn't available, the AWS CDK app notifies the AWS CDK CLI that the context information is missing\. The CLI then queries the current AWS account for the information, stores the resulting context information in the `cdk.context.json` file, and executes the AWS CDK app again with the context values\.

Don't forget to add the `cdk.context.json` file to your source control repository to ensure that subsequent synth commands will return the same result, and that your AWS account won't be needed when synthesizing from your build system\.

## Viewing and Managing Context<a name="context_viewing"></a>

Use the cdk context command to view and manage the information in your `cdk.context.json` file\. To see this information, use the cdk context command without any options\. The output should be something like the following\.

```
Context found in cdk.json:
      
┌───┬─────────────────────────────────────────────────────────────┬─────────────────────────────────────────────────────────┐
│ # │ Key                                                         │ Value                                                   │
├───┼─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
│ 1 │ availability-zones:account=123456789012:region=eu-central-1 │ [ "eu-central-1a", "eu-central-1b", "eu-central-1c" ]   │
├───┼─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
│ 2 │ availability-zones:account=123456789012:region=eu-west-1    │ [ "eu-west-1a", "eu-west-1b", "eu-west-1c" ]            │
└───┴─────────────────────────────────────────────────────────────┴─────────────────────────────────────────────────────────┘

Run cdk context --reset KEY_OR_NUMBER to remove a context key. It will be refreshed on the next CDK synthesis run.
```

To remove a context value, run cdk context \-\-reset, specifying the value's corresponding key number\. The following example removes the value that corresponds to the key value of `2` in the preceding example, which is the list of availability zones in the Ireland region\.

```
$ cdk context --reset 2
```

```
Context value
availability-zones:account=123456789012:region=eu-west-1
reset. It will be refreshed on the next SDK synthesis run.
```

Therefore, if you want to update to the latest version of the Amazon Linux AMI, you can use the preceding example to do a controlled update of the context value and reset it, and then synthesize and deploy your app again\.

```
$ cdk synth
```

```
...
```

To clear all of the stored context values for your app, run cdk context \-\-clear, as follows\.

```
$ cdk context --clear
```

## Example<a name="context_example"></a>

Below is an example of importing an existing Amazon VPC using AWS CDK context\.

```
import cdk = require('@aws-cdk/core');
import ec2 = require('@aws-cdk/aws-ec2');
export class ExistsvpcStack extends cdk.Stack {

  constructor(scope: cdk.Construct, id: string, props?: cdk.StackProps) {
  
    super(scope, id, props);
    
    const vpcid = this.node.tryGetContext('vpcid');
    const vpc = ec2.Vpc.fromLookup(this, 'VPC', {
      vpcId: vpcid,
    });
    
    const pubsubnets = vpc.selectSubnets({subnetType: ec2.SubnetType.PUBLIC});
    
    new cdk.CfnOutput(this, 'publicsubnets', {
      value: pubsubnets.subnetIds.toString(),
    });
  }
}
```

You can use cdk diff to see the effects of passing in a context value on the command line:

```
$ cdk diff -c vpcid=vpc-0cb9c31031d0d3e22
```

```
Stack ExistsvpcStack
Outputs
[+] Output publicsubnets publicsubnets: {"Value":"subnet-06e0ea7dd302d3e8f,subnet-01fc0acfb58f3128f"}
```

The resulting context values can be viewed as shown here\.

```
$ cdk context -j
```

```
{
  "vpc-provider:account=123456789012:filter.vpc-id=vpc-0cb9c31031d0d3e22:region=us-east-1": {
    "vpcId": "vpc-0cb9c31031d0d3e22",
    "availabilityZones": [
      "us-east-1a",
      "us-east-1b"
    ],
    "privateSubnetIds": [
      "subnet-03ecfc033225be285",
      "subnet-0cded5da53180ebfa"
    ],
    "privateSubnetNames": [
      "Private"
    ],
    "privateSubnetRouteTableIds": [
      "rtb-0e955393ced0ada04",
      "rtb-05602e7b9f310e5b0"
    ],
    "publicSubnetIds": [
      "subnet-06e0ea7dd302d3e8f",
      "subnet-01fc0acfb58f3128f"
    ],
    "publicSubnetNames": [
      "Public"
    ],
    "publicSubnetRouteTableIds": [
      "rtb-00d1fdfd823c82289",
      "rtb-04bb1969b42969bcb"
    ]
  }
}
```