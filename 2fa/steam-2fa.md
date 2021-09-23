# Using your standard 2FA app with Steam

*Author: Matt Bryant*

*Source: https://programsareproofs.com/articles/steam_2fa.html*


## Overview

We'll remove your existing Steam Guard, generate a shared secret via the Steam API, add it to your favorite 2FA application and verify that it works, then require the use of 2FA for Steam.

## Implementation

Remove your existing Steam Guard via the mobile app (Steam Guard > Remove Authenticator).

Install the Steam API, potentially after reviewing the code a bit:

```
pip install git+https://github.com/ValvePython/steam#egg=steam
```

*Editor's note: You can replace this step by running (Debian) `apt-get install python3-probuf python3-pip && pip3 install steam`. This is the only way I was able to get this to work.*

Open a Python shell (we'll want to run some code dynamically) and get a connection to steam:

```
$ python
>>> import steam
>>> import steam.webauth as wa
>>> import steam.guard as guard
>>> wa = wa.MobileWebAuth("USERNAME", "PASSWORD")
>>> wa.cli_login()
```

This will wait for you to enter a code that was emailed to you.

Initiate a Steam Authenticator for your account (this won't make it required yet):

```
>>> sa = guard.SteamAuthenticator(backend=wa)
>>> sa.add()
>>> print("Shared secret", sa.secrets["shared_secret"])
>>> print("Revocation code", sa.secrets["revocation_code"])
```

You'll want to save the revocation code somewhere before continuing. If you lose this code, you'll have no way to login to your account.

Now, add this key to your 2FA application of choice. If you are using Aegis, you'll need to convert it to base32 first:

```
>>> import base64
>>> base64.b32encode(base64.b64decode(sa.secrets["shared_secret"])).decode("utf-8")
```

Your 2FA application should now be generating codes, so let's confirm that they're correct. Run the following a few times and make sure the results match your application:

```
>>> sa.get_code()
```

## Enabling 2FA

If everything looks good, let's turn 2FA on for good. You should have received a code via SMS to use here:

```
>>> sa.finalize("SMS CODE HERE")
```

Logout and log back in, at which point you should be able to use your new 2FA app!
