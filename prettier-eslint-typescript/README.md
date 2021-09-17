# Prettier, ESLint and Typescript

I decided to write this article to sumup a struggle of mine. We've started a new project in the company, Prettier was setup, ESLint was setup and at some point we added Typescript. By the end Typescript was also setup. CI was linting, commit hooks were also linting, VSCode was fixing the code and so on (that is what I thought).
At some point I was playing with the project and realized that some files were being warned by my editor but not when running the linter (`npm run lint` in my case). I got triggered. I've a hard time accepting that something works but I can't understand, unless it is an absolutely external tool that I didn't have to setup myself but that was not the case here.

In this article I will summarize some understandings that I've about integrating all the tools above. The main focus is how to setup Prettier, how to setup ESLint, how to integrate both and by the end how to add Typescript to it.

## Prettier

The first tool I want to explore is [Prettier](https://prettier.io/). I would leave it to you to read more about what it is but in short it is a code formatter. What does it mean? It means that it will keep your codebase consistent (in terms of coding style). Do you use `;`? If yes, it will ensure that all your files have it, for example. I like it for two reasons: we barely have to discuss code formatting and it is easy to onboard new members to the team.

*At the time of this writing, Prettier is in version 2.4.1, so keep in mind things might change (especially formatting) in future versions.*

## How to setup Prettier?

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
