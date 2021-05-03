# heroku-buildpack-tor

This buildpack sets up a Tor v3 onion service for your application on Heroku. v2 will be disabled [later in 2021](https://blog.torproject.org/v2-deprecation-timeline).

I will be regularly and manually updating the Tor version, which may require you to clear your build cache upon your next git push for your application when I do so. Easy instructions can be found [here](https://help.heroku.com/18PI5RSY/how-do-i-clear-the-build-cache).

This is so that we get to stay current with Tor's bugfixes and patches.

## Disclaimer

This buildpack has a specific hobbyist use case, and is not intended for any use involving gravely serious requirements of anonymity. If you must use Tor for such a reason, please look into other ways of deploying web apps onto the dark web.

## Simplest, easiest way to get running

1. Install the buildpack like you would with any other buildpack
2. Modify your `Procfile` as follows:

    ```
    web: ./tor/bin/run_tor & <your-usual-dyno-cmd>
    ```

3. Deploy your application
4. Run `heroku logs --tail` to see the `.onion` address you generated; with this setup the address will change with each redeploy, but the next section will explain how to set up a persistent address

## To set your own permanent v3 .onion address

1. Obtain the following files for a `.onion` address by using [mkp224o](https://github.com/cathugger/mkp224o)
   - `hostname`
   - `hs_ed25519_public_key`
   - `hs_ed25519_secret_key`
2. Create a new environment variable for your application in Heroku
   - Key: `ONION_LOCATION`
   - Value: the `.onion` address inside the `hostname` file
3. Create a `config` directory in your application's root folder
4. Copy and paste your `hs_ed25519_public_key` and `hs_ed25519_secret_key` files into this new `config` directory
5. Create a `torrc.erb` file in the `config` directory and ensure  `HiddenServiceDir` has the value `/app/onion-service/` ([example](lib/torrc.erb))
6. Modify your `Procfile` as outlined in the previous section if you haven't yet done so
7. Deploy your application

## To use free .herokuapp.com domain

The free `.herokuapp.com` domain automatically redirects to HTTPS, which will normally break your visit to your onionsite, but you can use a custom port to circumvent this.

Add the following to your `torrc.erb` (referred to above):

```
HiddenServicePort <pick-a-port> 127.0.0.1:<your-port>
```

Your onionsite should then be available at `yourv3onionaddress.onion:your-port`

## Miscellaneous

If you get stuck, please defer to [jtschoonhoven's fork](https://github.com/jtschoonhoven/heroku-buildpack-tor) or associated [Medium story](https://medium.com/@jtschoonhoven/deploying-a-tor-hidden-service-to-heroku-in-5-minutes-60bc61583e58) for a much more in-depth tutorial. Most of the instructions apply except for the private_key ones specific to v2 .onion addresses, which I've updated for v3 `.onion` addresses here.
