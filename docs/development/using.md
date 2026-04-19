# Using the development environment

This page describes the basic edit/refresh workflows for working with
the Zulip development environment. Generally, the development
environment will automatically update as soon as you save changes
using your editor. Details for work on the [server](#server),
[web app](#web), and [mobile apps](#mobile) are below.

If you're working on authentication methods or need to use the [Zulip
REST API][rest-api], which requires an API key, see [authentication in
the development environment][authentication-dev-server].

## Common

- Zulip's `main` branch moves quickly, and you should rebase
  constantly with, for example,
  `git fetch upstream; git rebase upstream/main` to avoid developing
  on an old version of the Zulip codebase (leading to unnecessary
  merge conflicts).
- Remember to run `tools/provision` to update your development
  environment after switching branches; it will run in under a second
  if no changes are required.
- After making changes, you'll often want to run the
  [linters](../testing/linters.md) and relevant [test
  suites](../testing/testing.md). Consider using our [Git pre-commit
  hook](../git/zulip-tools.md#set-up-git-repo-script) to
  automatically lint whenever you make a commit.
- All of our test suites are designed to support quickly testing just
  a single file or test case, which you should take advantage of to
  save time.
- Many useful development tools, including tools for rebuilding the
  database with different test data, are documented in-app at
  `https://localhost:9991/devtools`.
- If you want to restore your development environment's database to a
  pristine state, you can use `./tools/rebuild-dev-database`.

## Server

- For changes that don't affect the database model, the Zulip
  development environment will automatically detect changes and
  restart:
  - The main Django/Tornado server processes are run on top of
    Django's [manage.py runserver][django-runserver], which will
    automatically restart them when you save changes to Python code
    they use. You can watch this happen in the `run-dev` console
    to make sure the backend has reloaded.
  - The Python queue workers will also automatically restart when you
    save changes, as long as they haven't crashed (which can happen if
    they reloaded into a version with a syntax error).
- If you change the database schema (`zerver/models/*.py`), you'll need
  to use the [Django migrations
  process](../subsystems/schema-migrations.md); see also the [new
  feature tutorial][new-feature-tutorial] for an example.
- While testing server changes, it's helpful to watch the `run-dev`
  console output, which will show tracebacks for any 500 errors your
  Zulip development server encounters (which are probably caused by
  bugs in your code).
- To manually query Zulip's database interactively, use
  `./manage.py shell` or `manage.py dbshell`.
- The database(s) used for the automated tests are independent from
  the one you use for manual testing in the UI, so changes you make to
  the database manually will never affect the automated tests.

## Web

- Once the development server (`run-dev`) is running, you can visit
  <http://localhost:9991/> in your browser.
- By default, the development server homepage just shows a list of the
  users that exist on the server and you can log in as any of them by
  just clicking on a user.
  - This setup saves time for the common case where you want to test
    something other than the login process.
  - You can test the login or registration process by clicking the
    links for the normal login page.
- Most changes will take effect automatically. Details:
  - If you change CSS files, your changes will appear immediately via
    webpack hot module replacement.
  - If you change JavaScript code (`web/src`) or Handlebars
    templates (`web/templates`), the browser window will be
    reloaded automatically.
  - For Jinja2 backend templates (`templates/*`), you'll need to reload
    the browser window to see your changes.
- Any JavaScript exceptions encountered while using the web app in a
  development environment will be displayed as a large notice, so you
  don't need to watch the JavaScript console for exceptions.
- Both Chrome and Firefox have great debuggers, inspectors, and
  profilers in their built-in developer tools.
- `debug.js` has some occasionally useful JavaScript profiling code.

## Mobile

See the mobile project's documentation on [getting set up to develop
and contribute to the mobile app][mobile-development-guide].

## External access to the dev server

By default, the dev server is reachable only from the machine it
runs on: Vagrant forwards port 9991 to the host's loopback
interface, and `run-dev` listens only on `localhost` otherwise.
Several development workflows need reachability from elsewhere, for example:

- **Testing integrations end-to-end.** Most Zulip integrations,
  such as [incoming
  webhooks](../webhooks/incoming-webhooks-overview.md) (for
  services like GitHub, Jira), cloud
  plugins like Zapier, and Python API clients running on another
  machine, work by having an external service send requests into
  your Zulip server.
- **Running the mobile app against your dev server.** A physical
  phone or an emulator needs a network-reachable address for the
  dev server; see the [mobile setup
  guide][mobile-development-guide] for the mobile-specific
  pieces.

:::{important}
The Zulip development environment is not hardened for exposure to
the public internet. Enable external access only for short testing
windows, and turn it off when you are done.
:::

Follow the steps below to expose the dev server to an external
system.

1. **(Vagrant only) Let the Vagrant forwarder listen on all
   interfaces.** Set `HOST_IP_ADDR 0.0.0.0` in
   `~/.zulip-vagrant-config` and run `vagrant reload`. Vagrant
   will then forward port 9991 from every host interface rather
   than just loopback. See [Using a different port for
   Vagrant][vagrant-host-ip] for background. `run-dev` inside the
   guest already listens on all interfaces for the `vagrant` user,
   so no further change is needed there. Direct installs on the
   host OS skip this step and use `--interface=''` at startup
   instead (see step 3).

1. **Give the service a public URL that reaches your development
   server.** Cloud services deliver webhooks only to publicly
   reachable addresses, and most (e.g., GitHub, Stripe) require
   `https://` URLs. Pick one:

   - **Behind NAT (e.g., a laptop):** use a tunneling tool such as
     [ngrok](https://ngrok.com/), [Cloudflare
     Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/),
     or [UltraHook](http://www.ultrahook.com/) (webhook-specific):

     ```bash
     ngrok http 9991
     ```

     The tool prints a public URL (e.g.,
     `https://<subdomain>.ngrok-free.app`); use that host in the
     next step.

   - **Public-IP server:** the server's public address and port
     9991 are already your URL. See [Developing
     remotely](remote.md) for the full remote-server recipe.

1. **Start `run-dev` with `EXTERNAL_HOST` and the flags your setup
   requires.** `EXTERNAL_HOST` tells Zulip what base URL to
   generate (strip any `https://` prefix; drop `:9991` if the
   tunnel serves on the default HTTPS port). Add `--interface=''`
   on a direct install so `run-dev` listens beyond `localhost`.
   Add `--behind-https-proxy` only when an HTTPS tunnel is
   fronting the server; it tells Zulip to generate `https://` URLs
   and to trust cross-origin POSTs from the tunnel. For example, a
   laptop running directly on the host OS behind an ngrok tunnel
   would use:

   ```bash
   EXTERNAL_HOST="<subdomain>.ngrok-free.app" ./tools/run-dev --interface='' --behind-https-proxy
   ```

   Omit either flag that does not apply. A Vagrant guest reaching a
   public-IP integration over plain HTTP uses just
   `EXTERNAL_HOST=<public-host> ./tools/run-dev`.

Before configuring the real third-party service, sanity-check
reachability from outside your dev box, for example, with `curl`
from another network:

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{}' \
  'https://<public-host>/api/v1/external/<integration>?api_key=<api_key>'
```

A `200` response (or a well-formed `400` from the integration's
payload parser) confirms the chain is wired up; a connection
timeout or TLS error points at the `HOST_IP_ADDR`, `--interface`,
or tunnel configuration. Once that works, point the third-party
service at the same URL and trigger an event; the request should
appear in the `run-dev` console, and the resulting message in
Zulip.

[rest-api]: https://zulip.com/api/rest
[authentication-dev-server]: authentication.md
[django-runserver]: https://docs.djangoproject.com/en/5.0/ref/django-admin/#runserver
[new-feature-tutorial]: ../tutorials/new-feature-tutorial.md
[testing-docs]: ../testing/testing.md
[mobile-development-guide]: https://github.com/zulip/zulip-flutter/blob/main/docs/setup.md
[vagrant-host-ip]: setup-recommended.md#using-a-different-port-for-vagrant
