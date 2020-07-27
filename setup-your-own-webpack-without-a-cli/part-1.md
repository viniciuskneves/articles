# Setup your own Webpack without a CLI

*This is the first time I write a series of articles. I decided to structure it like that as I don't have the full picture of what I'm going to be writing, I've just the first part of it written, and I also want to avoid a 10min reading article at the end.*

---

I've been doing Vue, professionally, for a while now. As usual, whenever I create/setup a new project, I use Vue CLI which abstracts a lot of the heavy weight of creating a new project, which is great, especially if you've ever tried to setup a full Webpack config by yourself. The same applies if you use other CLIs to generate your project.
Nonetheless, I want to walk through the process of setuping a Webpack config by myself and highlight some benefits and drawbacks of using Vue CLI on the way. I will try to mimic as much as possible the behavior of Vue CLI but avoid the usage of its plugins for example.

Why I want to do that? Well, as sort of documentation/study for myself (I can't remember the last time I went through each Webpack loader and customized it, which is great, don't get me wrong) and also, maybe, to help people to understand the full process and challenges that comes with such setup (so you might get some feeling about why CLIs do a great job for us).

The content will be split between Webpack, Vue and enhancements. First I will go through Webpack and its concepts, focusing more on examples than in the theory ([Webpack docs](https://webpack.js.org/concepts/) are great if we need to deep dive into some concepts). After that I want to be able to have a bare minimum Webpack config to build a Vue project. At the end I will explore some "enhancements", that is how I decided to call it, to my Webpack config. By enhancements I mean everything that "enhances" our developer experience or add some flavor to our build (file compressing, for example, or splitting...).

---

## Webpack

Most probably you've heard about Webpack before, if not, check their [official website](https://webpack.js.org) which has tons of documentation about it. I think this image, from the offical website, summarizes quite well what Webpack does for us:

![Webpack bundle process](https://i.ibb.co/Yyw2L2v/image-1.png)

Once you start reading the documentation you will find out that the first two dependencies you've to install in order to use Webpack are `webpack` and `webpack-cli`. From here we can start customizing our Webpack config which tells Webpack how it should process our files.

The Webpack config starts with the `entry` property, which specifies the entry file to be processed with Webpack. Let's create the following `webpack.config.js` file, which tells Webpack that the entry point is `./src/main.js`:

```javascript
module.exports = {
  entry: './src/main.js',
};
```

If we add any Javascript code to `main.js` it will get processed by Webpack and the output will be placed inside `dist/` folder (by default). To run Webpack we just need to `npx webpack --config webpack.config.js`.

It works (if we have a `main.js` file created) but it complains (warns) that the `mode` property is not set. From Webpack docs, the `mode` property:

> Providing the `mode` configuration option tells webpack to use its built-in optimizations accordingly.

The `mode` property can be `production`, `development` or `none`. To set the `mode` property we can just pass `--mode=MODE_WE_WANT` to our `npx webpack` call. To make things more structured we can add two commands to our NPM scripts, one to build for production and another one for development:

```json
"scripts": {
  "build:production": "webpack --config webpack.config.js --mode=production",
  "build:development": "webpack --config webpack.config.js --mode=development"
}
```

Ok, now we can add Javascript code to `main.js` and build it. Not that useful, but it is the basic setup and we made it work.

### Loaders

Loaders are the magic behind Webpack. They allow us to process different file types in different ways and the outcome will be a single (or multiple) bundled file. From the docs, loaders are defined as:

> webpack enables use of loaders to preprocess files. This allows you to bundle any static resource way beyond JavaScript.

Once we are developing for the web, one main concern that we usually have is: it should work cross browser. Which means, it should fallback features that are not present in old browsers. A project that does this kind of job is [Babel](https://babel.dev) and we can, via loader, use it with Webpack. We can tell Webpack to process Javascript files with Babel, which will do the job of compiling our code.

To use Babel with Webpack we need to setup its [loader](https://github.com/babel/babel-loader). From the Babel loader docs we endup with the following Webpack config:

```javascript
module.exports = {
  entry: './src/main.js',
  module: {
    rules: [{
      test: /\.js$/,
      exclude: /node_modules/,
      use: {
        loader: 'babel-loader',
      },
    }],
  },
};
```

Let's take a step back first. If the `main.js` file contained something that doesn't need to be compiled, it won't make a huge difference here. For example, if it contains `console.log('Hello World!')`, the output will be the same with or without the loader. It gets more interesting once we started adding features that are not supported in all browsers.

First lets setup Babel configuration file, also known as `babelrc`, which allows us to use special syntax if we want or use the well known [`@babel/preset-env`](https://babeljs.io/docs/en/babel-preset-env) which:

> is a smart preset that allows you to use the latest JavaScript without needing to micromanage which syntax transforms (and optionally, browser polyfills) are needed by your target environment(s). This both makes your life easier and JavaScript bundles smaller!

The setup needed is the following:

```json
{
  "presets": ["@babel/preset-env"]
}
```

Now our project is ready to compile "new" Javascript syntax. Let's take the following code as example:

```javascript
async function hey() {
  await Promise.resolve();

  return 'Hey!';
}

hey();
```

It uses a new syntax that won't work in old browsers. If we build our file without the Babel config, the output will be something like the following:

```javascript
eval("async function hey() {\n  await Promise.resolve();\n  return 'Hey!';\n}\n\nhey();\n\n//# sourceURL=webpack:///./src/main.js?");
```

If we setup the Babel config file, the output will be like:

```javascript
eval("function asyncGeneratorStep(gen, resolve, reject, _next, _throw, key, arg) { try { var info = gen[key](arg); var value = info.value; } catch (error) { reject(error); return; } if (info.done) { resolve(value); } else { Promise.resolve(value).then(_next, _throw); } }\n\nfunction _asyncToGenerator(fn) { return function () { var self = this, args = arguments; return new Promise(function (resolve, reject) { var gen = fn.apply(self, args); function _next(value) { asyncGeneratorStep(gen, resolve, reject, _next, _throw, \"next\", value); } function _throw(err) { asyncGeneratorStep(gen, resolve, reject, _next, _throw, \"throw\", err); } _next(undefined); }); }; }\n\nfunction hey() {\n  return _hey.apply(this, arguments);\n}\n\nfunction _hey() {\n  _hey = _asyncToGenerator( /*#__PURE__*/regeneratorRuntime.mark(function _callee() {\n    return regeneratorRuntime.wrap(function _callee$(_context) {\n      while (1) {\n        switch (_context.prev = _context.next) {\n          case 0:\n            _context.next = 2;\n            return Promise.resolve();\n\n          case 2:\n            return _context.abrupt(\"return\", 'Hey!');\n\n          case 3:\n          case \"end\":\n            return _context.stop();\n        }\n      }\n    }, _callee);\n  }));\n  return _hey.apply(this, arguments);\n}\n\nhey();\n\n//# sourceURL=webpack:///./src/main.js?");
```

Ok, let's break it into smaller pieces first. The first code is basically a copy and paste from the code we wrote. The second one has some magic so we don't see the usage of `async/await` for example. `@babel/preset-env` is doing its job here, we didn't need to tell it that we would be using a specific feature. Now the question that we might be asking is: what happens if I want to support only browsers that support `async/await`?

The answer is: [Browserslist](https://github.com/browserslist/browserslist). You might have seen a file called `.browserslistrc` in the root of a project or inside `package.json`. This file is responsible for specifying which browsers you want to support and is used by multiple tools to bundle your code. Babel is one of them. If you run `npx browserslist` in your project you will see a list of browsers it should support. By default, at the time of this writing, the list is the following (which is the `defaults` setting from `browserslist`):

```
and_chr 81
and_ff 68
and_qq 10.4
and_uc 12.12
android 81
baidu 7.12
chrome 83
chrome 81
chrome 80
edge 83
edge 81
edge 18
firefox 78
firefox 77
firefox 76
firefox 68
ie 11
ios_saf 13.4-13.5
ios_saf 13.3
ios_saf 12.2-12.4
kaios 2.5
op_mini all
op_mob 46
opera 69
opera 68
safari 13.1
safari 13
samsung 12.0
samsung 11.1-11.2
```

What we can do here is to setup our own `.browserslistrc` file with modern browsers, for example `last 2 Chrome versions`, and run our build again. The output will be the following:

```javascript
eval("async function hey() {\n  await Promise.resolve();\n  return 'Hey!';\n}\n\nhey();\n\n//# sourceURL=webpack:///./src/main.js?");
```

As the last 2 versions of Chrome support `async/await`, Babel won't transpile it so we will endup with "modern" code. As a side note, if you're not 100% sure of which browsers you need to support and so on, leave the default values and don't mess with your bundle.

Things are getting more interesting now. We are already using the Webpack magic to process our files. Now we are going to explore another Webpack concept, called plugins which will add more power to our build.

### Plugins

Plugins do what loaders can't do. Ok, not that easy to understand but they hook into Webpack's lifecycle and do something. Webpack itself has a broad list of [plugins](https://webpack.js.org/plugins/) that you can choose and each of them add a specific "power" to our build.

Let's take the example from Vue CLI, the environment variables. You can define some environment variables, they should begin with `VUE_APP`, and they will be available inside your application.

If we want to do something similar, let's say we want to define `BASE_URL` as an environment variable, we need to use a plugin for that. Webpack provides a plugin called [DefinePlugin](https://webpack.js.org/plugins/define-plugin/) which `allows you to create global constants which can be configured at compile time.`.

The Webpack config will look like:

```javascript
plugins: [
  new webpack.DefinePlugin({
    'process.env': {
      BASE_URL: JSON.stringify('localhost'),
    },
  }),
],
```

Now we can use `process.env.BASE_URL` inside our code and it will be replaced by `'localhost'`. Vue CLI does an extra job and allows you to define `.env` files which is out of scope for now.

### Final config

Our final config, after all the setup above, will look like the following:

```javascript
const webpack = require('webpack');

module.exports = {
  entry: './src/main.js',
  module: {
    rules: [{
      test: /\.js$/,
      exclude: /node_modules/,
      use: {
        loader: 'babel-loader',
      },
    }],
  },
  plugins: [
    new webpack.DefinePlugin({
      'process.env': {
        BASE_URL: JSON.stringify('localhost'),
      },
    }),
  ],
};
```

So far we will process the file `src/main.js`, it will use the DefinePlugin to replace `process.env.BASE_URL` by `'localhost'` and it will be processed by Babel which will take into account our `.browserslistrc` definition to add (or not) missing features.

That is the basic of Webpack setup. You can imagine the heavy lift CLIs do and I hope we can understand a bit better how things work under the hood.
