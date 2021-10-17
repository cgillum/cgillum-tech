---
title: "App Service Token Store"
date: "2016-03-08"
categories: 
  - "easy-auth"
---

The App Service Token Store is an advanced capability that was added to the Authentication / Authorization feature (a.k.a. "Easy Auth") of App Service. Like the name implies, the token store is a repository of OAuth tokens that are associated with the end-users of your app.

When a user logs into your app via an identity provider, such as Azure Active Directory or Facebook (or any of the other supported providers), the identity provider provides one or more tokens that 1) prove the user's identity and may also 2) provide access to resources owned by that user. By default, all new App Service apps with auth configured will have a built-in token store which your app code can immediately take advantage of. Support is included for Web Apps, Mobile Apps, API Apps, Function apps, and is available in all SKUs.

# Common Scenarios for the Token Store

Lets say you're building an app and you want the ability for users to log in with their Facebook account credentials. Lets also say that you want your app post to their Facebook timelines on their behalf. In order to call the Facebook API to perform such an action, you would need an OAuth token issued by Facebook with the proper permissions to do this. The token store is for automatically collecting and storing these tokens and making it easy for your app to access them.

Similarly if you are writing an app which needs to call into the [Azure Active Directory Graph API](https://msdn.microsoft.com/library/azure/hh974476.aspx) or even the [Microsoft Graph](https://graph.microsoft.io/) on behalf of a user in your corporate directory, your app can be configured to store the required access token in the token store automatically. More details on how to configure your AAD applications for Graph API access will come in a subsequent post.

While convenient, not all apps require these capabilities. If, for example, you're only using built-in authentication to [protect access to a staging slot](https://channel9.msdn.com/Series/Windows-Azure-Web-Sites-Tutorials/Protecting-Site-Slots-with-Web-App-Authentication-and-Authorization) and don't need to actually do anything with these extra tokens, then you likely don't need to use the token store and can safely disable the feature.

# Accessing the Tokens

From within your backend code, accessing these tokens is as easy as reading an HTTP request header. The headers are named like **X-MS-TOKEN-{provider}-{type}**. The possible token header names are listed below:

## Azure Active Directory Token Request Headers

- X-MS-TOKEN-AAD-ID-TOKEN
- X-MS-TOKEN-AAD-ACCESS-TOKEN
- X-MS-TOKEN-AAD-EXPIRES-ON
- X-MS-TOKEN-AAD-REFRESH-TOKEN

## Facebook Token Request Headers

- X-MS-TOKEN-FACEBOOK-ACCESS-TOKEN
- X-MS-TOKEN-FACEBOOK-EXPIRES-ON

## Google Token Request Headers

- X-MS-TOKEN-GOOGLE-ID-TOKEN
- X-MS-TOKEN-GOOGLE-ACCESS-TOKEN
- X-MS-TOKEN-GOOGLE-EXPIRES-ON
- X-MS-TOKEN-GOOGLE-REFRESH-TOKEN

## Microsoft Account Token Request Headers

- X-MS-TOKEN-MICROSOFTACCOUNT-ACCESS-TOKEN
- X-MS-TOKEN-MICROSOFTACCOUNT-EXPIRES-ON
- X-MS-TOKEN-MICROSOFTACCOUNT-AUTHENTICATION-TOKEN
- X-MS-TOKEN-MICROSOFTACCOUNT-REFRESH-TOKEN

## Twitter Token Request Headers

- X-MS-TOKEN-TWITTER-ACCESS-TOKEN
- X-MS-TOKEN-TWITTER-ACCESS-TOKEN-SECRET

Querying the request headers is the simplest method for obtaining these user-specific OAuth tokens. No SDKs, external API calls, database queries, or file access is required (this is a general theme you'll see repeated when it comes to Easy Auth). The values of these headers are the raw token values and can be used as-is when calling into the provider APIs. No parsing necessary.

To see which tokens are available to your app for a particular user, try logging in and enumerating the request headers. These headers are added by the local IIS module, which I discussed in a previous post on the [App Service Auth Architecture](architecture-of-azure-app-service-authentication-authorization.md).

If you want to access the tokens from a client (like JavaScript in a browser), or if you want to get a richer set of information about the logged in user, you can also send an authenticated GET request to the local **/.auth/me** endpoint. Whatever authentication method you use to access your app (cookies or tokens) can be used to access this API. The request format is very simple:

`GET /.auth/me`

...which returns a JSON response that looks like the following:

```json
[{
  "provider_name":"<provider>",
  "user_id": "<user_id>",
  "user_claims":[{"typ": "<claim-type>","val": "<claim-value>"}, ...],
  "access_token":"<access_token>",
  "access_token_secret":"<access_token_secret>",
  "authentication_token":"<authentication_token>",
  "expires_on":"<iso8601_datetime>",
  "id_token":"<id_token>",
  "refresh_token":"<refresh_token>"
}]
```

This API is what the App Service Server SDK uses internally to fetch information about the currently logged-in user.

No matter which method you choose, the actual tokens available depend on what provider was used to log into the app as well as which scopes were configured for that provider. For example, a Google login will always include an access token, but will only include a refresh token if the offline access scope is configured.

# Where the Tokens Live

Internally, all these tokens are stored in your app's local file storage under **D:\home\data\.auth\tokens**. The tokens themselves are all encrypted in user-specific .json files using app-specific encryption keys and cryptographically signed as per best practice. This is an internal detail that your app code does not need to worry about. Just know that they are secure. While you could write code to read and decrypt these token store files, we recommend against doing so as the format could be changed in a future service update.

The app's distributed file storage was chosen as the default location for storing tokens because it's freely available to all App Service apps without any additional configuration. However, the file system token store may not be ideal for all customers. In particular, it does not scale well when large numbers of concurrent logins are expected and is incompatible with the local disk caching feature of App Service.

As an alternative, you can provision an Azure Blob Storage container and configure your web app with a SaS URL (with read/write/list access) pointing to that blob container. This SaS URL can then be saved to the **WEBSITE_AUTH_TOKEN_CONTAINER_SASURL** app setting. When this app setting is present, all tokens will be stored in and fetched from the specified blob container.

**As a warning, this blob storage capability should be considered experimental until further notice.** Also note that any existing tokens are not migrated to the new store. If you make this configuration change, users will need to re-authenticate with the app in order for their tokens to become available.

# Refreshing Tokens

An important detail about using access tokens is that most of them will eventually expire. Some providers, like Facebook, have access tokens which expire after 60 days. Other providers, like Azure AD, Microsoft Account, and Google, issue access tokens which expire in 1 hour. In all cases, a fresh set of tokens can be obtained by forcing the user to re-authenticate. This is reasonable for Facebook since a re-auth would only need to happen once every 60 days. However, this is not practical for Azure AD, Microsoft Account, and Google, where the token expiration is 1 hour.

To avoid the need to re-authenticate the user to get a new access token, you can instead issue an authenticated GET request to the **/.auth/refresh** endpoint of your application. This is a built-in endpoint, just like **/.auth/me**. When called, the Easy Auth module will automatically refresh the access tokens in the token store for the authenticated user. Subsequent requests for tokens by your app code will then get the most up-to-date tokens. In order for this to work, the token store must contain refresh tokens for your provider. If you're not familiar with how to do this, here are some hints:

- **Google**: Append an **"access_type=offline"** query string parameter to your /.auth/login API call (if using the Mobile Apps SDK, you can add this to one of the LogicAsync overloads).
- **Microsoft Account**: Select the **wl.offline_access** scope in the Azure management portal.
- **Azure AD**: This is a little complex right now, but take a look at my next post on [enabling Graph API access](app-service-auth-aad-graph-api.md). Follow the setup steps and this will also enable you to get refresh tokens for Azure AD (you can omit the **Read directory data** and the **resource=...** parts if they don't apply to you). Work is being done to simplify this.

Here is an example snippet which refreshes tokens from a JavaScript client (with jQuery). Similar logic could have been written in other languages as well:

```javascript

function refreshTokens() {
    var refreshUrl = "/.auth/refresh";
    $.ajax(refreshUrl).done(function() {
        console.log("Token refresh completed successfully.");
    }).fail(function() {
        console.log("Token refresh failed. See application logs for details.");
    });
}
```

It's important to note that the **/.auth/refresh** API is not guaranteed to succeed. For example, a user is free to revoke the permissions granted to your app at any time. In such cases, any attempt to refresh existing access tokens will fail with a 403 Forbidden response. Also, if your auth provider is not configured to return refresh tokens (e.g. you did not request the appropriate offline scope), you can expect this to fail with a 400 Bad Request response. In either case, you can check your application logs for details (assuming you have already enabled application logging).

If you are a mobile app and use App Service Mobile SDK, which internally uses the **x-zumo-auth** header to authenticate with your App Service backend, you may be aware of the fact that your x-zumo-auth JWT may also have a short expiration. This mobile authentication token can also be refreshed using the **/.auth/refresh** endpoint. Just send a GET request to **/.auth/refresh** with the **x-zumo-auth** header (present by default when using the mobile client SDK), and the endpoint will respond with a new authentication token for use by your application. Note that the same restrictions regarding refresh tokens still apply.

If you're building your client using one of the Azure Mobile Client SDK flavors, then you can use the _RefreshUser_ method to refresh both the local authentication token as well as the provider tokens. More documentation on _RefreshUser_ can be found at [https://azure.microsoft.com/blog/mobile-apps-easy-authentication-refresh-token-support/](https://azure.microsoft.com/blog/mobile-apps-easy-authentication-refresh-token-support/).

Here is an example snippet in C#:

```csharp
async Task<string> GetDataAsync()
{
    try
    {
        return await App.MobileService.InvokeApiAsync<string>("values");
    }
    catch (MobileServiceInvalidOperationException e)
    {
        if (e.Response.StatusCode != HttpStatusCode.Unauthorized)
        {
            throw;
        }
    } 
  
    // Internally calls /.auth/refresh to update the tokens in the token store
    // and will also update the authentication token used by the client.
    await App.MobileService.RefreshUser();
  
    // Make the call again, this time with a fresh authentication token.
    return await App.MobileService.InvokeApiAsync<string>("values");
}
```

As shown in the above, the code handles the unauthorized response, refreshes the authentication token, and makes the call again. Note that this code sample is meant to be simple and trivial. Your implementation will certainly be more robust.

You may have also noticed that this example calls into the **/.auth/refresh** API with an expired authentication token. This is allowed in the first 72-hours after expiration and simplifies token management by not requiring you to track expirations yourself. If the authentication token is not refreshed within the 72-hour window, a fresh re-auth by the end-user will be required to get a new, non-expired authentication token.

_**Advanced**: Several folks have asked about this, so we've made an update such that if 72 hours is not enough for you, you can customize this expiration window by adding a **tokenRefreshExtensionHours** value in the site/config/authSettings configuration in [Azure Resource Explorer](https://azure.microsoft.com/documentation/articles/resource-manager-resource-explorer/). Be warned that extending the expiration over a long period could have significant security implications - e.g. if an authentication token is leaked or stolen, so it's highly recommended to either leave it at the default 72 or set it to the smallest value necessary. The default of 72 hours was chosen to account for cases where a mobile app is used primarily on weekdays and goes inactive over the weekend._

# Wrapping Up

If you have general questions or if you're running into issues, I suggest posting on [StackOverflow](http://stackoverflow.com) (please tag your questions with **azure-app-service** or **azure-web-sites**) so that others can easily find, rate, and participate in the discussion. For the highest level of support, take a look at some of our [support plans](https://azure.microsoft.com/support/plans/). This is absolutely the best way to get our attention and prioritize any issues you may encounter.

Also, feel free to reach out to me directly on [Twitter](https://twitter.com/cgillum).
