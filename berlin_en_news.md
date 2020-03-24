# Creating a Twitter BOT for Berlin English speakers

![Berlin panoramic](https://pbs.twimg.com/profile_banners/18760371/1494923251/1500x500)

I'm going to walk you through the process of creating [@Berlin_en_News](https://twitter.com/Berlin_en_News), a Twitter BOT that tweets news in English for non-German speakers.
The [project](https://github.com/viniciuskneves/berlin_en_news) was developed using Javscript. It is an AWS Lambda function that has an AWS CloudWatch scheduler as trigger. The function crawls Berlin's latest news and tweets it =]

## Motivation

I'm working from home since mid-March due the Corona outbreak. On the first week I've been constantly reading the news about it but I live in Berlin and I don't speak proper German.
Berlin provides has its official [English News channel](https://www.berlin.de/en/news/) which I think is super cool. It also has its official Twitter account [@Berlin_de_News](https://twitter.com/Berlin_de_News) which tweets their news in German.
The issue here is that they don't offer an English option. The Twitter account tweets only the German's news so if you want to have the "latest" English news you would have to open their website.
That was my main motivation to create [@Berlin_en_News])https://twitter.com/Berlin_en_News), a bot that would tweet Berlin's News in English. The idea is that you can get notified everytime that there is an update.

---

Enough of introduction and motivation. From now on I'm going to dive into how it was implemented and I would love to have your feedback. I hope the project keeps improving over time, I can see a lot of room for improvements, from tech to new ideas!

The project consists in 2 basic structures: Crawler and Twitter API =]
I'm going to also talk about the deployment, using AWS SAM in this case, and at the end I do also invite you to contribute (not just tech-wise) and share it =]

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

