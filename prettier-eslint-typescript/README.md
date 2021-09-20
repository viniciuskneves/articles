# Prettier, ESLint and Typescript

I decided to write this article to sum up a struggle of mine. We've started a new project in the company, Prettier was set up, ESLint was set up and at some point, we added Typescript. By the end, Typescript was also set up. CI was linting, commit hooks were also linting, VSCode was fixing the code, and so on (that is what I thought).
At some point I was playing with the project and realized that some files were being warned by my editor but not when running the linter (`npm run lint` in my case). I got triggered. I have a hard time accepting that something works but I can't understand unless it is an external tool that I didn't have to set up myself but that was not the case here.

In this article, I will summarize some understandings that I've about integrating all the tools above. The main focus is how to set up Prettier, how to set up ESLint, how to integrate both, and by the end how to add Typescript to it.

## Prettier

The first tool I want to explore is [Prettier](https://prettier.io/). I would leave it to you to read more about what it is but, in short, it is a code formatter. What does it mean? It means that it will keep your codebase consistent (in terms of coding style). Do you use `;`? If yes, it will ensure that all your files have it, for example. I like it for two reasons: we barely have to discuss code formatting and it is easy to onboard new members to the team.

*At the time of this writing, Prettier is in version 2.4.1, so keep in mind things might change (especially formatting) in future versions.*

### How to set up Prettier?

I will consider you have a project set up already so in short, you need to install it:

```bash
npm i prettier #--save-dev and --save-exact are recommended
```

Right now you can start using Prettier. You don't need any configuration (if you don't want it). You can run it against your codebase with:

```bash
npx prettier .
```

The `.` at the end means to run across your whole codebase. You can run for a specific file or pattern if you want.
This command will print the files formatted, nothing special. A more useful command happens when you add `--write` flag. Instead of printing the formatted code, it will write to the origin file.

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

Cool, that is the simplest setup we can have with Prettier. The [default rules are documented on their website](https://prettier.io/docs/en/options.html) and can be customized (a bit). We will take a look into it next but before I want to mention another flag: `--check`.

The `--check` flag, `npx prettier index.js --check`, is useful if you just want to check if a file (or the codebase with `.`) is Prettier compliant. It is useful for CIs and git hooks, for example, if you just want to warn the user (you can also enable `--write` in these scenarios).

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

You can configure Prettier to some extend. You can do it via the CLI or via a [configuration file](https://prettier.io/docs/en/configuration.html), which is more adequate. The configuration file could be in a variety of formats so you can choose the one that fits you best.

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

On top of that, you can also extend configuration and set up a few other things. Check their [configuration documentation](https://prettier.io/docs/en/configuration.html) for more details.

## ESLint

[ESLint](https://eslint.org/) has been around for a while. In short, it does a bit more than Prettier as it analyzes your code to find problems (or patterns that you don't want, like variables that are not used should be removed). Again, I invite you to read ESLint documentation if you want to go deeper into the topic. I like ESLint for the simple reason it helps me to find problems and configure some patterns in the project (it might be useful when onboarding new people). It is extremely extensible as well in case you're interested.

*At the time of this writing, ESLint is in version 7.32.0, so keep in mind things might change (especially formatting) in future versions. Version 8 is in beta at the moment.*

### How to set up ESLint?

In short, quite similar to Prettier but you need the configuration file. I will consider you have a project set up already so, in short, you need to install it:

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

This is simply the issue with a default ESLint configuration. It considers ES5 by default, so `const` is not allowed yet, and some older setup might make sense to your project but not in general.

We can spend hours configuring ESLint but in general, we get a default from a style guide (AirBnB for example) and apply it to our project. If you use the init command you can do so.

Let's install [Airbnb ESLint configuration](https://www.npmjs.com/package/eslint-config-airbnb-base), it also requires `eslint-plugin-import` to be installed (following their documentation) so:

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

ESLint allows you to extensively configure it if you want. It goes beyond the scope here and I will leave it up to you to explore and play with it: https://eslint.org/docs/user-guide/configuring/

## Prettier + ESLint

There are a lot of tutorials online on how to connect both. I want to take a different approach and try to reason about each tool and how they connect.

I will assume that we have the following Prettier configuration:

```json
// .prettierrc
{
  "semi": false
}
```

I will assume that we have the following ESLint configuration:

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

Not so cool! Running ESLint with `--fix` will fix these issues. Now if we run Prettier again we get:

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
It is fine if we want to ignore the rules from Prettier but in general, we want Prettier rules to override ESLint rules.
In case we delete the Prettier configuration file and use its defaults (which requires `;`), running Prettier check will result in:

```
Checking formatting...
[warn] index.js
[warn] Code style issues found in the above file(s). Forgot to run Prettier?
```

The file is not valid anymore as it is missing the `;` but the ESLint run won't fail, as the Prettier rules have been disabled when running ESLint.

One important thing to note here: the order used by `extends`, in the ESLint configuration, matters. If we use the following order we will get an error as AirBnB rules will override Prettier disabled rules when running ESLint:

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

We replaced `eslint-config-prettier` with `plugin:prettier/recommended`. Check ESLint docs about extending a plugin: https://eslint.org/docs/user-guide/configuring/configuration-files#using-a-configuration-from-a-plugin
I also recommend you to check what `eslint-plugin-prettier` is doing with our ESLint configuration: https://github.com/prettier/eslint-plugin-prettier/blob/a3d6a2259cbda7b2b4a843b6d641b298f03de5ad/eslint-plugin-prettier.js#L66-L75

Running ESLint again we will get:

```
1:12  error  Insert `;`  prettier/prettier
3:23  error  Insert `;`  prettier/prettier

✖ 2 problems (2 errors, 0 warnings)
  2 errors and 0 warnings potentially fixable with the `--fix` option.
```

Two things to note here:
1. We are getting `;` errors again, that have been disabled earlier with `eslint-config-prettier`;
2. The error is coming from the rule `prettier/prettier`, which is added by the plugin. All prettier validations will be reported as `prettier/prettier` rules.

## Typescript

Let's start from the very basic: running ESLint against TS files.
Right now, running ESLint against your codebase would be `npx eslint .`. That is fine until you want to run it against files that are not ending with `.js`.

Let's have these two files in our code base:

```javascript
// index.js
const a = 1
```

```typescript
// index.ts
const a = 1
```

Running `npx eslint .` we get:

```
1:7   error  'a' is assigned a value but never used  no-unused-vars
1:12  error  Insert `;`                              prettier/prettier

✖ 2 problems (2 errors, 0 warnings)
  1 error and 0 warnings potentially fixable with the `--fix` option.
```

It runs against our JS file but not our TS file. To run against TS files you need to add `--ext .js,.ts` to the ESLint command. By default, ESLint will only check for `.js` files.

Running `npx eslint . --ext .js,.ts`

```
/index.js
1:7   error  'a' is assigned a value but never used  no-unused-vars
1:12  error  Insert `;`                              prettier/prettier

/index.ts
1:7   error  'a' is assigned a value but never used  no-unused-vars
1:12  error  Insert `;`                              prettier/prettier

✖ 4 problems (4 errors, 0 warnings)
  2 errors and 0 warnings potentially fixable with the `--fix` option.
```

Working like a charm so far. Let's add some real TS code and run it again. The TS file will look like this:

```typescript
const a: number = 1
```

Running ESLint only against the `.ts` file:

```
1:8  error  Parsing error: Unexpected token :

✖ 1 problem (1 error, 0 warnings)
```

ESLint doesn't know, by default, how to parse Typescript files. It is a similar problem we faced when running ESLint for the first time with ES5 defaults.
ESLint has a configuration in which you can specify the parser you want to use. There is also a package, as you could imagine, that handles this parsing for us. It is called [`@typescript-eslint/parser`](https://github.com/typescript-eslint/typescript-eslint/tree/master/packages/parser).

Let's install it:

```bash
npm i @typescript-eslint/parser # --save-dev recommended
```

Now let's configure ESLint to use the new parser:

```javascript
module.exports = {
  parser: "@typescript-eslint/parser",
  extends: [
    'eslint-config-airbnb-base',
    'plugin:prettier/recommended',
  ]
};
```

Running ESLint again (`npx eslint index.ts`):

```
1:7   error  'a' is assigned a value but never used  no-unused-vars
1:20  error  Insert `;`                              prettier/prettier

✖ 2 problems (2 errors, 0 warnings)
  1 error and 0 warnings potentially fixable with the `--fix` option.
```

Cool! Now we can run ESLint on TS files. Nonetheless, we don't have any rules being used so we need to configure or use some styleguide, like the one we used by AirBnB before.
There is [`@typescript-eslint/eslint-plugin`](https://github.com/typescript-eslint/typescript-eslint/tree/master/packages/eslint-plugin) that offers us some defaults. Let's go with it for now:

```bash
npm i @typescript-eslint/eslint-plugin # --save-dev recommended
```

Adding it to our configuration:

```javascript
module.exports = {
  parser: "@typescript-eslint/parser",
  extends: [
    'eslint-config-airbnb-base',
    'plugin:@typescript-eslint/recommended',
    'plugin:prettier/recommended',
  ]
};
```

Now running `npx eslint index.ts`:

```
1:7   error    Type number trivially inferred from a number literal, remove type annotation  @typescript-eslint/no-inferrable-types
1:7   warning  'a' is assigned a value but never used                                        @typescript-eslint/no-unused-vars
1:20  error    Insert `;`                                                                    prettier/prettier

✖ 3 problems (2 errors, 1 warning)
  2 errors and 0 warnings potentially fixable with the `--fix` option.
```

Cool! Now we have also proper linting in our Typescript file. We can also see that the Prettier rule still applies as expected.

Bear in mind that `typescript-eslint` is overriding `eslint-config-airbnb-base` in this case. It means that some rules won't work in TS files that are still valid on JS files. Let's have the files below to see it in action:

```javascript
// index.js and index.ts
const a = 1;
a = 2;
```

Both files are identical. Running `npx eslint . --ext .js,.ts` we get:

```
/index.js
  2:1  error    'a' is constant                         no-const-assign
  2:1  warning  'a' is assigned a value but never used  @typescript-eslint/no-unused-vars

/index.ts
  2:1  warning  'a' is assigned a value but never used  @typescript-eslint/no-unused-vars

✖ 3 problems (1 error, 2 warnings)
```

The [`no-const-assign` rule is overwritten by `typescript-eslint` for `.ts` files](https://github.com/typescript-eslint/typescript-eslint/blob/1c1b572c3000d72cfe665b7afbada0ec415e7855/packages/eslint-plugin/src/configs/eslint-recommended.ts#L13) so we don't get the same error for both files.
To overcome it, we need to change the order of the extended configurations, `typescript-eslint` comes first and `eslint-config-airbnb-base` next. If we do so:

```javascript
module.exports = {
  parser: "@typescript-eslint/parser",
  extends: [
    "plugin:@typescript-eslint/recommended",
    "eslint-config-airbnb-base",
    "plugin:prettier/recommended"
  ]
};
```

Running `npx eslint . --ext .js,.ts`:

```
/index.js
  2:1  error    'a' is constant                         no-const-assign
  2:1  error    'a' is assigned a value but never used  no-unused-vars
  2:1  warning  'a' is assigned a value but never used  @typescript-eslint/no-unused-vars

/index.ts
  2:1  error    'a' is constant                         no-const-assign
  2:1  error    'a' is assigned a value but never used  no-unused-vars
  2:1  warning  'a' is assigned a value but never used  @typescript-eslint/no-unused-vars

✖ 6 problems (4 errors, 2 warnings)
```

Cool! Now we get the same error for both files.

*One side note: In this example, I have a codebase with JS/TS, it might not be your case and you might also use another style guide where conflicts won't happen.*

## That's all folks!

I hope this article helped you to learn or clarify some concepts behind ESLint, Prettier, and Typescript playing together.

In short, you've to understand which files ESLint will analyze and the order of the configurations you want. Image adding now this into a Vue project, for example, you need to add `.vue` to `--ext .js,.ts,.vue` and add (or configure) some style guide which will add some rules to your project.

Most boilerplates will already have some lint set up and you will mostly disable some rules but in case you want to customize it or update packages (especially major bumps), it is important to understand how to perform the changes and the impacts it might have in your project.

That's is all! Happy linting!
