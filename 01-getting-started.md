# React - Part 1 - Getting Started

## Introduction

Welcome to building a web application using [React](https://reactjs.org/)!

The target audience of this series of posts are people who have some prior experience using React, and ideally some experience using a state management library such as Redux or MobX. The main aim is to show an opinionated way that a non-toy application can be built.

During the series, we're going to building a site that will let users create and vote on polls. The site will call to an existing API server hosted at https://purpoll.azurewebsites.net which will mean we don't have to worry about setting it up ourselves.

## Before we start

Before we can start the real work of building the site, we need to make sure we have the appropriate tools to do the job.

- Install or update to the latest stable version of [Node](https://nodejs.org/)
  - Direct downloads are [available here](https://nodejs.org/en/download/)
  - Make sure your npm directory is added to your PATH so it's accessible via command line. You can test if it's working using `npm -v` to see the NPM version that's installed.
- Update to the latest NPM
  - `npm i -g npm@latest`
- Make sure you have an IDE or text editor installed and up to date
  - If in doubt, [Visual Studio Code](https://code.visualstudio.com/) is a great option.

### Optional

- React DevTools Browser Extension for [Chrome](https://chrome.google.com/webstore/detail/react-developer-tools/fmkadmapgofadopljbjfkapdkoienihi) or [Firefox](https://addons.mozilla.org/en-US/firefox/addon/react-devtools). The React DevTools should "just work" when you're developing a React application. It has useful tabs for see the current props and state of components, and finds what's actually rendered. It also has a performance profiler to help track down issues if anything happens to run slowly.

## Create the new project

The first step is to create the new React project. To do this we'll be using [`create-react-app`](https://create-react-app.dev), which is an opinionated way to structure a React application development environment.

> It's likely that many readers will have used `create-react-app` before, but we'll run through the process for completeness.

While being opinionated means that possible options may not be available, it also means that we don't have to deal with the details of the WebPack/Babel build pipeline. Additionally, the maintainers of this project work hard to make updating projects to new React versions as easy as possible, which is **very valuable** for projects that are going to be supported over the medium to long term.

[Run the commands below](https://create-react-app.dev/docs/getting-started) in your favourite terminal. Our new project will be called **"repoll"**.

> Note we are explicitly saying to use `npm` - if you'd prefer to use `yarn` and have it installed, you can remove that option. We are also going to [use TypeScript](https://create-react-app.dev/docs/adding-typescript), because why wouldn't you? 😉

```cmd
npx create-react-app@latest repoll --use-npm --template typescript
```

The [React docs](https://create-react-app.dev/docs/folder-structure) give a brief overview of the files that are generated.

Now check that it works by changing the current directory to the `repoll` folder and using `npm start`, which should display the new-generated web app in your default browser. You can also run the generated test using `npm test`.

> The available npm commands and what they do is described in the generated README.md file.

Note that `create-react-app` has also initialised the project as a git repository with a single commit containing the generated files.

## Git Attributes

Add a `.gitattributes` file to the root of the project with the following contents, so that line endings are treated the Unix way. Even on Windows any text editor/IDE should be fine with this.

```
* text=auto eol=lf
*.{cmd,[cC][mM][dD]} text eol=crlf
*.{bat,[bB][aA][tT]} text eol=crlf
```

> This code is taken from [this VSCode documentation](https://code.visualstudio.com/docs/remote/troubleshooting#_resolving-git-line-ending-issues-in-containers-resulting-in-many-modified-files) about sharing repos between Windows and Linux.

You can run `git add --renormalize .` to apply "renormalisation" to the already committed files. If any changes are staged they can then be committed to get everything consistent.

## Tweaking the generated app

The generated application does many things automatically for you, and generally has sensible defaults, but there are a few changes that I suggest are a good idea.

### Stop auto-opening a browser window

The default behaviour of `create-react-app` is to open a browser whenever `npm start` is run. This is great when you've not used `create-react-app` before, but personally I find it gets annoying as I'm stopping and starting the react dev server often and don't need a new browser tab each time.

To change this, add a new `.env` file to the root (`/repoll`) directory with the following contents:

```txt
BROWSER=none
```

The `.env` file sets environment variables to default values. [See the docs](https://create-react-app.dev/docs/adding-custom-environment-variables) for extensive details about how to set environment variables and how to use them within the app.

### Update dependencies

The generated project may not use the latest versions of all of its dependencies. But as we're just starting out, now's the time to use the latest and greatest!

Run `npm outdated` to see what newer versions are available.

To update everything to the latest versions, run the following:

```
npx npm-check-updates -u
npm install
```

> `npm-check-updates` is an independent package - it also has [many additional options](https://www.npmjs.com/package/npm-check-updates) if the command above is not suitable.

### Styling

The default project uses [regular css for styling](https://create-react-app.dev/docs/adding-a-stylesheet). We're going to change this to use [Css Modules](https://create-react-app.dev/docs/adding-a-css-modules-stylesheet) with [Sass](https://create-react-app.dev/docs/adding-a-sass-stylesheet). For this tutorial, we'll be going with the following:

- CSS Modules over global styles
- Preferring the use of React Components to apply styles
- SASS rather than plain CSS
- CSS Properties/Variables over SASS variables for colours

> _Why?_
>
> - [CSS Modules](https://create-react-app.dev/docs/adding-a-css-modules-stylesheet) automatically makes unique style class names, and allows styles to be imported as an object which may improve editor intellisense.
> - Using components (eg a `<Button>` rather than `<button className="button">`) means that global style names do not need to be remembered, and modifications (such as having a "primary" or "alternate" button) can be added as optional props, and discovered using intellisense.
> - Using SASS is a simple compile-time addition to the project. While many of SASS's features are not needed, some are still useful to use from time to time, such as to avoid repetition.
> - [CSS Properties](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties) can have their values declared in your typescript code, rather than in your SASS, and are scoped to the element that they're applied to and its children. This makes it really simple to change their values at runtime to apply things like theming. It does mean that some SASS features (like `darken`) cannot be used however (but there are usually suitable alternatives).

Css Modules are automatically supported, but for Sass we need to install a library to help. If you follow [the standard `create-react-app` docs](https://create-react-app.dev/docs/adding-a-sass-stylesheet/), they will suggest installing `node-sass`. However, [currently `node-sass` does not support the newer features of SASS, such as @use](https://github.com/sass/node-sass/issues/2886). Instead, [`create-react-app` now has support for `dart-sass`](https://github.com/facebook/create-react-app/issues/5282) and so we can use a [js implmentation of it](https://www.npmjs.com/package/sass).

> There are some trade-offs involved in this choice of which sass library to use: `sass` is more up-to-date with features, and is js-only with fewer dependencies, while `node-sass` uses a C++ implmentation, so it is faster to compile.

We want to use the latest SASS features [like `@use`](https://sass-lang.com/documentation/at-rules/use), so run the following:

```cmd
npm i sass
```

> You may also find that your editor has a Css Modules extension. For VSCode, there's the [CSS Modules](https://marketplace.visualstudio.com/items?itemName=clinyong.vscode-css-modules) extension that can help with intellisense and go-to-definition.

Now we'll update some of the existing pages to use Css Modules and Sass.

- Change the main index.css file to use Sass:
  - Rename `src/index.css` to `src/index.scss`. Having this one non-module css file gives a convenient location for apply style resets and any custom global styles (of which there should not be many as we'll be using React components!)
  - Change `src/index.tsx` to update the import of the style file to match the new name
- Change the App.css to use Css Modules
  - Rename `src/App.css` to `src/App.module.scss`
  - Update the styles within the file
    - change the `App` class to `root` (an optional change, but a useful convention for the style that will be applied to the root element)
    - remove the `App-` prefix from the remaining classes (as Modules automatically unique-ifies the names for us)
  - Update the existing css import in `src/App.tsx` to be:

```ts
import styles from './App.module.scss';
```

When using modules, styles are imported and used directly, rather than specifying a "magic string". So update the `className` attributes in the `App.tsx` file like the following:

- `className="App"` -> `className={styles.root}`
- `className="App-header"` -> `className={styles.header}`
- `className="App-logo"` -> `className={styles.logo}`
- `className="App-link"` -> `className={styles.link}`

If you installed a Css Modules editor extension, hopefully you had some intellisense on the `styles` object.

Note that when a class name is a single word, you can use dot-notation from the `styles` instance. If you have a name like `foo-bar`, you need to use an indexer `styles['foo-bar']`.

`create-react-app` also contains [PostCSS Normalize, that can be opted-into if desired](https://create-react-app.dev/docs/adding-css-reset/) by adding `@import-normalize;` to the top of `index.scss`.

> Note that in VSCode, you may get a warning about having an "unknown at rule" - this can be ignored, or turned off in VSCode's SCSS settings.

### Enable Absolute Imports

By default relative imports are used in the application. However absolute imports often can allow for more easy refactoring in future.

Personally, I recommend using absolute imports in preference to relative imports when "going up" (ie. `../`) would be required. This usually means that what's being imported is from a different area of the system. If I decide later to refactor this file into a subfolder for example, the absolute import path will not need to change.

To enable absolute imports, update `tsconfig.json` to include `baseUrl` in `compilerOptions`:

```json
{
  "compilerOptions": {
    ...
    "baseUrl": "src"
  }
  ...
}
```

To import a file at `/repoll/src/foo/bar`, you can now use `import Bar from 'foo/bar';`

### Prettier

Adding [Prettier](https://prettier.io/) to the project is optional, but personally I find it a good idea - handling the code layout and standards automatically can add consistency that's difficult to maintain manually.

Install Prettier from NPM:

```cmd
npm install prettier
```

Prettier is quite opinionated and [its options are deliberately kept sparse](https://prettier.io/docs/en/options.html). However I usually do customise them from the defaults. Add the following as `.prettierrc` in the root to configure the options, but feel free to choose the settings you wish.

```json
{
  "printWidth": 100,
  "singleQuote": true,
  "jsxBracketSameLine": true,
  "overrides": [
    {
      "files": ".prettierrc",
      "options": { "parser": "json" }
    }
  ]
}
```

It can be applied to all the project files at once to bring everything up to date. Add the following line to the `scripts` of `package.json`:

```json
"format": "prettier --write ."
```

> The files that this command formats [can be customised using glob patterns](https://prettier.io/docs/en/cli.html) if desired.

Now execute `npm run format` which should cause several files to be updated.

It's generally a good idea to add a Prettier Plugin to your IDE of choice so that changes are applied on every save. [Here's one for VSCode](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode).

`create-react-app` also [suggests applying Prettier as part of every commit into Git](https://create-react-app.dev/docs/setting-up-your-editor/#formatting-code-automatically) which can be implemented if desired.

## Example Code Tidy-up

Much of the example application will not be required, so we'll remove and restructure it a little to get ready for adding functionality we actually want!

Make the following changes:

- Delete `logo.svg`
- Move `App.tsx`, `App.test.tsx`, `App.module.scss` into a new `views` subfolder
- Update `App.module.scss` to:

```scss
.root {
  // Make the App be at least the full page (or longer) and disable margin-collapsing
  display: grid;
  min-height: 100vh;
}
```

> Setting `display: grid` disables [margin-collapsing](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Box_Model/Mastering_margin_collapsing), so if the app contents has a top margin set (such as a default `h1`), it will not cause the size of `App` to immediately exceed `100vh` and show a scrollbar.

- Update `App.tsx` to:

```tsx
import React from 'react';
import styles from './App.module.scss';

export const App: React.FC = function () {
  return <div className={styles.root}>App</div>;
};
```

- Update `index.tsx` to change the App component import to `import { App } from './views/App';`
- In `index.tsx`, you can delete the import of `serviceWorker` and the code that uses it, as well as the `/src/serviceWorker.ts` file itself.
  - The service workers code [available by default in `create-react-app`](https://create-react-app.dev/docs/making-a-progressive-web-app/) enables effective caching of the site's static assets. The trade-offs involved with this may or may not be appropriate for your particular use case, so we're going to leave it out to keep things a little simpler.

You should now have a folder structure that looks like this:

```txt
src
  - views
    - App.tsx
    - App.module.scss
    - App.test.tsx
  - index.scss
  - index.tsx
  - react-app-env.d.ts
  - setupTests.ts
```

> The idea of the `views` folder is to contain the "view logic" of the application (as opposed to "data logic"). This should be **the main location for any React code** - as React is primarily a view library. In future steps we'll be adding a place for the data logic code.

Create React App uses [Jest](https://jestjs.io/) as its test runner. Jest is a Node-based runner meaning that the tests always run in a Node environment and not in a real browser. The initial test that comes with Create React App looks for a Link element on the home page, but previously we removed all the elements from that page which will now cause the test to fail. To ensure the test still works, make the following changes to `App.test.tsx`:

```tsx
import React from 'react';
import { App } from './App';
import ReactDOM from 'react-dom';

test('renders without crashing', () => {
  const div = document.createElement('div');
  ReactDOM.render(<App />, div);
});
```

You should now run `npm test` to ensure the test still passes.

Now run up the app to ensure it runs with `npm start`. We turned off the default behaviour of opening the browser when `npm start` is run, so open your browser and navigate to `http://localhost:3000`. When you run up the application, you should see a very plain-looking page that just says "App".

## Summing Up

That was a fair bit of text to go through just to end up with an almost empty project! However this now sets us up with a good foundation to actually build our app, with many things automatically taken care of.
