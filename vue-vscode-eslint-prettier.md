*published on October 3rd, 2020*

# Vue.js, VSCode, ESLint and Prettier - [Why can't we be friends?](https://youtu.be/nmhgi665Oek?t=165)

## Wait but why?

You might wonder "There's already a good amount of articles (and even videos) out there explaining how to use ESLint and Prettier with Vue.js, why write yet another one?". It's a legitimate question. I went through some of them, and while there are a couple of good ones, in my opinion they all fall short in one or more things that I think any good article or tutorial should respect:

- start from a clean slate and state the exact versions of the tools used throughout;
- be up to date - too many developers taking up a new technology are discouraged by struggling with outdated articles;
- explain **how** things work and **why** you should do something, instead of showing pieces of code or configuration that you should just copy/paste.

If this article ends up helping (or frustrating) you in any way, I'd appreciate if you took a minute to leave me some [feedback below](#feedback-is-50).

## Life goals

So what can you expect from this article? Hopefully it will check all the boxes above, plus the ones below:

- have a consistent (and pleasant!) `eslint` linting experience in VSCode, `vue-cli-service` and on the command line, while working with a Vue.js project;
- use `prettier` to auto-format code on save or from the command line;
- make `eslint` and `prettier` behave like friends, not enemies fighting over your code.

I know this all sounds very ambitious (especially the last one!), but I promise we'll get there in the end!

We're gonna follow a hands-on approach, first playing around to get a feel of the behavior, and then digging into why and how things work behind the covers.

## Getting our feet wet

A quick list of the tools and versions I'm using for this article:

- Visual Studio Code 1.49.2
- VSCode extensions:
  - Vetur 0.28.0
  - ESLint 2.1.8
- @vue/cli 4.5.6 (used to create a simple app project)

I'm also using Git Bash from [Git for Windows](https://git-scm.com/download/win) as a terminal, because, well, I'm old and grumpy and still using Windows.

We'll start with a very simple Vue app created by running `vue create vue-simple-app` with the following settings:

```
Vue CLI v4.5.6
? Please pick a preset: Manually select features
? Check the features needed for your project: Choose Vue version, Babel, Linter
? Choose a version of Vue.js that you want to start the project with 2.x
? Pick a linter / formatter config: Basic
? Pick additional lint features: Lint on save
? Where do you prefer placing config for Babel, ESLint, etc.? In dedicated config files
? Save this as a preset for future projects? No
```

You can also get the exact app I'm using throughout this article from this [GitHub repository](https://github.com/lbogdan/vue-simple-app).

## First achievement: VSCode linting expert!

If you're already using VSCode, you can use this trick to start it with a clean slate while keeping your current settings untouched:

```sh
# run this in a *sh (ash, bash etc.) terminal
# create a new profile folder
$ mkdir -p ~/.vscode-vue/extensions
# define an alias to start VSCode using the new profile
$ alias code-vue="code --user-data-dir ~/.vscode-vue --extensions-dir ~/.vscode-vue/extensions"
$ code-vue
```

Just for reference, for this article I'm using the following minimal VSCode configuration:
```json
{
  "telemetry.enableCrashReporter": false,
  "telemetry.enableTelemetry": false,
  "workbench.list.openMode": "doubleClick",
  "terminal.integrated.shell.windows": "C:\\Program Files\\Git\\bin\\bash.exe",
  "editor.tabSize": 2,
  "files.eol": "\n",
  "terminal.integrated.rendererType": "experimentalWebgl",
  "editor.rulers": [
    80,
    120
  ],
  "editor.renderWhitespace": "all"
}
```

To open VSCode's settings as JSON, press Ctrl+Shift+P to open the Command Palette, and search for "Preferences: Open Settings (JSON)".

I also (only) have `Vetur` and `ESLint` extensions installed.

Let's open the app folder in VSCode and open the `src/App.vue` file. We should see a dialog asking us if we want to use our project's local `eslint` version from `node_modules`:

![](/images/vue-vscode-eslint-prettier/1.png)

This allows us to use a different version of `eslint` per project, so of course we're gonna allow it!

Let's now see what happens if we make the most common (and probably the most hated, too) code inaccuracy (because it's not really a mistake) - declaring an unused variable: add `const a = 10` on a new line right under the `import`. We can immediately see a red squiggly under `a`, signaling that there's something wrong with our code. Hovering over it we get:

![](/images/vue-vscode-eslint-prettier/2.png)

From the text "eslint(no-unused-vars)" we can deduce it's an `eslint` error because we're breaking the `no-unused-vars` rule. If a rule sounds weird to you (they usually do!) and want to know more about it you can click on "Quick Fix" -> "Show documentation for ...". Doing that in our case gets us to the [`no-unused-vars` rule documentation page](https://eslint.org/docs/rules/no-unused-vars).

Let's now check that linting also works inside the `<template>` block: add `<div v-for="i in 5">{{ i }}</div>` on a new line right under `<HelloWorld>`, we should see another red squiggly and on hover:

![](/images/vue-vscode-eslint-prettier/3.png)

I'll leave it to you to figure out what that [error](https://eslint.vuejs.org/rules/require-v-for-key.html) is about.

You've probably already noticed that we get the error twice and thought "What's up with that?!". One thing that it's not immediately obvious is that [`vetur` also validates](https://github.com/vuejs/vetur/blob/master/docs/linting-error.md) the `<template>`, `<script>` and `<style>` blocks in SFCs, and we end up getting the same error twice - one from `eslint` itself, and the other from `vetur`, using `eslint-plugin-vue` to validate the `<template>` block. This happens for historical reasons, back when `eslint` didn't know how to directly parse and validate `.vue` files, and it was `vetur`'s responsibility to parse them and run `eslint` separately for the found `<template>` and `<script>` blocks. Because `eslint` now supports parsing `.vue` files and validating `<template>` and `<script>` blocks, we should disable this redundant checking by `vetur`:

```jsonc
// .vscode/settings.json
{
  "vetur.validation.script": false,
  "vetur.validation.template": false
}
```

I usually place this project-dependent settings in the VSCode's Workspace config, that's stored in the `.vscode/settings.json` file, in the project folder. I also version them.

We'll still keep `vetur`'s `<style>` validation for now, which uses [VSCode's built-in CSS validation](https://code.visualstudio.com/docs/languages/css#_syntax-verification-linting). You can disable it using `"vetur.validation.style": false`.

If we did everything right, we should now see the previous error only once:

![](/images/vue-vscode-eslint-prettier/4.png)

And so we get our first achievement unlocked: VSCode linting expert! Actually, there's three more ways to lint your code that should now work out-of-the-box. We'll take a look at them next.

## Linting from the terminal

You can manually lint all your project files by running `yarn lint` (or `npm run lint`, if using `npm`) inside your project folder. Make sure you saved the two changes we did to `App.vue` and try it:

```sh
$ yarn lint
yarn run v1.22.5
$ vue-cli-service lint
error: Elements in iteration expect to have 'v-bind:key' directives (vue/require-v-for-key) at src\App.vue:5:5:
  3 |     <img alt="Vue logo" src="./assets/logo.png">
  4 |     <HelloWorld msg="Welcome to Your Vue.js App"/>
> 5 |     <div v-for="n in 5">{{ n }}</div>
    |     ^
  6 |   </div>
  7 | </template>
  8 |


error: 'a' is assigned a value but never used (no-unused-vars) at src\App.vue:11:7:
   9 | <script>
  10 | import HelloWorld from './components/HelloWorld.vue'
> 11 | const a = 10
     |       ^
  12 |
  13 | export default {
  14 |   name: 'App',


2 errors found.
error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
```

This is very useful to be run as a commit hook so that teams can stay confident that all committed code is consistent with the agreed-upon linting rules. It can also be used to auto-fix the whole project codebase at once. We'll talk more about auto-fixing in a bit.

## Linting during development

Let's now try to run the dev-server. First undo the two `App.vue` changes we did earlier, save and run`yarn serve`. The dev-server should start and the app should be accessible at `http://localhost:8080/`. Open it in a browser and let's reintroduce the `const a = 10` inaccuracy. You'll notice we get an error from the dev-server:

```sh
 ERROR  Failed to compile with 1 errors                                            18:17:04
 error  in ./src/App.vue

Module Error (from ./node_modules/eslint-loader/index.js):

D:\work\vue-simple-app\src\App.vue
  10:7  error  'a' is assigned a value but never used  no-unused-vars

✖ 1 problem (1 error, 0 warnings)


 @ ./src/main.js 6:0-28 10:13-16
 @ multi (webpack)-dev-server/client?http://192.168.1.16:8080&sockPath=/sockjs-node (webpack)/hot/dev-server.js ./src/main.js
```

and a similar error overlay in the browser:

![](/images/vue-vscode-eslint-prettier/5.png)

If we now remove the error and save, everything should go back to normal: the app compiles and the error overlay goes away.

This usually annoys the hell out of people (and eventually make them hate `eslint` and never use it again in their life, before giving up on web development entirely) when e.g. commenting out a piece of code and getting lots of `no-unused-vars` errors and that dreaded error overlay, so they now have to find and comment all those unused declarations before things go back to normal and they can continue coding away. In my opinion, this is an awful first-time developer experience. We'll see how we can elegantly tackle this soon, but until then, if you really want to feel better and disable `eslint` in the dev-server, create a `vue.config.js` (if not already there) in the project folder and set [`lintOnSave`](https://cli.vuejs.org/config/#lintonsave) like this:

```js
// vue.config.js
module.exports = {
  // ignore linting errors while in development
  // fail production build if there are linting errors
  lintOnSave: process.env.NODE_ENV === 'production' ? 'error' : false
}
```

This will only disable linting while in development, as I think it's still valuable to have it enabled when building a production bundle.

## Linting when building a production bundle

By default (and with the `lintOnSave` configuration from the previous section), running a build with `yarn build` fails if there are any linting errors:

```sh
$ yarn build
yarn run v1.22.5
$ vue-cli-service build
$ D:\work\vue-simple-app\node_modules\.bin\vue-cli-service build

/  Building for production...

 ERROR  Failed to compile with 1 errors                                                     18:49:57
 error  in ./src/App.vue

Module Error (from ./node_modules/eslint-loader/index.js):

D:\work\vue-simple-app\src\App.vue
  10:7  error  'a' is assigned a value but never used  no-unused-vars

✖ 1 problem (1 error, 0 warnings)


 @ ./src/main.js 6:0-28 10:13-16
 @ multi ./src/main.js

 ERROR  Build failed with errors.
error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
```

This is useful in CI, it allows us to do the linting and building in the same step vs. having an additional linting step.

## "Do you have this in another color?"

Now that we saw how `eslint` is supposed to behave with a default Vue CLI configuration, let's see how we can bend it to our will. While `eslint` supports a [handful of configuration file types](https://eslint.org/docs/user-guide/configuring#configuration-file-formats), Vue CLI uses `.eslintrc.js`, which by default has the following contents:

```js
// default .eslintrc.js
module.exports = {
  root: true,
  env: {
    node: true
  },
  'extends': [
    'plugin:vue/essential',
    'eslint:recommended'
  ],
  parserOptions: {
    parser: 'babel-eslint'
  },
  rules: {
    'no-console': process.env.NODE_ENV === 'production' ? 'warn' : 'off',
    'no-debugger': process.env.NODE_ENV === 'production' ? 'warn' : 'off'
  }
}
```

Why a `.js` config? Because it allows us to programmatically generate the config. This can be very useful if we want to enable, disable or change rules config depending on the environment, like we see above. We can also add comments explaining certain decisions.

The most important part of the `eslint` config - the only one we'll be fiddling with - is the linting rules. Instead of listing all the rules in our project's config, we can extend preset configs carefully crafted by people with more experience. We can see the default Vue CLI `eslint` config extends two of such presets: [`node_modules/eslint-plugin-vue/lib/config/essential.js`](https://unpkg.com/browse/eslint-plugin-vue@6.2.2/lib/configs/essential.js) and [`node_modules/eslint/conf/eslint-recommended.js`](https://unpkg.com/browse/eslint@6.8.0/conf/eslint-recommended.js). If we want to customize these presets, we can override the rules config within the `rules: { ... }` object. We can see in the default config the rules `no-console` and `no-debugger` are turned off in development, and configured to generate a warning when building for production.

> **Note:** by default, the production build will still fail if we use any `console.log()` or `debugger` statements in our code, even if they're configured as warnings.

We can check the list of all available `eslint` rules [here](https://eslint.org/docs/rules/) and of Vue.js (added by the `eslint-plugin-vue` package) [here](https://eslint.vuejs.org/rules/).

Following the configs and presets trying to figure out which `eslint` rules are active at any given moment is quite tedious (not to say error-prone). The good news is there's an `eslint` helper command that allows us to see the exact config `eslint` uses for linting a certain file: `yarn eslint --print-config filename`. Let's check which Vue.js rules are currently enabled while linting our `App.vue` file:

```sh
# Vue.js rules begin with 'vue/'
# grep -A2 shows two more line after the matched one
$ yarn eslint --print-config src/App.vue | grep 'vue/' -A2
    "vue/no-async-in-computed-properties": [
      "error"
    ],
    "vue/no-dupe-keys": [
      "error"
    ],
[...]
```

## Fix me, baby, one more time!

As I said earlier, some linting rules can be auto-fixed. To see how this works, let's clear all errors from `App.vue` and add a new line right under `import` - `const re = new RegExp(/  /)`. You'll notice that besides `no-unused-vars` we get a new error: `no-regex-spaces`. If we hover over it and click on "Quick Fix..." we see a new menu option "Fix this no-regex-spaces problem":

![](/images/vue-vscode-eslint-prettier/6.png)

If we click on it the error goes away and our newly added code is replaced with `const re = new RegExp(/ {2}/)`. Magic! There aren't that many rules that support auto-fixing, but this will come in handy when meeting `prettier` soon.

To go a step further, because clicking on those small links and menus to fix things is quite annoying, we can configure VSCode to do this for us every time we save:

```jsonc
// .vscode/settings.json
{
  // ...
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  }
}
```

Now if we try to re-add the initial code and save, VSCode should fix all auto-fixable rules for us. More magic!

## Some better defaults, maybe?

Let's now try to make the whole `eslint` experience more enjoyable (or at least less frustrating). The idea is to make common rules, which might trigger while you're editing code - like `no-console`, `no-unused-vars` etc. - be warnings (which show with yellow squiggles in VSCode, and also don't break your development builds, like errors do), or even completely disable them, if you prefer:

```js
// .eslintrc.js
function disableInDevelopment(rules) {
  return rules.reduce((acc, rule) => {
    // warn in development, error in production
    // to completely disable in development, use 'off' instead of 'warn'
    acc[rule] = process.env.NODE_ENV === 'production' ? 'error' : 'warn';
    return acc;
  }, {});
}

module.exports = {
  root: true,
  env: {
    node: true,
  },
  extends: ['plugin:vue/recommended', 'eslint:recommended'],
  parserOptions: {
    parser: 'babel-eslint',
  },
  rules: {
    // add other annoying rules to the array below
    ...disableInDevelopment(['no-console', 'no-debugger', 'no-unused-vars']),
    // other custom rules here
  },
};
```

We're using [`plugin:vue/recommended`](https://unpkg.com/browse/eslint-plugin-vue@6.2.2/lib/configs/recommended.js) here, which is a more opinionated preset than `plugin:vue/essential`, but you can change it back if it's too opinionated for you. Otherwise, we can already see some new warnings in `src/App.vue` and `src/components/HelloWorld.vue` - a few template ones and this:

![](/images/vue-vscode-eslint-prettier/8.png)

which can be fixed by:

```js
// src/components/HelloWorld.vue
// ...
<script>
export default {
  name: 'HelloWorld',
  props: {
    msg: {
      type: String,
      default: ''
    }
  }
}
</script>
// ...
```

and now if we save, the template warnings are also auto-fixed. Even more magic!

## Exceptions, exceptions...

We have a saying in Romania, "It's the exception that confirms the rule". So you have a nice `eslint` config, everything is good in the world, but suddenly you stumble over an exception to one of linting rules. What to do? Should you disable the rule for your whole codebase? Of course not! `eslint` offers ways to temporarily disable one or all rules for a specific piece of code by adding [special-syntax comments](https://eslint.org/docs/user-guide/configuring#using-configuration-comments). Because I love explicitness, the one I use the most is `eslint-disable-next-line rule-name`, and it works like this:

```js
// we really, really, REALLY want to declare a and not use it anywhere!
// eslint-disable-next-line no-unused-vars
const a = 10;
```

and inside the `<template>`:

```html
    <!-- if there's no lock, why do we need a key?! -->
    <!-- eslint-disable-line vue/require-v-for-key -->
    <div v-for="i in 5">{{ i }}</div>
```

> **Note:** this is only for demonstration purposes, you should **always use a key with `v-for`**!

You should not over-use this - if you find yourself locally disabling the same rule over and over again, it's probably wiser to just disable it globally in the `eslint` config.

## Introducing Prettier - your handsome code pal

So we now have a working `eslint` config perfectly catered to our needs. Do we also need `prettier`? Some developers argue `eslint` already has formatting rules, so you don't really need yet another code formatting tool. While the first part is true, `eslint`'s formatting capabilities are quite limited - for example, `eslint` will never reflow your code if one line exceeds a number of characters, or even touch `<template>` formatting.

This is where `prettier` comes in. It is a very (and I mean extremely!) opinionated code formatting tool that nicely reflows your code based on its opinionated defaults and a handful of [configuration options](https://prettier.io/docs/en/options.html). This should lower your cognitive overhead while writing code, never having to ask yourself "Should I put a trailing comma here? What about a semicolon there?" ever again, while also ensuring formatting consistency across your codebase.

This leads to an endless source of developer frustration - trying to setup `prettier` separately from `eslint`. Because `eslint` has its own formatting rules, what usually happens is that `prettier` formats your code, `eslint` sees the changes as auto-fixable errors, but fixing them upsets `prettier`, which will format your code again, and so on, and so forth. This is the point at which developers just give up and decide to never use `prettier` alongside `eslint`, as they can't coexist together.

So here's a brilliant idea: what if instead of running `eslint` and `prettier` as separate tools, we could use them together? That's how `eslint-plugin-prettier` was born, a plugin that adds a new `prettier/prettier` rule to `eslint` which will run `prettier` on your code and transform the differences into auto-fixable `eslint` warnings. This is usually used together with `eslint-config-prettier`, which disables all `eslint` formatting rules that could conflict with `prettier` formatting.

This might sound a bit confusing, so let's see an example:

![](/images/vue-vscode-eslint-prettier/7.png)

Here we messed a bit with the `components: {...}` formatting, and we see `eslint` giving us an auto-fixable `prettier/prettier` warning saying we should insert two spaces in front of `HelloWorld` to fix formatting. Now, if we have auto-fix on save enabled, we can just save and our code goes back to being all nice and cute and cuddly.

To add `prettier` to our simple Vue.js app, we first have to install the needed packages - `yarn add --dev prettier eslint-plugin-prettier @vue/eslint-config-prettier`, and then tell `eslint` to use them, by adding `@vue/prettier` to the `extends: [...]` array in our `.eslintrc.js`:

```js
// .eslintrc.js
module.exports = {
  // ...
  'extends': [
    'plugin:vue/recommended', 'eslint:recommended', '@vue/prettier'],
  // ...
}
```

Yes, it's that simple! Just one more extra-step to set some `prettier` options by creating a `.prettierrc` file inside our projects' folder (these are just my personal preferences):

```jsonc
// .prettierrc
{
  "printWidth": 120,
  "semi": true,
  "singleQuote": true,
  "trailingComma": "es5"
}
```

and we can now re-format our whole project according to these options by running `yarn lint`:

```sh
$ yarn lint
yarn run v1.22.5
$ vue-cli-service lint
The following files have been auto-fixed:

  src\App.vue
  src\components\HelloWorld.vue
  src\main.js
  .eslintrc.js
  babel.config.js

 DONE  All lint errors auto-fixed.
Done in 1.59s.
```

> **Note:** `yarn lint` fixes all `eslint` auto-fixable rules by default, if that's not what you want, you can use `yarn lint --no-fix`.

And with this last bit of magic, we're done!

You can check out the final version of our simple app with all the configs in the [final branch](https://github.com/lbogdan/vue-simple-app/tree/final).

## Feedback is 50

If you have questions, or any kind of feedback, you can either [open an issue](https://github.com/lbogdan/ramblings/issues/new) or find me on the [Vue.js's official Discord server](https://chat.vuejs.org/) - I'm `BogdanL` there.
