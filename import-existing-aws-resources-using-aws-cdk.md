# Importing existing AWS resources using AWS CDK

I like infrastructure and I advocate AWS CDK for one reason: it allows us to use Typescript (or other languages) which is less of a blocker for new contributors in frontend. It is verbose but it does a lot of the weight lifting for you and is extremely flexible as well.

I've always been curious about the following situation: most companies start setting up their infrastructure through AWS Console. At some point Infrastructure as Code is introduced. How do we close the gap between them? Shall we code everything from scratch?

For a while (end of 2019) we can [import existing AWS resources into a CloudFormation Stack](https://aws.amazon.com/blogs/aws/new-import-existing-resources-into-a-cloudformation-stack/). The blog post from AWS does a good job explaining how it works. There is another [blog post](https://rusyasoft.github.io/aws,%20cdk/2021/05/23/cdk-existing-resource/) that explains how to do it with AWS CDK.

My goal here is to make it visual and with code examples. If you follow the steps from both blog posts you are good to go but I found myself confused with some names in between and decided to summarize it for myself (as I'm pretty sure I won't remember how to do it next time) and maybe help you as well.

I'm assuming that you're familiar with AWS CDK and also with CloudFormation. If not, don't hesitate to ask questions and I can try to answer them or guide you to other resources.

## AWS CDK output

AWS CDK "simply" translates some code we write into a CloudFormation template in this case, a `.json` file by the end. It is this file that is used by AWS CloudFormation to setup our infrastructure.

The output from AWS CDK can be found inside the `cdk.out/` folder. The CloudFormation template from your stack is in the file `MyStackName.template.json`.

AWS CDK creates this file whenever we run `synth` or `deploy` (which runs `synth` beforehand). The main difference is that `deploy` uploads this file to AWS CloudFormation, while `synth` "only" creates it.

## Importing existing AWS resources

I will manually create a bucket in my account and run a small example (the steps are similar for more resources but it takes more time). I called it `viniciuskneves-aws-cdk-real-bucket` and you can see it in the image below:

<img width="509" alt="Created S3 Bucket" src="https://user-images.githubusercontent.com/1734873/154742321-51ab18d5-8c7f-4cbf-a53e-18ba86b773c5.png">

In order to import an existing resource to our stack, we first need to create the stack. We can create a stack with a dummy resource as follows:

```typescript
import { RemovalPolicy, Stack, StackProps } from "aws-cdk-lib";
import { Construct } from "constructs";
import { Bucket } from "aws-cdk-lib/aws-s3";

export class AwsCdkTypescriptStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const DummyBucket = new Bucket(this, "DummyBucket", {
      removalPolicy: RemovalPolicy.DESTROY, // Easier to remove later
    });
  }
}

```

_The bucket removal policy was set to delete as this bucket will be deleted later and it is not our real bucket, which will be imported later on._

Now we can run `deploy` and, after deploying our Stack, we should be able to see it in AWS CloudFormation console:

<img width="513" alt="Created CloudFormation Stack" src="https://user-images.githubusercontent.com/1734873/154743207-44004b28-f184-4859-ba66-d5f0c03c6e16.png">

Following the instructions from AWS, we need to create a template with the resource we would like to import in this case, a S3 bucket. We can rely on AWS CDK to do the job for us. To create a S3 bucket, we need the following piece of code:

```typescript
import { RemovalPolicy, Stack, StackProps } from "aws-cdk-lib";
import { Construct } from "constructs";
import { Bucket } from "aws-cdk-lib/aws-s3";

export class AwsCdkTypescriptStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const DummyBucket = new Bucket(this, "DummyBucket", {
      removalPolicy: RemovalPolicy.DESTROY,
    });
    const RealBucket = new Bucket(this, "RealBucket", {
      removalPolicy: RemovalPolicy.RETAIN, // Recommended in order to avoid data loss
    });
  }
}

```

Now we can run `synth` to obtain our CloudFormation template. This template contains a bucket but it is not yet deployed, we will use it to manually import the resource through the AWS console.

We can go to our stack in AWS CloudFormation console, click `Stack Actions` and `Import resources into stack`:

<img width="363" alt="Import resources into AWS CloudFormation stack" src="https://user-images.githubusercontent.com/1734873/154744057-9ef509f0-9ec5-4571-8590-4a500236e363.png">

There we select the file created by `synth` in the step above (`cdk.out/MyStackName.template.json`):

<img width="744" alt="Select template file from synth" src="https://user-images.githubusercontent.com/1734873/154744208-52564a78-f548-4065-beb3-30ef71537c0a.png">

This step tells CloudFormation which new template to use.
The next step is how to link the CloudFormation template with an existing resource. AWS S3 uses the bucket name for linking, so we need to input the bucket name (from the bucket created earlier) here:

<img width="746" alt="Identify S3 bucket resource in CloudFormation" src="https://user-images.githubusercontent.com/1734873/154747626-136e900c-94af-4b92-8d3f-27c34d0ab2e1.png">

In the last step you will see the applied changes and our imported bucket will be there:

<img width="740" alt="Changes applied to CloudFormation stack after import resource" src="https://user-images.githubusercontent.com/1734873/154749075-23e4e0f8-43b2-41b8-8406-e050322d3f9e.png">

After a while (in our example it happens almost immediately as we're talking about a single resource) you will find the resource imported in your stack:

<img width="612" alt="CloudFormation events importing resource" src="https://user-images.githubusercontent.com/1734873/154749260-511fe8d2-bb56-4a37-9646-606893b5c961.png">
<img width="574" alt="List of all S3 resources from CloudFormation template" src="https://user-images.githubusercontent.com/1734873/154749541-216cb1d4-c8d4-4899-a901-6d64c3a8f94b.png">

Now we can try to run deploy again and we will get a message that no changes have been found in the stack, which means success to us:

<img width="348" alt="AWS CDK deploy no changes" src="https://user-images.githubusercontent.com/1734873/154749980-27f15879-3c0e-48f4-a15b-8551fc177ad5.png">

From now on we can make changes in the stack and it will be applied as expected, as if the resource had been created by us, in AWS CDK, since the beginning:

```typescript
const RealBucket = new Bucket(this, "RealBucket", {
  websiteIndexDocument: "index.html",
  removalPolicy: RemovalPolicy.RETAIN,
});
```

After deploying, we've:

<img width="451" alt="Static web hosting enabled for bucket" src="https://user-images.githubusercontent.com/1734873/154750620-c859236a-6bdf-451f-af16-d836c703e770.png">

We can finally remove the `DummyBucket` and as it had a destroy policy, the bucket will be removed as expected (no need to manually delete it):

```typescript
import { RemovalPolicy, Stack, StackProps } from "aws-cdk-lib";
import { Construct } from "constructs";
import { Bucket } from "aws-cdk-lib/aws-s3";

export class AwsCdkTypescriptStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const RealBucket = new Bucket(this, "RealBucket", {
      websiteIndexDocument: "index.html",
      removalPolicy: RemovalPolicy.RETAIN,
    });
  }
}
```

After deploying, we've:

<img width="604" alt="Resources list with only one bucket" src="https://user-images.githubusercontent.com/1734873/154750988-1b0c198c-f0d6-498b-8c2b-f8dc1ca18a3f.png">

## Importing resources that don't follow defaults

Our example above followed a clean approach: the real bucket had no extra configuration, it was the default, the same setup from a bucket created from AWS CDK without any configuration.

**What would happen if the bucket had, for example, static webhosting setup?**

It will allow us to import the resource as we did in the example above. One curiosity: the defaults, from AWS CDK, won't override already set configuration.

If we deploy the stack, it won't disable static webhosting, for example. The only way to disable it (overwrite it), is to enable it through CDK, deploy, and then remove it from CDK and deploy again.

Even if you run `cdk diff` you won't get this as a diff as it will compare the output of `synth` with the template that has been uploaded to CloudFormation. Stack drift won't highlight it as well which is weird.

It is a good approach as it won't magically overwrite everything that has been working. It could be tricky though if you don't know that a resource had been imported and some defaults don't apply to it.

**What would happen if the template had some setup, like static webhosting, when importing an existing resource?**

It is the opposite of the situation above. The resource we create in AWS CDK to be imported has some setup and is not the default resource.

There are two cases:
1. Imported resource and template have the same setup (static webhosting enabled): no problem. The resource is imported and changes made in CDK will apply once you deploy.
2. Imported resource and template have a different setup (static webhosting enabled in template): the difference won't be applied but drift will warn you that there is a difference.

<img width="1129" alt="Drift from imported resource" src="https://user-images.githubusercontent.com/1734873/154757270-48799f6f-a112-40bd-8e83-b6e33e50dfe7.png">

---

I hope it helps you, as it helped me, to visualize the process described in both blog posts mentioned in the introduction. I also went a bit further to explore drifts in the CloudFormation stack but it is a bit weird (check the examples above).

If you've further insights/questions, feel free to drop them in the comments!
