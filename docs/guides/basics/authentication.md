---
outline: deep
---

# Authentication

We now have a fully functional chat application consisting of [services](./services.md) and [schemas](./schemas.md). The services come with authentication enabled by default, so before we can use it we need to create a new user and learn how Feathers authentication works. We will look at authenticating our REST API, and then how to authenticate with Feathers in the browser. Finally we will implement a "Login with GitHub".

## Registering a user

The HTTP REST API can be used directly to register a new user. We can do this by sending a POST request to `http://localhost:3030/users` with JSON data like this as the body:

```js
// POST /users
{
  "email": "hello@feathersjs.com",
  "password": "supersecret"
}
```

Try it:

```sh
curl 'http://localhost:3030/users/' \
  -H 'Content-Type: application/json' \
  --data-binary '{ "email": "hello@feathersjs.com", "password": "supersecret" }'
```

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/6bcea48aac6c7494c2ad)

<BlockQuote type="info">

For SQL databases, creating a user with the same email address will only work once, then fail since it already exists. With the default database selection, you can reset your database by removing the `feathers-chat.sqlite` file and running `npm run migrate` to initialise it again.

</BlockQuote>

This will return something like this:

```json
{
  "id": "<user id>",
  "email": "hello@feathersjs.com",
  "avatar": "https://s.gravatar.com/avatar/ffe2a09df37d7c646e974a2d2b8d3e03?s=60"
}
```

Which means our user has been created successfully.

<BlockQuote type="info">

The password is stored securely in the database but will never be included in an external response.

</BlockQuote>

## Logging in

By default, Feathers uses [JSON web token](https://jwt.io/) for authentication. It is an access token that is valid for a limited time (one day by default) that is issued by the Feathers server and needs to be sent with every API request that requires authentication. Usually a token is issued for a specific user and in our case we want a JWT for the user we just created.

<BlockQuote type="tip">

If you are wondering why Feathers is using JWT for authentication, have a look at [this FAQ](../../help/faq.md#why-are-you-using-jwt-for-sessions).

</BlockQuote>

Tokens can be created by sending a POST request to the `/authentication` endpoint (which is the same as calling the `create` method on the `authentication` service set up in `src/authentication`) and passing the authentication strategy you want to use. To get a token for an existing user through a username (email) and password login we can use the built-in `local` authentication strategy with a request like this:

```js
// POST /authentication
{
  "strategy": "local",
  "email": "hello@feathersjs.com",
  "password": "supersecret"
}
```

Try it:

```sh
curl 'http://localhost:3030/authentication/' \
  -H 'Content-Type: application/json' \
  --data-binary '{ "strategy": "local", "email": "hello@feathersjs.com", "password": "supersecret" }'
```

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/6bcea48aac6c7494c2ad)

This will return something like this:

```json
{
  "accessToken": "<JWT for this user>",
  "authentication": {
    "strategy": "local"
  },
  "user": {
    "id": "<user id>",
    "email": "hello@feathersjs.com",
    "avatar": "https://s.gravatar.com/avatar/ffe2a09df37d7c646e974a2d2b8d3e03?s=60"
  }
}
```

The `accessToken` can now be used for other REST requests that require authentication by sending the `Authorization: Bearer <accessToken>` HTTP header. For example to create a new message:

```sh
curl 'http://localhost:3030/messages/' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <accessToken>' \
  --data-binary '{ "text": "Hello from the console" }'
```

<BlockQuote type="tip">

Make sure to replace the `<accessToken>` in the above request. For more information about the direct usage of the REST API see the [REST client API](../../api/client/rest.md) and for websockets the [Socket.io client API](../../api/client/socketio.md).

</BlockQuote>

## Login with GitHub

OAuth is an open authentication standard supported by almost every major platform and what gets us the log in with Facebook, Google, GitHub etc. buttons. From the Feathers perspective the authentication flow is pretty similar. Instead of authenticating with the `local` strategy by sending a username and password, Feathers directs the user to authorize the application with the login provider. If it is successful Feathers authentication finds or creates the user in the `users` service with the information it got back from the provider and then issues a token for them.

To allow login with GitHub, first, we have to [create a new OAuth application on GitHub](https://github.com/settings/applications/new). You can put anything in the name, homepage and description fields. The callback URL **must** be set to

```sh
http://localhost:3030/oauth/github/callback
```

![Screenshot of the GitHub application screen](./assets/github-app.png)

<BlockQuote type="info">

You can find your existing applications in the [GitHub OAuth apps developer settings](https://github.com/settings/developers).

</BlockQuote>

Once you clicked "Register application" we have to update our Feathers app configuration with the client id and secret copied from the GitHub application settings:

![Screenshot of the created GitHub application client id and secret](./assets/github-keys.png)

Find the `authentication` section in `config/default.json` replace the `<Client ID>` and `<Client Secret>` in the `github` section with the proper values:

```js
{
  "authentication": {
    "oauth": {
      "github": {
        "key": "<Client ID>",
        "secret": "<Client Secret>"
      }
    },
    // Other authentication configuration is here
    // ...
  }
}
```

This tells the OAuth strategy to redirect back to our index page after a successful login and already makes a basic login with GitHub possible. Because of the changes we made in the `users` service in the [services chapter](./services.md) we do need a small customization though. Instead of only adding `githubId` to a new user when they log in with GitHub we also include their email and the avatar image from the profile we get back. We can do this by extending the standard OAuth strategy and registering it as a GitHub specific one and overwriting the `getEntityData` method:

Update `src/authentication.ts` as follows:

```ts{1,5,14-26,33}
import type { Params } from '@feathersjs/feathers'
import { AuthenticationService, JWTStrategy } from '@feathersjs/authentication'
import { LocalStrategy } from '@feathersjs/authentication-local'
import { oauth, OAuthStrategy } from '@feathersjs/authentication-oauth'
import type { OAuthProfile } from '@feathersjs/authentication-oauth'
import type { Application } from './declarations'

declare module './declarations' {
  interface ServiceTypes {
    authentication: AuthenticationService
  }
}

class GitHubStrategy extends OAuthStrategy {
  async getEntityData(profile: OAuthProfile, existing: any, params: Params) {
    const baseData = await super.getEntityData(profile, existing, params)

    return {
      ...baseData,
      // The GitHub profile image
      avatar: profile.avatar_url,
      // The user email address (if available)
      email: profile.email
    }
  }
}

export const authentication = (app: Application) => {
  const authentication = new AuthenticationService(app)

  authentication.register('jwt', new JWTStrategy())
  authentication.register('local', new LocalStrategy())
  authentication.register('github', new GitHubStrategy())

  app.use('authentication', authentication)
  app.configure(oauth())
}
```

<BlockQuote type="info">

For more information about the OAuth flow and strategy see the [OAuth API documentation](../../api/authentication/oauth.md).

</BlockQuote>

To log in with GitHub, visit

```
http://localhost:3030/oauth/github
```

It will redirect to GitHub and ask to authorize our application. If everything went well, we get redirected to our homepage with the Feathers logo with the token information in the location hash. This will be used by the Feathers authentication client to authenticate our user.

## What's next?

Sweet! We now have an API that can register new users with email/password. This means we have everything we need for a frontend for our chat application. You have your choice of
