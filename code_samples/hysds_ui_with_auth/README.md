# HySDS UI Authentication with OAuth2 Authorization Code Grant with the PKCE (Proof Key for Code Exchange)

This code sample was created based on the HySDS UI source code available at https://github.com/hysds/hysds_ui .

Few additional features were implemented to enable authentication, to demonstrate this as a reference implementation
to use OAuth2 Authorization Code Grant with the PKCE (Proof Key for Code Exchange) in React.JS based Unity UI applications.

## Background

Previously the OAuth 2.0 Implicit Grant was widely used in many Single Page Applications (SPA) such as React.JS applications.
However, now it is not recommended to  use Implicit Grant with SPA (https://oauth.net/2/grant-types/implicit/).
Public clients such as native apps and JavaScript apps should now use the authorization code flow with the PKCE extension instead.

This code sample implements Auth 2.0 Authorization Code Grant with the PKCE based on following references.
- [RFC7636 - Proof Key for Code Exchange by OAuth Public Clients](https://datatracker.ietf.org/doc/html/rfc7636) - By Internet Engineering Task Force (IETF)
- [Call Your API Using the Authorization Code Flow with PKCE](https://auth0.com/docs/get-started/authentication-and-authorization-flow/call-your-api-using-the-authorization-code-flow-with-pkce) - By Auth0
- [Amazon Cognito user pools Auth API reference](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-userpools-server-contract-reference.html) - By Amazon
- [Setup an OAuth2 PKCE flow for a React.JS application](https://rmannibucau.metawerx.net/react-oauth2-pkce-flow-setup.html) - By Romain Manni-Bucau
- [react-oauth2-pkce library](https://github.com/gardner/react-oauth2-pkce) - By Gardner Bickford
- [The npm page of the react-oauth2-pkce library](https://www.npmjs.com/package/react-oauth2-pkce) - By Gardner Bickford
- [OAuth 2.0: Authorization Code Grant Flow with PKCE for Web Applications By Example](https://codeburst.io/oauth-2-0-authorization-code-grant-flow-with-pkce-for-web-applications-by-example-4dbcc089e805) - By John Tucker
- [Sample code related with the above article](https://github.com/larkintuckerllc/todosrus-fe/tree/cognito) - By John Tucker


## Pre-Requisites

- Node (>=6 & < 13) installed on machine
- npm installed

## Create Config Files

```bash
# create config file (if developing locally)
cp src/config/index.local-development.template.js src/config/index.js

cp src/config/figaro.template.js src/config/figaro.js
cp src/config/tosca.template.js src/config/tosca.js
```

## Update index.js Config File

Update the following settings in `config/index.js` to match with your setup.

#### **`config/index.js`**
```js

// Mozart's Rest API
exports.MOZART_REST_API_BASE = "<AWS_API_GATEWAY_URL>/mozart_rest_api";

// OAuth2 configs
exports.OAUTH2_CLIENT_ID = "<ADD_CLIENT_ID_HERE>";
exports.OAUTH2_REDIRECT_URI = "http://localhost:8080";
exports.OAUTH2_PROVIDER_URL = "<COGNITO_USER_POOL_DOMAIN_URL>/oauth2";
exports.APP_VIEWER_GROUP_NAME = "Unity_Viewer";
exports.APP_ADMIN_GROUP_NAME = "Unity_Viewer";


```

## Building and Running Application Locally

```bash
# install nodejs dependencies
npm install

# builds and compiles to dist/index.js
npm run build

# run application in development mode with hot-reloading
npm start

# run application in "production mode"
npm run prod
```

## Dockerizing React Application

- uses nginx as a webserver to serve application as a static file

```bash
# Building the docker image
docker build . -t hysds_ui:latest

# Running image
docker run -p 8080:80 hysds_ui:latest
```


## Code Changes Made to the Existing HySDS UI

1. Added a new 'src/App.js' file with following contents.

#### **`src/App.js`**
```js
import React from "react";
import Routes from "./pages/Routes/index.jsx";
import { Provider } from "react-redux";
import store from "./redux/store";
function App() {
  return (
      <Provider store={store}>
        <Routes/>
      </Provider>
  );
}
export default App;

```

2. Implemented a new AppWithAuthentication as follows to wrap the original App. This implementation can be considered as a reference implementation and can be reused to wrap any other React.JS App. The UI elements can be improved further.

#### **`src/AppWithAuthentication.js`**
```js

/**
 * This AppWithAuthentication.js is a reference implementation to use OAuth2 Authorization Code Grant with PKCE (Proof Key for Code Exchange)
 * in React.JS based Unity UI applications.
 *
 * This code is implemented based on the following references:
 *  - https://datatracker.ietf.org/doc/html/rfc7636
 *  - https://auth0.com/docs/get-started/authentication-and-authorization-flow/call-your-api-using-the-authorization-code-flow-with-pkce
 *  - https://docs.aws.amazon.com/cognito/latest/developerguide/authorization-endpoint.html
 *  - https://rmannibucau.metawerx.net/react-oauth2-pkce-flow-setup.html
 *  - https://github.com/gardner/react-oauth2-pkce
 *  - https://www.npmjs.com/package/react-oauth2-pkce
 *  - https://codeburst.io/oauth-2-0-authorization-code-grant-flow-with-pkce-for-web-applications-by-example-4dbcc089e805
 *  - https://github.com/larkintuckerllc/todosrus-fe/tree/cognito
 */
import React from 'react';
import App from './App';
import {AuthProvider, AuthService, useAuth} from 'react-oauth2-pkce'
import {
    OAUTH2_PROVIDER_URL,
    OAUTH2_CLIENT_ID,
    OAUTH2_REDIRECT_URI,
    APP_VIEWER_GROUP_NAME,
    APP_ADMIN_GROUP_NAME
} from "./config";
import { Button } from "./components/Buttons";
import jwt_decode from "jwt-decode";

// Variable to store tokens
let tokens;

// Initialize AuthService
const authService = new AuthService({
    provider: OAUTH2_PROVIDER_URL,
    clientId: OAUTH2_CLIENT_ID,
    redirectUri: OAUTH2_REDIRECT_URI,
    scopes: ['openid', 'profile'],
});

/**
 * Checks if a given access token has a given userGroup.
 *
 * @param accessToken
 * @param userGroup
 * @returns true if the given access token has the given userGroup, else false
 */
function hasUserGroup(accessToken, userGroup) {
    let accessTokenDecoded = jwt_decode(accessToken);
    let cognitoGroups = accessTokenDecoded["cognito:groups"].toString();
    let userGroupsArray = cognitoGroups.split(',').map(function(item) {
        return item.trim();
    });

    return userGroupsArray.includes(userGroup);
}

/**
 * Checks for user authentication, only shows the App, if the user is Authenticated AND belongs to valid user groups.
 */
function AppWithAuthentication() {
    const { authService } = useAuth();

    /**
     * Invokes login for the the user
     */
    const login = async () => authService.authorize();

    /**
     * Invokes logout for the the user and clears all tokens
     */
    const logout = async () => {
        tokens = null;
        window.localStorage.removeItem('accessToken');
        window.localStorage.removeItem('idToken');
        window.localStorage.removeItem('refreshToken');
        window.localStorage.removeItem('auth');
        await authService.logout();
    }

    // If the user authentication is pending,show a Reset button to retry logout and login
    if (authService.isPending()) {
        return <div align="center">
            <h4>Authenticating...</h4>
            <Button size="large" color="fail" label="Reset"
                    onClick={() => {
                        logout().then();
                        login().then();
                    }} />
        </div>
    }

    // If user is not authenticated, then show the Login button
    if (!authService.isAuthenticated()) {
        return (
            <div align="center">
                <h4>User Not Logged-in</h4>
                <Button size="large" color="fail" label="Login with Cognito"
                        onClick={login} />
            </div>
        )
    }

    // If user is authenticated, then read tokens from the authService
    let accessToken = authService.getAuthTokens().access_token;
    let idToken = authService.getAuthTokens().id_token;
    let refreshToken = authService.getAuthTokens().refresh_token;

    tokens = accessToken === null ? null : {
        accessToken,
        idToken,
        refreshToken
    };

    // Get logged in username, email and user groups
    let accessTokenDecoded = jwt_decode(tokens.accessToken);
    let idTokenDecoded = jwt_decode(tokens.idToken);
    let loggedInUserName = accessTokenDecoded.username;
    let loggedInUserEmail = idTokenDecoded.email;
    let logoutLabel = "Logout : " + loggedInUserName;
    let userGroups = accessTokenDecoded["cognito:groups"].toString();

    // Check if the access token has the user groups required to access the App
    if (!hasUserGroup(accessToken, APP_VIEWER_GROUP_NAME)
        && !hasUserGroup(accessToken, APP_ADMIN_GROUP_NAME)) {
        return (
            <div align="center">
                <h4>  User {loggedInUserName} ({loggedInUserEmail}) is not authorized to access this application.</h4>
                <h4>  Please check your user groups [{userGroups}].</h4>
                <Button size="large" color="fail" label="Logout"
                        onClick={logout} />
            </div>
        );
    }

    // Clear token variables variable after use
    accessTokenDecoded = null;
    idTokenDecoded = null;
    accessToken = null;
    idToken = null;
    refreshToken = null;

    return (

        // Return the App
        <div align="right">
            <Button size="large" color="fail" label={logoutLabel}
                    onClick={logout} />
            <App />
        </div>
    );
}

/**
 * Returns tokens
 */
export const getTokens = () => tokens;

// AppWithAuthentication wrapped by the AuthProviderWrapper
const AuthProviderWrapper = () => {
    return (
        <AuthProvider authService={authService} >
            <AppWithAuthentication />
        </AuthProvider>
    )
}

export default AuthProviderWrapper;


```

3. Updated the 'src/index.js' file as follows.

#### **`src/index.js`**
```js
import React from 'react';
import ReactDOM from 'react-dom';
import AppWithAuthentication from './AppWithAuthentication';

ReactDOM.render(<AppWithAuthentication />, document.getElementById('app'));

```


4. Updated the `getJobCounts` function at `src/redux/actions/figaro/index.js` to access a [light weight mock endpoint of Mozart REST API](https://github.com/unity-sds/unity-cs-security/tree/main/code-samples/hysds_mozart_mock) with an access token.

#### **`src/redux/actions/figaro/index.js`**
```js
import { getTokens } from "../../../AppWithAuthentication";

export const getJobCounts = () => (dispatch) => {
    const jobCountsEndpoint = `${MOZART_REST_API_BASE}/job_count`;

    return fetch(jobCountsEndpoint, {
        method: 'get',
        headers: new Headers({
            'Authorization': 'Bearer ' + getTokens().accessToken
        })
    })
        .then((res) => res.json())
        .then((data) =>
            dispatch({
                type: JOB_COUNTS,
                payload: data.counts,
            })
        );
};

```

5. Updated the `module.exports` , `entry` in `webpack.config.js` as follows.

#### **`webpack.config.js`**
```js
module.exports = {
    entry: ["regenerator-runtime/runtime.js", "./src/index.js"],
```

6. Added `"jwt-decode": "^3.1.2"` and `"react-oauth2-pkce": "^2.0.7"` as dependencies.

#### **`package.json`**
```json
"dependencies": {
"@fortawesome/fontawesome-svg-core": "1.2.28",
"@fortawesome/free-solid-svg-icons": "5.13.0",
"@fortawesome/react-fontawesome": "0.1.9",
"jwt-decode": "^3.1.2",
"react": "16.9.0",
"react-dom": "16.9.0",
"react-helmet": "5.2.1",
"react-json-view": "1.19.1",
"react-oauth2-pkce": "^2.0.7"
}

```


## Basic Example with Some U-SPS Components Integrated

More details and screenshots of this example available at https://github.com/unity-sds/unity-cs/wiki/Unity-Security-Guidelines#basic-example-with-some-u-sps-components-integrated
