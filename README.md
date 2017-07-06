# Using Webhooks as an Add-on Partner

## Introduction

>We are gradually rolling this feature out to partners, if you would like to be considered for inclusion as we add more partners, please fill out [this form](https://goo.gl/forms/iXY4PFRbFwvQu5ZC2).

Heroku webhooks provide subscription to HTTP notifications when things change. Add-on Partners may track many kinds of events relating to their resources on apps, domains, builds, releases, attachments, dynos, and more.

Webhook integration requires subscribing via the [Unified API](https://devcenter.heroku.com/articles/unified-api) and implementing endpoints to receive events. This document provides details on how to subscribe and receive notifications. We welcome questions, feedback and suggestions via [ecosystem-feedback@heroku.com](mailto:ecosystem-feedback@heroku.com).

>note During the alpha, the webhook endpoints exist must be addressed via a version variant and will not work without setting the `Accept` header to `application/vnd.heroku+json; version=3.webhooks`.

## Creating a Webhook Subscription

Before you will receive any events, you subscribe via:

```
POST api.heroku.com/addons/<uuid>/webhooks
Accept: application/vnd.heroku+json; version=3.webhooks
```
```json
{
  "authorization":  "Bearer 01234567-89ab-cdef-0123-456789abcdef",
  "include":  [
    "api:app",
  ],
  "level":  "sync",
  "url":  "https://myaddon.example.com/webhooks/01234567-89ab-cdef-0123-456789abcdef"
}
```

Replace `<uuid>` with the add-on instance uuid returned during resource [provisioning request](https://devcenter.heroku.com/articles/add-on-partner-api#provision).

Webhook [subscription requests](https://github.com/heroku/webhooks-docs/blob/master/docs/platform-api-reference.md#webhook-create) include these parameters:

### Required

* `include` - One or more event types your server will receive. For instance, [`api:app`](https://devcenter.heroku.com/articles/platform-api-reference#app-info) events provide notification for changes to app name, web url, and more. See below for event options and details.
* `level` - Delivery behavior, either `notify` or `sync`. `notify` provides a single “fire and forget” delivery attempt, while `sync` attempts multiple deliveries until successful or timed out.
* `url` - URL for receiver, should include the UUID identity of add-on instance for disambiguation..

>note Specifics of the `sync` notification level are still under development. As of this writing, `sync` retries 180 times over roughly seven days. Retry delay increases exponentially until it reaches an hour, after which retries occur hourly until the timeout is reached. As deliveries are intended to be ordered, further deliveries for that subscription will be delayed during retries. If this causes the backlog to grow overly large, we may fail additional events. If the overall failure rate grows high enough we might also disable the subscription.
> Delivery status can be checked via the `webhook-deliveries` endpoint, described below.

### Optional

* `authorization` - A secret shared with the receiver. Deliveries will set this as `Authorization` header to allow protection from unauthorized posting. The example uses [the `Bearer` authentication scheme](http://self-issued.info/docs/draft-ietf-oauth-v2-bearer.html), but a string of your choice is allowed.
* `secret` - Value to sign delivery with. Deliveries will set the HMAC-SHA256 of the body using this secret as the `Heroku-Webhook-Hmac-SHA256` header.

>note If the `secret` is omitted, a generated value will be returned in the `Heroku-Webhook-Secret` header. After creation, this value is not repeated, so it must be captured at this point. Otherwise you may PATCH the webhook to update the secret later.

### Partner Event Includes

As of this writing, partners may include these events in subscriptions:

<!--event-json-examples-start-->
- api:addon-attachment [create](docs/event-examples/api-addon-attachment-create.json), [destroy](docs/event-examples/api-addon-attachment-destroy.json)
- api:addon [create](docs/event-examples/api-addon-create.json), [destroy](docs/event-examples/api-addon-destroy.json), [update](docs/event-examples/api-addon-update.json)
- api:app [create](docs/event-examples/api-app-create.json), [destroy](docs/event-examples/api-app-destroy.json), [update](docs/event-examples/api-app-update.json)
- api:build TODO
- api:collaborator [create](docs/event-examples/api-collaborator-create.json), [destroy](docs/event-examples/api-collaborator-destroy.json), [update](docs/event-examples/api-collaborator-update.json)
- api:domain [create](docs/event-examples/api-domain-create.json), [destroy](docs/event-examples/api-domain-destroy.json)
- api:dyno [create](docs/event-examples/api-dyno-create.json)
- api:formation [destroy](docs/event-examples/api-formation-destroy.json), [update](docs/event-examples/api-formation-update.json)
- api:release [create](docs/event-examples/api-release-create.json), [update](docs/event-examples/api-release-update.json)
- api:sni-endpoint [create](docs/event-examples/api-sni-endpoint-create.json), [destroy](docs/event-examples/api-sni-endpoint-destroy.json), [update](docs/event-examples/api-sni-endpoint-update.json)
- api:ssl-endpoint [create](docs/event-examples/api-ssl-endpoint-create.json), [destroy](docs/event-examples/api-ssl-endpoint-destroy.json), [update](docs/event-examples/api-ssl-endpoint-update.json)
<!--event-json-examples-end-->

>note some events require additional permissions, [request access](mailto:ecosystem@heroku.com) to include following events:

* [`api:build`](docs/event-examples/build-list)
* [`api:dyno`](docs/event-examples/dyno-list)
* [`api:formation`](docs/event-examples/formation-list)
* [`api:release`](docs/event-examples/release-list)

These events will be sent when updates happen to the add-on or an app it is associated with, for details see [Webhook Events for Partners](https://github.com/heroku/webhooks-docs/blob/master/docs/events.md)

### Managing Subscriptions

After creating subscriptions, you may need to review or delete them.

You can request the list of webhooks on an add-on:

```
GET api.heroku.com/addons/<uuid>/webhooks
Accept: application/vnd.heroku+json; version=3.webhooks
```
```json
[{
  "addon": {
    "id":  "01234567-89ab-cdef-0123-456789abcdef",
    "name":  "acme-inc-primary-database"
  },
  "created_at":  "2015-01-01T12:00:00Z",
  "id":  "01234567-89ab-cdef-0123-456789abcdef",
  "include":  [
    "api:app",
  ],
  "level":  "sync",
  "url":  "https://myaddon.example.com/webhooks/01234567-89ab-cdef-0123-456789abcdef",
  "updated_at":  "2015-01-01T12:00:00Z",
}]
```

You can look up details for a particular webhook on an add-on:

```
GET api.heroku.com/addons/<uuid>/webhooks/<uuid>
Accept: application/vnd.heroku+json; version=3.webhooks
```
```json
{
  "addon": {
    "id":  "01234567-89ab-cdef-0123-456789abcdef",
    "name":  "acme-inc-primary-database"
  },
  "created_at":  "2015-01-01T12:00:00Z",
  "id":  "01234567-89ab-cdef-0123-456789abcdef",
  "include":  [
    "api:app",
  ],
  "level":  "sync",
  "url":  "https://myaddon.example.com/webhooks/01234567-89ab-cdef-0123-456789abcdef",
  "updated_at":  "2015-01-01T12:00:00Z",
}
```

Finally, you can delete an existing webhook from an add-on:

```json
DELETE api.heroku.com/addons/<uuid>/webhooks/<uuid>
Accept: application/vnd.heroku+json; version=3.webhooks
```
```json
{
  "addon": {
    "id":  "01234567-89ab-cdef-0123-456789abcdef",
    "name":  "acme-inc-primary-database"
  },
  "created_at":  "2015-01-01T12:00:00Z",
  "id":  "01234567-89ab-cdef-0123-456789abcdef",
  "include":  [
    "api:app",
  ],
  "level":  "sync",
  "url":  "https://myaddon.example.com/webhooks/01234567-89ab-cdef-0123-456789abcdef",
  "updated_at":  "2015-01-01T12:00:00Z",
}
```

## Receiving webhooks

When webhook events are matched, they will make a post request to your server. The `Authorization` header will match the `authorization` value from webhook creation and `Heroku-Webhook-Hmac-SHA256` will contain the signature obtained from signing the body with the `secret` value from webhook creation. The result will be a request similar to this:

```
POST myaddon.com/webhooks/01234567-89ab-cdef-0123-456789abcdef
Authorization: Bearer 01234567-89ab-cdef-0123-456789abcdef
Heroku-Webhook-Hmac-SHA256: cLM15U5x+jHEkANnVasRwBw2yEHu94pXdjcT9Hajc1M=
```
```json
{
  "action": "update",
  "actor": {
    "email": "user-0436@example.com",
    "id": "096370c8-f3ad-4e20-a309-fb4f06ec0a89"
  },
  "created_at": "2016-10-26T22:50:14Z",
  "id": "d472a8bb-1a3c-4f78-aad1-995e6d0022ec",
  "data": {
    "archived_at": null,
    "buildpack_provided_description": "Ruby/Rails",
    "build_stack": {
      "id": "5e854079-da33-475b-a209-77f527c0e2fb",
      "name": "cedar-14"
    },
    "created_at": "2016-10-26T22:50:14Z",
    "id": "d4714cc8-aa56-4314-817c-0c6a66ff3d41",
    "git_url": "https://git.heroku.com/sample-app-0301.git",
    "maintenance": false,
    "name": "sample-app-0301",
    "owner": {
      "email": "user-0436@example.com",
      "id": "096370c8-f3ad-4e20-a309-fb4f06ec0a89"
    },
    "region": {
      "id": "7044c05a-0873-42c2-abbe-6841c5481ba7",
      "name": "us"
    },
    "organization": null,
    "space": null,
    "released_at": "2016-10-26T22:50:14Z",
    "repo_size": 1048576,
    "slug_size": null,
    "stack": {
      "id": "5e854079-da33-475b-a209-77f527c0e2fb",
      "name": "cedar-14"
    },
    "updated_at": "2016-10-26T22:50:14Z",
    "web_url": "https://sample-app-0301.herokuapp.com/"
  },
  "previous_data": {
  },
  "published_at": null,
  "resource": "app",
  "sequence": null,
  "updated_at": "2016-10-26T22:50:14Z",
  "version": "application/vnd.heroku+json; version=3",
  "webhook_metadata": {
    "attempt": {
      "id": "8a44f820-2354-489d-9a11-a793cbf49979"
    },
    "delivery": {
      "id": "d244009a-670f-4340-88e9-789a4f9002d5"
    },
    "event": {
      "id": "d472a8bb-1a3c-4f78-aad1-995e6d0022ec",
      "include": "api:app"
    },
    "webhook": {
      "id": "01234567-89ab-cdef-0123-456789abcdef"
    }
  }
}
```

You should always respond with a 200 series status code to indicate delivery success. We will ignore the body of the response, so 204 with an empty body is ideal:

```
204 No Content
```

## Introspecting deliveries

You can list what we have (or had) enqueued to deliver to you via:

```json
GET api.heroku.com/addons/<uuid>/webhook-deliveries/<uuid>
Accept: application/vnd.heroku+json; version=3.webhooks
```
```json
[
  {
    "created_at": "2015-01-01T12:00:00Z",
    "event": {
      "id": "01234567-89ab-cdef-0123-456789abcdef"
    },
    "id": "01234567-89ab-cdef-0123-456789abcdef",
    "status": "pending",
    "updated_at": "2015-01-01T12:00:00Z",
    "webhook": {
      "id": "01234567-89ab-cdef-0123-456789abcdef"
    }
  }
]
```

By looking at `status`, which will be one of `failure`, `pending` or `success` and checking the most recent `updated_at` value you can check to see how many deliveries are queued and how successful delivery has been.

## Introspecting events

## Examples

When subscribing, you may include one or more include values. Here are some concrete examples of specific includes you might use and what to expect from each.

### App Webhooks

To keep track of app details, such as app name changes, subscribe to `api:app` events via:

```
POST api.heroku.com/addons/:uuid/webhooks
Accept: application/vnd.heroku+json; version=3.webhooks
```
```json
{
  "authorization":  "Bearer 01234567-89ab-cdef-0123-456789abcdef",
  "include":  [
    "api:app",
  ],
  "level":  "sync",
  "url":  "https://myaddon.com/webhooks/01234567-89ab-cdef-0123-456789abcdef"
}
```

The response would look something like this:

```
200 OK
Heroku-Webhook-Secret: 0123456789abcdef0123456789abcdef0123456789abcdef0123456789ab
```
```json
{
  "addon": {
    "id": "01234567-89ab-cdef-0123-456789abcdef",
    "name": "acme-inc-primary-database"
  },
  "created_at": "2015-01-01T12:00:00Z",
  "filter": [
    "api:addon-attachment"
  ],
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "include": [
    "release"
  ],
  "level": "notify",
  "updated_at": "2015-01-01T12:00:00Z",
  "url": "example"
}
```

For subsequent formation changes, you would receive something like:

```
POST myaddon.com/webhooks/01234567-89ab-cdef-0123-456789abcdef
Authorization: Bearer 01234567-89ab-cdef-0123-456789abcdef
Heroku-Webhook-Hmac-SHA256: cLM15U5x+jHEkANnVasRwBw2yEHu94pXdjcT9Hajc1M=
```
```json
{
  "action": "update",
  "actor": {
    "email": "user-0436@example.com",
    "id": "096370c8-f3ad-4e20-a309-fb4f06ec0a89"
  },
  "created_at": "2016-10-26T22:50:14Z",
  "id": "d472a8bb-1a3c-4f78-aad1-995e6d0022ec",
  "data": {
    "archived_at": null,
    "buildpack_provided_description": "Ruby/Rails",
    "build_stack": {
      "id": "5e854079-da33-475b-a209-77f527c0e2fb",
      "name": "cedar-14"
    },
    "created_at": "2016-10-26T22:50:14Z",
    "id": "d4714cc8-aa56-4314-817c-0c6a66ff3d41",
    "git_url": "https://git.heroku.com/sample-app-0301.git",
    "maintenance": false,
    "name": "sample-app-0301",
    "owner": {
      "email": "user-0436@example.com",
      "id": "096370c8-f3ad-4e20-a309-fb4f06ec0a89"
    },
    "region": {
      "id": "7044c05a-0873-42c2-abbe-6841c5481ba7",
      "name": "us"
    },
    "organization": null,
    "space": null,
    "released_at": "2016-10-26T22:50:14Z",
    "repo_size": 1048576,
    "slug_size": null,
    "stack": {
      "id": "5e854079-da33-475b-a209-77f527c0e2fb",
      "name": "cedar-14"
    },
    "updated_at": "2016-10-26T22:50:14Z",
    "web_url": "https://sample-app-0301.herokuapp.com/"
  },
  "previous_data": {
  },
  "published_at": null,
  "resource": "app",
  "sequence": null,
  "updated_at": "2016-10-26T22:50:14Z",
  "version": "application/vnd.heroku+json; version=3",
  "webhook_metadata": {
    "attempt": {
      "id": "8a44f820-2354-489d-9a11-a793cbf49979"
    },
    "delivery": {
      "id": "d244009a-670f-4340-88e9-789a4f9002d5"
    },
    "event": {
      "id": "d472a8bb-1a3c-4f78-aad1-995e6d0022ec",
      "include": "api:app"
    },
    "webhook": {
      "id": "01234567-89ab-cdef-0123-456789abcdef"
    }
  }
}
```

You should respond with a 200 series status code to indicate delivery success. We will ignore the body of the response, so in most cases you should respond something like this:

```
204 No Content
```

### Release Webhooks
To keep track of app releases, similar to existing [deployhooks](https://devcenter.heroku.com/articles/deploy-hooks), you can subscribe to `api:release` events via:

```
POST api.heroku.com/addons/:uuid/webhooks
Accept: application/vnd.heroku+json; version=3.webhooks
```
```json
{
  "authorization":  "Bearer 01234567-89ab-cdef-0123-456789abcdef",
  "include":  [
    "api:release",
  ],
  "level":  "sync",
  "url":  "https://myaddon.com/webhooks/01234567-89ab-cdef-0123-456789abcdef"
}
```

The response would look something like this:

```
200 OK
Heroku-Webhook-Secret: 0123456789abcdef0123456789abcdef0123456789abcdef0123456789ab
```
```json
{
  "addon": {
    "id": "01234567-89ab-cdef-0123-456789abcdef",
    "name": "acme-inc-primary-database"
  },
  "created_at": "2015-01-01T12:00:00Z",
  "filter": [
    "api:addon-attachment"
  ],
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "include": [
    "release"
  ],
  "level": "notify",
  "updated_at": "2015-01-01T12:00:00Z",
  "url": "example"
}
```

For subsequent formation changes, you would receive something like:

```
POST myaddon.com/webhooks/01234567-89ab-cdef-0123-456789abcdef
Authorization: Bearer 01234567-89ab-cdef-0123-456789abcdef
Heroku-Webhook-Hmac-SHA256: cLM15U5x+jHEkANnVasRwBw2yEHu94pXdjcT9Hajc1M=
```
```json
{
  "action": "create",
  "actor": {
    "email": "heroku-postgresql@addons.heroku.com",
    "id": "f87b5e7e-4631-46cc-9912-9fcdbbb6c985"
  },
  "created_at": "2016-10-26T22:50:29Z",
  "id": "b6a68e77-8c13-41c8-b30c-b50cca7a608a",
  "data": {
    "addon_plan_names": [
      "heroku-postgresql:hobby-dev"
    ],
    "app": {
      "id": "44dc9c7a-771c-4a9e-bfe9-a8dbc1c55f4d",
      "name": "sample-app-0339"
    },
    "created_at": "2016-10-26T22:50:29Z",
    "description": "Update DATABASE by heroku-postgresql",
    "status": "succeeded",
    "id": "5f1463a4-0210-43ba-b906-52599e246482",
    "slug": null,
    "updated_at": "2016-10-26T22:50:29Z",
    "user": {
      "email": "heroku-postgresql@addons.heroku.com",
      "id": "f87b5e7e-4631-46cc-9912-9fcdbbb6c985"
    },
    "version": 3,
    "current": true
  },
  "previous_data": {
  },
  "published_at": null,
  "resource": "release",
  "sequence": null,
  "updated_at": "2016-10-26T22:50:29Z",
  "version": "application/vnd.heroku+json; version=3",
  "webhook_metadata": {
    "attempt": {
      "id": "8a44f820-2354-489d-9a11-a793cbf49979"
    },
    "delivery": {
      "id": "d244009a-670f-4340-88e9-789a4f9002d5"
    },
    "event": {
      "id": "b6a68e77-8c13-41c8-b30c-b50cca7a608a",
      "include": "api:release"
    },
    "webhook": {
      "id": "01234567-89ab-cdef-0123-456789abcdef"
    }
  }
}
```

You should respond with a 200 series status code to indicate delivery success. We will ignore the body of the response, so in most cases you should respond something like this:

```
204 No Content
```

### Formation Webhooks **(extra permission required)**

To keep track of the quantity and size of running dynos that are attached to an add-on instance, subscribe to `api:formation` events via:

```
POST api.heroku.com/addons/:uuid/webhooks
Accept: application/vnd.heroku+json; version=3.webhooks
```
```json
{
  "authorization":  "Bearer 01234567-89ab-cdef-0123-456789abcdef",
  "include":  [
    "api:formation",
  ],
  "level":  "sync",
  "url":  "https://myaddon.com/webhooks/01234567-89ab-cdef-0123-456789abcdef"
}
```

The response would look something like this:

```
200 OK
Heroku-Webhook-Secret: 0123456789abcdef0123456789abcdef0123456789abcdef0123456789ab
```
```json
{
  "addon": {
    "id": "01234567-89ab-cdef-0123-456789abcdef",
    "name": "acme-inc-primary-database"
  },
  "created_at": "2015-01-01T12:00:00Z",
  "filter": [
    "api:addon-attachment"
  ],
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "include": [
    "release"
  ],
  "level": "notify",
  "updated_at": "2015-01-01T12:00:00Z",
  "url": "example"
}
```

For subsequent formation changes, you would receive something like:

```
POST myaddon.com/webhooks/01234567-89ab-cdef-0123-456789abcdef
Authorization: Bearer 01234567-89ab-cdef-0123-456789abcdef
Heroku-Webhook-Hmac-SHA256: cLM15U5x+jHEkANnVasRwBw2yEHu94pXdjcT9Hajc1M=
```
```json
{
  "action": "update",
  "actor": {
    "email": "user-0297@example.com",
    "id": "2530e242-a531-4d37-99ab-d8997e2335d0"
  },
  "created_at": "2016-10-26T22:49:29Z",
  "id": "89d9e649-1ecf-464e-a15d-86c15365fc40",
  "data": {
    "app": {
      "id": "b225a8d2-1602-42c3-a1a2-98c3c266cc0d",
      "name": "sample-app-0197"
    },
    "command": "ruby server.rb",
    "created_at": "2016-10-26T22:49:29Z",
    "id": "13f27e47-df06-48a1-92be-f521babd9060",
    "type": "web",
    "quantity": 0,
    "size": "1X",
    "updated_at": "2016-10-26T22:49:29Z"
  },
  "previous_data": {
    "quantity": 1
  },
  "published_at": null,
  "resource": "formation",
  "sequence": null,
  "updated_at": "2016-10-26T22:49:29Z",
  "version": "application/vnd.heroku+json; version=3",
  "webhook_metadata": {
    "attempt": {
      "id": "8a44f820-2354-489d-9a11-a793cbf49979"
    },
    "delivery": {
      "id": "d244009a-670f-4340-88e9-789a4f9002d5"
    },
    "event": {
      "id": "89d9e649-1ecf-464e-a15d-86c15365fc40",
      "include": "api:formation"
    },
    "webhook": {
      "id": "01234567-89ab-cdef-0123-456789abcdef"
    }
  }
}
```

You should respond with a 200 series status code to indicate delivery success. We will ignore the body of the response, so in most cases you should respond something like this:

```
204 No Content
```

## FAQ

#### Are collaborator webhooks sufficient to mirror user access of our add-on with the Heroku app?

No. For apps owned by enterprise and team accounts, you also would need to track user roles. We do not expose this functionality as a webhook yet. See [this article](https://devcenter.heroku.com/articles/syncing-user-access-as-an-ecosystem-partner) for a polling-based approach.

Note that mirroring permissions is usually only necessary if you are giving access to your add-on via an additional authentication beyond [the Heroku-provided Add-on SSO](https://devcenter.heroku.com/articles/add-on-single-sign-on).
