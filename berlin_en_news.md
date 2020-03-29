# Creating a Twitter BOT for Berlin English speakers

UPDATE IMAGE: BERLIN BOT PROFILE PICTURE
![Berlin panoramic](https://pbs.twimg.com/profile_banners/18760371/1494923251/1500x500)

I'm going to walk you through the process of creating [@Berlin_en_News](https://twitter.com/Berlin_en_News), a Twitter BOT that tweets Berlin's news in English for non-German speakers.
The [project](https://github.com/viniciuskneves/berlin_en_news) was developed using Javscript. It is an AWS Lambda function that has an AWS CloudWatch scheduler as trigger. The function crawls Berlin's latest news and tweets it =]

## Motivation

I'm working from home since mid-March due the Corona outbreak. On the first days I had been constantly reading the news about it but there is a problem: I live in Berlin and I don't speak proper German.
Berlin has its official [English News channel](https://www.berlin.de/en/news/) which I think is super cool. It also has its official Twitter account [@Berlin_de_News](https://twitter.com/Berlin_de_News) which tweets their news in German.
The issue here is that they don't offer an English option. The Twitter account tweets only the German's news so if you want to have the "latest" English news you would have to open their website.
That was my main motivation to create [@Berlin_en_News](https://twitter.com/Berlin_en_News), a bot that would tweet Berlin's News in English. The idea is that you can get notified everytime that there is an update.

---

Enough of introduction and motivation. From now on I'm going to dive into how it was implemented and I would love to have your feedback. I hope the project evolves over time, I can see a lot of room for improvements, from tech to new ideas!

The project consists in 2 basic structures: Crawler and Twitter API =]
I'm going to also talk about the deployment, using AWS SAM in this case, and at the end I invite you to contribute (not just tech-wise) and share it =]

## Crawler

First let me mention which webpage I'm crawling: https://www.berlin.de/en/news/

The idea is to go and grab the URL and title of every article in this page and tweet it. Thankfully this page is statically generated so I don't need to worry about any async requests made to extract the data that I need. This means that I need to download the source of the page and then parse it somehow.

### Downloading page source

There are a lot of different ways of doing it. You can even do it from your terminal if you want: `curl https://www.berlin.de/en/news/`.
I chose [axios](https://github.com/axios/axios) as I do use it almost everyday at work. You don't need a library to do it and axios is indeed and overkill here.

Nevertheless, the code with axios looks like the following:

```javascript
const axios = require('axios');

const BASE_URL = 'https://www.berlin.de';
const NEWS_PATH = '/en/news/';

async function fetchArticles() {
  const response = await axios(`${BASE_URL}${NEWS_PATH}`);
  
  console.log(response.data); //<!DOCTYPE html><html ...
}
```

The code is quite straightforward. I'm using `BASE_URL` and `NEWS_PATH` because I will need them later. The HTML that we want is under `.data` property from axios response.

That is all we need to do to grab the data that we need, now we need to parse it!

### Parsing page source

The parsing step should be simple. Given an HTML document as input I want to extract some structured information out of it. My first idea is: take the article title and the article link. So every tweet will contain the title and the link to the original article. It is similar to what [@Berlin_de_News](https://twitter.com/Berlin_de_News) does:

![Berlin DE News tweet example](https://i.imgur.com/b1qU1Jd.png)

To parse the HTML, I've choosen [cheerio](https://cheerio.js.org) which allows you to "jQuery" the input. This way I can navigate and select parts of the HTML document that I want to extract the data from.

The parsing code looks like the one below:

```javascript
const cheerio = require('cheerio');

async function parseArticles(html) { // HTML is `response.data` from `fetchArticles`
  const $ = cheerio.load(html);
  // `.special` might include some "random" articles
  const articles = $('#hnews').parent().find('article').not('.special').map(function() {
    const heading = $(this).find('.heading');
    return {
      title: heading.text(),
      link: `${BASE_URL}${heading.find('a').attr('href')}`,
    };
  }).toArray();

  console.log('Fetched articles: ', articles);

  return articles;
}
```

I navigate through all the `<article>` from an specific part of the page and `.map` them. There are some specific things like `#hnews`, `.parent()` and `.not()` that are rules I followed to find the articles section. This is a sensitive part but it does the job for now. The same result could be achieved using other selectors as well.

The result is the following structre:

```javascript
[
  {
    title: 'Article title',
    link: 'https://www.berlin.de/path/to/article/title'
  },
  {
    title: 'Article title 2',
    link: 'https://www.berlin.de/path/to/article/title-2'
  }
]
```

This concludes our crawler: it fetches the page and parses so we have a more structure data to work.

Next step is to tweet the extracted articles.

## Tweeting

First step was to create a Twitter account/app. Thankfully the handler `@Berlin_en_News` was not yet taken and it would be perfect for this case as the German version (official) is called `@Berlin_de_News`.

Now I've all the necessary keys to use the [Twitter API](https://developer.twitter.com/en/docs/basics/getting-started) and I just need to start tweeting.

I ended up using a library called [twitter](https://github.com/desmondmorris/node-twitter) to do so. It is not necessary to use a library as Twitter API seems really friendly but my goal was not to optimize or so in the beginning, I wanted to make it work first =]

This is the code needed to get the library ready to use (all Twitter keys are environment varibles):

```javascript
const Twitter = require('twitter');
const client = new Twitter({
  consumer_key: process.env.TWITTER_API_KEY,
  consumer_secret: process.env.TWITTER_API_SECRET_KEY,
  access_token_key: process.env.TWITTER_ACCESS_TOKEN,
  access_token_secret: process.env.TWITTER_ACCESS_TOKEN_SECRET,
});
```

To tweet we need to use the following API: [POST statuses/update](https://developer.twitter.com/en/docs/tweets/post-and-engage/api-reference/post-statuses-update). It has a lot of different parameters. At the beginning I'm ignoring most of them. I'm just using the `place_id` so it shows that the tweet is from Berlin.

ADD IMAGE: IMAGE THAT SHOW A TWEET FROM BERLIN --> BERLIN_EN_NEWS

The following code walks through the process of tweeting:

```javascript
const placeId = '3078869807f9dd36'; // Berlin's place ID

async function postTweet(status) {
  const response = await client.post('statuses/update', { // `client` was instantiated above
    status, // Tweet content
    place_id: placeId,
  });

  return response;
}

for (const article of newArticles) { // `newArticles` come from the crawler
  const response = await postTweet([
    article.title,
    `Read more: ${article.link}`,
  ].join('\n'));

  console.log('Tweet response: ', response);
}
```

The BOT is almost ready. It misses on important aspect: it shouldn't tweet the same article again. So far it doesn't know which articles it already tweeted.

### Filtering new articles

This process denifitely needs to be improved but it does the job for now (again) =]

I fetch the BOT's timeline and compare it with the articles' titles. The only tricky thing is that Twitter won't exactly use the article URL in the tweet itself, so some dirty "magic" had to be written for now. As I said, it does the job for now =]

```javascript
async function homeTimeline() {
  const response = await client.get('statuses/user_timeline', {});
  const responseTitles = response.map((tweet) => tweet.text.split('\n')[0]); // Dirty "magic" 🙈

  console.log('Last tweets titles: ', responseTitles);

  return responseTitles;
}

const [articles, tweets] = await Promise.all([fetchArticles(), homeTimeline()]);
const newArticles = articles.filter(article => !tweets.includes(article.title));
```

With that in place I'm "sure" that it will tweet only the new articles.

Now the BOT itself is done. There is one major issue: I need to run it on my machine. The next step is to deploy it so it runs automatically =]

## Deployment

I chose to deploy it to Lambda by convenience as I'm more familiar to it and this BOT won't run all day long. It will run every 30min (using a CloudWatch scheduler) at the moment which means that it would be a good use case for Lambda.

Everything was deployed using [AWS SAM](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html) as I wanted to try the tool in a real project. It gives you a lot of flexibility but also some challenges if you compare it to [Serverless Framework](https://serverless.com) for example.

You can check the PR where I added the deployment here: https://github.com/viniciuskneves/berlin_en_news/pull/4

The configuration file `template.yaml` (which is used by SAM) is divided into 3 important blocks that I'm going to explore: Resources, Globals and Parameters.

### Resources

In my case I'm using a Lambda Function and a CloudWatch scheduler as resources. The CloudWatch scheduler is automatically created for us once we define it as an event source for our function. The trickiest part here is to know how to define a schedule, which you would have to go through the docs if you want to understand it a bit better: https://docs.aws.amazon.com/eventbridge/latest/userguide/scheduled-events.html

```yaml
Resources:
 TwitterBotFunction: # Defining an AWS Lambda Function
   Type: AWS::Serverless::Function
   Properties:
     Handler: index.handler
     Events:
       Scheduler: # CloudWatch Scheduler automatically created
         Type: Schedule
         Properties:
           Description: Schedule execution for every 30min
           Enabled: true
           Schedule: 'rate(30 minutes)' # Runs every 30min
```

### Globals

Those are global settings applied to our resources. I could have defined them inside each resource for example but it doesn't make sense for the project so far.

I'm setting my runtime, which is Node.js for this project, a timeout for Lambda and also my environment variables which are used by my function (Twitter keys).

```yaml
Globals:
 Function:
   Runtime: nodejs12.x
   Timeout: 5
   Environment:
     Variables:
       TWITTER_API_KEY: !Ref TwitterApiKey
       TWITTER_API_SECRET_KEY: !Ref TwitterApiSecretKey
       TWITTER_ACCESS_TOKEN: !Ref TwitterAccessToken
       TWITTER_ACCESS_TOKEN_SECRET: !Ref TwitterAccessTokenSecret
```

What is missing now is where do those keys come from, that is why I've added a Parameters block.

### Parameters

Those are the parameters my build is expecting. I've decided to setup it like that in a way to avoid hardcoding the keys. There are different strategies here and I went for the quickest one for now.

```yaml
Parameters:
 TwitterApiKey:
   Description: Twitter API Key
   NoEcho: true
   Type: String
 TwitterApiSecretKey:
   Description: Twitter API Secret Key
   NoEcho: true
   Type: String
 TwitterAccessToken:
   Description: Twitter Access Token
   NoEcho: true
   Type: String
 TwitterAccessTokenSecret:
   Description: Twitter Access Token Secret
   NoEcho: true
   Type: String
```

Now, once I call the deployment command I need to pass those parameters as arguments:

```bash
sam deploy --parameter-overrides TwitterApiKey=$TWITTER_API_KEY TwitterApiSecretKey=$TWITTER_API_SECRET_KEY TwitterAccessToken=$TWITTER_ACCESS_TOKEN TwitterAccessTokenSecret=$TWITTER_ACCESS_TOKEN_SECRET
```

## Contribute and share

I hope I could briefly share the idea behind the BOT and I also hope you could understand it. Please, don't hesitate to ask, I will try my best to help you.

It has been a fun process, some learnings as Twitter account blocked by mistake, but at the end it was useful, at least to me. Now I don't need to open the news website every day and can just wait until I get notified about a new tweet =]

I would appreciate if you could share the project so it would help other people as well, especially in Berlin =]
I would also appreciate if you want to contribute to the project:
- New ideas: add images to tweets, add comments... Anything that could be done on Twitter level to enhance the experience.
- Project maintenance: I've setup some [issues on GitHub](https://github.com/viniciuskneves/berlin_en_news/issues) and you're more than welcome to give it a try.
- New sources: do you have any other sources that are worth adding? Let me know and we can work on it.
- New city/topic: would you like to have it in your city as well? For a specific topic? Let's make it happen =]

Thank you and #StayHome =]
