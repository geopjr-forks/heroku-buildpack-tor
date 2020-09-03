# heroku-buildpack-tor

This buildpack sets up a Tor V3 hidden service for your app on Heroku.

I will be regularly manually updating the Tor version, which may require you to clear your build cache upon your next git push for your app when I do so. Easy instructions in this link:

https://help.heroku.com/18PI5RSY/how-do-i-clear-the-build-cache

This is so that we get to stay current with Tor's bugfixes and patches. However, if you do not want to have to keep up with this, you can pin a specific past version of this buildpack to use for yourself:

```
heroku buildpacks:add https://github.com/EJTheSnail/heroku-buildpack-tor.git#v<version-number-here>
```

# Disclaimer

This buildpack has a specific hobbyist use-case scenario, and is not intended for any use involving gravely serious requirements of anonymity. If you must use Tor for such a reason, please look into other ways of deploying web apps onto the dark web.

# Simplest, easiest way to get running

1. Install the buildpack like you would with any other buildpack
2. Modify your procfile as follows:
```
web: ./tor/bin/run_tor & <your usual dyno cmd>
```
2. Deploy your app
3. Run "heroku logs --tail" to see the .onion address you generated

# To set your own permanent V3 .onion address

1. Generate/find your own hostname/hs_ed25519_public_key/hs_ed25519_secret_key files
2. Set the .onion address inside the hostname file as an environment variable named "HIDDEN_DOT_ONION"
3. Create a folder named "config" in your app's root folder
4. Copy and paste your hs_ed25519_public_key/hs_ed25519_secret_key files here, and rename them so that ".erb" is at the end of each
5. Create/put your custom torrc file in this config folder, and also rename it so that ".erb" is at the end.  The HiddenServiceDir should point to /app/hidden_service/
6. Modify your procfile as outlined in the previous section if you haven't yet done so
7. Deploy your app

# To use free .herokuapp.com domain

The free .herokuapp.com domain automatically redirects to HTTPS, but you can use a custom port to circumvent this.
Add the following to your torrc.erb (referred to above):
```
HiddenServicePort <pick-a-port> 127.0.0.1:<your-port>
```
Your onionsite should then be available at yourv3onionaddress.onion:your-port

# Misc

If you get stuck, please feel free to defer to jtschoonhoven's buildpack/fork/Medium story for a much more in-depth tutorial. Most of the instructions apply except for the private_key ones specific to V2 .onion addresses, which I've updated for V3 .onion addresses here.
