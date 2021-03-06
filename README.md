# ♚ [castling.club]

This is the codebase for [castling.club], an [ActivityPub] server with a single
hardcoded 'King' service actor that acts as a chess arbiter.

This code is MIT-licensed, see `LICENSE.md`.

Not a requirement, but if you use this code for anything, I'd love to hear about
it! Send me a toot: [@kosinus@mastodon.social]

## How to run

To run the app, you need [Node.js] (targeting 10.x) and [PostgreSQL] (targeting
9.6.x). [Yarn] is also recommended, but optional.

First, install the Node.js library dependencies:

```sh
# For development:
yarn
# For production:
yarn --prod
# Or use npm, if you don't have Yarn.
```

Next, you should configure the app.

Configuration is provided through environment variables, which can also be
provided in a `.env` file. The defaults may work for development, but usually
need to be tweaked. The following variables are used (with their defaults):

- `APP_SCHEME="http"`

  The scheme used to access the app. The app only talks plain HTTP, but this
  should be set to `https` when behind a reverse proxy providing HTTPS.

- `APP_DOMAIN="chess.test"`

  The domain used to access the app. You should set up a virtual host matching
  this, and have it proxy to the app.

- `APP_ADMIN_URL=""`

  Profile URL of the server admin, publicly accessible through the [NodeInfo]
  API.

- `APP_ADMIN_EMAIL=""`

  Email of the server admin, publicly accessible through the [NodeInfo] API.

- `APP_KEY_FILE="signing-key"`

  Filename of the RSA private key (in PEM format) used to sign activities for
  the 'King' actor. A matching file with a `.pub` suffix MUST be present,
  containing the RSA public key (also in PEM format).

  The default matches the `tools/gen-signing-key.sh` script, and should be fine
  in most cases.

- `APP_HMAC_SECRET="INSECURE"`

  The secret used to sign public data generated by the app that must later be
  verified. The default MUST NOT ever be used in a public instance.

  Currently, this is only used to sign image URLs, so that they cannot be
  tampered with.

  One can be generated with, for example: `head -c 63 /dev/random | base64`

- `NODE_ENV="development"`

  One of `development` or `production`.

  When in `production`, the app will reject federation attempts not over HTTPS,
  or from origins that should not appear on the open Fediverse.

  This variable may also change various behaviour in library dependencies.

- `PORT="5080"`

  The TCP port the app will listen on for HTTP requests.

  The app will always bind on all interfaces, and MUST be properly firewalled in
  a public instance.

- PostgreSQL connection variables follow the standard set defined at:
  https://www.postgresql.org/docs/9.6/static/libpq-envars.html

Now generate the signing key:

```sh
./tools/gen-signing-key.sh
```

Make sure the database exists in PostgreSQL, then create the schema:

```sh
./tools/migrate.js up
```

The app can now be started with:

```sh
./server.js
```

Debug logging is provided by the [debug] module. For development,
castling.club's own debug logging can be enabled using, for example:

```sh
DEBUG='chess:*' ./server.js
```

## Testing interaction in development

With some work, it's possible to run local castling.club, Mastodon and Pleroma
instances talking to each other. A simple setup might have:

- castling.club running as `chess.test`
- Mastodon running as `mastodon.test`
- Pleroma running as `pleroma.test`

To get this working, you want a webserver configured with these virtual hosts
running in front, and correctly reverse proxying to each app. The hosts can then
be defined in your `/etc/hosts` file, or a custom DNS server with a wildcard
`*.test` domain, to point at the local machine (`127.0.0.1` / `::1`).

Both Mastodon and Pleroma perform Webfinger requests over HTTPS. You can either
setup HTTPS for all your local testing instances, or patch either project to use
plain HTTP. ([Mastodon patch], [Pleroma patch]. You MUST NOT ever run such a
patched codebase in a public instance.)

## App structure

The `server.js` entry point loads configuration, then starts the two main tasks
of the app:

- `src/front/` contains the frontend code handling HTTP requests.
- `src/deliver/` contains the outbox delivery queue processing code.

These could easily be split into separate processes, and could also individually
be scaled horizontally, because all state lives in PostgreSQL.  In practice,
that kind of scale is not reached, so things are kept simple and single-process
for now.

The codebase heavily uses async/await, with the [Koa] framework for its
frontend.

Instances of library dependencies and our own services (such as Koa, the
PostgreSQL client, etc.) live on an `app` object. This structure can be best
seen in `src/shared/createApp.js` and the `index.js` files of each task
directory. This is basically poor-man's dependency injection; there's no
automated system to resolve the dependencies between objects, we just create
them in the order we know works.

The castling.club codebase sticks to ActivityPub naming. Notably, this means:
not 'toots', but 'notes'.

When interacting with the bot, there are roughly 6 steps that take place:

1. The user looks up the bot by name.
2. The user publishes a note.
3. The user's instance delivers it to the bot's and others' instances.
4. The bot acts on the note and publishes a reply.
5. We deliver the reply to the users' instances.
6. The users views our reply.

Step 1 involves an optional Webfinger request, and fetching our Actor object.
Both of these are implemented in `src/front/actor.js`. In a server
implementation like Mastodon, these would be dynamic, because there's an account
system. But in castling.club, a single account is simply hardcoded.

Step 3 is when we receive a signed request `POST /inbox`. This is implemented in
`src/front/inbox.js`. The signature is verified by code in
`src/shared/signing.js`, then dispatched as an internal event `noteCreated`.

Step 4 starts when the event is picked up in `src/front/dispatch.js`. Here, the
note is parsed and methods are called on `src/front/game.js` and
`src/front/challengeBoard.js`, which are roughly our controllers when thinking
in MVC. When these decide a reply needs to be sent, they call `createObject()`
of `src/front/outbox.js`, which queues deliveries to user inboxes.

Step 5 is where outbox deliveries are picked up, implemented in
`src/deliver/deliver.js`. Each delivery is a two-step process, where first the
addressed Actor is fetched to discover the inbox, then the note is actually
delivered to the inbox. These are implemented as essentially two separate jobs
folded into a single table `deliveries`. Keeping these separate allows combining
deliveries that have the same shared inbox.

Step 6 is of note, because images are lazily rendered. A signed image URL is
generated in step 4, and when requested, the image is fetched from cache or
rendered on the spot. This code lives in `src/front/draw.js`.

The ActivityPub objects are all accessible by browser as HTML. This is
important, because the actor URL and note URLs are often visible to the user.
Castling.club renders templates for these when accessed through a browser, as
well as providing various other browser-only pages in `src/front/misc.js`.  The
actual template files are [EJS] files, found in `assets/tmpl/`.

[castling.club]: https://castling.club
[activitypub]: http://activitypub.rocks
[@kosinus@mastodon.social]: https://mastodon.social/@kosinus
[node.js]: https://nodejs.org/
[postgresql]: https://www.postgresql.org/
[yarn]: https://yarnpkg.com/
[nodeinfo]: http://nodeinfo.diaspora.software/
[debug]: https://github.com/visionmedia/debug#readme
[mastodon patch]: ./tools/ext/mastodon-plain-http.patch
[pleroma patch]: ./tools/ext/pleroma-plain-http.patch
[koa]: https://koajs.com/
[ejs]: http://ejs.co/
