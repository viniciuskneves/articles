# Prettier, ESLint and Typescript

I decided to write this article to sumup a struggle of mine. We've started a new project in the company, Prettier was setup, ESLint was setup and at some point we added Typescript. By the end Typescript was also setup. CI was linting, commit hooks were also linting, VSCode was fixing the code and so on (that is what I thought).
At some point I was playing with the project and realized that some files were being warned by my editor but not when running the linter (`npm run lint` in my case). I got triggered. I've a hard time accepting that something works but I can't understand, unless it is an absolutely external tool that I didn't have to setup myself but that was not the case here.

In this article I will summarize some understandings that I've about integrating all the tools above. The main focus is how to setup Prettier, how to setup ESLint, how to integrate both and by the end how to add Typescript to it.

## Prettier

The first tool I want to explore is [Prettier](https://prettier.io/). I would leave it to you to read more about what it is but in short it is a code formatter. What does it mean? It means that it will keep your codebase consistent (in terms of coding style). Do you use `;`? If yes, it will ensure that all your files have it, for example. I like it for two reasons: we barely have to discuss code formatting and it is easy to onboard new members to the team.

*At the time of this writing, Prettier is in version 2.4.1, so keep in mind things might change (especially formatting) in future versions.*

### How to setup Prettier?

I will consider you've a project setup already so in short you need to install it:

```bash
npm i prettier #--save-dev and --save-exact are recommended
```

Right now you can start using Prettier. You don't need any configuration (if you don't want). You can run it against your codebase with:

```bash
npx prettier .
```

The `.` at the end means run across your whole codebase. You can run for a specific file or pattern if you want.
This command will print the files formatted, nothing special. A more useful command happens when you add `--write` flag. Instead of printind the formatted code, it will write to the origin file.

Let's create a file called `index.js` with the following code:

```javascript
// index.js
const a = 1
```

If we run `npx prettier index.js`, the output will be:

```javascript
const a = 1;
```

It automatically adds the `;` for us but it is not saved in the file. If we run `npx prettier index.js --write` though, the file will change and the `;` will be added to it.

Cool, that is the simplest setup we can have with Prettier. The [default rules are documented in their website](https://prettier.io/docs/en/options.html) and can be customized (a bit). We will take a look into it next but before I want to mention another flag: `--check`.

The `--check` flag, `npx prettier index.js --check`, is useful if you just want to check if a file (or the codebase with `.`) are Prettier compliant. It is useful for CIs and git hooks, for example, if you just want to warn the user (you can also enable `--write` in these scenarios).

If we consider the following code again:

```javascript
// index.js
const a = 1
```

And run `npx prettier index.js --check`, we get the following output:

```
Checking formatting...
[warn] index.js
[warn] Code style issues found in the above file(s). Forgot to run Prettier?
```

### Prettier configuration

You can configure Prettier at some extend. You can do it via the CLI or via a [configuration file](https://prettier.io/docs/en/configuration.html), which is more adequate. The configuration file could be in a variety of formats so you can choose the one that fits you best.

Add the configuration file to the root of your project (you can have configurations/folder but I would leave it up to you to explore this path) and start adding rules to it:

```json
// .prettierrc
{
  "semi": false
}
```

With this configuration file and the following code, again, the `--check` run will succeed:

```javascript
// index.js
const a = 1
```

`npx prettier index.js --check`:

```
Checking formatting...
All matched files use Prettier code style!
```

On top of that, you can also extend configuration and setup a few other things. Check their [configuration documenation](https://prettier.io/docs/en/configuration.html) for more details.

## ESLint

[ESLint](https://eslint.org/) has been around for a while. In short it does a bit more than Prettier as it analyzes your code in order to find problems (or patterns that you don't want, like variables that are not used should be removed). Again, I invite you to read ESLint documentation if you want to go deeper into the topic. I like ESLint for the simple reason it helps me to find problems and configure some patterns in the project (it might be useful when onboarding new people). It is extremely extensible as well in case you're interested.

*At the time of this writing, ESLint is in version 7.32.0, so keep in mind things might change (especially formatting) in future versions. Version 8 is in beta at the moment.*

### How to setup ESLint?

In short, quite similar to Prettier but you need the configuration file. I will consider you've a project setup already so in short you need to install it:

```bash
npm i eslint #--save-dev is recommended
```

You need a configuration file. You can create one by yourself or you can run the command below that bootstraps one for you (it can add a lot of presets already):

```bash
npx eslint --init
```

Let's start with an empty configuration file though, it is enough to run ESLint:

```javascript
// .eslintrc.js
module.exports = {
};
```

We can now run it, similar to Prettier:

```bash
npx eslint .
```

*One thing to note here: ESLint runs only on `.js` files (by default).*

Let's consider the same example as before:

```javascript
// index.js
const a = 1
```

`npx eslint index.js` and we get:

```
1:1  error  Parsing error: The keyword 'const' is reserved
✖ 1 problem (1 error, 0 warnings)
```

This is simply the issue with a default ESLint configuration. It considers ES5 by default, so `const` is not allowed yet, and some older setup that might make sense to your project but not in general.

We can spend hours configuring ESLint but in general we get a default from a styleguide (AirBnB for example) and apply to our project. If you use the init command you can do so.

Let's install [AirBnB ESLint configuration](https://www.npmjs.com/package/eslint-config-airbnb-base), it also requires `eslint-plugin-import` to be installed (following their documentation) so:

```bash
npm i eslint-config-airbnb-base eslint-plugin-import # --save-dev is recommended
```

Then we extend it in our configuration, so it will look like:

```javascript
module.exports = {
  extends: [
    'eslint-config-airbnb-base', // or `airbnb-base`, you can omit `eslint-config-`
  ]
};
```

Running `npx eslint index.js` again we get:

```
1:7   error  'a' is assigned a value but never used  no-unused-vars
1:12  error  Missing semicolon                       semi

✖ 2 problems (2 errors, 0 warnings)
  1 error and 0 warnings potentially fixable with the `--fix` option.
```

Cool! Now we get errors defined by the AirBnB guide. We can use the `--fix` option, which works similar to `--write` from Prettier, in case we want to fix the errors when possible.

ESLint allows you to extensively configure it if you want. It goes a way beyond the scope here and I will leave it up to you to explore and play with it: https://eslint.org/docs/user-guide/configuring/

## Prettier + ESLint

There are a lot of tutorials online on how to connect both. I want to take a different approach and try to reason about each tool and how they connect.

I will assume that we've the following Prettier configuration:

```json
// .prettierrc
{
  "semi": false
}
```

I will assume that we've the following ESLint configuration:

```javascript
// .eslintrc.js
module.exports = {
  extends: [
    'eslint-config-airbnb-base',
  ]
};
```

I will assume the following script to run both tools:

```javascript
// index.js
const a = 1

module.exports = { a }
```

If we run Prettier check, we get:

```
Checking formatting...
All matched files use Prettier code style!
```

Cool! If we run ESLint, we get:

```
1:12  error  Missing semicolon  semi
3:23  error  Missing semicolon  semi

✖ 2 problems (2 errors, 0 warnings)
  2 errors and 0 warnings potentially fixable with the `--fix` option.
```

Not so cool! Running ESLint with fix will fixes these issues. Now if we run Prettier again we get:

```
Checking formatting...
[warn] index.js
[warn] Code style issues found in the above file(s). Forgot to run Prettier?
```

If we run Prettier with `--write` it will fix but then ESLint will fail again. It will be like this forever. If the goal were just formatting, I would say pick one of the tools and ignore the other, but we want the power of both tools, especially as ESLint is more than just formatting your code.

Prettier provides two packages that integrate with ESLint.
- [`eslint-config-prettier`](https://github.com/prettier/eslint-config-prettier): turns off rules that might conflict with Prettier.
- [`eslint-plugin-prettier`](https://github.com/prettier/eslint-plugin-prettier): adds Prettier rules to ESLint.

Let's go step by step. First let's go and install `eslint-config-prettier`:

```bash
npm i eslint-config-prettier # --save-dev recommended
```

Our new `.eslintrc.js` will look like:

```javascript
module.exports = {
  extends: [
    'eslint-config-airbnb-base',
    'eslint-config-prettier',
  ]
};
```

Considering the file below, again:

```javascript
const a = 1

module.exports = { a }
```

It was a valid file for Prettier but invalid for ESLint. Using the new configuration, it becomes valid as the conflicting [rule `semi` has been disabled](https://github.com/prettier/eslint-config-prettier/blob/main/index.js#L73).
It is fine if we want to ignore the rules from Prettier but in general we want that Prettier rules override ESLint rules.
In case we delete the Prettier configuration file and use its defaults (which requires `;`), running Prettier check will result in:

```
Checking formatting...
[warn] index.js
[warn] Code style issues found in the above file(s). Forgot to run Prettier?
```

The file is not valid anymore as it is missing the `;` but ESLint run won't fail, as the Prettier rules have been disabled when running ESLint.

One important thing to note here: the order used by `extends`, in the ESLint configuration, matters. If we apply the following order we will get an error as AirBnB rules will override Prettier disabled rules when running ESLint:

```javascript
module.exports = {
  extends: [
    'eslint-config-prettier',
    'eslint-config-airbnb-base',
  ]
};
```

Running `npx eslint index.js`:

```
1:12  error  Missing semicolon  semi
3:23  error  Missing semicolon  semi

✖ 2 problems (2 errors, 0 warnings)
  2 errors and 0 warnings potentially fixable with the `--fix` option.
```

To mitigate this issue let's install the plugin:

```bash
npm i eslint-plugin-prettier # --save-dev recommended
```

We can then update our `.eslintrc.js` file to:

```javascript
module.exports = {
  extends: [
    'eslint-config-airbnb-base',
    'plugin:prettier/recommended',
  ]
};
```

We replaced `eslint-config-prettier` by `plugin:prettier/recommended`. Check ESLint docs about extending a plugin: https://eslint.org/docs/user-guide/configuring/configuration-files#using-a-configuration-from-a-plugin

Running ESLint again we will get:

```
1:12  error  Insert `;`  prettier/prettier
3:23  error  Insert `;`  prettier/prettier

✖ 2 problems (2 errors, 0 warnings)
  2 errors and 0 warnings potentially fixable with the `--fix` option.
```

Two things to note here:
1. We are getting `;` errors again, that have been disabled earlier with `eslint-config-prettier`;
2. The error is coming from the rule `prettier/prettier`, which is added by the plugin. All prettier validations will be reported as `prettier/prettier` rule.
