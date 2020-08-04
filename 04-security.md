# React - Part 4 - Security

The API that we'll be using for this project has the majority of its endpoints secured, so our application will need to obtain an appropriate Access Token in order to call them. The first step to obtaining an access token is to allow users to be able to sign in and authenticate themselves.

To do this we're going to use [Auth0](https://auth0.com/)'s free tier and OpenIdConnect in order to allow us to use [Authorization Code Flow with Proof Key for Code Exchange (PKCE)](https://auth0.com/docs/flows/concepts/auth-code-pkce). This flow is recommended over [Implicit Flow](https://auth0.com/docs/flows/concepts/implicit) which was often used by Single Page Apps in the past.

## Https

In order to usse Auth0 correctly, our site should be served using `https`. `create-react-app` supports using a self-signed certificate to provide `https` locally by setting an environment variable.

Update the `.env` file to contain the following:

```
BROWSER=none
HTTPS=true
```

Stop your local development server (if it's running) and then restart it with `npm start`. You should now see that the addresses specified in the console have a leading `https` rather than `http`.

If you navigate to `https://localhost:3000` now, you should see a warning as the certificate is not trusted - this can be ignored as you can trust the site that you are creating locally!

> If you prefer to be able to use a specific certificate (and then potentially add it to your system's trust list) you can do so using [additional environment variables](https://create-react-app.dev/docs/using-https-in-development/#custom-ssl-certificate).

## Setting up Auth0

In order to use Auth0, you must create an account and a tenant (if you do not already have one)

> To create an account head to [auth0.com/signup](https://auth0.com/signup) and follow the steps. When prompted to create a tenant, you can use whatever domain name you like.

Once you are signed in to Auth0, create a new Application via the "Applications" menu on the dashboard and "Create Application". Then:

- Enter a name for your application ("Repoll" perhaps, but anything is fine).
- Select "Single Page Web Applications" as the type.
- Click "Create"

In the Settings for the new application, configure the following fields:

- In "Application URIs":
  - Allowed Callback URLs: **https://localhost:3000/authorise**
  - Allowed Logout URLs: **https://localhost:3000/signed-out**
  - Allowed Web Origins: **https://localhost:3000**
  - Allowed Origins (CORS): **https://localhost:3000**
- In "Application Tokens"
  - Change "Refresh Token Behavior" to **"Rotating"** (this will allow us to use refresh tokens if we wish)
- Under "Advanced Settings" (via the "Show Advanced Setings" link at the bottom)
  - Under "Grant Types", **untick "Implicit"** (just to ensure that we dont' accidentially use it)

Then click **"Save Changes"**.

### Further Optional Configuration

> These extra settings just lock down the application a little more, but are not required.

Switch to the "Connections" tab for the Application:

- **Untick the "Google" option** under "Social", as we'll just use defined username/passwords for this demo.

Disable sign ups, so that only accounts that you define explicitly can sign in. Navigate to the "Authentication - Database" option in the menu, and click on the item to configure it.

- **Enable the "Disable Sign Ups"** option.

Finally, create a user to sign in as. Click on the "User Management - Users" menu option, and click "Create User".

- Enter an email and password to use for signing in, and click "Create".

## React Configuration

In order to use our new Auth0 application for signing in, we'll need to pass a couple of configuration values for our site to use. The easiest way to pass configuration values is to use environment variables.

Add the following to the `.env` file:

```
...
REACT_APP_AUTH_DOMAIN=
REACT_APP_AUTH_CLIENT_ID=
```

> Note that the ["REACT_APP\_" prefix](https://create-react-app.dev/docs/adding-custom-environment-variables) is **required**.

The change doesn't actually _do anything_ as far as our site goes, as all we've done is define a couple of empty variables. However it's often useful to use the `.env` file as a list of all the environment variables that the site can use.

However, we do not want to have to manually configure these variables every time we start the application, and if we have other developers working with us, it would also be good if they did not have to manually configure their own systems either.

Create a [new file called `.env.development`](https://create-react-app.dev/docs/adding-custom-environment-variables/#what-other-env-files-can-be-used) and add the following content:

```
REACT_APP_AUTH_DOMAIN=<your tenant name>.<your auth0 region>.auth0.com
REACT_APP_AUTH_CLIENT_ID=<your application client id>
```

Both of these values ("Domain" and "Client ID") are visible in the Settings of your Auth0 appliction.

For example:

```
REACT_APP_AUTH_DOMAIN=my-co.au.auth0.com
REACT_APP_AUTH_CLIENT_ID=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

> Note: Make sure you do not use your "Client Secret" by mistake! You should only ever expose something in a SPA codebase if it is considered "public".

By committing `.env.development` to Git, other developers will automatically use the same values, but they'll only be used for `development` builds (ie. when running `npm start`).

Now that we're setting our environment variables, we can make them available in our code. Add a new file at `src\environment.ts` with the following content:

```ts
export const AUTH_DOMAIN = process.env.REACT_APP_AUTH_DOMAIN ?? '';
export const AUTH_CLIENT_ID = process.env.REACT_APP_AUTH_CLIENT_ID ?? '';
```

These exported `const` values will be set to the environment variable values at build time.

## Creating a Security Service

Most of the work requried in order to use our Auth0 application is provided in an existing library, which we'll install from NPM:

```cmd
npm i @auth0/auth0-spa-js
```

But in order to make using it fit a little more nicely without our React application, we'll create some extra types. Create a new file (module) at `src/infrastructure/security.ts`.

> The `infrastructure` folder is intended for code that would be considered not to be "domain-specific" - that is, it could potentially be reused in another unrelated application with minimal changes.

In the new `security` file, add the following:

```ts
import { Auth0Client } from '@auth0/auth0-spa-js';
import { AUTH_CLIENT_ID, AUTH_DOMAIN } from 'environment';
import { History } from 'history';
import React from 'react';

const DEFAULT_RETURN_PATHNAME = '/';
const AUTHORISE_PATHNAME = '/authorise';

export type SignedInUser = {
  name: string;
  picture: string;
};

export class SecurityManager {
  private readonly _history: History;
  private readonly _client: Auth0Client;
  private readonly _redirectUri = `${window.location.origin}${AUTHORISE_PATHNAME}`;

  private _user: SignedInUser | undefined;

  constructor(history: History) {
    this._history = history;
    this._client = new Auth0Client({
      domain: AUTH_DOMAIN,
      client_id: AUTH_CLIENT_ID,
      redirect_uri: this._redirectUri,
      useRefreshTokens: true,
    });
  }
}
```

The intention of the `SecurityManager` class is to encapsulate our security state and behaviour. With the code above, we are:

- capturing an instance of the site's `history`
- creating an instance of the Auth0 client
  - the values from our environment variables are used here
  - we are also defining `useRefreshTokens` as `true` so that refreshing the user's tokens does not require the use of a hidden iframe. This is optional though, if require the `refresh_token` claim is undesirable for some reason.
- defining a private field to hold the user's details when they are available

We've covered the state - and now we'll add some behaviour to the class.

### Signing in

There are two steps involved in signing in:

1. The application triggers the sign in, which directs the browser to the Auth0 sign in page.
2. After the user has authenticated, Auth0 redirects the browser back to our application (with a token)

For step 1, add the following public method to the `SecurityManager` class:

```ts
export class SecurityManager {
  ...

  signIn = () => {
    this._client.loginWithRedirect({
      redirect_uri: this._redirectUri,
      appState: { returnUrl: this._history.createHref(this._history.location) },
    });
  };
}
```

This method triggers `loginWithRedirect` to navigate to Auth0, while using the `appState` to capture the user's current location (which will be used to ensure that the user ends up where they started).

For step 2, add this method as well:

```ts
export class SecurityManager {
  ...

  init = async () => {
    if (this._history.location.pathname === AUTHORISE_PATHNAME) {
      try {
        const result = await this._client.handleRedirectCallback();
        const returnUrl = result.appState?.returnUrl ?? DEFAULT_RETURN_PATHNAME;
        this._history.replace(returnUrl);
      } catch (e) {
        console.warn("Security handleRedirectCallback failed", e);
      }
    }

    await this._client.checkSession();
    this._user = await this._client.getUser();
  };
}
```

This method is called `init`, as it will be involved as part of initialising the `SecurityManager` instance. In its implementation, it does a few things:

- If the browser location is our `/authorise` endpoint (which you may recognise from the "Allowed Callback URLs" we configured in Auth0), we invoke the `handleRedirectCallback`. This method should handle everything we need to verify that the user signed in correctly. We can also extract our `returnUrl` and use the `history` instance to set the current url.
  - if something goes wrong, we just log a warning, and allow the system to continue.
- Regardless of the browser url, we ensure that we've loaded the current session correctly (if there is one), and get the current user (if there is one).
  - This handles the situation if the user just refreshes the page without signing in or out.

We also want to be able to check who is signed in. Add the following properties to the `SecurityManager` class:

```ts
export class SecurityManager {
  ...

  get isAuthenticated() {
    return !!this._user;
  }

  get currentUser() {
    return this._user;
  }
}
```

Now that we've added these methods and properties, we should be able to trigger a sign in _if_ they're invoked at the correct time. In order to do that, there are two more things we'll add to the `security.ts` file.

Add the following after the `SecurityManager` class:

```ts
const SecurityContext = React.createContext<SecurityManager | undefined>(undefined);

export const SecurityContextProvider = SecurityContext.Provider;

export function useSecurity() {
  const value = React.useContext(SecurityContext);
  if (!value) throw new Error('Security context has not been initialised');
  return value;
}
```

This will create exports to allow us to create a "Security Context" and a custom React hook so that we can make an instance of `SecurityManager` available throughout our application.

Now we'll use the new `SecurityManager`. Update `index.tsx` to the following:

```tsx
...
import { SecurityManager } from 'infrastructure/security';

async function init() {
  const history = createBrowserHistory();
  const securityManager = new SecurityManager(history);

  await securityManager.init();

  ReactDOM.render(
    <React.StrictMode>
      <App history={history} securityManager={securityManager} />
    </React.StrictMode>,
    document.getElementById('root')
  );
}

init().catch((e) => console.error('Application bootstrapping failed', e));

```

These changes create an instance of `SecurityManager`, invoke the `SecurityManager.init` method on it, and then pass it to the `App` component.

> This does mean that the site will not render until the `SecurityManager.init` method has completed. This should be quick, but if the "time to first paint" delay is important, you could render an "authenticating ..." page before calling init.

Now update the `App` component to take the `SecurityManager` instance and apply it to the Security Context using the `SecurityContextProvider`.

```tsx
...
import { SecurityContextProvider, SecurityManager } from 'infrastructure/security';

interface IAppProps {
  ...
  securityManager: SecurityManager;
}

export const App: React.FC<IAppProps> = function ({ history, securityManager }) {
  return (
    <div ...>
      <SecurityContextProvider value={securityManager}>
        <Router ...>
          ...
        </Router>
      </SecurityContextProvider>
    </div>
  );
};
```

Our security infrastructure is now ready to use! We just need to invoke the sign in and verification at the correct times.

## Protecting Routes

There are a number of ways that you can prevent unauthenticated users from accessing particular routes - as an extreme example, you can choose to not render the relevant `Route` components at all, so they do not even exist as far as the site is concerned. However we are going to create a "Private" version of the `Route` component that will send unauthenticated users off to sign in, which will allow the application to redirect them back to their original url when they're done.

> Note: Keep in mind that all client-side source code is available to any would-be attacker, so you should never put anything sensitive anywhere directly. Instead, it should be loaded from the server via an endpoint that does its own authentication. The "protected routes" we're adding here can be thought of more as for just providing a good user experience for legitimate users.

In the `src\views\features\Routes.tsx` file, add the following new private component:

```tsx
...
import { Switch, Redirect, Route, useLocation, RouteProps } from 'react-router-dom';
import { useSecurity } from 'infrastructure/security';
...

const PrivateRoute: React.FC<RouteProps> = function ({ children, ...rest }) {
  const securityService = useSecurity();

  React.useEffect(() => {
    if (!securityService.isAuthenticated) {
      securityService.signIn();
    }
  }, [securityService]);

  return <Route {...rest} render={() => (securityService.isAuthenticated ? children : null)} />;
};
```

This new version of `Route` will only render its child components if the user is authenticated, and if they're not, it will also trigger the sign in.

Now update the routes for `/polls` and `/polls/create` to be private:

```tsx
...

export const Routes: React.FC = function () {
  ...

  return (
    ...
      <PrivateRoute exact path="/polls">
        <Polls />
      </PrivateRoute>
      <PrivateRoute exact path="/polls/create">
        <CreatePoll />
      </PrivateRoute>
    ...
  );
};
```

If you run the site now, you should find that your are bounced to the Auth0 sign in page, as all the routes require the user to be signed in. You can sign in with the user account you created as part of the Auth0 setup, or use the "Sign Up" if you didn't complete the optional steps.

> You can experiment with having on one of the two routes as a `PrivateRoute` to see the effect of not having all routes secured.

## Displaying the Signed In User

While we know what's going on, there's currently nothing being displayed to the user to indicate whether they're signed in or not.

Add the following package from NPM:

```cmd
npm i react-responsive-modal
```

We'll also import the default style sheet for this library in `index.tsx` - add the following to the imports:

```tsx
import 'react-responsive-modal/styles.css';
```

> We could entirely style the modal ourselves (the default stylesheet is not long), but the defaults are good enough for this application.

Now, create a new file at `views\components\application\UserInfoButton.tsx` with the following content:

```tsx
import { useSecurity } from 'infrastructure/security';
import React from 'react';
import { Modal } from 'react-responsive-modal';
import styles from './UserInfoButton.module.scss';

export const UserInfoButton: React.FC = function () {
  const securityService = useSecurity();
  const [open, setOpen] = React.useState(false);

  return securityService.currentUser ? (
    <>
      <button type="button" onClick={() => setOpen(true)}>
        {securityService.currentUser.name}
      </button>
      <Modal
        open={open}
        center
        onClose={() => setOpen(false)}
        classNames={{
          modal: styles.userModal,
          closeButton: styles.closeButton,
        }}>
        <div className={styles.userModalContent}>
          <img src={securityService.currentUser.picture} alt="User avatar" />
          <div className={styles.userLabel}>Signed in as:</div>
          <strong className={styles.userName}>{securityService.currentUser.name}</strong>
        </div>
      </Modal>
    </>
  ) : (
    <button type="button" className={styles.signin} onClick={() => securityService.signIn()}>
      Sign In
    </button>
  );
};
```

Then create a corresponding `UserInfoButton.module.scss` file with the following content:

```scss
@use 'src/styles/colors' as c;

.signin {
  background-color: transparent;
  color: c.$on-highlight;
  border: none;
}

.avatar {
  padding: 0;
  overflow: hidden;
  width: 2rem;
  height: 2rem;
  border: solid 2px c.$on-highlight;
  border-radius: 1rem;

  > img {
    width: 100%;
    height: 100%;
  }
}

.userModal {
  display: grid;
  grid-template-columns: 1fr auto;
  grid-template-areas: 'content close';
  gap: 8px;
  align-items: start;

  .userModalContent {
    grid-area: content;

    display: grid;
    grid-template-columns: 2fr 3fr;
    grid-template-rows: auto 1fr auto;
    grid-template-areas:
      'picture label'
      'picture name';
    gap: 8px;
    justify-items: center;
    align-items: center;
    text-align: center;

    > img {
      grid-area: picture;
      width: 100%;
      max-width: 100px;
    }

    > .userLabel {
      grid-area: label;
    }

    > .userName {
      grid-area: name;
    }
  }

  .closeButton {
    grid-area: close;
    position: initial;
  }
}
```

This new component will show a "Sign In" button for unauthenticated users, and will show a button with the user's name for authenticated users. When the user's name is clicked, it shows a modal with the user's name and picture.

To display the new component, add it to the end of our `Nav` list in `src\views\components\application\Shell.tsx`:

```tsx
...
import { UserInfoButton } from './UserInfoButton';

const Nav: React.FC = function () {
  return (
    <nav ...>
      <ul>
        ...
        <li>
          <UserInfoButton />
        </li>
      </ul>
    </nav>
  );
};
...
```

You should now be able to see who is signed in.

## Signing Out

Finally, we want signed in users to be able to sign out if they wish. To add this feature, there are a few changes to make:

- Create a "Signed Out" page to show the user when they have been signed out.
- Update the `SecurityManager` to provide the sign out functionality
- Update the `UserInfoButton` to show a "Sign Out" button in the popup

Create a new file at `views\features\signed-out\SignedOut.tsx` with the following content:

```tsx
import React from 'react';

export const SignedOut: React.FC = function () {
  return (
    <>
      <h1>Signed out</h1>
      <p>You have been signed out.</p>
    </>
  );
};
```

Now update `views\features\Routes.tsx` with the following:

```tsx
...
import { SignedOut } from './signed-out/SignedOut';

...

export const Routes: React.FC = function () {
  ...
  const securityService = useSecurity();

  return (
    <Shell>
      <ErrorBoundary ...>
        <Switch>
          ...
          <Route exact path="/signed-out">
            {securityService.isAuthenticated ? <Redirect to="/" push={false} /> : <SignedOut />}
          </Route>
          ...
        </Switch>
      </ErrorBoundary>
    </Shell>
  );
};
```

> If someone who is signed in attempts to visit `/signed-out`, they will be redirected to `/`, instead of being shown a page with incorrect information.
>
> The `push={false}` just means that the url will be "replace" rather than "push", so that `/signed-out` is not retained in the browser history.

Then, update the `SecurityManager` class to add the following:

```ts
...
const SIGNED_OUT_PATHNAME = '/signed-out';

...

export class SecurityManager {
  ...

  signOut = () => {
    this._user = undefined;
    this._client.logout({
      returnTo: `${window.location.origin}${SIGNED_OUT_PATHNAME}`,
    });
  };
}
```

Now update the `UserInfoButton`:

```tsx
...

export const UserInfoButton: React.FC = function () {
  ...

  return securityService.currentUser ? (
    <>
      ...
      <Modal ...>
        <div className={styles.userModalContent}>
          ...
          <button className={styles.signout} type="button" onClick={securityService.signOut}>
            Sign Out
          </button>
        </div>
      </Modal>
    </>
  ) : (
    ...
  );
};
```

And update the styles in `UserInfoButton.module.scss`:

```scss
...

.userModal {
  ...

  .userModalContent {
    ...
    grid-template-areas:
      'picture label'
      'picture name'
      'picture signout';

    ...

    > .signout {
      grid-area: signout;
      justify-self: stretch;
      margin: 0 16px;
    }
  }

  ...
}

```

When the "Sign Out" button is clicked, the user should end up on the `/signed-out` page. You should now be able to sign in and out as you wish!

## Summing Up

Data security is a very important part of many applications, and hopefully the steps above have shown at least one straight-foward way to implmented it. Now that it's in place, in the next post we can look to retrieving the user's data from the API.
