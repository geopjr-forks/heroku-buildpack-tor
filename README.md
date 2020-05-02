# heroku-buildpack-tor

This buildpack sets up a Tor V2 hidden service for your app on Heroku.

**Neither Tor nor this buildpack guarantee your privacy. Use at your own risk.**

## Setup

Create a Heroku app as normal, with any buildpacks you typically use.

Then:

```sh
# Add heroku-buildpack-tor as a custom buildpack
$ heroku buildpacks:add https://github.com/jtschoonhoven/heroku-buildpack-tor.git

# Optionally pin a specific version (strongly recommended for production)
$ heroku buildpacks:add https://github.com/jtschoonhoven/heroku-buildpack-tor.git#v0.1.4
```

With the buildpack installed, you'll need to modify your Procfile such that the hidden service will be setup when the app runs.

Procfile:

```
web: ./tor/bin/run_tor & <cmd you'd normally run>
```

While `web` works "just fine" (*see warning below*) , so too will any other dyno name. Use `web` if you want the app to be accessible generally, as well as over Tor. Use `<any other type>` (e.g. `tor`), to avoid Heroku's router routing to your app like so:

```
tor: $PORT=8080 ./tor/bin/run_tor & <cmd you'd normally run>
```

Your app will only be accessible over Tor, through your configured `.onion` address.

##### WARNING
If your app is hosted on the `<appname>.herokuapp.com` domain, you should NOT use a `web` dyno unless you have a plan to circumvent Heroku's automatic HTTPS redirects. More info below.

## Tutorial (Linux/OSX)

*This guide will provide step-by-step instructions for creating a new Tor hidden service from scratch. We'll use create-react-app for simplicity, but your webserver can be whatever you want.*

##### Step 1: Create a New Heroku App

First we need to create a new project and connect it to Heroku. Using `create-react-app` will initialize `git` for us and set up a working webserver. The `mars/create-react-app` buildpack will allow us to run the project on Heroku without any configuration.

```sh
APP_NAME="my-secret-app"
npx create-react-app $APP_NAME  # Create project
cd $APP_NAME
heroku create $APP_NAME --buildpack mars/create-react-app  # Connect to Heroku
git push heroku master  # Deploy to Heroku
```

Three commands and we've already deployed our app to Heroku! Type `heroku open` to see it in your browser.

##### Step 2: Install and Use Buildpack

With our app online, now is a fine time to add heroku-buildpack-tor.

```sh
heroku buildpacks:add https://github.com/jtschoonhoven/heroku-buildpack-tor.git
```

To actually use the buildpack, we'll need to create a [`Procfile`](https://devcenter.heroku.com/articles/procfile). This is the file that tells Heroku what commands to run when the app starts.

```sh
START_SERVER_COMMAND="./tor/bin/run_tor & /bin/boot"
echo "tor: $START_SERVER_COMMAND" > Procfile
```

Let's break that down. `./tor/bin/run_tor` starts Tor. `/bin/boot` comes from [create-react-app-buildpack](https://github.com/mars/create-react-app-buildpack#procfile) and starts the webserver. So the above creates a new Procfile that defines a worker dyno named "tor", and specifies that it should start Tor plus the webserver when the app starts.

Deploy your new Procfile the usual way.

```sh
git add .
git commit -m "run app as Tor hidden service"
git push heroku master  # Deploy changes
```

##### Step 3: Access Your New Tor Hidden Service

In the previous step we created a Procfile with a worker dyno named "tor". Heroku doesn't start worker dynos automatically so we need to do it ourselves:

```sh
heroku ps:scale tor=1  # Start our new webserver
```

Congrats! You have just deployed a Tor hidden service to Heroku. But where is it? Since we didn't specify a .onion address ourselves, Tor went ahead and generated one for us. We can pull it from the logs using `heroku logs --tail`. Scroll up until you see a section that looks like this:

```
[TOR] ===================================================================
[TOR] Starting hidden service at jdiop35tbhgnwlqi.onion
[TOR] You may dangerously print your private_key by running this command:
[TOR] heroku ps:exec --dyno=tor.1 'cat "/app/hidden_service/private_key"'
[TOR] ===================================================================
```

In this case our auto-generated .onion address is `jdiop35tbhgnwlqi.onion`. Plop that address into your Tor Browser and gape in breathless wonder at your brand new Tor service!

##### Step 4: Get a Permanent .onion Address (Optional)

Since you're still not specifying a specific .onion address and private key, Tor will generate a new one for you on each deploy. If you want to make the current address permanent you'll need to follow the instructions in the **Environment Variables** section. Happy hacking!

## Configuring your torrc

Tor hidden services are configured by a config file called a [`torrc`](https://github.com/torproject/tor/blob/master/src/config/torrc.sample.in). A minimal one looks like this:

```
HiddenServiceDir /app/hidden_service
HiddenServicePort 80 127.0.0.1:8080
```

This buildpack assumes that your webserver is already configured to listen on Heroku's assigned `$PORT` and it automatically generates a `torrc` for you. If you need more control over your `torrc`, you can override with your own custom file by defining `config/torrc.erb` in your app root. You can copy and paste the [default torrc.erb](https://github.com/jtschoonhoven/heroku-buildpack-tor/blob/master/lib/torrc.erb) to get started.

## Environment Variables

Unlike other forks, this buildpack works without setting any environment variables. If you don't specify your own, a private key and .onion hostname will be automatically generated for you on each build. However you might find it inconvenient to be assigned a brand new .onion address every time you deploy, so you may optionally set these permanently using the following two environment variables:

* `HIDDEN_PRIVATE_KEY`
* `HIDDEN_DOT_ONION`

For example the `HIDDEN_DOT_ONION` address `testxlvhldg55pwd.onion` is derived from the following `HIDDEN_PRIVATE_KEY`:

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

**OBVIOUSLY YOU SHOULD NOT ACTUALLY USE THE EXAMPLE KEY FROM THE README.** I hope that I don't have to say that. No, not even for testing.

If you want more control over you .onion address, I recommend checking out [Eschalot](https://github.com/ReclaimYourPrivacy/eschalot).

## Handling HTTPS Redirects on Heroku

**TL:DR: If your app is using free Heroku hosting at `<app-name>.herokuapp.com` you cannot use the `web` dyno in your Procfile without doing extra work.**

Apps hosted under `herokuapp.com` are forced to use HTTPS. This is normally a good thing, but it breaks our default `torrc`. You may remember that HTTP requests use port 80 and HTTPS uses port 443. The default `torrc` intentionally does not define a virtual port for port 443, which means any request made using HTTPS will fail. While it _is_ possible to configure your app to handle HTTPS requests itself, you will either have to self-sign your own SSL cert (which no browser will trust) or find a certificate authority that accepts .onion domains and pay them a few hundred dollars for the privilege.

It will be much easier to simply keep your Tor hidden service running on a separate worker dyno (i.e. use anything other than `web` in your Procfile) but if you absolutely _must_ use a web dyno, you can work around the problem by using a custom `torrc.erb` that listens on nonstandard virtual port (I like 8080). Then your site will be accessible over regular HTTP at `y0uroni0nd0ma1n.onion:8080`.
