---
title: "Setting Up Microsoft Entra External ID with an ASP.NET Core MVC App"
date: "2025-05-05"
lastmod: "2025-05-05"
draft: false
weight: 604
toc: true
---

## 🎯 Goal

Create a Microsoft Entra External ID tenant and integrate it with a sample ASP.NET Core MVC application to enable secure user sign-up and sign-in experiences.

## 📋 Prerequisites

Before beginning this exercise, you should:

* Have an active Azure account with a subscription.
* Ensure your Azure account has been assigned at least the **Contributor** role, scoped either to the Azure subscription or to a specific resource group within the subscription (See 1:04 - 1:10 in the video).
* Have the [.NET SDK](https://dotnet.microsoft.com/download) installed on your local machine to run the ASP.NET Core MVC application.
* Have a basic understanding of ASP.NET Core MVC concepts.

## 📚 Learning Objectives

By the end of this exercise, you will:

* Create a new **Microsoft Entra External ID tenant** using the Microsoft Entra Admin Center.
* Utilize the **"Get Started" guide** to configure initial sign-up/sign-in settings and basic branding.
* Download, configure, and run a sample **ASP.NET Core MVC application** integrated with your External ID tenant.
* Understand the purpose and basic configuration of **Application Registrations** and **User Flows** within External ID.
* Test the end-to-end user **sign-up, sign-in, and sign-out** process in the MVC application.

## 🔍 Why This Matters

Microsoft Entra External ID provides a robust Customer Identity and Access Management (CIAM) solution. Learning to integrate it allows you to:

* Secure your customer-facing web applications effectively.
* Provide customizable and user-friendly sign-up and sign-in experiences.
* Separate customer identities from your internal workforce identities.
* Leverage advanced features like social logins, MFA, and conditional access policies.

### Step 1: Create a Microsoft Entra External ID Tenant

1. Navigate to the Microsoft Entra Admin Center at [entra.microsoft.com](https://entra.microsoft.com) and sign in with your Azure account.

2. In the Microsoft Entra Admin Center, go to **Identity** → **Overview** → **Manage tenants**.

3. From the toolbar, select **Create**.

4. In the "Choose a configuration for your tenant" page, select **External** and then select **Continue**.

5. On the "Basics" tab, enter the following information:

    * **Tenant name**: Enter a descriptive name (e.g., "MyApp Customers")
    * **Domain name**: Enter a unique name (e.g., "myapp-customers")
    * **Location**: Select your country and the corresponding data center

6. Select **Next: Add a subscription**.

7. On the "Add a subscription" tab:
   * Select your Azure subscription
   * Choose an existing resource group or create a new one
   * If creating a new resource group, enter a name and select a geographic location

8. Select **Next: Review and create**.

9. Review your configuration and select **Create**.

10. Once the tenant is created, access it from the notification link or by selecting your profile from the header and choosing **Switch directory**.

> 💡 **Information**
>
> * **Tenant Creation**: When you create a new Microsoft Entra tenant, you automatically become the first user and global administrator
> * **Data Location**: The geographic location selection determines where user data will be stored and cannot be changed after creation
> * **Domain Name**: The domain name becomes part of your authentication endpoints and cannot be changed later
>
> ⚠️ **Common Mistakes**
>
> * Forgetting to ensure you have the necessary permissions (contributor role) in your Azure subscription
> * Not noting the domain name you selected (you'll need it later for configuration)
> * Creating a standard (workforce) tenant instead of an external tenant

### Step 2: Configure Your Authentication Experience

1. In your new tenant, select **Overview** from the main menu.

2. On the "Get started" tab, select **Start the guide**.

3. Choose your preferred sign-in method:
   * **Email and password**: Traditional username/password authentication

4. Customize the branding experience:
   * Upload a company logo (optional)
   * Choose a background color
   * Select your preferred sign-in page layout

5. Wait for the tenant to be configured with your selections, then select **Run it now** to test the experience.

6. A new browser tab will open with the sign-up and sign-in page.

7. Select **No account, create one** and follow the steps to create a test user account:
   * Enter an email address (different from your admin account)
   * Verify with the code sent to your email
   * Create a password if using the password option

8. After sign-in completes, you'll see the JWT.ms application showing your token information.

9. Return to the Microsoft Entra Admin Center and select **Continue** to proceed.

10. Download Sample App

    1. **Continue to set up a simple application**
    2. **Select Application Type:**
       * WebbApp -> ASP.NET Core
    3. **Download sample app:**

       * Unzip the downloaded file to a location on your computer
       * Run the application using the .NET CLI command:

         ```bash
         dotnet run --launch-profile https
         ```

       * The command will build and start the web server. Note the URL where the application is listening (e.g., `https://localhost:7123` or `http://localhost:5123`).

    4. Test the MVC Application Sign-in

      * Open a web browser and navigate to the https URL
      * Find and click the **Sign In** button or link in the sample MVC application
      * If prompted, sign in using the **external user account** (email and password) you created before
      * Once signed in, the application should indicate your logged-in status, possibly displaying your email or other details from the token.
      * Test the **Sign Out** functionality in the application.
      * Close the browser tab for the sample application. You can stop the `dotnet run` command in the terminal (usually by pressing `Ctrl+C`).

> 💡 **Information**
>
> * **User Flow**: The guide automatically creates a sign-up and sign-in user flow with your selected settings
> * **JWT.ms**: This is a Microsoft tool that displays the contents of JWT tokens for debugging purposes
> * **Branding**: You can further customize the branding later with additional options
>
> ⚠️ **Common Mistakes**
>
> * Using the same email for the test account as your admin account
> * Not completing the guide which automatically configures essential components
> * Missing the verification code email (check your spam folder)

### Step 4: Register Your Application in Entra External ID

1. Return to the Microsoft Entra Admin Center in your external tenant.

2. In the left menu, select **Applications** → **App registrations**.

3. Select **New registration**.

4. Enter the following information:
   * **Name**: EntraAuthDemo
   * **Supported account types**: "Accounts in this organizational directory only"
   * **Redirect URI**: Web, [`https://localhost:7128/signin-oidc`](https://localhost:7128/signin-oidc) (note: port may vary, check your launchSettings.json)

5. Click **Register**.

6. After the application is registered, note the **Application (client) ID** from the Overview page.

7. Select **Authentication** from the left menu.

8. Under "Implicit grant and hybrid flows", enable **ID tokens**.

9. In the "Front-channel logout URL" field, enter: [`https://localhost:7128/signout-oidc`](https://localhost:7128/signout-oidc)

10. Click **Save**.

11. Update your `appsettings.json` file with the Application ID and your tenant name.

> 💡 **Information**
>
> * **Application Registration**: Creates an identity for your application in Entra External ID
> * **Application ID**: A unique identifier that your application uses to identify itself to the authentication service
> * **Redirect URIs**: The allowed URLs where the authentication service will redirect after authentication
> * **ID Tokens**: Contain claims about the user's identity and are used for authentication
>
> ⚠️ **Common Mistakes**
>
> * Incorrect redirect URI (must match exactly with your application's callback path)
> * Forgetting to enable ID tokens in the authentication settings
> * Not configuring the front-channel logout URL

## 🧪 Final Tests

### Run the Application and Validate Your Work

1. Start the application:

   ```bash
   dotnet run --launch-profile https
   ```

2. Open a browser and navigate to:

   ```sh
   https://localhost:7128/
   ```

3. Test the authentication flow by:

   * Clicking the "Sign in" link
   * Observing the redirect to your Entra External ID tenant
   * Creating a new account or signing in with your test account
   * After successful authentication, being redirected back to your application
   * Verifying that your name appears in the navigation bar
   * Accessing the Profile page which should display your claims

4. Test the sign-out flow:

   * Click "Sign out"
   * Verify you're logged out and redirected back to the application
   * Attempt to access the Profile page (should redirect to sign-in)

✅ **Expected Results**

* The sign-in link redirects to your custom Entra External ID login page
* After signing in, you should see your username in the navigation bar
* The Profile page should be accessible and show your claims
* Sign-out should clear your session and redirect you back to the application
* Attempting to access protected resources should redirect to the login page

## 🔧 Troubleshooting

If you encounter issues:

* **Redirect URI Mismatch Error**:
  * Verify the redirect URI in your app registration exactly matches your application's callback path
  * Check the port number in the redirect URI (it might be different from 7128)

* **Policy Not Found Error**:
  * Ensure your policy ID in appsettings.json matches what was created in your tenant
  * The default policy created by the guide is usually "B2C_1_susi"

* **CORS or Cookie Issues**:
  * Ensure you're using HTTPS for local development
  * Check that the front-channel logout URL is properly configured

* **Claim Access Issues**:
  * If you need additional claims, you may need to modify the user flow in the Azure portal

## 🚀 Optional Challenge

Want to take your learning further? Try:

1. **Add Social Identity Providers**:
   * Configure Google, Facebook, or Microsoft accounts as login options
   * Modify your user flow to include these identity providers
   * Test the multi-provider authentication experience

2. **Customize the User Interface**:
   * Use the Company Branding feature to fully customize the sign-in experience
   * Upload a background image and additional brand elements
   * Create a custom CSS template for advanced styling

3. **Implement Role-Based Authorization**:
   * Create app roles in your application registration
   * Assign users to different roles
   * Implement role-based access control in your application

## 📚 Further Reading

* [Microsoft Entra External ID Documentation](https://learn.microsoft.com/en-us/entra/external-id/) - Official documentation
* [Microsoft Identity Web GitHub](https://github.com/AzureAD/microsoft-identity-web) - Open-source library for authentication
* [ASP.NET Core Authentication](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/) - Authentication fundamentals
* [OpenID Connect Protocol](https://openid.net/developers/how-connect-works/) - Understanding the authentication protocol

## Done! 🎉

Great job! You've successfully created a Microsoft Entra External ID tenant and implemented secure authentication in your ASP.NET Core MVC application. This modern identity solution provides a scalable, secure, and customizable authentication experience for your users! 🚀
