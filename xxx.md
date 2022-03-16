# XXX

It all started with the following question: How do we [safely store AWS IAM User Keys (Access and Secret) created by IaC](https://serverfault.com/questions/1096129/safely-store-aws-iam-user-keys-access-and-secret-created-by-iac)?

Imagine the following scenario: you've a Bucket that will host your Frontend assets. Your Frontend lives in another repository and you use, in my example, GitHub Actions to deploy (move) those files to the Bucket. You want to give permission to your GitHub Actions to perform that and only that.

You can, of course, add your credentials to the repository but you will give it too many permissions.

You can create an User, in your infrastructure, that has only the permission needed by GitHub Actions. This sounds a way better already. The issue is: where do we store this User's keys?
1. We can have the keys as Output of our Stack. It is not safe, everyone that has access to the logs from your GitHub Action can see it (imagine an open source repository).
2. We can store these keys in SSM. It works but everyone that has access to SSM has access to it (not a big deal in my case).
3. We can store these keys in Secrets Manager. It works but everyone that has access to Secrets Manager has access to it (not a big deal in my case).
4. We can generate the keys manually. It works and only the person that generated the keys will know it (hopefully store them in GitHub Secrets and forget about it).

All the steps above work at some extent. The only one that doesn't require any manual effort is option 1. which is the least secure one.

The best approach, from my understanding, is another one. I would like to go through it step by step and how to setup it using AWS CDK.

## OpenID Connect (OIDC)

Since October of 2021 GitHub Actions allow it: https://github.blog/changelog/2021-10-27-github-actions-secure-cloud-deployments-with-openid-connect/

OpenID Connect allows you to connect an external provider (GitHub Actions) with you AWS account. You can read more about it here: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html

The external provider will assume a defined role which has the permissions (policies) we need. You can read more about IAM roles here: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html

Breaking it into steps:
1. We need a policy that allows us to move files into the Bucket (nothing more than that);
2. We need a role that will be assumed by GitHub Actions and this role needs to use the policy from step 1.
3. We need an OIDC identity provider to connect our GitHub Actions.

GitHub and AWS have an extensive documentation about it:
- [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [Creating OpenID Connect (OIDC) identity providers](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
- [aws-actions/configure-aws-credentials assuming role](https://github.com/aws-actions/configure-aws-credentials#assuming-a-role)
- [AWS CDK IAM Module - OpenID Connect Providers](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam-readme.html#openid-connect-providers)

How does it look in the code?

1. Let's imagine we've a Bucket
```typescript
const bucket = new Bucket(this, "Bucket");
```

2. Let's create the OIDC following the documentation
```typescript
const provider = new OpenIdConnectProvider(this, "GitHubProvider", {
  url: "https://token.actions.githubusercontent.com",
  clientIds: ["sts.amazonaws.com"], // Referred as "Audience" in the documentation
});
```

3. Let's create our role (AWS CDK makes it easy to link the role with the identity provider)
```typescript
const role = new Role(this, "Role", {
  assumedBy: new OpenIdConnectPrincipal(provider, {
    StringEquals: {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com", // "Audience" from above
    },
    StringLike: {
      "token.actions.githubusercontent.com:sub":
        "repo:<ORGANIZATION>/<REPOSITORY>:*", // Organization and repository that are allowed to assume this role
    },
  }),
});
```

_The example above is a bit different from the one documented by GitHub. Here I'm making it more flexible (all branches, for example, can use it)._

The `assumedBy` property can be customized:
- You can play with the conditions: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition.html
- You can use other properties from the payload: https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token

4. Let's grant the necessary permission to our role
```typescript
bucket.grantPut(role);
```

Once deployed everything is ready on AWS side, now we go to GitHub side.

5. Let's update our GitHub Action to assume the role
```yaml
# Before
...
- name: Configure AWS Credentials 🔧
  uses: aws-actions/configure-aws-credentials@v1.6.1 # Version at the time of this writing
  with: # We are loading keys stored in secrets
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: eu-central-1
- run: npm run deploy # Just an example

# After
...
permissions:
  id-token: write
  contents: read # This is required for actions/checkout (GitHub documentation mentions that)
...
- name: Configure AWS Credentials 🔧
  uses: aws-actions/configure-aws-credentials@v1.6.1 # Version at the time of this writing
  with: # We are loading keys stored in secrets
    role-to-assume: arn:aws:iam::<ACCOUNT>:role/<ROLE-NAME> # You can specify the role name when creating it or AWS will automatically generate one for you
    aws-region: eu-central-1
- run: npm run deploy # Just an example
```

We can now safely remove the secrets from GitHub and also don't need any "special" user to deploy through GitHub Actions.

---

This helps me a lot to "wrap" the knowledge and I believe it can help other people too.

References:
- https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html
- https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam-readme.html
- https://github.com/aws-actions/configure-aws-credentials
- https://aws.plainenglish.io/no-more-aws-access-keys-in-github-secrets-6fea25c9e1a4
- https://awsteele.com/blog/2021/09/15/aws-federation-comes-to-github-actions.html
- https://dev.to/mmiranda/github-actions-autenticando-na-aws-via-oidc-2cf
