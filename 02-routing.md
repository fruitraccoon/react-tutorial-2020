# React - Part 2 - Routing

## Creating some pages

In order for our site to be useful, we have to have some pages that people can view and navigate between.

- At `/polls` we'll have a page to list existing Polls
- At `/polls/create` we'll have a page to create a new Poll
- We'll also create a "Not Found" page for any invalid urls

To get to that point, the first thing we'll do is create some components that will become the pages.

### Polls List

Create a new file at `src\views\features\polls\Polls.tsx` with the following content:

```tsx
import React from "react";

export const Polls: React.FC = function () {
  return <>Polls</>;
};
```

> To create this file, we are also adding a new `features` directory to `views`. This folder is where we'll have a folder for each of the major features that the system provides, and these will roughly correspond to the routes as well.

### Create Poll

Creating a Poll is still part of the large "Polls" feature, so create a new file at `src\views\features\polls\create\CreatePoll.tsx` with the following content:

```tsx
import React from "react";

export const Polls: React.FC = function () {
  return <>Create Poll</>;
};
```

### Not Found

Create a new file at `src\views\features\not-found\NotFound.tsx` with the following content:

```tsx
import React from "react";

export const NotFound: React.FC = function () {
  return (
    <>
      <h1>:(</h1>
      <p>The page you were looking for cannot be found.</p>
    </>
  );
};
```

## Routing

Now that we have some pages, let's create some routes so that we're able to view them! We are going to use [the `react-router` library](https://reacttraining.com/react-router/web/guides/quick-start) which we install from NPM.

```cmd
npm i react-router-dom @types/react-router-dom @types/history
```

Create another new file at `src\views\features\Routes.tsx` and make set its contents to:

```tsx
import React from "react";

export const Routes: React.FC = function () {
  return <div>Routes</div>;
};
```

Update the contents of `App.tsx` to

```tsx
import React from "react";
import styles from "./App.module.scss";
import { Router } from "react-router-dom";
import { History } from "history";
import { Routes } from "views/features/Routes";

interface IAppProps {
  history: History;
}

export const App: React.FC<IAppProps> = function ({ history }) {
  return (
    <div className={styles.root}>
      <Router history={history}>
        <Routes />
      </Router>
    </div>
  );
};
```

Here we are creating a `react-router` "Router" that will allow routing to work for components below it - hence we're adding it at the top of the component tree. The Router takes a `History` instance, which is passed in as a prop. Passing it in allows different uses of the App component to pass different kinds of History.

Update the `index.tsx` file with the following import:

```ts
import { createBrowserHistory } from "history";
```

and use it to create a new instance to pass to the App component

```tsx
const history = createBrowserHistory();

ReactDOM.render(
  <React.StrictMode>
    <App history={history} />
  </React.StrictMode>,
  document.getElementById("root")
);
```

The `App` component is also used in the `App.test.tsx` file. Here, include this different import:

```ts
import { createMemoryHistory } from "history";
```

Then use it to create an instance and pass it to the App component in the same way.

> If you run your application now, you should see "Routes" displayed.

### Routing to our pages

Next we want to be able to access our pages at different urls. Update the content of the `Routes` component to the following:

```tsx
import React from "react";
import { Switch, Route, Redirect } from "react-router-dom";
import { Polls } from "./polls/Polls";
import { CreatePoll } from "./polls/create/CreatePoll";
import { NotFound } from "./not-found/NotFound";

export const Routes: React.FC = function () {
  return (
    <Switch>
      <Redirect exact from="/" to="/polls" />
      <Route exact path="/polls">
        <Polls />
      </Route>
      <Route exact path="/polls/create">
        <CreatePoll />
      </Route>
      <Route>
        <NotFound />
      </Route>
    </Switch>
  );
};
```

The different parts of the updated component are as follows:

- We're using a `<Switch>` component so that only the first matching child component is used.
- The `<Redirect>` will redirect users at the root url (`/`) to the Polls url.
- If the url is `/polls`, the `Polls` component will be rendered
- If the url is `/polls/create` the `CreatePoll` component will be rendered
- If there are no other matches, the `NotFound` component will be rendered (as it has no `to` prop). Note that the url will not be changed, as we're not redirecting)

> If you run your application now, you should be able to experiment with typing different urls as see the different pages rendered.

## Creating a Shell

The Shell provides the consistent navigation and framing across all pages of the application, but we only want to build it once (of course). As the Shell is not linked to any particular feature for the user, it doesn't belong in the `features` folder. We'll create a new `components` sibling folder for components that are not feature-specific. We'll also create an `application` subfolder for components that are only used once in the application (as opposed to a non-feature component that might be used multiple times).

Create a new file at `views/components/application/Shell.tsx` and create a new function component with the following contents:

```tsx
import React from "react";

export const Shell: React.FC = function () {
  return (
    <div>
      <header></header>
      <nav></nav>
      <main></main>
      <footer></footer>
    </div>
  );
};
```

Update the contents of the `header` to the following (which needs this import - `import { Link } from 'react-router-dom';`) to create standard click-the-title-to-go-to-root behaviour.

```html
<Link to="/">
  <em>Re</em>
  <span>poll</span>
</Link>
```

Update the footer how you would like:

```html
<footer>React - Polls - Repoll!</footer>
```

The `main` element is to contain the current route page - the main focus of the application! The Shell does not care what this is, but it needs to position it appropriately in its JSX hierarchy. We are going to use the `children` prop for this, so that usage of the Shell can be `<Shell><PageContent /></Shell>`.

Update the declaration of the `Shell` function to this:

```tsx
...
export const Shell: React.FC = function({ children }) {
  ...
  <main>{children}</main>
  ...
```

### Site Navigation

We could just create some site navigation anchor tags inline in the `Shell` component, like we have for the others. However, as we'll have a list of links that may get a bit long, instead we'll create a separate _private_ component. A private component is just one in the same file (js module) that is not exported. If the component ends up being a good candidate for reuse, it can easily be exported or moved to its own file in an appropriate location at the time it is needed. By making it private until we need it to not be, refactoring the `Shell` module will be easier.

> I like to approach refactoring of React components in the same way that I would approach refactoring other code - I start with writing the main block of code, and when it becomes big and/or confusing, I refactor out private functions/methods to create simpler, well-named code blocks. And as components are just functions, this is usually straight-forward.

Create a new empty component called `Nav` in `Shell.tsx`, above the `Shell` function but below the imports, with the contents below to create links to our two pages. Note that you'll need to import `NavLink` from `react-router-dom` - if you just used regular anchor tags, the site would entirely reload when the links were clicked.

```tsx
import ...

const Nav: React.FC = function () {
  return (
    <nav>
      <ul>
        <li>
          <NavLink exact to="/polls">
            Polls
          </NavLink>
        </li>
        <li>
          <NavLink exact to="/polls/create">
            Create Poll
          </NavLink>
        </li>
      </ul>
    </nav>
  );
};

export const Shell: React.FC = function ({ children }) {
...
```

Once the `Nav` component has been added, use it instead of the existing `<nav></nav>` in `Shell`.

### Using the Shell

In order to use the Shell, it has to be rendered somewhere. In the `routes/Routes.tsx` file, wrap the `Switch` tags with the `Shell`, so that the `Switch` is the `Shell`'s child. This will require `import`ing the Shell component.

```html
<Shell>
  <Switch>...</Switch>
</Shell>
```

Running the site now should display a bit more than before, complete with functioning links.

> _Why put the `Shell` component in `Routes`, rather than in `App` (that renders `Routes`)?_
>
> You certainly can make that exchange if you wish to. The main reason that I put it in `Routes` is that I've found that sometimes I want to render the Shell differently for different routes, such as when the user is Authenticated vs. not.

## Errors

React has the concept of [ErrorBoundaries](https://reactjs.org/docs/error-boundaries.html). This can be thought of as `try/catch` blocks, but for React Components. Ultimately, they allow unexpected errors to be captured and isolated, rather than have the whole application crash (or continue running in an unknown state).

We are going to wrap the display of our pages in an error boundary so that if one page crashes, the Shell (and thus the Nav) is still available, and the user can navigate to a different page that (hopefully) is still working.

Install the following from NPM:

```cmd
npm i react-error-boundary
```

> While implmeenting an error boundary does not take a lot of code, using this library just frees us from having to implement some of the nicer features, such as being able to easily "reset" the boundary - see below.

Add the following as a private component in `src\views\features\Routes.tsx`. This will be what is displayed, should a particular page crash.

```jsx
const ErrorFallback: React.FC = function () {
  return (
    <div>
      <h1>:(</h1>
      <p>Unfortunately something went wrong. Please try again later.</p>
    </div>
  );
};
```

Now update the `Routes` component to wrap the `Switch` in the `ErrorBoundary` component from `react-error-boundary`:

```jsx
...
import { ErrorBoundary } from 'react-error-boundary';
import { Redirect, Route, Switch, useLocation } from 'react-router-dom';

...

export const Routes: React.FC = function () {
  const location = useLocation();

  return (
    <Shell>
      <ErrorBoundary FallbackComponent={ErrorFallback} resetKeys={[location.pathname]}>
        <Switch>
          ...
        </Switch>
      </ErrorBoundary>
    </Shell>
  );
};
```

This will "catch" any unhandled exceptions that thrown when the site pages are rendered, and display our fallback, while still showing the `Shell`. By setting the `resetKeys` prop to the current url path, the error boundary will be reset when the route changes, so navigating to a different page will work.

Test out the error boundary by adding a `throw new Error()` into one of the page components, then run the site to see how it behaves as you navigate back and forth between the working and broken page.

## Summing Up

We now have the _very_ beginnings of our site, with a Shell that wraps our pages, and a nav that allows us to navigate between them. Next up, we'll add some styling.
