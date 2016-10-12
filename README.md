# Zapier Platform CLI

> This is currently pre-release software! You can fill out https://zapier.typeform.com/to/Z4TZBm if you'd like early access.

Zapier is a platform for creating integrations and workflows. This CLI is your gateway to creating custom applications on the Zapier platform.

## Table of Contents

<!-- toc -->

- [Requirements](#requirements)
- [Tutorial](#tutorial)
  * [Installing the CLI](#installing-the-cli)
  * [Starting an App](#starting-an-app)
  * [Your `index.js`](#your-indexjs)
  * [Adding a Trigger](#adding-a-trigger)
  * [Modifying a Trigger](#modifying-a-trigger)
  * [Deploying an App](#deploying-an-app)
  * [Adding Authentication](#adding-authentication)
  * [Tutorial Next Steps](#tutorial-next-steps)
- [Quickstart](#quickstart)
- [Creating a Local App](#creating-a-local-app)
  * [Local Project Structure](#local-project-structure)
  * [Local App Definition](#local-app-definition)
- [Registering an App](#registering-an-app)
- [Deploying an App Version](#deploying-an-app-version)
  * [Private App Version (default)](#private-app-version-default)
  * [Sharing an App Version](#sharing-an-app-version)
  * [Promoting an App Version](#promoting-an-app-version)
- [Authentication](#authentication)
  * [Basic](#basic)
  * [Custom](#custom)
  * [Digest](#digest)
  * [Session](#session)
  * [OAuth2](#oauth2)
- [Resources](#resources)
  * [Resource Definition](#resource-definition)
- [Triggers/Searches/Creates](#triggerssearchescreates)
- [Making HTTP Requests](#making-http-requests)
  * [Shorthand HTTP Requests](#shorthand-http-requests)
  * [Manual HTTP Requests](#manual-http-requests)
    + [POST and PUT Requests](#post-and-put-requests)
  * [Using HTTP middleware](#using-http-middleware)
  * [HTTP Request Options](#http-request-options)
  * [HTTP Response Object](#http-response-object)
- [Z Object](#z-object)
- [Bundle Object](#bundle-object)
- [Environment](#environment)
  * [Defining Environment Variables](#defining-environment-variables)
  * [Accessing Environment Variables](#accessing-environment-variables)
- [Logging](#logging)
  * [Console Log Statements](#console-log-statements)
  * [Viewing Logs](#viewing-logs)
- [Testing](#testing)
  * [Writing Unit Tests](#writing-unit-tests)
  * [Running Unit Tests](#running-unit-tests)
  * [Viewing HTTP Logs in Unit Tests](#viewing-http-logs-in-unit-tests)
- [Example Apps](#example-apps)

<!-- tocstop -->

## Requirements

The Zapier CLI and Platform require Node.js `v4.3.2`. We recommend using [nvm](https://github.com/creationix/nvm) to manage your Node.js installation.

On Mac (via [homebrew](http://brew.sh/)):

```bash
brew install nvm
nvm install v4.3.2
nvm use v4.3.2
```


## Tutorial

Welcome to the Zapier Platform! In this tutorial, we'll walk you through the process of building, testing, and pushing an app to Zapier.


### Installing the CLI

To get started, first make sure that your dev environment meets the [requirements](#requirements) for running the the platform. Once you have the proper version of Node.js, install the Zapier CLI tool.

```bash
# install the CLI globally
npm install -g zapier-platform-cli
```

The CLI is the primary tool for managing your apps. With it, you can validate and test apps locally, push apps so they are available on Zapier, and view logs for debugging. To see a list of all the available commands, try `zapier help`.

Now that your CLI is installed - you'll need to identify yourself via the CLI.

```bash
# auth to Zapier's platform with your deploy key. To obtain a key contact partners@zapier.com
zapier auth
```

Now your CLI is installed and ready to go!


### Starting an App

To begin building an app, use the `init` command to setup the needed structure.

```bash
# create a directory with the minimum required files
zapier init example-app
# move into the new directory
cd example-app
```

Inside the directory, you'll see a few files. `package.json` is a typical requirements file of any Node.js application. It's pre-populated with a few dependencies, most notably the `zapier-platform-core`, which is what makes your app work with the Zapier Platform. There is also an `index.js` file and a test directory (more on those later).

Before we go any further, we need to install the dependencies for our app:

```
npm install
```

### Your `index.js`

Right next to `package.json` should be `index.js`, which is the entry point to your app. This is where the Platform will look for your app definition. Open it up in your editor of choice and let's take a look!

You'll see a few things in `index.js`:

 * we export a single `App` definition which will be interpreted by Zapier
 * in `App` definition, `beforeRequest` & `afterResponse` are hooks into the HTTP client
 * in `App` definition, `triggers` will describe ways to trigger off of data in your app
 * in `App` definition, `searches` will describe ways to find data in your app
 * in `App` definition, `creates` will describe ways to create data in your app
 * in `App` definition, `resources` are purely optional but convenient ways to describe CRUD-like objects in your app

### Adding a Trigger

Let's start by adding a **trigger.** We will configure it to read data from a mocked API:

```bash
mkdir triggers
touch triggers/recipe.js
```

Open `triggers/recipe.js` and paste in:

```javascript
const listRecipes = (z, bundle) => {
  z.console.log('hello from a console log!');
  const promise = z.request('http://57b20fb546b57d1100a3c405.mockapi.io/api/recipes');
  return promise.then((response) => z.JSON.parse(response.content));
};

module.exports = {
  key: 'recipe',
  noun: 'Recipe',
  display: {
    label: 'New Recipe',
    description: 'Trigger when a new recipe is added.'
  },
  operation: {
    perform: listRecipes
  }
};
```

Let's break down what is happening in this snippet!

First, look first at the function definition for `listRecipes`. You see that it handles the API work, making the HTTP request and returning a promise that will eventually yield a result.

It receives two arguments, a `z` object and a `bundle` object.

* The [Z Object](#z-object) is a collection of utilities needed when working with APIs. In our snippet, we use `z.request` to make the HTTP call and `z.JSON` to parse the response.
* The [Bundle Object](#bundle-object) contains any data needed to make API calls, like authentication credentials or data for a POST body. In our snippet the Bundle is not used, since we don't require any of those to make our simple GET request.

> Note about Z Object: While it is possible to accomplish the same tasks using alternate Node.js libraries, it's preferable to use the `z` object as there are features built into these utilities that augment the Zapier experience. For example, logging of HTTP calls and better handling of JSON parsing failures. [Read the docs](#z-object) for more info.

Second, look at the second part of our snippet; the export. Essentially, we export some metadata plus our `listRecipes` function. We'll explain later how Zapier uses this metadata. For now, know that it satisfies the minimum info required to define a trigger.

With our trigger defined, we need to incorporate it into our app.

In `index.js`, edit the file to include:

```javascript
// Place this at the top of index.js
const recipe = require('./triggers/recipe');
```

With the trigger imported, we need to register it on our app by editing the existing `triggers` property.

In `index.js`, edit `App` to include:

```javascript
// Edit the App definition to register our trigger
const App = {
  // ...
  triggers: {
    [recipe.key]: recipe // new line of code
  },
  // ...
};
```

Now, let's add a test to make sure our code is working properly.

In `test/index.js`, replace the file with:

```javascript
const should = require('should');

const zapier = require('zapier-platform-core');

const appTester = zapier.createAppTester(require('../index'));

describe('My App', () => {

  it('should load recipes', (done) => {
    const triggerPointer = 'triggers.recipe';
    const bundle = {};

    appTester(triggerPointer, bundle)
      .then(results => {
        should(results.length).above(1);

        const firstResult = results[0];
        console.log('test result: ', firstResult)
        should(firstResult.name).eql('name 1');
        should(firstResult.directions).eql('directions 1');

        done();
      })
      .catch(done);
  });

});
```

You should be able to run the test with `zapier test` and see it pass:

```
zapier test
#
#   triggers
# 200 GET http://57b20fb546b57d1100a3c405.mockapi.io/api/recipes
#     ✓ should load recipes (312ms)
#
#   1 passing (312ms)
#
```

### Modifying a Trigger

Let's say we want to let our users tweak the cuisine style of recipes they are triggering on. A classic way to do that with Zapier is to provide an input field a user can fill out.

In `triggers/recipe.js`, replace the file with:

```javascript
const listRecipes = (z, bundle) => {
  z.console.log('hello from a console log!');
  const promise = z.request('http://57b20fb546b57d1100a3c405.mockapi.io/api/recipes', {
    // NEW CODE
    params: {
      style: bundle.inputData.style
    }
  });
  return promise.then((response) => JSON.parse(response.content));
};

module.exports = {
  key: 'recipe',
  noun: 'Recipe',
  display: {
    label: 'New Recipe',
    description: 'Trigger when a new recipe is added.'
  },
  operation: {
    // NEW CODE
    inputFields: [
      {key: 'style', type: 'string'}
    ],
    perform: listRecipes
  }
};
```

Notice that we now include and use an input field called "style". We have to add it in two places:

* In the `inputFields` on `operation` - this defines the field as exposed in the Zapier UI. The user will see a form asking them to provide a "Style".
* In the `listRecipes` function - we use the provided style via the bundle `bundle.inputData.style`. Since the field is not required - it could be null!

Since we are developing locally, let's tweak the test to verify everything still works.

In `test/index.js`, replace the file with:

```javascript

const should = require('should');

const zapier = require('zapier-platform-core');

const appTester = zapier.createAppTester(require('../index'));

describe('My App', () => {

  it('should load recipes', (done) => {
    const triggerPointer = 'triggers.recipe';
    const bundle = {
      // NEW CODE
      inputData: {
        style: 'mediterranean'
      }
    };

    appTester(triggerPointer, bundle)
      .then(results => {
        should(results.length).above(1);

        const firstResult = results[0];
        console.log('test result: ', firstResult)
        should(firstResult.name).eql('name 1');
        should(firstResult.directions).eql('directions 1');

        done();
      })
      .catch(done);
  });

});
```

You can run your test again and make sure everything still works:

```
zapier test
#
#   triggers
# 200 GET http://57b20fb546b57d1100a3c405.mockapi.io/api/recipes
#     ✓ should load recipes (312ms)
#
#   1 passing (312ms)
#
```

Looking good locally! Let's move on.

### Deploying an App

So far, everything we have done has been local, on your machine. It's been fun, but we want our app on Zapier.com so we can use it with the thousands of other integrations! To do so, we need to take our working local app and push it to Zapier.

Let's push a version of your app! You can can have many versions of an app, which simplifies making breaking changes and testing in the future. For now, we just need a single version pushed.

> If this is your first time pushing your app version - we will ask you to provide a name so we can register your app - this is a one time thing!

```bash
zapier push
# Preparing to build and upload your app.
#
#   Copying project to temp directory - done!
#   Installing project dependencies - done!
#   Applying entry point file - done!
#   Validating project - done!
#   Building app definition.json - done!
#   Zipping project and dependencies - done!
#   Cleaning up temp directory - done!
#   Uploading version 1.0.0 - done!
#
# Build and upload complete! You should see it in your Zapier editor at https://testing.zapier.com/app/editor now!
```

Now that your app version is properly pushed, log in and visit [https://testing.zapier.com/app/editor](https://testing.zapier.com/app/editor) to create a Zap and check our progress.

You'll see the app listed as an available option for the first step. Selecting it, you'll see the "New Recipe" trigger. At this point, we've come full circle on the trigger definition from earlier. Remember that, as part of the metadata, we defined a `display` property with a label and help text. Those properties control the info you see inside the Zapier UI.

As you click through, you'll see our input field "style" appear, which you can fill out. Once you finish setting up the step and test it, Zapier will run the `listReceipes` function associated with the trigger, which will make the API request and return the result to Zapier. If you are curious to see what HTTP requests Zapier makes at any point, you can use the `zapier logs` command to find out.

```bash
zapier logs --type=http

# The logs of your app "Example App" listed below.
#
# ┌────────┬────────┬────────────────────────────────────────────────────────┬───────────────┬─────────┬──────────────────────────────────────┬───────────────────────────┐
# │ Status │ Method │ URL                                                    │ Querystring   │ Version │ Step                                 │ Timestamp                 │
# ├────────┼────────┼────────────────────────────────────────────────────────┼───────────────┼─────────┼──────────────────────────────────────┼───────────────────────────┤
# │ 200    │ GET    │ http://57b20fb546b57d1100a3c405.mockapi.io/api/recipes │ style=italian │ 1.0.0   │ a9055e16-fc0d-4fb2-a3e6-9d442d1f70e8 │ 2016-09-13T15:11:30-05:00 │
# └────────┴────────┴────────────────────────────────────────────────────────┴───────────────┴─────────┴──────────────────────────────────────┴───────────────────────────┘
#   Most recent logs near the bottom.
```

Good work, we've built a trigger locally and pushed it to Zapier.

### Adding Authentication

Up to this point we've ignored something that is usually crucial to APIs: authentication. Zapier supports a number of different [authentication schemes](#authentication). For our app, we are going to set it up to include an API Key in a header.

The first thing we need to do is define the `authentication` section on the app.

In `index.js`, edit `App` to include:

```javascript
const App = {
  // ...
  authentication: {
    type: 'custom',
    fields: [
      {key: 'apiKey', type: 'string'}
    ],
    test: (z, bundle) => {
      const promise = z.request('http://57b20fb546b57d1100a3c405.mockapi.io/api/me');
      return promise.then((response) => {
        if (response.status !== 200) {
          throw new Error('Invalid API Key');
        }
      });
    }
  },
  // ...
};
```

In the above snippet, we define the two required properties of `authentication`:

* `fields` is where we define our auth fields. This works similar to the `inputFields` of triggers. When users connect their account to Zapier, they'll be prompted to fill in this field, and the value they enter becomes available in the `bundle`.
* `test` is a function used during the account connection process to verify that the user entered valid credentials. The goal of the function is to make an authenticated API request whose response indicates if the credentials are correct. If valid, the test function can return anything. On invalid credentials, the test needs to raise an error.

With that setup, we now need to make sure that our API key is included in all the requests our app makes.

In `index.js`, edit the file to include:

```javascript
// Add this helper function above the App definition
const addApiKeyToHeader = (request, z, bundle) => {
  request.headers['MY-AUTH-HEADER'] = bundle.authData.apiKey;
  return request;
};

// const App = ...
```

Above we define a helper function, `addApiKeyToHeader`, that puts the user-provided API key in a request header called `MY-AUTH-HEADER`. The name chosen is illustrative, it can be whatever the API you are integrating with requires. Alternatively, you could put it in a query parameter instead of a header, and/or encoded it first.

To make our helper function take effect, we need to register it on our app.

In `index.js`, edit `App` to include:

```javascript
const App = {
  // ...
  beforeRequest: [
    addApiKeyToHeader // new line of code
  ],
  // ...
};
```

`beforeRequest` is a list of functions that are called before every HTTP request, letting you add headers, query params, etc. to all outbound requests. In our case, every HTTP request will now have the API key added in a header.

To check our progress, we need to re-push our app.

```bash
zapier push
```

Go back to your Zap at `https://testing.zapier.com`. You'll see a new 'Connect Account' item in your 'New Recipe' trigger. Add an account for our app (enter any value you like for the API key, the mock API does not care).

As soon as you add the account, Zapier will run our app's `authentication.test` function to confirm the credentials are valid.

We can verify the header is present in the request by looking at the logs again.

```bash
zapier logs --type=http --detailed --limit=1

# The logs of your app "Example App" listed below.
# 
# ┌─────────────────────┬────────────────────────────────────────────────────────────────────────────────────────┐
# │ Status              │ 200                                                                                    │
# │ Method              │ GET                                                                                    │
# │ URL                 │ http://57b20fb546b57d1100a3c405.mockapi.io/api/me                                      │
# │ Querystring         │                                                                                        │
# │ Version             │ 1.0.0                                                                                  │
# │ Step                │                                                                                        │
# │ Timestamp           │ 2016-09-16T15:57:51-05:00                                                              │
# │ Request Headers     │ user-agent: Zapier                                                                     │
# │                     │ MY-AUTH-HEADER: :censored:6:b1af149262:                                                │
# │ Request Body        │                                                                                        │
# │ Response Headers    │ server: Cowboy                                                                         │
# │                     │ connection: close                                                                      │
# │                     │ x-powered-by: Express                                                                  │
# │                     │ access-control-allow-origin: *                                                         │
# │                     │ access-control-allow-methods: GET,PUT,POST,DELETE,OPTIONS                              │
# │                     │ access-control-allow-headers: X-Requested-With,Content-Type,Cache-Control,access_token │
# │                     │ content-type: application/json                                                         │
# │                     │ content-length: 59                                                                     │
# │                     │ etag: "-80491811"                                                                      │
# │                     │ vary: Accept-Encoding                                                                  │
# │                     │ date: Fri, 16 Sep 2016 20:57:51 GMT                                                    │
# │                     │ via: 1.1 vegur                                                                         │
# │ Response Body       │ [{"id":"1","createdAt":1473801403,"userName":"userName 1"}]                            │
# └─────────────────────┴────────────────────────────────────────────────────────────────────────────────────────┘
```

You can see from the detailed log that the request included our auth header. The value appears as `:censored:6:b1af149262:`, which is intentional. Zapier does not log authentincation credentials in plain text.

With that, we've successfully added authentication to our app!

### Tutorial Next Steps

Congrats, you've completed the tutorial! At this point we recommend reading up on the [Z Object](#z-object) and [Bundle Object](#bundle-object) to get a better idea of what is possible within the `perform` functions. You can also check out the other [example apps](#example-apps) to see how to incorporate different authentication schemes into your app and how to implement things like searches and creates.


## Quickstart

> Be sure to check the [Requirements](#requirements) before you start! Also, we recommend the [Tutorial](#tutorial) for a more thorough introduction.

First up is installing the CLI and setting up your auth to create a working "Zapier Example" application. It will be private to you and visible in your live [Zap editor](https://testing.zapier.com/app/editor).

```bash
# install the CLI globally
npm install -g zapier-platform-cli

# auth to Zapier's platform with your deploy key. To obtain a key, email partner@zapier.com
zapier auth
```

Your Zapier CLI should be installed and ready to go at this point. Next up, we'll create our first app!

```bash
# create a directory with the minimum required files
zapier init example-app

# move into the new directory
cd example-app

# install all the libraries needed for your app
npm install
```

> Note: there are plenty of templates & example apps to choose from! [View all Example Apps here.](#example-apps)

You should now have a working local app. You can run several local commands to try it out.

```bash
# run the local tests
# the same as npm test
zapier test
```

Next, you'll probably want to upload app to Zapier itself so you can start testing live.

```bash
# push your app to Zapier
zapier push
```

> Go check out our [full CLI reference documentation](docs/cli.md) to see all the other commands!


## Creating a Local App

> Tip: check the [Quickstart](#quickstart) if this is your first time using the platform!

Creating an App can be done entirely locally and they are fairly simple Node.js apps using the standard Node environment and should be completely testable. However, a local app stays local until you `zapier register`.

```bash
# make your folder
mkdir zapier-example
cd zapier-example

# create the needed files from a template
zapier init --template=trigger

# install all the libraries needed for your app
npm install
```

If you'd like to manage your **local App**, use these commands:

* `zapier init --template=resource` - initialize/start a local app project
* `zapier scaffold resource Contact` - auto-injects a new resource, trigger, etc.
* `zapier test` - run the same tests as `npm test`
* `zapier validate` - ensure your app is valid
* `zapier describe` - print some helpful information about your app

### Local Project Structure

In your app's folder, you should see this general recommended structure. The `index.js` is Zapier's entry point to your app. Zapier expects you to export an `App` definition there.

```plain
$ tree .
.
├── README.md
├── index.js
├── package.json
├── triggers
│   └── contact-by-tag.js
├── resources
│   └── Contact.js
├── test
│   ├── basic.js
│   ├── triggers.js
│   └── resources.js
├── build
│   └── build.zip
└── node_modules
    ├── ...
    └── ...
```

### Local App Definition

The core definition of your `App` will look something like this, and is what your `index.js` should provide as the _only_ export:

```javascript
const App = {
  // both version strings are required
  version: require('./package.json').version,
  platformVersion: require('zapier-platform-core').version,

  // see "Authentication" section below
  authentication: {
  },

  // see "Making HTTP Requests" section below
  requestTemplate: {
  },
  beforeRequest: [
  ],
  afterResponse: [
  ],

  // See "Resources" section below
  resources: {
  },

  // See "Triggers/Searches/Creates" section below
  triggers: {
  },
  searches: {
  },
  creates: {
  }
};

module.export = App;

```

> Tip: you can use higher order functions to create any part of your App definition!


## Registering an App

Registering your App with Zapier is a necessary first step which only enables basic administrative functions. It should happen before `zapier push` which is to used to actually expose an App Version in the Zapier interface and editor.

```bash
# register your app
zapier register "Zapier Example"

# list your apps
zapier apps
```

> Note: this doesn't put your app in the editor - see the docs on pushing an App Version to do that!

If you'd like to manage your **App**, use these commands:

* `zapier apps` - list the apps in Zapier you can administer
* `zapier register "Name"` - creates a new app in Zapier
* `zapier link` - lists and links a selected app in Zapier to your current folder
* `zapier history` - print the history of your app
* `zapier collaborate [user@example.com]` - add admins to your app who can push
* `zapier invite [user@example.com]` - add users to try your app before promotion


## Deploying an App Version

An App Version is related to a specific App but is an "immutable" implementation of your app. This makes it easy to run multiple versions for multiple users concurrently. By default, **every App Version is private** but you can `zapier promote` it to production for use by over 1 million Zapier users.

```bash
# push your app version to Zapier
zapier push

# list your versions
zapier versions
```

If you'd like to manage your **Version**, use these commands:

* `zapier versions` - list the versions for the current directory's app
* `zapier push` - push the current version the of current directory's app & version (read from `package.json`)
* `zapier promote [1.0.0]` - mark a version as the "production" version
* `zapier migrate [1.0.0] [1.0.1] [100%]` - move users between versions, regardless of deployment status
* `zapier deprecate [1.0.0] [YYYY-MM-DD]` - mark a version as deprecated, but let users continue to use it (we'll email them)
* `zapier env 1.0.0 [KEY] [value]` - set an environment variable to some value


### Private App Version (default)

A simple `zapier push` will only create the App Version in your editor. No one else using Zapier can see it or use it.


### Sharing an App Version

This is how you would share your app with friends, co-workers or clients. This is perfect for quality assurance, testing with active users or just sharing any app you like.

```bash
# sends an email this user to let them view the app in the UI privately
zapier invite user@example.com

# sends an email this user to let being an admin of the app
zapier collaborate user@example.com
```


### Promoting an App Version

Promotion is how you would share your app with every one of the 1 million+ Zapier users. If this is your first time promoting - you may have to wait for the Zapier team to review and approve your app.

If this isn't the first time you've promoted your app - you might have users on older versions. You can `zapier migrate` to either move users over (which can be dangerous if you have breaking changes). Or, you can `zapier deprecate` to give users some time to move over themselves.

```bash
# promote your app version to all Zapier users
zapier promote 1.0.1

# OPTIONAL - migrate your users between one app version to another
zapier migrate 1.0.0 1.0.1

# OR - mark the old version as deprecated
zapier deprecate 1.0.0 2017-01-01
```


## Authentication

Most applications require some sort of authentication - and Zapier provides a handful of methods for helping your users authenticate with your application. Zapier will provide some of the core behaviors, but you'll likely need to handle the rest.

> Hint: You can access the data tied to your authentication via the `bundle.authData` property in any method called in your app.

### Basic

Useful if your app requires two pieces of information to authentication: `username` and `password` which only the end user can provide. By default, Zapier will do the standard Basic authentication base64 header encoding for you (via an automatically registered middleware).

> Note: if you do the common API Key pattern like `Authorization: Basic APIKEYHERE:x` you should look at the "Custom" authentication method instead.

```javascript
const authentication = {
  type: 'basic',
  // "test" could also be a function
  test: {
    url: 'https://example.com/api/accounts/me.json'
  }
  // you can provide additional fields, but we'll provide `username`/`password` automatically
};

const App = {
  // ...
  authentication: authentication,
  // ...
};

```

### Custom

This is what most "API Key" driven apps should default to using. You'll likely provide some some custom `beforeRequest` middleware or a `requestTemplate` to complete the authentication by adding/computing needed headers.

```javascript
const authentication = {
  type: 'custom',
  // "test" could also be a function
  test: {
    url: 'https://{{bundle.authData.subdomain}}.example.com/api/accounts/me.json'
  },
  fields: [
    {key: 'subdomain', type: 'string', required: true, helpText: 'Found in your browsers address bar after logging in.'},
    {key: 'api_key', type: 'string', required: true, helpText: 'Found on your settings page.'}
  ]
};

const addApiKeyToHeader = (request, z, bundle) => {
  request.headers['X-Subdomain'] = bundle.authData.subdomain;
  const basicHash = Buffer(`${bundle.authData.api_key}:x`).toString('base64');
  request.headers.Authorization = `Basic ${basicHash}`;
  return request;
};

const App = {
  // ...
  authentication: authentication,
  beforeRequest: [
    addApiKeyToHeader,
  ],
  // ...
};

```

### Digest

Very similar to the "Basic" authentication method above, but uses digest authentication instead of Basic authentication.

```javascript
const authentication = {
  type: 'digest',
  // "test" could also be a function
  test: {
    url: 'https://example.com/api/accounts/me.json'
  }
  // you can provide additional fields, but Zapier will provide `username`/`password` automatically
};

const App = {
  // ...
  authentication: authentication,
  // ...
};

```

### Session

TODO.

### OAuth2

Zapier will handle most of the logic around the 3 step OAuth flow, but you'll be required to define how the steps work on your own. You'll also likely want to set your `CLIENT_ID` and `CLIENT_SECRET` as environment variables:

```bash
# setting the environment variables on Zapier.com
$ zapier env 1.0.0 CLIENT_ID 1234
$ zapier env 1.0.0 CLIENT_SECRET abcd

# and when running tests locally, don't forget to define them!
$ CLIENT_ID=1234 CLIENT_SECRET=abcd zapier test
```

Your auth definition would look something like this:

```javascript
const authentication = {
  type: 'oauth2',
  test: {
    url: 'https://example.com/api/accounts/me.json'
  },
  // you can provide additional fields for inclusion in authData
  oauth2Config: {
    // "authorizeUrl" could also be a function returning a string url
    authorizeUrl: {
      method: 'GET',
      url: 'https://example.com/api/oauth2/authorize',
      params: {
        client_id: '{{process.env.CLIENT_ID}}',
        state: '{{bundle.inputData.state}}',
        redirect_uri: '{{bundle.inputData.redirect_uri}}',
        response_type: 'code'
      }
    },
    // Zapier expects a response providing {access_token: 'abcd'}
    // "getAccessToken" could also be a function returning an object
    getAccessToken: {
      method: 'POST',
      url: 'https://example.com/api/v2/oauth2/token',
      body: {
        code: '{{bundle.inputData.code}}',
        client_id: '{{process.env.CLIENT_ID}}',
        client_secret: '{{process.env.CLIENT_SECRET}}',
        redirect_uri: '{{bundle.inputData.redirect_uri}}',
        grant_type: 'authorization_code'
      },
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded'
      }
    }
  }
};

const addBearerHeader = (request, z, bundle) => {
  request.headers.Authorization = `Bearer ${bundle.authData.access_token}`;
  return request;
};

const App = {
  // ...
  authentication: authentication,
  beforeRequest: [
    addBearerHeader,
  ]
  // ...
};

```

## Resources

A `resource` is a representation (as a JavaScript object) of one of the REST resources of your API. Say you have a `/recipes`
endpoint for working with recipes; you can define a recipe resource in your app that will tell Zapier how to do create,
read, and search operations on that resource.

```javascript
const Recipe = {
  // `key` is the unique identifier the Zapier backend references
  key: 'recipe',
  // `noun` is the user-friendly name displayed in the Zapier UI
  noun: 'Recipe',
  // `list` and `create` are just a couple of the methods you can define
  list: {
      //...
  },
  create: {
      //...
  }
};

```

The quickest way to create a resource is with the `zapier scaffold` command:

```bash
zapier scaffold resource "Recipe"
```

This will generate the resource file and add the necessary statements to the `index.js` file to import it.


### Resource Definition

A resource has a few basic properties. The first is the `key`, which allows Zapier to identify the resource on our backend.
The second is the `noun`, the user-friendly name of the resource that is presented to users throughout the Zapier UI.

After those, there is a set of optional properties that tell Zapier what methods can be performed on the resource.
The complete list of available methods can be found in the [Resource Schema Docs](https://github.com/zapier/zapier-platform-schema/blob/master/docs/build/schema.md#resourceschema).
For now, let's focus on two:

 * `list` - Tells Zapier how to fetch a set of this resource. This becomes a Trigger in the Zapier Editor.
 * `create` - Tells Zapier how to create a new instance of the resource. This becomes an Action in the Zapier Editor.

Here is a complete example of what the list method might look like

```javascript
const listRecipesRequest = {
  url: 'http://example.com/recipes'
};

const Recipe = {
  //...
  list: {
    display: {
      label: 'New Recipe',
      description: 'Triggers when a new recipe is added.'
    },
    operation: {
      perform: listRecipesRequest
    }
  }
};

```

The method is made up of two properties, a `display` and an `operation`. The `display` property ([schema](https://github.com/zapier/zapier-platform-schema/blob/master/docs/build/schema.md#basicdisplayschema)) holds the info needed to present the method as an available Trigger in the Zapier Editor. The `operation` ([schema](https://github.com/zapier/zapier-platform-schema/blob/master/docs/build/schema.md#resourceschema)) provides the implementation to make the API call.

Adding a create method looks very similar.

```javascript
const createRecipeRequest = {
  url: 'http://example.com/recipes',
  method: 'POST',
  body: {
    name: 'Baked Falafel',
    style: 'mediterranean'
  }
};

const Recipe = {
  //...
  list: {
    //...
  },
  create: {
    display: {
      label: 'Add Recipe',
      description: 'Adds a new recipe to our cookbook.'
    },
    operation: {
      perform: createRecipeRequest
    }
  }
};

```

Every method you define on a `resource` Zapier converts to the appropriate Trigger, Create, or Search. Our examples
above would result in an app with a New Recipe Trigger and an Add Recipe Create.


## Triggers/Searches/Creates

Triggers, Searches, and Creates are the way an app defines what it is able to do. Triggers read
data into Zapier (i.e. watch for new recipes). Searches locate individual records (find recipe by title). Creates create
new records in your system (add a recipe to the catalog).

The definition for each of these follows the same structure. Here is an example of a trigger:

```javascript
const recipeListRequest = {
  url: 'http://example.com/recipes',
};

const App = {
  //...
  triggers: {
    new_recipe: {
      key: 'new_recipe', // uniquely identifies the trigger
      noun: 'Recipe',    // user-friendly word that is used to refer to the resource
      // `display` controls the presentation in the Zapier Editor
      display: {
        label: 'New Recipe',
        helpText: 'Triggers when a new recipe is added.'
      },
      // `operation` implements the API call used to fetch the data
      operation: {
        perform: recipeListRequest
      }
    },
    another_trigger: {
      // Another trigger definition...
    }
  }
};

```

You can find more details on the definition for each by looking at the [Trigger Schema](https://github.com/zapier/zapier-platform-schema/blob/master/docs/build/schema.md#triggerschema),
[Search Schema](https://github.com/zapier/zapier-platform-schema/blob/master/docs/build/schema.md#searchschema), and [Create Schema](https://github.com/zapier/zapier-platform-schema/blob/master/docs/build/schema.md#createschema).


## Making HTTP Requests

There are two primary ways to make HTTP requests in the Zapier platform:

1. **Shorthand HTTP Requests** - these are simple object literals that make it easy to define simple requests.
1. **Manual HTTP Requests** - this is much less "magic", you use `z.request()` to make the requests and control the response.

There are also a few helper constructs you can use to reduce boilerplate:

1. `requestTemplate` which is an shorthand HTTP request that will be merged with every request.
2. `beforeRequest` middleware which is an array of functions to mutate a request before it is sent.
2. `afterResponse` middleware which is an array of functions to mutate a response before it is completed.


### Shorthand HTTP Requests

For simple HTTP requests that do not require special pre or post processing, you can specify the HTTP options as an object literal in your app definition.

This features:

1. Lazy `{{curly}}` replacement.
2. JSON de-serialization.
3. Automatic non-2xx error raising.

```javascript
const triggerShorthandRequest = {
  method: 'GET',
  url: 'http://{{bundle.authData.subdomain}}.example.com/v2/api/recipes.json',
  params: {
    sort_by: 'id',
    sort_order: 'DESC'
  }
};

const App = {
  // ...
  triggers: {
    example: {
      // ...
      operation: {
        // ...
        perform: triggerShorthandRequest
      }
    }
  }
};

```

In the url above, `{{bundle.authData.subdomain}}` is automatically replaced with the live value from the bundle. If the call returns a non 2xx return code, an error is automatically raised. The response body is automatically parsed as JSON and returned.

An error will be raised if the response is not valid JSON, so _do not use shorthand HTTP requests with non-JSON responses_.

### Manual HTTP Requests

When you need to do custom processing of the response, or need to process non-JSON responses, you can make manual HTTP requests. This approach does not perform any magic - no status code checking, no automatic JSON parsing. Use this method when you need more control. Manual requests do perform lazy `{{curly}}` replacement.

To make a manual HTTP request, use the `request` method of the `z` object:

```javascript
const listExample = (z, bundle) => {
  const customHttpOptions = {
    headers: {
      'my-header': 'from zapier'
    }
  };

  return z.request('http://example.com/api/v2/recipes.json', customHttpOptions)
    .then(response => {
      if (response.status >= 300) {
        throw new Error(`Unexpected status code ${response.status}`);
      }

      const recipes = JSON.parse(response.content);
      // do any custom processing of recipes here...

      return recipes;
    });
};

const App = {
  // ...
  triggers: {
    example: {
      // ...
      operation: {
        // ...
        perform: listExample
      }
    }
  }
};

```

#### POST and PUT Requests

To POST or PUT data to your API you can do this:

```javascript
const App = {
  // ...
  triggers: {
    example: {
      // ...
      operation: {
        // ...
        perform: (z, bundle) => {
          const recipe = {
            name: 'Baked Falafel',
            style: 'mediterranean',
            directions: 'Get some dough....'
          };

          const options = {
            method: 'POST',
            body: JSON.stringify(recipe)
          };

          return z.request('http://example.com/api/v2/recipes.json', options)
            .then(response => {
              if (response.status !== 201) {
                throw new Error(`Unexpected status code ${response.status}`);
              }
            });
        }
      }
    }
  }
};

```

Note that you need to call `JSON.stringify()` before setting the `body`.

### Using HTTP middleware

If you need to process all HTTP requests in a certain way, you may be able to use one of utility HTTP middleware functions, by putting them in your app definition:

```javascript
const addHeader = (request) => {
  request.headers['my-header'] = 'from zapier';
  return request;
};

const mustBe200 = (response) => {
  if (response.status !== 200) {
    throw new Error(`Unexpected status code ${response.status}`);
  }
  return response;
};

const autoParseJson = (response) => {
  response.json = JSON.parse(response.content);
  return response;
};

const App = {
  // ...
  beforeRequest: [
    addHeader,
  ],
  afterRequest: [
    mustBe200,
    autoParseJson,
  ]
  // ...
};

```

A `beforeRequest` middleware function takes a request options object, and returns a (possibly mutated) request object. An `afterResponse` middleware function takes a response object, and returns a (possibly mutated) response object. Middleware functions are executed in the order specified in the app definition, and each subsequent middleware receives the request or response object returned by the previous middleware.

Middleware functions can be asynchronous - just return a promise from the middleware function.

### HTTP Request Options

Shorthand requests and manual `z.request()` calls support the following HTTP options:

* `method`: HTTP method, default is `GET`.
* `headers`: request headers object, format `{'header-key': 'header-value'}`.
* `params`: URL query params object, format `{'query-key': 'query-value'}`.
* `body`: request body, can be a string, buffer, or readable stream.
* `redirect`: set to `manual` to extract redirect headers, `error` to reject redirect, default is `follow`.
* `follow`: maximum redirect count, set to `0` to not follow redirects. default is `20`.
* `compress`: support gzip/deflate content encoding. Set to `false` to disable. Default is `true`.
* `agent`: Node.js `http.Agent` instance, allows custom proxy, certificate etc. Default is `null`.
* `timeout`: request / response timeout in ms. Set to `0` to disable (OS limit still applies), timeout reset on `redirect`. Default is `0` (disabled).
* `size`: maximum response body size in bytes. Set to `0`` to disable. Default is `0` (disabled).

### HTTP Response Object

The response object returned by `z.request()` supports the following fields and methods:

* `status`: The response status code, i.e. `200`, `404`, etc.
* `content`: The raw response body. For JSON you need to call `JSON.parse(response.content)`.
* `headers`: Response headers object. The header keys are all lower case.
* `getHeader`: Retrieve response header, case insensitive: `response.getHeader('My-Header')`
* `options`: The original request options object (see above).

## Z Object

We provide several methods off of the `z` object, which is provided as the first argument to all function calls in your app.

* `request`: An HTTP client with some Zapier-specific goodies. See [Making HTTP Requests](#making-http-requests).
* `console`: Logging console, similar to Node.js `console` but logs remotely, as well as to stdout in tests. See [Log Statements](#console-log-statements)
* `JSON`: Similar to the JSON built-in, but catches errors and produces nicer tracebacks.
* `hash`: Crypto tool for doing things like `z.hash('sha256', 'my password')`
* `errors`: Error classes that you can throw in your code, like `throw new z.errors.HaltedError('...')`
* `dehydrate`: Dehydrate a function
* `dehydrateRequest`: Dehydrate a request
* `dehydrateFile`: Dehydrate a file

## Bundle Object

This object holds the user's auth details and the data to for the API requests.

* `authData` - user-provided authentication data, like `api_key` or `access_token`. [(Read more on authentication)[#authentication]]
* `inputData` - user-provided data for this particular run of the trigger/search/create, as defined by the inputFields. For example:
```javascript
{
  createdBy: 'Bobby Flay'
  style: 'mediterranean'
}
```
* `inputDataRaw` - like `inputData`, but before rendering `{{curlies}}`.
```javascript
{
  createdBy: '{{chef_name}}'
  style: '{{style}}'
}
```

## Environment

Apps can define environment variables that are available when the app's code executes. They work just like environment
variables defined on the command line. They are useful when you have data like an OAuth client ID and secret that you
don't want to commit to source control. Environment variables can also be used as a quick way to toggle between a
a staging and production environment during app development.

It is important to note that **variables are defined on a per-version basis!** When you push a new version, the
existing variables from the previous version are copied, so you don't have to manually add them. However, edits
made to one version's environment will not affect the other versions.

### Defining Environment Variables

To define an environment variable, use the `env` command:

```bash
# Will set the environment variable on Zapier.com
zapier env 1.0.0 MY_SECRET_VALUE 1234
```

You will likely also want to set the value locally for testing.

```bash
export MY_SECRET_VALUE=1234
```

### Accessing Environment Variables

To view existing environment variables, use the `env` command.

```bash
# Will print a table listing the variables for this version
zapier env 1.0.0
```

Within your app, you can access the environment via the standard `process.env` - any values set via local `export` or `zapier env` will be there.

For example, you can access the `process.env` in your perform functions:

```javascript
const listExample = (z, bundle) => {
  const httpOptions = {
    headers: {
      'my-header': process.env.MY_SECRET_VALUE
    }
  };
  return z.request('http://example.com/api/v2/recipes.json', httpOptions);
};

const App = {
  // ...
  triggers: {
    example: {
      // ...
      operation: {
        // ...
        perform: listExample
      }
    }
  }
};

```

## Logging

There are two types of logs for a Zapier app, console logs and HTTP logs. The console logs are created by your app through the use of the `z.console` method ([see below for details](#console-log-statements)). The HTTP logs are created automatically by Zapier whenever your app makes HTTP requests (as long as you use `z.request()` or shorthand request objects).

### Console Log Statements

To manually print a log statement in your code, use `z.console`:

```javascript
  z.console.log('Here are the input fields', bundle.inputData);
```

The `z.console` object has all the same methods and works just like the Node.js [`Console`](https://nodejs.org/dist/latest-v4.x/docs/api/console.html) class - the only difference is we'll log to our distributed datastore and you can view them via `zapier logs` (more below).

### Viewing Logs

To view the logs for your application, use the `zapier logs` command. There are two types of logs, `http` (logged automatically by Zapier on HTTP requests) and `console` (manual logs via `z.console.log()` statements). To see the HTTP logs do:

```
zapier logs --type=http
```
To see detailed http logs including headers, request and response bodies, etc, do:

```
zapier logs --type=http --detailed
```

To see your `z.console` logs do:

```
zapier logs --type=console
```

For more advanced logging options including only displaying the logs for a certain user or app version, look at the help for the logs command:

```
zapier help logs
```

## Testing

You can write unit tests for your Zapier app that run locally, outside of the zapier editor.
You can run these tests in a CI tool like [Travis](https://travis-ci.com/).

### Writing Unit Tests

We recommend using the [Mocha](https://mochajs.org/) testing framework. After running
`zapier init` you should find an example test to start from in the `test` directory.

To

```javascript
// we use should assertions
const should = require('should');
const zapier = require('zapier-platform-core');

// createAppTester() makes it easier to test your app. It takes your
// raw app definition, and returns a function that will test you app.
const appTester = zapier.createAppTester(require('../index'));

describe('triggers', () => {

  describe('new recipe trigger', () => {
    it('should load recipes', (done) => {
      // This is what Zapier will send to your app as input.
      // It contains trigger options the user choice in their zap.
      const bundle = {
        inputData: {
          style: 'mediterranean'
        }
      };

      // Pass appTester the path to the trigger you want to call,
      // and the input bundle. appTester returns a promise for results.
      appTester('triggers.recipe', bundle)
        .then(results => {
          // Make assertions

          results.length.should.eql(10);

          const firstRecipe = results[0];
          firstRecipe.name.should.eql('Baked Falafel');

          done();
        })
        .catch(done);
    });
  });

});

```

### Running Unit Tests

To run all your tests do:

```
zapier test
```

### Viewing HTTP Logs in Unit Tests


When running a unit test via `zapier test`, `z.console` statements and detailed HTTP logs print to `stdout`:

```
zapier test
```

Sometimes you don't want that much logging, for example in a CI test. To suppress the detailed HTTP logs do:

```
zapier test --quiet
```

To also suppress the HTTP summary logs do:

```
zapier test --very-quiet
```

## Example Apps

Check out the following example applications to help you get started:

* [Resource Example](https://github.com/zapier/zapier-platform-example-app-resource) - `zapier init --template=resource`
* [Trigger Example](https://github.com/zapier/zapier-platform-example-app-trigger) - `zapier init --template=trigger`
* [Search Example](https://github.com/zapier/zapier-platform-example-app-search) - `zapier init --template=search`
* [Create Example](https://github.com/zapier/zapier-platform-example-app-create) - `zapier init --template=create`
* [Middleware Example](https://github.com/zapier/zapier-platform-example-app-middleware) - `zapier init --template=middleware`
* [Basic Auth Example](https://github.com/zapier/zapier-platform-example-app-basic-auth) - `zapier init --template=basic-auth`
* [Custom Auth Example](https://github.com/zapier/zapier-platform-example-app-custom-auth) - `zapier init --template=custom-auth`
* [OAuth2 Example](https://github.com/zapier/zapier-platform-example-app-oauth2) - `zapier init --template=oauth2`
