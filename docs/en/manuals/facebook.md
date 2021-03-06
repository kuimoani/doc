---
title: Defold Facebook manual
brief: This manual explains how to setup and integrate Facebook functionality in a Defold game.
---

# Facebook

The Facebook APIs allows you to interact with Facebook's game connectivity features in a uniform way for games on iOS, Android and HTML5.

The Defold Facebook APIs brings the various platform specific Facebook APIs under a unified set of functions that work the same on iOS, Android and HTML5 (through Facebook Canvas). To get started with Facebook connectivity in your games, you need a Facebook account.

::: important
As of Defold 1.2.92, the Facebook API is redesigned with a new way of interacting with Facebook. The previous API:s still work so no breaking changes are introduced. However, the old API:s should be considered deprecated and not used in new applications.
:::

## Registering as a Facebook developer

To develop for Facebook requires that you sign up as a Facebook developer. This allows you to create Facebook applications that your Defold game can communicate with.

* Head over to [Facebook for developers](https://developers.facebook.com)
* Log in with your Facebook account.
* Follow the instructions to register and verify your developer account.

![Register as a developer](images/facebook/register_dev.png)
![ developer](images/facebook/register_verify.png)

## Creating a Facebook app

The next step is to create a Facebook application. The <kbd>My Apps</kbd> menu in the corner lists your apps, and there is a choice <kbd>Add a New App</kbd>.

![Add new app](images/facebook/add_new_app_menu.png)

You are presented with a selection of target platforms. Click *basic setup* to skip the wizards.

::: sidenote
Most information provided through the wizards are irrelevant when developing on Defold. In particular, you usually don't need to edit *Info.plist* or *AndroidManifest.xml* yourself. Defold does that work for you.
:::

![Add new app platform](images/facebook/add_new_app_platform.png)

You can easily add, remove and change platform settings in the app dashboard. You are now asked to name your app and select *Display Name*, *Namespace* and *Category*. Again, these can all be edited in the app dashboard. When you have gone through this, Facebook creates an app with a unique app identifier for you. The *App ID* is not possible to change since it identifies this specific app.

![New app id](images/facebook/new_app_id.png)

![App dashboard settings](images/facebook/add_platform.png)

Click the *Settings* tab. Notice the numerical *App ID*. That identifier needs to go into the [project settings](/manuals/project-settings) of your Defold game. Unfortunately this setting is hidden in the editor (this will change soon), but you easily add it. Right-click the *game.project* file in the *Project Explorer* and select <kbd>Open With ▸ Text Editor</kbd>.

![Open project settings with](images/facebook/project_open_with.png)

Add a section `[facebook]` and an entry `appid = 456788687846098` but with your application's *App ID*. Make sure you get the number right. Then save the file.

![Game project](images/facebook/game_project.png)

Now, back in the *Settings* tab on the Facebook app page, click *+ Add Platform* to add a new platform to the app. Each platform has a set of settings to fill in.

![Select platform](images/facebook/select_platform.png)

## iOS

For iOS you need to specify the game's `bundle_identifier` as specified in *game.project*.

![iOS settings](images/facebook/settings_ios.png)

## Android

For Android you need to specify *Google Play Package Name*, which is the game's *package* identifier specified in *game.project*. You should also generate hashes of the certificate(s) you use and enter them into the *Key Hashes* field. You can generate a hash from a *certificate.pem* with openssl:

```sh
$ cat certificate.pem | openssl x509 -outform der | openssl sha1 -binary | openssl base64
```

(See [Creating certificates and keys](/manuals/android/#_creating_certificates_and_keys) in the Android manual for details how to create your own signing files)

![Android settings](images/facebook/settings_android.png)

## Facebook Canvas

For HTML5 games, the process is a bit different. Facebook needs access to your game content from somewhere. There are two options:

![Facebook Canvas settings](images/facebook/settings_canvas.png)

1. Use Facebook's *Simple Application Hosting*. Click *Yes* to select managed hosting. Select *uploaded assets* to open the hosted asset manager.

    ![Simple hosting](images/facebook/simple_hosting.png)
    
    Select that you want to host a "HTML5 Bundle":
    
    ![HTML5 bundle](images/facebook/html5_bundle.png)
    
    Compress your HTML5 bundle into a .7z or .zip archive and upload it to Facebook. Make sure to click *Push to production* to start serving the game.

2. The alternative to Facebook hosting is to upload a HTML5 bundle of your game to some server of your choice that serves the game through HTTPS. Set the *Secure Canvas URL* to the URL of your game.

The game now works through the Facebook URL provided as *Canvas Page*.

## Testing the setup

The following basic test can be used to see if things are set up properly.

1. Create a new game object and attach a script component with a new script file to it.
2. Enter the following code in the script file:

```lua
local function get_me_callback(self, id, response)
    -- The response table includes all the response data
    pprint(response)
end

local function fb_login(self, data)
    if data.status == facebook.STATE_OPEN then
        -- Logged in ok. Let's try reading some "me" data through the
        -- HTTP graph API.
        local token = facebook.access_token()
        local url = "https://graph.facebook.com/me/?access_token=" .. token
        http.request(url, "GET", get_me_callback)
    elseif data.status == facebook.STATE_CLOSED_LOGIN_FAILED then
        -- Do something to indicate that login failed
    end
    if data.error then
        -- An error occurred
    else
        -- No error
    end
end

function init(self)
    -- Log in with read permissions.
    local permissions = { "public_profile", "email" }
    facebook.login_with_read_permissions(permissions, fb_login)
end
```

Running this simple test should display something like the following in the console:

```txt
DEBUG:SCRIPT: 
{
  status = 200,
  headers = {
    connection = keep-alive,
    date = Fri, 04 Nov 2016 13:54:33 GMT,
    etag = "0725a4f703fe6af27da183cfec0bb22637e331e0",
    access-control-allow-origin = *,
    content-length = 53,
    expires = Sat, 01 Jan 2000 00:00:00 GMT,
    content-type = text/javascript; charset=UTF-8,
    x-fb-debug = Pr1qUssb8Xa3x3r1t913hHMdefh69DSYYV5vcxeOB7O33mcfShIw+r7BoLpn147I2wzLF2CZRTpnR3/VYOtFpA==,
    facebook-api-version = v2.5,
    cache-control = private, no-cache, no-store, must-revalidate,
    pragma = no-cache,
    x-fb-trace-id = F03S5dtsdaS,
    x-fb-rev = 2664414,
  }
  response = {"name":"Max de Fold ","id":"14159265358979323"},
}
```

* The full Defold Facebook APIs are documented in the [Facebook reference documentation](/ref/facebook).
* The Facebook Graph API is documented here: https://developers.facebook.com/docs/graph-api

## Development caveats

When developing it is very convenient to use the dev application. Unfortunately, the Facebook API does not yet work on the dev app due to how the bundled *Info.plist* file has to be constructed. However, any debug bundle works as a dev app so the workaround is to build the game with the proper Facebook project settings, put it on the device and then connect to the running game and stream data to it from the editor as usual.

