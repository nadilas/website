---
sidebar_position: 2
---

# NPM Library

Jackson is available as an [npm package](https://www.npmjs.com/package/@boxyhq/saml-jackson) that can be integrated into any web application framework (like Express.js for example). Please file an issue or submit a PR if you encounter any issues with your choice of framework.

```bash
npm i @boxyhq/saml-jackson
```

Integrating SAML Jackson with a Node.js app involves the following steps.

See the [Github repo](https://github.com/boxyhq/express-jackson-demo) to see the source code for the Express integration

## Express.js example

### Requirements

- Node.js version 14 or newer
- A [supported database](./service.md#database)
- An Express.js based app to add SAML Jackson

### Install SAML Jackson library

```bash
npm i --save @boxyhq/saml-jackson
```

### Add Express Routes

```javascript
// express
const express = require('express');
const router = express.Router();
const cors = require('cors'); // needed if you are calling the token userinfo endpoints from the frontend

// Set the required options. Refer to `Environment Variables` for the full list
const opts = {
  externalUrl: 'https://my-cool-app.com',
  samlAudience: 'https://my-cool-app.com',
  samlPath: '/sso/oauth/saml',
  db: {
    engine: 'mongo',
    url: 'mongodb://localhost:27017/my-cool-app',
  },
};

let apiController;
let oauthController;
let logoutController;
// Please note that the initialization of @boxyhq/saml-jackson is async, you cannot run it at the top level
// Run this in a function where you initialize the express server.
async function init() {
  const ret = await require('@boxyhq/saml-jackson').controllers(opts);
  apiController = ret.apiController;
  oauthController = ret.oauthController;
  logoutController = ret.logoutController;
}
```

- Add your app base URL as `externalUrl`
- `samlPath` becomes part of the ACS URL. The ACS URL is an endpoint on the SP where the IdP will redirect to with its authentication response. For example: If `externalUrl` is `http://localhost`, and `samlPath` is `/sso/acs`, the ASC URL will be `http://localhost/sso/acs`

### Add SAML Config API route

[API Reference](../saml-flow.md#2-saml-config-api)

```javascript
// express.js middlewares are needed to parse json and x-www-form-urlencoded
router.use(express.json());
router.use(express.urlencoded({ extended: true }));

// SAML config API. You should pass this route through your authentication checks, do not expose this on the public interface without proper authentication in place.
router.post('/api/v1/saml/config', async (req, res) => {
  try {
    // apply your authentication flow (or ensure this route has passed through your auth middleware)
    ...

    // only when properly authenticated, call the config function
    res.json(await apiController.config(req.body));
  } catch (err) {
    res.status(500).json({
      error: err.message,
    });
  }
});
// update config
router.patch('/api/v1/saml/config', async (req,res) => {
   try {
    // apply your authentication flow (or ensure this route has passed through your auth middleware)
    ...

    // only when properly authenticated, call the config function
    res.json(await apiController.updateConfig(req.body));
  } catch (err) {
    res.status(500).json({
      error: err.message,
    });
  }
})
// fetch config
router.get('/api/v1/saml/config', async (req, res) => {
  try {
    // apply your authentication flow (or ensure this route has passed through your auth middleware)
    ...

    // only when properly authenticated, call the config function
    res.json(await apiController.getConfig(req.query));
  } catch (err) {
    res.status(500).json({
      error: err.message,
    });
  }
});
// delete config
router.delete('/api/v1/saml/config', async (req, res) => {
  try {
    // apply your authentication flow (or ensure this route has passed through your auth middleware)
    ...

    // only when properly authenticated, call the config function
    await apiController.deleteConfig(req.body);
    res.status(200).end();
  } catch (err) {
    res.status(500).json({
      error: err.message,
    });
  }
});
```

### OAuth: Authorize URL

The OAuth flow begins with redirecting your user to the authorize URL. The response contains the `redirect_url` to which you should redirect the user.

[API Reference](../saml-flow.md#4-authorize)

```javascript
// OAuth 2.0 flow
router.get('/oauth/authorize', async (req, res) => {
  try {
    const { redirect_url } = await oauthController.authorize(req.query);

    res.redirect(redirect_url);
  } catch (err) {
    const { message, statusCode = 500 } = err;

    res.status(statusCode).send(message);
  }
});
```

### Handle SAML Response

Add a method to handle the SAML Response from IdP.

:::info
SAML Response - IdP issues an HTTP POST request to SP's Assertion Consumer Service (ACS URL) with 2 fields `SAMLResponse` and `RelayState`.
:::

```javascript
router.post('/oauth/saml', async (req, res) => {
  try {
    const { redirect_url } = await oauthController.samlResponse(req.body);

    res.redirect(redirect_url);
  } catch (err) {
    const { message, statusCode = 500 } = err;

    res.status(statusCode).send(message);
  }
});
```

### OAuth: Code Exchange

The code can then be exchanged for a token by making the following request. You should validate that the state matches the one you sent in the authorize request.

[API Reference](../saml-flow.md#5-code-exchange)

```javascript
router.post('/oauth/token', cors(), async (req, res) => {
  try {
    const result = await oauthController.token(req.body);

    res.json(result);
  } catch (err) {
    const { message, statusCode = 500 } = err;

    res.status(statusCode).send(message);
  }
});
```

### OAuth: Get User Profile

The short-lived access token can now be used to request the user's profile.

[API Reference](../saml-flow.md#6-profile-request)

```javascript
router.get('/oauth/userinfo', async (req, res) => {
  try {
    let token = extractAuthToken(req);

    // check for query param
    if (!token) {
      token = req.query.access_token;
    }

    if (!token) {
      res.status(401).json({ message: 'Unauthorized' });
    }

    const profile = await oauthController.userInfo(token);

    res.json(profile);
  } catch (err) {
    const { message, statusCode = 500 } = err;

    res.status(statusCode).json({ message });
  }
});

// set the router
app.use('/sso', router);
```

### SLO: Create Logout Request

Create the logout request by calling the method `createRequest()`.

```javascript
router.get('/logout', async (req, res, next) => {
  const { logoutUrl, logoutForm } = await logoutController.createRequest({
    nameId: 'google-oauth2|108149256146623609101',
    tenant: 'boxyhq.com',
    product: 'demo',
    redirectUrl: 'http://localhost:3000',
  });

  // HTTP-Redirect binding
  if (logoutUrl) {
    return res.redirect(logoutUrl);
  }

  // HTTP-POST binding
  if (logoutForm) {
    return res.send(logoutForm);
  }
});
```

### SLO: Handle the Response

IdP will send a response back to a specific URL. You need to register this URL on the IdP to handle the response properly.

```javascript
router.post('/logout/callback', async (req, res, next) => {
  const { SAMLResponse, RelayState } = req.body;

  const { redirectUrl } = await logoutController.handleResponse({
    SAMLResponse,
    RelayState,
  });

  return res.redirect(redirectUrl);
});
```
