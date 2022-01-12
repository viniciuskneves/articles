# S3: Beyond simple storage

S3 stands for Simple Storage Service. It is indeed simple to use but it has a lot of features that go beyond the word "simple".

It has some quite interesting features once it comes to static websites which is, in short, hosting a few assets somewhere which will be downloaded by someone. The assets are our website, somewhere is S3 and someone are our users.

S3, out of the box, offers support to static hosting and provides you with an URL that you can use by the end. It also allows you to add some metadata to your assets and uses this metadata as headers in the response. It also allows you to use the metadata to add some simple redirections (it has a more sophisticated way to do redirections as well).

S3, out of the box, doesn't offer good support for dealing with some edge cases on redirections and behaves not as expected when dealing with query strings. We had to overcome that as they were important to us.

I will share some code examples below, and as mentioned in the first post, they will be using AWS CDK.

## Hosting some assets

The first thing you need, after having your assets ready, is a place to host them. We are talking about S3, so you need a Bucket.

From AWS:

> Amazon S3 is an object storage service that stores data as objects within buckets. An object is a file and any metadata that describes the file. A bucket is a container for objects.

The following piece of code is what you need to create a Bucket in AWS CDK with a custom name (otherwise a random name will be given):

```typescript
import { Bucket } from 'aws-cdk-lib/aws-s3';
...
const bucket = new Bucket(this, 'Bucket', {
  bucketName: 'my-bucket-name',
});
```

Now that we've our Bucket ready, we need to move some files there. This step is usually performed during CI/CD, where we build our applications and deploy them (in this case move files) somewhere.

We can do so using AWS CLI. Consider that our application is built under `/dist`. The following command will copy all files from `/dist` to our Bucket:

```bash
aws s3 cp dist/ s3://my-bucket-name --recursive
```

Once we've our files there, we can connect it to CloudFront to serve it or use the static hosting feature from S3. First we will explore the static hosting feature and then we will connect it to CloudFront in the next post.

## Enabling static hosting

S3 allows you to customize DNS and use a different domain. In our case, these things are done through CloudFront.

Right now, if you try to access the files you uploaded, you will get an error:

```
GET https://my-bucket-name.s3.eu-central-1.amazonaws.com/index.html

403 Forbidden
```

The first problem we've is related to permissions:
1. The Bucket allows Objects to be public by default (where public means that anyone can access them);
2. The uploaded Objects are, by default, private;

There are two ways of tackling that:
1. Upload each Object (file) with public access, which means that the access is setup individually by file;
2. Make all objects in the Bucket public by default;

We went for 2 as it hosts our static site and all its files should be available. It might not be your case and is not the case for every Bucket, where a mix between public/private might be desired.

To do that, we need the following piece of code:

```typescript
import { Bucket } from 'aws-cdk-lib/aws-s3';
...
const bucket = new Bucket(this, 'Bucket', {
  bucketName: 'my-bucket-name',
  publicReadAccess: true,
});
```

With that we get:

```
GET https://my-bucket-name.s3.eu-central-1.amazonaws.com/index.html

200 OK
```

This is good but we need to specifically point to the `index.html` file within the Bucket (or any other file). If we remove it (whenever we open an website we do not append `/index.html` to its URL), we end up with an error again:

```
GET https://my-bucket-name.s3.eu-central-1.amazonaws.com/

403 Forbidden
```

This is the default behavior of a Bucket, not a problem. Buckets aren't just used to host websites but we can configure that. To configure that we need the following piece of code:

```typescript
import { Bucket } from '@aws-cdk/aws-s3';
...
const bucket = new Bucket(this, 'Bucket', {
  bucketName: 'my-bucket-name',
  publicReadAccess: true,
  websiteIndexDocument: 'index.html',
});
```

This enables the static hosting feature from S3. It tells S3 that whenever we hit a "folder" it should search for the `index.html` document inside and return it. It also gives us a new URL to access the bucket with the desired behavior.

If we make a similar request (there is a `-website` after `s3`) from the last one, we get:

```
GET https://my-bucket-name.s3-website.eu-central-1.amazonaws.com/

200 OK
```

We can still request a specific file, like `index.html` or `favicon.ico`. This feature is important if your deployment has multiple folders. It will try to serve the `index.html` of each folder once a request is made. In short: if it can't find the file, it will try the `index.html` "inside" the folder.

Bear in mind that we still need the `publicReadAccess` as it targets something else, removing it will make everything stop working as files are private again.

To illustrate a bit what happens, let's consider the following structure:

```
dist
├── index.html
└── path
    └── index.html
```

Now let's make a request to `path`, without a trailing slash so S3 doesn't know we're requesting from a folder:

```
GET https://my-bucket-name.s3-website.eu-central-1.amazonaws.com/path

302 Moved Temporarily
Location: /path/
```

It redirects us to the folder as it exists. The next request will then get the `index.html` inside it. If there is no `index.html` inside the folder, it won't even redirect us and will return 404 directly.

Similar to `websiteIndexDocument` there is `websiteErrorDocument`. It defines which document should be forwarded to the user in case of an error (4xx). To add it to our bucket:

```typescript
import { Bucket } from '@aws-cdk/aws-s3';
...
const bucket = new Bucket(this, 'Bucket', {
  bucketName: 'my-bucket-name',
  publicReadAccess: true,
  websiteIndexDocument: 'index.html',
  websiteErrorDocument: 'error.html',
});
```

If we request a path that doesn't exist, it will return 404 to us and the document linked. It is especially useful if you're dealing with Single Page Applications where you should also point to `index.html` in case of error as most probably the routing will be handled in the Frontend:

```
GET https://my-bucket-name.s3-website.eu-central-1.amazonaws.com/wrong-path

404 Not Found --> With `error.html` as a response
```

This is the basic setup we can get from static hosting on S3. It has a bunch of other features that we didn't use in our process but it can be useful to you. Now let's explore how to use metadata to redirect and set cache headers.

## Metadata

Every object in S3 can store some metadata. This metadata can be used by a lot of different things, from deleting objects based on some rules to set response headers, which is our case.

### HTTP response headers

When using AWS CLI you can use some flags which set some predefined metadata to you, like cache control. You can also manually set it as custom metadata called `Cache-Control` but we will use what AWS CLI gives us out of the box.

To add some cache control to our deployment, we need to:

```bash
aws s3 cp dist/ s3://my-bucket-name --recursive --cache-control "public"
```

All objects copied to S3 with the command above will have the metadata `Cache-Control` set to `public`. It is then used in the response and the HTTP header `Cache-Control` is automatically set for us.

Keep in mind: this setting is on an Object level, not on the Bucket level. It means that we can customize it for different kinds of files: some files are not cached, some are cached forever, some are cached for a day... This is done during deployment and not during infrastructure setup.

In our case, bundled files like [hash].js are cached "forever", meanwhile files like `index.html` are not. We have different steps in the deployment process to set the cache headers accordingly.

### Redirect

You can set a variety of different headers and it will work out of the box. There is also another custom header to specify redirections. Setting the metadata `x-amz-website-redirect-location` will make S3 redirect the object to somewhere else. Imagine the following structure:

```
dist
├── error.html
├── index.html
├── path
│   └── index.html
└── redirect
    └── index.html
```

Uploading `/redirect` enabling website redirection:

```bash
aws s3 cp dist/redirect s3://my-bucket-name/redirect --recursive --website-redirect "https://www.homeday.de"
```

Now requesting `/redirect`:

```
GET https://my-bucket-name.s3-website.eu-central-1.amazonaws.com/redirect/

301 Moved Permanently
Location: https://www.homeday.de
```

This is useful but you most probably won't be using it a lot. We've one use case that is to redirect our `error.html` to a default error page from our old system as we didn't migrate the error page yet.

S3 offers some more advanced setup in case you want redirection rules, overwrite parts of the path, and so on but end up not using them so far.

## Missing parts

So far we've seen how S3 can do some heavy lifting for us but as mentioned earlier, it "fails" (does not handle) some aspects.

How we solved those issues are out of the scope of this article and we will explore them in a follow-up post.

### Case insensitive

S3 doesn't know your paths are case insensitive. If we get our example and do the following:

```
GET https://my-bucket-name.s3-website.eu-central-1.amazonaws.com/patH/

404 Not Found --> With `error.html` as a response
```

We would like to handle the case above so, no matter if the user mistakenly types a letter in the upper case, we route them to the right path.

### Query strings

Query strings are really important for us. We use them a lot for marketing campaigns so it is not uncommon to see `?utm_xxx=yyy` somewhere.

By default, query strings are persisted if we don't need a redirect:

```
GET https://my-bucket-name.s3-website.eu-central-1.amazonaws.com/path/?utm_xxx=yyy

200 OK
```

If we rely on a redirect from S3, the query strings are lost:

```
GET https://my-bucket-name.s3-website.eu-central-1.amazonaws.com/path?utm_xxx=yyy

302 Moved Temporarily
Location: /path/
```

This is problematic because we lose important information once this redirect happens.

---

This is all about S3 for now. We managed to deploy an application and make it work.

Next step is to add CloudFront and tackle some missing parts, as the DNS mentioned earlier. This will be done in the next post.
