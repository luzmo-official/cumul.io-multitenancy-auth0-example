# Multi-tenant integration example with use of Auth0

Follow the steps below to setup a simple webapp that displays a [Cumul.io](https://cumul.io) dashboard with multi tenancy. Setting this app will allow you to define rules that determine what each user has access to on your dashboard.

Before you begin, you will need a [Cumul.io](https://cumul.io) account.

## I. Create a dashboard

You can create as many dashboards as you'd like. Then, you should add the dashboards you want to use in this application all into one 'Integration' in Cumul.io. We will use this Integration ID to create an SSO token and embed the dashboards into the application.

## II. Auth0 setup

1.  Create an account [here](https://auth0.com/)

2.  In the Applications menu create a new Application and select Single Page Web Applications and in Settings:

    - copy 'Domain' & 'Client ID' to the same attributes in the `auth_config.json` file

    - set the parameters of:
      > Allowed Callback URLs: `http://localhost:3000`
      > Allowed Logout URLs: `http://localhost:3000`
      > Allowed Web Origins: `http://localhost:3000`
    - Save the changes

    in Connections: deactivate google-oauth2 (to hide social)

3.  In Applications -> APIs copy 'API audience' next to Auth0 Management API to the audience attribute in the `auth_config.json` file

4.  Add some users in User Management -> Users:

    - Go to users & create a user

    - You should add the following properties to the `user_metadata` of a user:

      - `firstName`: will be used to display name in application
      - `language`: will be used to initially show application is language set, currently only "fr" and "en" are supported.
      - `base-color`: will be used to style the sidebar and to use in the dashboards (should be in hex!)
      - `colors`: will be used in the dashboard (should be in hex!)
      - `logoUrl`: will be used in the dashboard (optional: if not specified, it will not be used). Keep in mind that it will replace all image widgets' internal images: in case you want to override a specific image, specify the variable `logo_widget_chart_id` in the `server.js` file.

      An example of the `user_metadata`:

      ```json
      {
        "firstName": "Brad",
        "language": "en",
        "base-color": "#ff784f",
        "colors": [
          "#880065",
          "#b3005e",
          "#b3005e",
          "#ef513e",
          "#fd7b27",
          "#ffa600",
          "#fdae6b"
        ],
        "logoUrl": "https://cumul.io/assets/favicon/logo.svg"
      }
      ```

    - You should add the following properties to the `app_metadata` of a user:
      \_ `parameters`: object containing parameter names and their values. These parameter filters will **ALWAYS** be applied in the authorization token, so e.g. useful for row-level security per client.

          An example of the `app_metadata`:

          ```json
          {
              "parameters": {
                  "<parameter name>": ["<parameter value 1>", "<parameter value 2>"]
              },
              "role": "viewer"
          }
          ```

      (`user_metadata` is meant for user preferences that they could easily change, whereas `app_metadata` is for user information that an admin would control)

5.  In order for the metadata to be able to be extracted from the jwt tokens we need to add a rule.

    - Go to Auth Pipeline -> Rules and create a rule with name 'Add metadata to token' and use the following function:

      ```javascript
      let namespace = "http://namespace.app/";
      function (user, context, callback) {
      user.user_metadata = user.user_metadata || {};
      Object.keys(user.user_metadata).forEach((k) => {
        context.idToken[namespace + k] = user.user_metadata[k];
        context.accessToken[namespace + k] = user.user_metadata[k];
      });
      Object.keys(user.app_metadata).forEach((k) => {
        context.idToken[namespace + k] = user.app_metadata[k];
        context.accessToken[namespace + k] = user.app_metadata[k];
      });
      callback(null, user, context);
      }
      ```

    - copy the namespace value used in the rule function above to the namespace property in the `auth_config.json` file. The namespace is required for Auth0 as an arbitrary identifier (see [here](https://auth0.com/docs/tokens/create-namespaced-custom-claims)).

## III. App install

(This project uses webpack. You may have to run `npx webpack` to begin with.)

`npm install`

Create a file called `.env` in the root directory with two keys. Replace the `CUMULIO_API_KEY` & `CUMULIO_API_TOKEN` with ones from your Cumul.io account. You can create them in your Profile settings under API Tokens. Also add the `INTEGRATION_ID` for the integration you will be using in this app:

```
CUMULIO_API_KEY=XXX
CUMULIO_API_TOKEN=XXX
INTEGRATION_ID=XXX
```

## IV. Adapt `server.js` according to your needs

- Optionally, you can disable custom theming and/or custom css, by setting the `custom_theme` and `custom_css` in `server.js` to false. You can also specify a specific image and/or text widget chart id where the logo and first name of the client should appear, otherwise it will add it to all image and text widgets.

## V. Run the app

1. `npm run start` or if you do not have nodemon, use: `node server.js`
2. each time you add/remove dashboards, or change something to `server.js` you will have to restart the server. Changes to e.g. `public/js/app.js` do not require a server restart, but could be cached on the client side so it could be that you have to hard-refresh your browser in order to see the changes.
