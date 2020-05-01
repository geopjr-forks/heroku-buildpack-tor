# heroku-buildpack-tor

This buildpack sets up a Tor V2 hidden service for your app on Heroku.

**Tested and working as of latest commit.**

## Setup

Create a Heroku app as normal, with any buildpacks you typically use.

Then:

```sh
$ heroku buildpacks:add https://github.com/jtschoonhoven/heroku-buildpack-tor.git

# Pin a specific release if running in production:
$ heroku buildpacks:add https://github.com/jtschoonhoven/heroku-buildpack-tor.git#v0.0.15
```

With the buildpack installed, you'll need to modify your Procfile such that
the hidden service will be setup when the app runs.

Procfile:

```
web: ./tor/bin/run_tor & <cmd you'd normally run>
```

While `web` works just fine, so too will any other dyno type. Use `web`
if you want the app to be accessible generally, as well as over Tor. Use
`<any other type>` (e.g. `foo`), to avoid Heroku's router routing to your app like so:

```
foo: $PORT=8080 ./tor/bin/run_tor & <cmd you'd normally run>
```

Your app will only be accessible over Tor, through your configured
`.onion` address.

## Configuring your torrc

Tor hidden services are configured by a config file called [`torrc`](https://github.com/torproject/tor/blob/master/src/config/torrc.sample.in). They look like this:

```
# torrc

# HiddenServiceDir: where tor looks for "private_key" and "hostname" files
HiddenServiceDir /app/hidden_service

# HiddenServicePort: a VIRTUAL port mapped to the local port for your webserver
HiddenServicePort 80 127.0.0.1:8080

# HiddenServiceVersion: This buildpack currently only supports V2 hidden services
HiddenServiceVersion 2
```

This buildpack assumes that your webserver is already configured to listen on Heroku's assigned `$PORT` and generates a `torrc` for you. If you need more control over your `torrc`, you can override with your own custom file by defining `config/torrc.erb` in your app root. You can copy and paste the [default torrc.erb](https://github.com/jtschoonhoven/heroku-buildpack-tor/blob/master/lib/torrc.erb) to get started.

## Variables

Of course, Tor hidden services require that you provide a private_key and it's
SHA, for the .onion name. You'll need to provide these as env vars:

* `HIDDEN_PRIVATE_KEY`: The contents of a private_key file
* `HIDDEN_DOT_ONION`: The onion name for the private_key.

I recommend [Eschalot](https://github.com/ReclaimYourPrivacy/eschalot) for generating V2 .onion addresses. For example the `HIDDEN_DOT_ONION` address `testxlvhldg55pwd.onion` is derived from the following `HIDDEN_PRIVATE_KEY`:

```
-----BEGIN RSA PRIVATE KEY-----
MIICXQIBAAKBgQDB0iaEYdt0ihL05TS+LEi+VIFuPWh1+tEh/orFF70zFnnMHcJV
hxBb5//f1r46/AY6vAAD/C+S/guKNnc0hD3N5xzT3rh+UKEu/+UDAxhlcm+u2TCD
19ELx0LZUX+gCrAJ5WG1Y/9Yjupu/BjxIINlyN6xDDWfd54DFVrUUbh8YwIEAQ9F
4QKBgESqefFJId80BJAoik5LTyugMuhlgDYV8RSbda8Djugdcyhp82kc9U5nh82N
HUVgnJ0Qa/DwHBoyiiEQIj5U96fW6yYP92WezwpNL6mc0m4P8vl2BUZVELmQJXnz
jiqpQPr9h1COKxS+zVhHDtJXxSHLzputXTxYRaXU7Ztw5EypAkEA7/ARFZ1ky21d
fNEJhlVouiI53xbVA89FXuAw/tQtoNkqCisk7jjgkPfbQZyHKVcsFi0j5D3Fj1o+
4KcNKCiBywJBAM7LwYxA+YpXxLExsvsf7YWnlEgJG4Flw3J1w1QR+v7bGQoA+z6A
boRDXpN6w011SnqZa8ByQ+dBd6JoXdadPMkCQFhY+iWY0V8nlj9pWSuziPR03AtY
Ko0WFEzb+Qfjj3llvuEhVJeU59H6GWLbPyO5qEnnJKk2MYLonhV3qdFuEAsCQENu
MrN4f1TQWmx4cNnNvEwDvxkj3BCxEWzzutGPfRRYQPbUke6Sd3EoaiK0vgS6fyKB
vgRnUrnljZIiXrxJoEECQQDn4LlrB8gQwAZLoEFtXhSzHN1pEnjgFZAeFFHuDPgJ
MeIm0wXs4CYNj9Gx+2hPHfx/v+eoBQNjkHKoebLLqa69
-----END RSA PRIVATE KEY-----
```

**OBVIOUSLY YOU SHOULD NOT ACTUALLY USE THE EXAMPLE KEYPAIR FROM THE README.** I hope that I don't have to say that. Yes, not even for testing.

## Handling HTTPS on Heroku

**IMPORTANT: If your app is using free Heroku hosting at `your-app-name.herokuapp.com` you cannot use the `web:` dyno in your Procfile without doing extra work.**

Apps hosted under `herokuapp.com` are forced to use HTTPS. This is normally a good thing, but it breaks our default `torrc`. You may remember that HTTP requests use port 80 and HTTPS uses port 443. The default `torrc` intentionally does not define a virtual port for port 443, which means any request made using HTTPS will fail. While it _is_ possible to configure your app to handle HTTPS requests itself, you will either have to self-sign your own SSL cert (which no browser will trust) or find a certificate authority that accepts .onion domains and pay them a few hundred dollars for the privilege.

It will be much easier to simply keep your Tor hidden service running on a separate worker dyno (i.e. use anything other than `web:` in your Procfile) but if you absolutely _must_ use a web dyno, you can work around the problem by using a custom `torrc.erb` that listens on nonstandard virtual port (I like 8080). Then your site will be accessible over regular HTTP at `y0uroni0nd0ma1n.onion:8080`.
