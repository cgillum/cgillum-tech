---
title: "App Service Auth and the Azure AD Graph API"
date: "2016-03-26"
categories: 
  - "easy-auth"
---

This post demonstrates how an App Service Web, Mobile, or API app can be configured to call the Azure Active Directory Graph API on behalf of the logged-in user. If you haven't read it already, this post extends from my previous one on the [Azure App Service Token Store](app-service-token-store.md).

# Configuration

The default setup for Azure AD that we use does not include the configuration required for your app to call into the Graph API. However, there are a couple one-time configuration changes that you can make to enable this. In the future, we plan to make this as simple as clicking a button or checkbox in the portal, but until that happens, there is some additional setup required.

## Step 1: Update Azure AD Configuration in Azure AD Portal

You can find and manage your Azure AD application in the legacy Azure Portal at [https://manage.windowsazure.com](https://manage.windowsazure.com). If you used the **Express** setup when configuring Azure AD on your App Service app, you can search for your Azure AD app using either your app name or the client ID of your Azure AD application. Once there, you will need to make two changes: 1) add the "Read directory data" delegated permission and 2) add a key to your Azure AD application.

![Legacy Azure Portal](/images/GraphAPI-AAD-Setup.png)

If your app is already configured with the "Read directory data" and already has an existing key, then no further changes are necessary.

## Step 2: Update App Service Auth Configuration via REST API

Next you need to update the App Service Auth settings for your web, mobile, or API app with your Azure AD key plus some additional login parameters. This is a little tricky because there is no official UI to assist with this. Until we get around to building that UI, I usually recommend that people use [Azure Resource Explorer](https://resources.azure.com), using the following steps:

- Search for your web, mobile or API app using the search bar.
- Under your site node, navigate to **/config/authsettings**.
- Click **Edit** to enable making changes.
- Set **clientSecret** (a string property) to the key value that was generated in the Azure AD portal.
- Set **additionalLoginParams** to the following:

(This is a JSON array value)

```json
["response_type=code id_token", "resource=https://graph.windows.net"]
```

- Click the **Read/Write** button at the top of the page to enable making changes.
- Click the **PUT** button to save your changes.

The JSON configuration for your auth settings should look something like the screenshot below.

![Screenshot](/images/GraphAPI-EasyAuth-Setup.png)

Once this is done, the next time users log into your web app, there will be a one-time prompt to consent to graph API access. Once consented, the App Service Auth infrastructure will start populating access tokens and refresh tokens into your app's [Token Store](app-service-token-store.md), which can be used for making Azure AD Graph API calls.

# Calling the Graph API as the End-User

Once the app is properly configured, the code to obtain the token and call into the Azure AD Graph API using the user's identity is relatively trivial. Here is a C# example of how to obtain the user's profile photo from the Azure AD Graph from within your Web, Mobile, or API app:

```csharp
// The access token can be fetched directly from a built-in request
// header! If this is null, it means you either haven't completed
// the setup prerequisites or you have disabled the token store in
// the Azure portal.
string accessToken = this.Request.Headers[
    "X-MS-TOKEN-AAD-ACCESS-TOKEN"];

// Call into the Azure AD Graph API using HTTP primitives and the
// Azure AD access token.
var url = "https://graph.windows.net/me/thumbnailPhoto?api-version=1.6";
var request = WebRequest.CreateHttp(url);
var headerValue = "Bearer " + accessToken;
request.Headers.Add(HttpRequestHeader.Authorization, headerValue);
 
using (var response = request.GetResponse())
using (var responseStream = response.GetResponseStream())
using (var memoryStream = new MemoryStream())
{
    responseStream.CopyTo(memoryStream);
    string encodedImage = Convert.ToBase64String(
        memoryStream.ToArray());

  // do something with encodedImage, like embed it into your HTML...
}
```

The above example is intentionally trivial, but you could imagine customizing it to use other Azure AD Graph APIs, including group membership APIs for building your own security group ACLs. I'll also point out that I used HTTP primitives rather than using [the official Graph API Client SDK](https://blogs.msdn.microsoft.com/aadgraphteam/2014/12/11/announcing-azure-ad-graph-api-client-library-2-0/). This was mainly to emphasize that access to the Azure AD Graph API can be written in any language and doesn't require any SDKs (though you can certainly use the SDKs if you like).

If you'd like more information on the available Azure AD Graph APIs, please consult the [documentation on MSDN](https://msdn.microsoft.com/library/azure/hh974476.aspx). If you have any questions or are running into issues, I recommend posting them on [StackOverflow](http://stackoverflow.com/) and adding the tags **azure-app-service** and **azure-active-directory**. Also, be sure to reference this post in your question. I hope this is helpful!
