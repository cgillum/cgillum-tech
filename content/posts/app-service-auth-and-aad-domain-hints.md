---
title: "App Service Auth and Azure AD Domain Hints"
date: "2016-07-13"
categories: 
  - "aad"
  - "easy-auth"
---

When creating web, mobile, API, or Function apps for use by members of your organization, it's often the case that you're using Azure Active Directory and you want to remove the option to log in with non-organizational credentials. For example, you want to prevent users from accidentally logging in with MSA credentials (hotmail.com, live.com, outlook.com, etc.). This can be done by leveraging what's known as a **domain hint** when navigating users to the Azure AD login page.

Domain hints will do two things for you: 1) remove the home realm discovery page from the login flow and 2) ensure that users can't accidentally auto-log into your app using wrong credential types (for example, MSA credentials). More background information on Azure AD's support for domain hints can be found on the [Microsoft Enterprise Mobility blog](https://blogs.technet.microsoft.com/enterprisemobility/2015/02/11/using-azure-ad-to-land-users-on-their-custom-login-page-from-within-your-app/).

[Vittorio Bertocci](https://twitter.com/vibronet) also talks about domain hints in his post on [Skipping the Home Realm Discovery Page in Azure AD](http://www.cloudidentity.com/blog/2014/11/17/skipping-the-home-realm-discovery-page-in-azure-ad/), demonstrating how to use them when using [ADAL](https://github.com/AzureAD/azure-activedirectory-library-for-dotnet) and the [OpenID Connect Middleware](http://katanaproject.codeplex.com/) to build your web app. In this post, however, I'll describe how enable domain hints when using App Service's integrated Easy Auth feature.

# Default Login Parameters

Most web apps will want to configure domain hints to be used for all logins. Unfortunately you cannot configure default domain hints in this way using the Azure portal today. Instead, you must use the App Service Management API. Until we get around to building a portal experience, I recommend that most people configure default domain hints in [Azure Resource Explorer](https://resources.azure.com). This can be done using the following steps:

1. Search for your web, mobile or API app using the search bar. Alternatively, you can navigate directly to your app if you click on the Resource Explorer link in the _tools_ section of the portal.
2. Under your site node, navigate to **/config/authsettings.**
3. Click **Edit** to enable making changes.
4. Set **additionalLoginParams** to the following (This is a JSON array value): `["domain_hint=microsoft.com"]`
5. Click the **Read/Write** button at the top of the page to enable making changes.
6. Click the **PUT** button to save your changes.

The JSON configuration for your auth settings should look something like the screenshot below. In my case, I specified _domain\_hint=microsoft.com_ since the app shown here is intended to be used by Microsoft employees.

![DomainHints_ResourceExplorer](/images/DomainHints_ResourceExplorer.jpg)

Once this is done, users will no longer see the home realm discovery page when logging into the app. Instead, users will be immediately directed to the organizational login page, ensuring they cannot intentionally or accidentally log in with the wrong credentials.

# Using the Login API

If you're building an app that invokes the built-in _/.auth/login/aad_ REST API, you can alternatively specify `domain_hint={domain}` as a query string parameter to get the same effect.

For example, if I'm writing a mobile client and using the App Service .NET SDK, I could write the following code to initiate a login using a domain hint for _contoso.com_.

```csharp
var user = App.MobileClient.LoginAsync(
  MobileServiceAuthenticationProvider.WindowsAzureActiveDirectory,
  new Dictionary<string, string>
  {
    { "domain_hint", "contoso.com" }
  }
```

Similarly, I could create a login link in a web page using HTML that includes a _domain\_hint_ parameter.

```html
<a href="/.auth/login/aad?domain_hint=contoso.com">Login</a>
```

This allows me to specify login hints without changing the auth settings of the app as I showed in the first part of this post. In theory, this would also allow me to create multiple login links, one for each domain that my Azure AD application supports. Note that if an app is already configured with a default domain hint, the query string parameter will override that default.

# Conclusion

In conclusion, most single-tenant applications will want to use domain hints to optimize the login experience. These hints allows you to skip the home realm discovery page in the Azure AD login sequence and mitigates the common problem where the browser will try to log into the app using MSA credentials via SSO. Depending on the type of application you are building, you can use either the default login parameter method or you can explicitly specify login hints via the built-in _/.auth/login/aad_ endpoint.
