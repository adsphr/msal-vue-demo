# MSAL / Vue Demo

adsphr note : had to run with sudo npm run dev 
	(still errors out with 400 bad request....env variables not set up right proabably)

![image](https://user-images.githubusercontent.com/132681/221994392-5b42ed54-93c7-4b54-8d23-40c37de0c182.png)

## Overview

This repository is the accompanying code for my article [A guide to MSAL authentication in Vue](https://davestewart.co.uk/blog/msal-vue/).

The app has the following aims:

- working end-to-end Vue / MSAL implementation
- production-friendly yet bare-minimal dependencies, markup and code 
- easily understood via a [modular](https://github.com/davestewart/msal-vue-demo/tree/main/src) structure and clean [abstractions](https://github.com/davestewart/msal-vue-demo/blob/main/src/services/api.ts)
- easily migrated via strong [separation](https://github.com/davestewart/msal-vue-demo/blob/main/src/services/auth.ts) of [concerns](https://github.com/davestewart/msal-vue-demo/blob/main/src/store/auth.ts)

Next steps:

1. [read](https://davestewart.co.uk/blog/msal-vue/#a-working-authentication-setup) the article walkthrough
2. [configure](#configuration) as per the steps below
3. [run](#running-the-app) the app

## Configuration

> Note that the app is a minimal reproduction of my own production-optimised setup, and you should review and if necessary modify configuration to suit your requirements!

The app has four places you'll need to check and configure before running:

- [env variables](#env-variables) - to make it production ready
- [auth config](#auth-config) - to configure the MSAL library
- [vite config](#vite-config) - to optionally change the application port
- [user store](#user-store) - to call your `user` endpoint

### Env variables

Copy [`example.env`](https://github.com/davestewart/msal-vue-demo/blob/main/example.env) to `.env` and populate with your API and MSAL config:

```
# your api
VITE_API_ROOT=https://xxx.com/api/v1

# client id
VITE_AUTH_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# authority
VITE_AUTH_AUTHORITY=https://xxx.b2clogin.com/xxx.onmicrosoft.com/xxx

# scope
VITE_AUTH_SCOPE=https://xxx.onmicrosoft.com/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/xxx
```

Note that I had only one authority and scope.

### Auth config

The [auth config](https://github.com/davestewart/msal-vue-demo/blob/main/src/config/auth.ts) pulls these values in like so:

```ts
export const config: Configuration = {
  // required
  auth: {
    // must match info in dashboard
    clientId: import.meta.env.VITE_AUTH_CLIENT_ID,
    authority: import.meta.env.VITE_AUTH_AUTHORITY,
    knownAuthorities: [
      import.meta.env.VITE_AUTH_AUTHORITY,
    ],

    // login redirect; must match path in dashboard
    redirectUri: `${location.origin}/auth/redirect/login`,
  },
```

Note:

- I needed to pass our only authority as a "known" authority; you may not need this
- your redirect URL will probably be different to this; make sure you change it

### Vite config

Because OAuth 2 requires that redirect URIs in the app match those in the dashboard, you may need to [configure](https://github.com/davestewart/msal-vue-demo/blob/main/vite.config.ts) the port Vite runs on:

```js
export default defineConfig({
  plugins: [vue()],
  server: {
    port: 8080
  },
```

Our (legacy) application runs on port 8080, so I modified this.

### User store

Finally, check the [user store](https://github.com/davestewart/msal-vue-demo/blob/main/src/stores/user.ts) for the app's call to get user data:

```ts
export async function load () {
  data.value = await Api.get('/users/current') // change this for your endpoint
}
```

You will want to tweak this, so it calls your own user endpoint.

## Running the app

Once completely configured, you can run the app with:

```
npm run dev
```

View it at:

- http://localhost:8080

When loaded:

- click `login` to login
- the route navigation should complete and a welcome message should display
- once logged in, click the `user` link to show account and API data
- if you cancel the login an error message will show
- if you log out and return to `/user` you'll be prompted to log in again

## Troubleshooting

If the login popup opens then immediately closes, check the console for errors.

If the login popup opens but doesn't show the Microsoft login form, it's likely that your redirect URI doesn't match what is entered in the dashboard. Make sure it matches and that it's a full URI.

Note that the popup's URL will contain information about the error in the **hash**:

```
https://<your_domain>/<your_redirect_path>#error=redirect_uri_mismatch&error_description=AADB2C90006%3a+The+redirect+URI+%27http%3a%2f%2flocalhost%3a8080%2fauth%2fredirect%2flogin%27+provided+in+the+request+is+not+registered+for+the+client+id+%27385685cf-8080-44d5-a67a-e192a4324351%27.%0d%0aCorrelation+ID%3a+c277eb37-ec03-4ae7-8d06-ee6a69cf60f9%0d%0aTimestamp%3a+2023-07-21+08%3a35%3a38Z%0d%0a&state=eyJpZCI6ImQwMmY4YTFmLTBmNTctNDFjZi04YjdhLTk0ODgxNTY0MmE2OSIsIm1ldGEiOnsiaW50ZXJhY3Rpb25UeXBlIjoicG9wdXAifX0%3d
```

To make the messages readable, run the following code in the popup's DevTools console:

```js
var u = new URL(location.href)
var p = new URLSearchParams(u.hash.substr(1))
Array.from(p.entries()).forEach(([key, value]) => console.log(`[${key}]: ${value}`))
```
```
[error]: redirect_uri_mismatch
[error_description]: AADB2C90006: The redirect URI 'http://localhost:8080/auth/redirect/login' provided in the request is not registered for the client id '385685cf-8080-44d5-a67a-e192a4324351'.
Correlation ID: c277eb37-ec03-4ae7-8d06-ee6a69cf60f9
Timestamp: 2023-07-21 08:35:38Z
[state]: eyJpZCI6ImQwMmY4YTFmLTBmNTctNDFjZi04YjdhLTk0ODgxNTY0MmE2OSIsIm1ldGEiOnsiaW50ZXJhY3Rpb25UeXBlIjoicG9wdXAifX0=
```

Note that if using the [redirect strategy](https://github.com/AzureAD/microsoft-authentication-library-for-js/blob/dev/lib/msal-browser/docs/initialization.md#choosing-an-interaction-type), then the URL with the error may only show for a split second before redirecting. In that case, you'll have to be quick to select and copy the URL (!) then you can use the code above to display the error. 

## Next steps

This repo is aimed to be a good jumping-off point for an MSAL app.

Here are some ideas for possible next steps:

- migrate this code to your own app
- convert and iron out the [gotchas](https://davestewart.co.uk/blog/msal-vue/#msal-gotchas) to be work with redirect flow
- move all auth code to a `/vendor/msal` folder and provide an entry point `index.ts`
- create one or more composables that could do the same job, but be intuitive to use
- convert to a plugin, and set up with an `install()` function
- back-port to Vue 2
