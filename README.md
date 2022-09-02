# Steam Session Manager

[![npm version](https://img.shields.io/npm/v/steam-session.svg)](https://npmjs.com/package/steam-session)
[![npm downloads](https://img.shields.io/npm/dm/steam-session.svg)](https://npmjs.com/package/steam-session)
[![license](https://img.shields.io/npm/l/steam-session.svg)](https://github.com/DoctorMcKay/node-steam-session/blob/master/LICENSE)
[![paypal](https://img.shields.io/badge/paypal-donate-yellow.svg)](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=N36YVAT42CZ4G&item_name=node%2dsteam%2dsession&currency_code=USD)

This module enables you to negotiate Steam tokens by authenticating with the Steam login server. **This is for use with
your own accounts.** This is not to be used to authenticate other Steam users or to gain access to their accounts. For
that use-case, please use the [Steam OpenID service](https://steamcommunity.com/dev) and the many available
[WebAPIs](https://steamapi.xpaw.me/).

Node.js v12.22.0 or later is required to use this module.

# Concepts

Logging into Steam is a two-step process.

1. You start a login session either using your account credentials (username and password) or by generating a QR code
	- Use [`startWithCredentials`](#startwithcredentialsdetails) to start a login session using your account credentials
2. Assuming any credentials you provided when you started the session were correct, Steam replies with a list of login guards
	- See [EAuthSessionGuardType](https://github.com/DoctorMcKay/node-steam-session/blob/master/src/enums-steam/EAuthSessionGuardType.ts)
	- If your account doesn't have Steam Guard enabled or you provided a valid code upfront, there may be 0 guards required
    - Only one guard must be satisfied to complete the login. For example, you might be given a choice of providing a TOTP code or confirming the login in your Steam mobile app
3. When you satisfy any guards, Steam sends back an access token and a refresh token. These can be used to:
    - Log on with node-steam-user
    - [Obtain web session cookies](#getwebcookies)
    - Authenticate with WebAPI methods used by the mobile app

# Example Code

See the [examples directory on GitHub](https://github.com/DoctorMcKay/node-steam-session/tree/master/examples) for example code.

# Exports

When using CommonJS (`require()`), steam-session exports an object. When using ES6 modules (`import`), steam-session does
not offer a default export and you will need to import specific things.

The majority of steam-session consumers will only care about the [`LoginSession`](#loginsession) class and enums.

## Enums

### EAuthSessionGuardType

```js
const {EAuthSessionGuardType} = require('steam-session');
import {EAuthSessionGuardType} from 'steam-session';
```

[View on GitHub](https://github.com/DoctorMcKay/node-steam-session/blob/master/src/enums-steam/EAuthSessionGuardType.ts)

Contains the possible auth session guards.

### EAuthTokenPlatformType

```js
const {EAuthTokenPlatformType} = require('steam-session');
import {EAuthTokenPlatformType} from 'steam-session';
```

[View on GitHub](https://github.com/DoctorMcKay/node-steam-session/blob/master/src/enums-steam/EAuthTokenPlatformType.ts)

Contains the different platform types that can be authenticated for. You should specify the correct platform type when
you instantiate a [`LoginSession`](#loginsession) object.

### EResult

```js
const {EResult} = require('steam-session');
import {EResult} from 'steam-session';
```

[View on GitHub](https://github.com/DoctorMcKay/node-steam-session/blob/master/src/enums-steam/EResult.ts)

Contains possible result codes. This is a very large enum that used throughout Steam, so most values in this enum will
not be relevant when authenticating.

### ESessionPersistence

```js
const {ESessionPersistence} = require('steam-session');
import {ESessionPersistence} from 'steam-session';
```

[View on GitHub](https://github.com/DoctorMcKay/node-steam-session/blob/master/src/enums-steam/ESessionPersistence.ts)

Contains possible persistence levels for auth sessions.

## Custom Transports

It's possible to define a custom transport to be used when interacting with the Steam login server. By default, the
standard `WebApiTransport` will be used to interact with the Steam login server using api.steampowered.com. **It is very
likely that you won't need to mess with this.**

Everything in this category is TypeScript interfaces, so even if you're implementing a custom transport, you don't need
these unless you're using TypeScript.

```js
const {ITransport, ApiRequest, ApiResponse} = require('steam-session');
import {ITransport, ApiRequest, ApiResponse} from 'steam-session';
```

[View on GitHub](https://github.com/DoctorMcKay/node-steam-session/blob/master/src/transports/ITransport.ts)

# LoginSession

```js
const {LoginSession} = require('steam-session');
import {LoginSession} from 'steam-session';
```

The `LoginSession` class is the primary way to interact with steam-session.

## Properties

### steamID

**Read-only.** A [`SteamID`](https://www.npmjs.com/package/steamid) instance containing the SteamID for the
currently-authenticated account. Populated immediately after [`startWithCredentials`](#startwithcredentialsdetails)
resolves, or immediately after [`accessToken`](#accesstoken) or [`refreshToken`](#refreshtoken) are set.

### loginTimeout

A `number` specifying the time, in milliseconds, before a login attempt will [`timeout`](#timeout). The timer begins
after [`polling`](#polling) begins.

### accountName

**Read-only.** A `string` containing your account name. This is populated just before the [`authenticated`](#authenticated)
event is fired.

### accessToken

A `string` containing your access token. This is populated just before the [`authenticated`](#authenticated) event is
fired. You can also assign an access token to this property if you already have one, although at present that wouldn't
do anything useful.

Setting this property will throw an Error if:

- You set it to a token that isn't well-formed, or
- You have already called [`startWithCredentials`](#startwithcredentialsdetails) and you set it to a token that doesn't belong to the same account, or
- You have already set [`refreshToken`](#refreshtoken) and you set this to a token that doesn't belong to the same account as the refresh token

### refreshToken

A `string` containing your refresh token. This is populated just before the [`authenticated`](#authenticated) event is
fired. You can also assign a refresh token to this property if you already have one.

Setting this property will throw an Error if:

- You set it to a token that isn't well-formed, or
- You have already called [`startWithCredentials`](#startwithcredentialsdetails) and you set it to a token that doesn't belong to the same account, or
- You have already set [`accessToken`](#accesstoken) and you set this to a token that doesn't belong to the same account as the access token

## Methods

### Constructor([platformType][, transport])
- `platformType` - A value from [`EAuthTokenPlatformType`](#eauthtokenplatformtype). You should set this to the
	appropriate platform type for your desired usage. If omitted, defaults to `WebBrowser`.
- `transport` - An `ITransport` instance, if you need to specify a [custom transport](#custom-transports).
	If omitted, defaults to a `WebApiTransport` instance. In all likelihood, you don't need to use this.

Constructs a new `LoginSession` instance. Example usage:

```js
import {LoginSession, EAuthTokenPlatformType} from 'steam-session';

let session = new LoginSession(EAuthTokenPlatformType.WebBrowser);
```

### startWithCredentials(details)
- `details` - An object with these properties:
	- `accountName` - Your account's login name, as a string
    - `password` - Your account's password, as a string
    - `deviceFriendlyName` - Optional. A name to identify this device. Defaults to the Chrome user-agent.
    - `persistence` - Optional. A value from [ESessionPersistence](#esessionpersistence). Defaults to `Persistent`.
    - `websiteId` - Optional. A string containing a valid website ID.
    - `steamGuardMachineToken` - Optional. If you have a valid Steam Guard machine token, supplying it here will allow
      you to bypass email code verification.
    - `steamGuardCode` - Optional. If you have a valid Steam Guard code (either email or TOTP), supplying it here will
      attempt to use it during login.

Starts a new login attempt using your account credentials. Returns a Promise.

On failure, the Promise will be rejected with its message being equal to the string representation of an [EResult](#eresult)
value. There will also be an `eresult` property on the Error object equal to the numeric representation of the relevant
EResult value. For example:

```
Error: InvalidPassword
  eresult: 5
```

On success, the Promise will be resolved with an object containing these properties:

- `actionRequired` - A boolean indicating whether action is required from you to continue this login attempt.
  If false, you should expect for [`authenticated`](#authenticated) to be emitted shortly.
- `validActions` - If `actionRequired` is true, this is an array of objects indicating which actions you could take to
  continue this login attempt. Each object has these properties:
	- `type` - A value from [EAuthSessionGuardType](#eauthsessionguardtype)
    - `detail` - An optional string containing more details about this guard option. Right now, the only known use for
      this is that it contains your email address' domain for `EAuthSessionGuardType.EmailCode`.

Here's a list of which guard types might be present in this method's response, and how you should proceed:

- `EmailCode`: An email was sent to you containing a code (`detail` contains your email address' domain, e.g. `gmail.com`).
  You should get that code and either call [`submitSteamGuardCode`](#submitsteamguardcodeauthcode), or create a new
  `LoginSession` and supply that code to the `steamGuardCode` property when calling [`startWithCredentials`](#startwithcredentialsdetails).
- `DeviceCode`: You need to supply a TOTP code from your mobile authenticator (or by using [steam-totp](https://www.npmjs.com/package/steam-totp)).
  Get that code and either call [`submitSteamGuardCode`](#submitsteamguardcodeauthcode), or create a new `LoginSession`
  and supply that code to the `steamGuardCode` property when calling [`startWithCredentials`](#startwithcredentialsdetails).
- `DeviceConfirmation`: You need to approve the confirmation prompt in your Steam mobile app. If this guard type is
  present, [polling](#polling) will start and [`loginTimeout`](#logintimeout) will be in effect.
- `EmailConfirmation`: You need to approve the confirmation email sent to you. If this guard type is
  present, [polling](#polling) will start and [`loginTimeout`](#logintimeout) will be in effect.

Note that multiple guard types might be available, for example both `DeviceCode` and `DeviceConfirmation` can be
available at the same time.

When this method resolves, [`steamID`](#steamid) will be populated.

### submitSteamGuardCode(authCode)
- `authCode` - Your Steam Guard code, as a string

If a Steam Guard code is needed, you can supply it using this method. Returns a Promise.

On failure, the Promise will be rejected with its message being equal to the string representation of an [EResult](#eresult)
value. There will also be an `eresult` property on the Error object equal to the numeric representation of the relevant
EResult value. For example:

```
Error: TwoFactorCodeMismatch
  eresult: 88
```

Note that an incorrect email code will fail with EResult value InvalidLoginAuthCode (65), and an incorrect TOTP code
will fail with EResult value TwoFactorCodeMismatch (88).

On success, the Promise will be resolved with no value. In this case, you should expect for [`authenticated`](#authenticated)
to be emitted shortly.

### cancelLoginAttempt()

Cancels [polling](#polling) for an ongoing login attempt. Once canceled, you should no longer interact with this
`LoginSession` object, and you should create a new one if you want to start a new attempt.

### getWebCookies()

Once successfully [authenticated](#authenticated), you can call this method to get cookies for use on the Steam websites.
You can also manually set [`refreshToken`](#refreshtoken) and then call this method without going through another login
attempt if you already have a valid refresh token. Returns a Promise.

On failure, the Promise will be rejected. Depending on the nature of the failure, an EResult may or may not be available.

On success, the Promise will be resolved with an array of strings. Each string contains a cookie, e.g.
`'steamLoginSecure=blahblahblahblah'`.

## Events

### polling

This event is emitted once we start polling Steam to periodically check if the login attempt has succeeded or not.
Polling starts when any of these conditions are met:

- A login session is successfully started with credentials and no guard is required (e.g. Steam Guard is disabled)*
- A login session is successfully started with credentials and you supplied a valid code to `steamGuardCode`*
- A login session is successfully started with credentials, you're using email Steam Guard, and you supplied a valid `steamGuardMachineToken`*
- A login session is successfully started with credentials, then you supplied a valid code to [`submitSteamGuardCode`](#submitsteamguardcodeauthcode)*
- A login session is successfully started, and `DeviceConfirmation` or `EmailConfirmation` are among the valid guards

\* = in these cases, we expect to only have to poll once before login succeeds.

After this event is emitted, if your [`loginTimeout`](#logintimeout) elapses and the login attempt has not yet succeeded,
[`timeout`](#timeout) is emitted and the login attempt is abandoned. You would then need to start a new login attempt
using a fresh `LoginSession` object.

### timeout

This event is emitted when the time specified by [`loginTimeout`](#logintimeout) elapses after [polling](#polling) begins,
and the login attempt has not yet succeeded. When `timeout` is emitted, [`cancelLoginAttempt`](#cancelloginattempt) is
called internally.

### remoteInteraction

This event is emitted when Steam reports a "remote interaction" via [polling](#polling). This is observed to happen
when the approval prompt is viewed in the Steam mobile app for the `DeviceConfirmation` guard.

### authenticated

This event is emitted when we successfully authenticate with Steam. At this point, [`accountName`](#accountname),
[`accessToken`](#accesstoken), and [`refreshToken`](#refreshtoken) are populated. If the [EAuthTokenPlatformType](#eauthtokenplatformtype)
passed to the [constructor](#constructorplatformtype-transport) is appropriate, you can now safely call [`getWebCookies`](#getwebcookies).

### error

This event is emitted if we encounter an error while [polling](#polling). The first argument to the event handler is
an Error object. If this happens, the login attempt has failed and will need to be retried.

Node.js will crash if this event is emitted and not handled.
