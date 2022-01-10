# The road to Static Site Generation on AWS

At Homeday we are responsible for maintaining our main website: http://www.homeday.de. There are a few reasons for that but it is our main entry point to get new customers.

Until earlier this year we used a CMS called [Kirby](https://getkirby.com/) which is PHP-based and gives you a lot of flexibility.

Quoting Uncle Ben:
> _With great power comes great responsibility._

It translates to "with great flexibility comes great responsibility" in our context.

We always had issues with Kirby: from, sometimes, lack of documentation (both, from our customizations and official docs); lack of knowledge on how its infrastructure works (we are good at HTML/CSS/JS); and issues on applying our [component library](https://github.com/homeday-de/homeday-blocks) to the project (it is Vue-based and although we managed to integrate it, the experience is not great).

Over the end of last year and the beginning of this year, we started researching how to move away from Kirby. We ended up picking [Nuxt](https://nuxtjs.org/) (which is Vue-based and would allow us to achieve some SEO requirements) and [Contentful](https://www.contentful.com/) (headless CMS with some useful features). We also decided to approach it using Static Site Generation (SSG) instead of maintaining servers, as it is (in terms of infrastructure) similar to all our Vue applications and beneficial in terms of performance.

In this series of posts (stay tuned for the next ones), I want to highlight how we approached this migration in terms of infrastructure, which challenges we faced, and what we learned on the way.

## Infrastructure as Code

In short: keep your infrastructure setup coded.
The main problem I see it tackling is the following:
- Usually, not everyone is aware of how your infrastructure is setup: someone, one day, made it work and told others, that might not be in the company anymore, how they made it. They might have written some docs, somewhere, where they told some people that also left the company where these docs live... Well, we know this story for a lot of other things.

In short (again): I think it is essential once you go out of the MVP phase.
In not so short: Although I consider it essential, I know it is challenging. If you’ve time/knowledge you can begin with it, as adding it later (same applies to a lot of other topics) is harder. If you can’t begin with it, ship first and come back to it later but do not neglect it if you work in a team.

I consider it important for multiple reasons (besides the one mentioned in the first paragraph):
- Version control: If someone makes a change directly on the AWS console, it will be hard for the team to know what has changed. Version control will allow you to review changes before they’re applied and keep your infrastructure consistent over time.
- Documentation: It is code, you can add comments, you can go through old discussions… It helps!

I also understand it from the following perspective: not everyone will be interested in it, especially in Frontend, so I don't expect everyone to work with it in a team but it helps, a lot, to onboard new people to it and let them explore.

Enough reasons/comments on Infrastructure as Code (IaC) for now.
We end up using [AWS CDK](https://aws.amazon.com/cdk/) for one main reason: it allows us to use Typescript (or other languages) which is less of a blocker for new contributors in Frontend. It is verbose but it does a lot of the weight lifting for you and is extremely flexible as well.

Throughout the rest of this and the next articles, I will be using AWS CDK as example.

## Which AWS services are we using?

We didn't finish the migration yet which means we still have the old servers from Kirby running. They are not part of the IaC but referenced there.

We rely on three main services:
- S3: storage but we also use it for setting up cache and redirection. It also gave us some headaches 😅
- CloudFront: it is our CDN so it is essential for caching but also helps us to setup our routing so we can split the traffic between Kirby and the new system.
- Lambda functions: we use a few Lambda@Edge functions as we don't have servers so we need to run some pieces of code throughout the request-response lifecycle. It will be hard to go full serverless and not need to customize a few things as not all services will tackle all your needs.

Other services like Route 53, Certificate Manager, and WAF are also part of the infrastructure but they are not the main focus as they can come last in your setup and are more references, in our case, than custom configuration.

---

In the next articles we will go into details of each service and which features we ended up using from each of them. We will also explore some limitations and how we overcame them with our current setup.
