---
title: "App Service Auth and Azure AD B2C"
date: "2016-05-27"
categories: 
  - "aad"
  - "easy-auth"
---

An exciting new preview feature which was recently added to Azure Active Directory is [Azure Active Directory B2C](https://azure.microsoft.com/services/active-directory-b2c/). "B2C" stands for "Business to Consumer" and allows a developer to add user and login management to their application with very little (if any) coding. This also includes login integration with social identity providers like Facebook, Amazon, LinkedIn, etc. Check out their [documentation](https://aka.ms/aadb2c) and [blog posts](https://blogs.technet.microsoft.com/ad/2016/05/05/more-preview-enhancements-for-azure-ad-b2c/) for more details. My colleague Swaroop from the Azure AD team also has a nice [//build video](https://channel9.msdn.com/Events/Build/2016/P423) where you can see it in action.

From my perspective, App Service Authentication / Authorization (Easy Auth) shares a similar goal of B2C, which is to make it really easy to build identity into your application. We saw a great opportunity to make these features work well together, giving you both an identity management system as well as login and OAuth token management without requiring a single line of code.

In this post, I'll describe how you can use Easy Auth to add Azure AD B2C capabilities to your App Service Web App.

# Creating an App Service Web App

Hopefully you know how to do this by now. Go ahead and create a web app (or an API/mobile/function app - they all work the same way) and make a note of the URL. For example, when drafting this blog post and walking through the steps, I created `https://cgillum-b2c-preview.azurewebsites.net`. Use your own web app URL in place of mine wherever you see it in these instructions. However, don't configure Authentication / Authorization yet. We'll do that in a later step.

# Creating the Azure AD B2C Tenant and Application

We don't currently support an "Express" setup of B2C like we do for classic Azure AD, so these steps will need to be done manually. You can find detailed instructions for this below:

1. [Create an Azure AD B2C tenant](https://azure.microsoft.com/documentation/articles/active-directory-b2c-get-started/)
2. [Register your application](https://azure.microsoft.com/documentation/articles/active-directory-b2c-app-registration/)

Note that in step 2, you'll need to use the **https** address of the web app you previously created as the Reply URL and you must suffix it with "/.auth/login/aad/callback" (again, in my case this is `https://cgillum-b2c-preview.azurewebsites.net/.auth/login/aad/callback`). Once this is done, you should have an application in the B2C portal which looks something like the following:

![B2C Application Blade](/images/B2C-ApplicationBlade.png)

Make a note of the **Application Client ID** that you see in the Application blade. You'll need this in a later step.

# Adding a Sign-Up/Sign-In Policy

For simplicity, we'll create a single B2C "policy" which allows the user to sign in or sign up if they don't already have an account. In the portal, this is the **Sign-up or sign-in policies** selection. Add a new policy (assuming you don't have one already). The details of the policy don't matter too much, so I won't provide any specific guidance here. There are a lot of options, including whether to configure social identity providers. In my case, I set up email login as well as Google and Facebook. To get started quickly, I suggest you use the email sign-up policy. When you're done, make a note of the **Metadata Endpoint for this policy** URL which gets generated, like in the screenshot below:

![B2C Policy Blade](/images/B2C-PolicyBlade.png)

Azure AD B2C supports other policy types as well, but the combination sign-up/sign-in is currently the one best suited for Easy Auth login integration.

# Configure Easy Auth

Now let's go back to the web app we previously created. We'll configure Easy Auth with Azure AD using the **Advanced** configuration option. The steps are:

1. In the portal in the context of your web app, click the **Settings** icon.
2. Set **App Service Authentication** to **On**
3. Configure **Azure Active Directory**
4. Select the **Advanced** management mode
5. Set the **Client ID** to be the **Application Client ID** from before.
6. Set the **Issuer URL** to be the **Metadata Endpoint for this policy** URL value that was generated from your sign-in/sign-on B2C policy.
7. Click **OK** and then the **Save** icon to save your changes.

Your Authentication / Authorization settings blade should look something like the following:

![Authentication / Authorization Blade](/images/B2C-EasyAuthBlade.png)

Now if you navigate to your site, you should see the B2C login page that you configured previously. Depending on how it was configured, you can sign up using social identity credentials or you can sign up using username (or email) and password. You will also be prompted for additional registration information, such as your name, etc (again, all dictated by the policies you configured).

Here is an example of what your initial sign-in page might look like. Notice the link on the bottom of the image which allows users to register:

![B2C Login Page](/images/B2C-LoginPage.png)

Below is an example of what your "sign-up" registration page will look like. If you selected the email option, Azure AD B2C will even implement the email verification workflow for you.

![B2C Sign-Up Page](/images/B2C-SignUpPage.png)

That's it! You've now created a skeleton B2C web application that allows users to sign-up and sign-in without writing any code or deploying any databases! I've used all the defaults in terms of styling, but Azure AD B2C does allow you to customize the look and feel of these pages to match your own application branding, if you choose. See the [B2C UI customization documentation](https://azure.microsoft.com/documentation/articles/active-directory-b2c-reference-ui-customization/) for more information.

Once signed in, you can write code which inspects the inbound HTTP headers and/or the /.auth/me endpoint (both described in [my earlier Token Store post](app-service-token-store.md)) to get more information about the user. If you're running an ASP.NET application, you can also enumerate the claims on the current [ClaimsPrincipal](https://msdn.microsoft.com/library/system.security.claims.claimsprincipal%28v=vs.110%29.aspx). Specifically, you should be able to see all the claims that you configured in your B2C policy. This is great because you don't need to provision your own database which contains this information - it's built into the B2C directory and can be accessed using the features of Easy Auth.

# Azure AD B2C Social Providers vs. Easy Auth Social Providers

One thing you may have noticed is that there are now two ways to incorporate social logins into your web app: using B2C policies or configuring them directly in the Authentication / Authorization settings. Ideally, there would be just one which is common between the two technologies. Unfortunately we're not there yet. Until then, here are some important differences between social providers in Azure AD B2C and Easy Auth:

- **Different identity providers**: B2C and Easy Auth support different providers. At the time of writing, B2C supports MSA, Facebook, Google, LinkedIn, and Amazon identities. Easy Auth, however, supports MSA, Facebook, Google, and Twitter.
- **Client-Directed Login**: Both B2C and Easy Auth support server-directed social logins where the login flow is controlled by the browser. However, only Easy Auth supports client-directed logins where the login flow is controlled by the client operating system (typically a mobile OS or a JavaScript client).
- **User Claims**: B2C provides a somewhat normalized set of user claims for each login. These claims are similar to what you'd see in an ordinary AAD login. The claims are also configurable. With Easy Auth, however, the claims are static and in many cases are different for each identity provider.
- **OAuth Tokens**: With Easy Auth, the application code has direct access to the provider-specific OAuth tokens. This is useful if you want to make graph API calls on behalf of the logged-in user (for example, calling the Facebook Graph to post a photo to the user's timeline). B2C, however, does not expose the provider OAuth tokens to your application code.

If social identity integration is important to your app, then consider these differences very carefully when deciding between the two options. Note that these limitations will certainly change as both Easy Auth and Azure AD B2C evolve over time, hopefully in a way that better aligns them together (this is certainly our goal, internally).

# Taking it to the Next Level

This post demonstrated using a single, global B2C policy within a web app in a way that doesn't require any code or database setup. If you are a mobile, API, or SPA app developer, I have written a followup post which goes into more details about how to use code to dynamically select B2C policies, how to set up token refresh, and even included a sample SPA app which demonstrates these capabilities. Check it out in [Part 2 of the App Service + Azure AD B2C series](app-service-auth-and-azure-ad-b2c-part-2.md).
