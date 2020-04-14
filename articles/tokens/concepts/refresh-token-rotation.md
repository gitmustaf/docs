---
description: Understand how Refresh Token rotation provides greater security by issuing a new Refresh Token with each request made to Auth0 for a new Access Token by a client using Refresh Tokens.
topics:
  - tokens
  - refresh-token-rotation
contentType: concept
useCase:
  - refresh-token-rotation
  - silent-authentication
---
# Refresh Token Rotation

With Refresh Token Rotation enabled, every time a client exchanges a [Refresh Token](/tokens/concepts/refresh-tokens) to get a new [Access Token](/tokens/cocncepts/access-tokens), a new Refresh Token is also returned. Therefore, you no longer have a long-lived Refresh Token that, if compromised, could provide illegitimate access to resources. As Refresh Tokens are continually exchanged and invalidated, the threat is reduced. 

The way Refresh Token rotation works in Auth0 conforms with the [OAuth 2.0 BCP](https://tools.ietf.org/html/draft-ietf-oauth-security-topics-13#section-4.12) and works with the following flows:
* [Authorization Code Flow](/flows/concepts/auth-code)
* [Authorization Code Flow with Proof Key for Code Exchange (PKCE)](/flows/concepts/auth-code-pkce)
* [Device Authorization Flow](/flows/concepts/device-auth)
* [Resource Owner Password Credentials Exchange](/api-auth/tutorials/adoption/password)

## Maintaining user sessions in SPAs

Until very recently, SPAs overcame the challenge of maintaining the user’s session by using the Authorization Code Flow with PKCE in conjunction with [Silent Authentication](/api-auth/tutorials/silent-authentication). Recent developments in browser privacy technology, such as Intelligent Tracking Prevention (ITP) prevent access to the Auth0 session cookie, thereby requiring users to reauthenticate. 

Refresh Token Rotation offers a remediation to end-user sessions being lost due to side-effects of browser privacy mechanisms. Because Refresh Token Rotation does not rely on access to the Auth0 session cookie, it is not affected by ITP or similar mechanisms.

The following state diagram illustrates how Refresh Token Rotation is used in conjunction with the Authorization Code Flow with PKCE, but the general principle of getting a new refresh token with each exchange applies to all supported flows.

![Refresh Token Rotation State Diagram](/media/articles/tokens/rtr-state-diagram.png)

This means that you can safely use Refresh Tokens to mitigate the adverse effects of browser privacy tools and provide continuous access to end-users without disrupting the user experience.

## Automatic reuse detection

When a client needs a new Access Token, it sends the Refresh Token with the request to Auth0 to get a new token pair. As soon as the new pair is issued by Auth0, the Refresh Token used in the request is invalidated. This helps safeguard your app from replay attacks resulting from compromised tokens.

Without enforcing sender-constraint, it’s impossible for the authorization server to know which actor is legitimate or malicious in the event of a replay attack. Therefore, it’s important that when a previously-used Refresh Token (already invalidated) is sent to the authorization server that the most recently issued Refresh Token is immediately invalidated as well, preventing any Refresh Tokens in the same token family (all Refresh Tokens descending from the original Refresh Token issued for the client) from being used to get new Access Tokens.

For example, consider the following scenario: 
1. Legitimate Client has **Refresh Token 1**, and it is leaked to or stolen by Malicious Client. 
2. Legitimate Client uses **Refresh Token 1** to get a new Refresh Token/Access Token pair.
3. Auth0 returns **Refresh Token 2/Access Token 2**.
4. Malicious Client then attempts to use **Refresh Token 1** to get an Access Token. Auth0 recognizes that Refresh Token 1 is being reused, and immediately invalidates the Refresh Token family, including **Refresh Token 2**.
5. Auth0 returns an **Access Denied** response to Malicious Client.
6. **Access Token 2** expires and Legitimate Client attempts to use **Refresh Token 2** to request a new token pair. Auth0 returns an **Access Denied** response to Legitimate Client.
7. Re-authentication is required.

![Reuse Detection](/media/articles/tokens/reuse-detection.png)

This protection mechanism works regardless of whether the legitimate client or the malicious client is able to exchange **Refresh Token 1** for a new token pair before the other. As soon as reuse is detected, all subsequent requests will be denied until the user re-authenticates. When reuse is detected, Auth0 captures this event (`ferrt` indicating a failed exchange) in logs to provide visibility for security reviews and audits. This can be especially useful in conjunction with Auth0’s log streaming capabilities.

## SDK support

The following SDKs include support for Refresh Token Rotation and automatic reuse detection. 

* [Auth0 SPA SDK](/libraries/auth0-spa-js)
* [Auth0 Swift (iOS) SDK](/libraries/auth0-swift)
* [Auth0 Android SDK](/libraries/auth0-android).

You can opt-in to storing tokens in either local storage or browser memory, the default being in browser memory. See [Token Storage](/tokens/concepts/token-storage) for details.

## Keep reading

* [Configure Refresh Token Rotation](/tokens/guides/configure-refresh-token-rotation)
* [Use Refresh Token Rotation](/tokens/guides/use-refresh-token-rotation)
* [Revoke Refresh Tokens](/tokens/guides/revoke-refresh-tokens)
* [Disable Refresh Token Rotation](/tokens/guides/disable-refresh-token-rotation)
* [OAuth 2.0 BCP](https://tools.ietf.org/html/draft-ietf-oauth-security-topics-13#section-4.12)