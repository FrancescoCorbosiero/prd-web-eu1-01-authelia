# Authelia on prd-web-eu1-01

The sign-in in front of the sites on this server. Caddy (caddy-docker-proxy)
asks Authelia about every request to a protected address, before the site
sees it. Everyone signs in with their own name and password. Five wrong
passwords lock that name for 15 minutes, which stops bots guessing.

One Authelia per server, like Caddy. It lives here, cloned once to
`/srv/authelia`. Every site behind it is listed in
[config/configuration.yml](config/configuration.yml). Today that is Store Hub,
for two shops:

| Account | Group | Opens |
| --- | --- | --- |
| the admin (e.g. `operator`) | `operators` | every address below |
| Shoes Clothing Store's (e.g. `shoesclothing`) | `shoesclothingstore` | `hub.shoesclothingstore.com`, `vetrina.shoesclothingstore.com` |
| Resell Piacenza's (e.g. `resellpiacenza`) | `resellpiacenza` | `hub.resellpiacenza.shop`, `vetrina.resellpiacenza.shop` |

The group decides what an account opens, not its name.

A site listed here but not running yet is harmless: nothing reaches Authelia
for it until its container is up behind Caddy.

How a request goes through, and why the sign-in pages live on each address
under `/authelia`: [Store Hub's docs/auth.md](https://github.com/FrancescoCorbosiero/kicks-js-proxy/blob/main/docs/auth.md).

## What is where

| | |
| --- | --- |
| `docker-compose.yml` | Authelia, and the Redis that keeps its sessions |
| `config/configuration.yml` | the settings: which addresses, who may open them, how sign-in works. In git |
| `config/users.yml` | the people. Made by `bin/setup` from `users.example.yml`. **Not in git** |
| `secrets/` | two random keys, made by `bin/setup`. **Not in git** |
| `bin/setup`, `bin/hash`, `bin/check-login` | first run, password hashes, why a sign-in fails |

Authelia's database (bans after wrong passwords) and
`notification.txt` live in the `authelia_data` volume. The sessions live in
`authelia_sessions`.

## Install

Needs caddy-docker-proxy running on the network `caddy`. If yours has another
name, `cp .env.example .env` and set `CADDY_NETWORK`.

```bash
git clone https://github.com/FrancescoCorbosiero/prd-web-eu1-01-authelia /srv/authelia
cd /srv/authelia
bin/setup                    # secrets/ and config/users.yml
bin/hash                     # once per person: type their password, copy the line it prints
nano config/users.yml        # see "Add a person" for each field
docker compose up -d
docker compose logs -f authelia     # wait for "Startup complete", then Ctrl+C
bin/check-login operator     # tests a name and password from the server
```

Keep a copy of `secrets/STORAGE_ENCRYPTION_KEY` off the server: Authelia's
database can't be read without it.

A warning that it could not reach the NTP server is harmless. A warning that
the clock is off is not: fix the server's time.

### Starting over

To throw away an earlier Authelia (its database and sessions) before
installing:

```bash
docker rm -f authelia-authelia-1 authelia-redis-1
docker volume rm authelia_data authelia_sessions
```

Then install again with new secrets (`rm -r secrets` first, if this folder had
some). New secrets can't open the old `authelia_data` volume: its database stays
locked with the old key, and Authelia won't start.

## Add a person

In `config/users.yml`:

```yaml
users:
  francesco:                   # what they type to sign in: lowercase, no spaces
    displayname: Francesco     # what the pages greet them with
    email: you@example.com     # required, even without an email server
    password: '$argon2id$...'  # the line bin/hash printed
    groups:
      - operators              # or the shop's group, e.g. shoesclothingstore
```

It applies at once, with no restart. Then:

1. Give them the address, their name (the key under `users:`, not the email)
   and their password.
2. They sign in and tick **Remember me** (*Ricordami*). With it, the session
   lasts a month (three on the Vetrina), used or not. Without it, an idle hour
   or 12 hours sign them out.

Each address signs in on its own: signing in on the Hub doesn't sign you in
on the Vetrina. To test another account, use a private window, or sign out
first at `/authelia/logout` on that address.

## Sign-in doesn't work

```bash
bin/check-login <name>
```

It checks, in order:

- that Authelia runs;
- that the name is in `users.yml` (names are case-sensitive, and it spots a
  display name or email typed instead);
- that the account isn't disabled;
- that the password field holds a hash;
- the password you type (hidden) against that hash;
- bans after wrong passwords;
- Authelia's own log lines about that person.

If everything is fine but the browser still refuses, look at the page you sign
in on. Authelia's says "Powered by Authelia" under the form. A page with only a
password box is the site's own old login: that address isn't switched to
Authelia yet.

## Day to day

From `/srv/authelia`:

| To | Do |
| --- | --- |
| Remove someone | delete their entry in `config/users.yml`, or add `disabled: true`. Their sessions end at their next click |
| Reset a forgotten password | `bin/hash`, then replace their `password:` line |
| Unlock someone after wrong passwords | it lifts after 15 minutes, or now: `docker compose exec authelia authelia storage bans user revoke <name>` |
| Sign everyone out | `docker compose exec redis redis-cli flushall` |
| Change the settings | commit to `config/configuration.yml`, then here: `git pull && docker compose restart authelia` (nobody is signed out) |
| Check the settings | `docker compose run --rm authelia authelia config validate` |
| Update Authelia | `docker compose pull && docker compose up -d`. The `4.39` tag follows its fixes; read the release notes before moving to `4.40` |
| See what happened | `docker compose logs authelia` |

Five wrong passwords within ten minutes lock that name for 15 minutes. Bans
are per person, not per IP: behind Cloudflare every visitor arrives from
Cloudflare's addresses, and banning one of those would lock out everyone using
it.

When someone changes their own password, Authelia saves `users.yml` again in
its own layout: comments are dropped and every field is listed. That's
expected.

**Back up** the `authelia_data` volume, `secrets/` and `config/users.yml` now
and then, together. Losing them is a chore, not a disaster: everyone
registers again.

## Protect another site

1. In `config/configuration.yml`, add its addresses under `session.cookies`
   (copy an entry) and a rule under `access_control` (copy a shop's, with
   the group that may open it). Commit, then here:
   `git pull && docker compose restart authelia`.
2. Give its people that group in `config/users.yml`.
3. In the site's compose file, put these Caddy labels on its container, with
   its own address and port:

   ```yaml
   caddy: site.example.com
   caddy.handle_0: /authelia*
   caddy.handle_0.reverse_proxy: authelia:9091
   caddy.handle_1.route.0_request_header: "-Remote-*"
   caddy.handle_1.route.1_forward_auth: authelia:9091
   caddy.handle_1.route.1_forward_auth.uri: /api/authz/forward-auth
   caddy.handle_1.route.1_forward_auth.copy_headers: Remote-User Remote-Groups Remote-Email Remote-Name
   caddy.handle_1.route.2_reverse_proxy: "{{upstreams 8080}}"
   ```

   The first route line drops any `Remote-*` header a visitor sends, so only
   Authelia says who someone is. A site that reads `Remote-User` should also
   check that the request came through Caddy, the way Store Hub does with its
   `AUTH_PROXY_SECRET`.

## A second factor (optional)

Authelia can also ask for a second step after the password: a code from an
authenticator app (Google Authenticator, Aegis, 1Password...) or a passkey
(Face ID, a fingerprint, a security key). It's off today. A stolen or guessed
password is then not enough on its own, which matters most for the admin
account, since it opens every shop.

**1. Turn the methods on.** In `config/configuration.yml`, replace the `totp`
and `webauthn` blocks (they say `disable: true`) with:

```yaml
totp:
  issuer: Store Hub            # the name shown in the authenticator app
webauthn:
  display_name: Store Hub
  enable_passkey_login: true   # "Sign in with a passkey" fills in the name too
identity_validation:
  elevated_session:
    # Adding or removing a device: confirm with one already registered.
    require_second_factor: true
    skip_second_factor: true
```

**2. Choose who must use it.** Rules apply top to bottom and the first that
matches wins. For the admin only, add this rule above the shops' rules in
`access_control`, listing every address:

```yaml
    - domain:
        - hub.shoesclothingstore.com
        - vetrina.shoesclothingstore.com
        - hub.resellpiacenza.shop
        - vetrina.resellpiacenza.shop
      subject: 'group:operators'
      policy: two_factor
```

For everyone, change each shop rule's `policy: one_factor` to `two_factor`
instead.

**3. Apply:** commit, then here `git pull && docker compose restart authelia`.

**4. Register a device.** At their next sign-in, those people are asked to
register one (*Registra dispositivo*). The first needs a one-time code "sent
by email". There is no email server, so it's written to a file; it's valid
for 5 minutes:

```bash
docker compose exec authelia tail -n 25 /config/notification.txt
```

A passkey only works on the address it was registered on, so an admin
registers one per address. An authenticator app's codes work on every
address: register both, so a lost phone isn't a lockout.

**A lost phone or passkey:** delete their devices, and they register new ones
at the next sign-in:

```bash
docker compose exec authelia authelia storage user webauthn delete <name> --all
docker compose exec authelia authelia storage user totp delete <name>
```

To turn it off again, put `disable: true` back and the rules to `one_factor`:
registered devices stay in the database, unused.

## Email (optional)

With an SMTP server, people can reset a forgotten password themselves, and
the one-time codes of [a second factor](#a-second-factor-optional) arrive by
email. In `config/configuration.yml`, replace the `notifier`
block:

```yaml
notifier:
  smtp:
    address: submission://smtp.example.com:587
    username: hub@example.com
    sender: Store Hub <hub@example.com>
```

Also set `authentication_backend.password_reset.disable: false`. That needs
two more secrets: the SMTP password, and a key for the reset links
(`openssl rand -hex 64`). Put each in a file in `secrets/`, and add both to
`docker-compose.yml` like the other two, as
`AUTHELIA_NOTIFIER_SMTP_PASSWORD_FILE` and
`AUTHELIA_IDENTITY_VALIDATION_RESET_PASSWORD_JWT_SECRET_FILE`. Then validate
and restart.

## Troubleshooting

- **Every protected address answers 502:** Authelia is down, or not on Caddy's
  network. Check `docker compose ps`; `docker compose logs authelia` says why.
- **Authelia restarts in a loop:** its log names the problem. Usually it's a
  `password:` that isn't a hash yet, or a YAML indentation slip in
  `users.yml`.
- **403 after signing in:** that account's group doesn't open this address.
  Authelia's log names the account (`docker compose logs authelia | grep
  forbidden`): often it's another account still signed in on that address.
  Sign out at `/authelia/logout` and sign in again.
- **The sign-in page comes back after signing in:** the address is missing
  from `session.cookies`, or the request did not arrive over https.
