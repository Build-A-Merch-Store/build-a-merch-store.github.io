---
title: "Adding Azure Entra External ID Authentication"
date: "2025-05-05"
lastmod: "2025-05-05"
draft: false
weight: 605
toc: true
---

## 🎯 Goal

Enhance your application by implementing Azure Entra External ID (formerly Azure AD B2C) as a third authentication method alongside your existing cookie authentication and ASP.NET Core Identity, creating a flexible multi-authentication system.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 3 (Adding ASP.NET Core Identity with EF Core)
- Have an Azure Entra External ID tenant configured
- Understand basic OAuth/OpenID Connect concepts
- Have Azure tenant credentials (Authority URL and Client ID)
- Be familiar with ASP.NET Core authentication concepts

## 📚 Learning Objectives

By the end of this exercise, you will:

- Implement **Azure Entra External ID** authentication using Microsoft Identity Web
- Configure **multiple authentication schemes** working in parallel
- Use a **policy scheme** to dynamically select between three authentication methods
- Understand **cookie naming** and how it affects authentication routing
- Create a user interface that shows which authentication method is active
- Debug authentication issues with a custom middleware
- Implement proper **sign-out** for each authentication method

## 🔍 Why This Matters

In real-world applications, multiple authentication methods are crucial because:

- They provide flexibility for users to choose their preferred sign-in method
- They enable gradual migration from one authentication system to another
- Enterprise applications often require support for both local and federated authentication
- Understanding authentication scheme routing is a valuable skill for ASP.NET Core developers
- Properly managing cookies and sign-out flows prevents security issues

## 📝 Step-by-Step Instructions

### Step 1: Install Required NuGet Packages

1. Open a terminal in your project directory.
2. Install the following NuGet packages:

   ```bash
   dotnet add package Microsoft.Identity.Web
   dotnet add package Microsoft.Identity.Web.UI
   ```

3. Verify the packages were added to your project file:

    > `MerchStore.csproj`

    ```xml
    <ItemGroup>
      <PackageReference Include="BCrypt.Net-Next" Version="4.0.3" />
      <PackageReference Include="Microsoft.AspNetCore.Identity.EntityFrameworkCore" Version="9.0.4" />
      <PackageReference Include="Microsoft.AspNetCore.Identity.UI" Version="9.0.4" />
      <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="9.0.4">
        <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
        <PrivateAssets>all</PrivateAssets>
      </PackageReference>
      <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="9.0.4" />
      <PackageReference Include="Microsoft.Identity.Web" Version="3.8.4" />
      <PackageReference Include="Microsoft.Identity.Web.UI" Version="3.8.4" />
    </ItemGroup>
    ```

> 💡 **Information**
>
> - **Microsoft.Identity.Web**: Provides integration with Azure AD and Azure AD B2C
> - **Microsoft.Identity.Web.UI**: Includes UI components and controllers for login/logout
> - These packages handle the complex OAuth 2.0 and OpenID Connect protocols for you
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to add Microsoft.Identity.Web.UI will cause routing issues
> - Using incompatible package versions can lead to runtime errors
> - Not updating your Program.cs to use these packages after installation

### Step 2: Configure Azure Entra External ID in appsettings.json

1. Open `appsettings.json`.
2. Add the Azure AD configuration section:

    > `appsettings.json`

    ```json
    {
      "ConnectionStrings": {
        "DefaultConnection": "Data Source=merchstore.db"
      },
      "AzureAd": {
        "Authority": "https://entrademo123.ciamlogin.com/",
        "ClientId": "80ba6e0a-90e5-40f7-95a3-cff7f4edacdf",
        "CallbackPath": "/signin-oidc",
        "SignedOutCallbackPath": "/signout-callback-oidc"
      },
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft.AspNetCore": "Warning"
        }
      },
      "AllowedHosts": "*"
    }
    ```

> 💡 **Information**
>
> - **Authority**: The domain of your Entra External ID tenant
> - **ClientId**: The application ID from your Azure portal registration
> - **CallbackPath**: Where Entra will redirect after successful authentication (default: `/signin-oidc`)
> - **SignedOutCallbackPath**: Where Entra will redirect after sign-out (default: `/signout-callback-oidc`)
>
> ⚠️ **Common Mistakes**
>
> - Using incorrect Authority URL format (should end with .ciamlogin.com for CIAM tenants)
> - Not registering these redirect URIs in the Azure portal
> - Using incorrect ClientId value or format

### Step 3: Configure JWT Token Handling in Program.cs

1. Open `Program.cs`.
2. Add the JWT Security Token Handler configuration before other authentication services:

    > `Program.cs` (beginning)

    ```csharp
    using System.IdentityModel.Tokens.Jwt;
    using MerchStore.Data;
    using MerchStore.Middleware;
    using MerchStore.Models;
    using Microsoft.AspNetCore.Authentication.Cookies;
    using Microsoft.AspNetCore.Authentication.OpenIdConnect;
    using Microsoft.AspNetCore.Identity;
    using Microsoft.EntityFrameworkCore;
    using Microsoft.Identity.Web;
    using Microsoft.Identity.Web.UI;
    
    var builder = WebApplication.CreateBuilder(args);
    
    // Add services to the container.
    builder.Services.AddControllersWithViews()
        .AddMicrosoftIdentityUI(); // Add this for the Azure AD UI components
    
    // Add Razor Pages (required for Identity UI)
    builder.Services.AddRazorPages();
    
    // This is required to be instantiated before the OpenIdConnectOptions starts getting configured.
    // By default, the claims mapping will map claim names in the old format to accommodate older SAML applications.
    // For instance, 'http://schemas.microsoft.com/ws/2008/06/identity/claims/role' instead of 'roles' claim.
    JwtSecurityTokenHandler.DefaultMapInboundClaims = false;
    
    // Configure Entity Framework Core with SQLite
    builder.Services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));
    
    // Add ASP.NET Core Identity
    builder.Services.AddDefaultIdentity<IdentityUser>(options => 
        options.SignIn.RequireConfirmedAccount = false)
        .AddEntityFrameworkStores<ApplicationDbContext>();
    ```

> 💡 **Information**
>
> - **JwtSecurityTokenHandler.DefaultMapInboundClaims = false**: Prevents automatic claim type mapping
> - **AddMicrosoftIdentityUI()**: Adds controllers and views for Microsoft Identity Web
> - Setting this before configuring authentication ensures proper JWT token handling
>
> ⚠️ **Common Mistakes**
>
> - Setting DefaultMapInboundClaims after configuring OpenID Connect options won't work
> - Forgetting to add Identity UI components will cause routing errors
> - Not adding the necessary using statements for Microsoft Identity Web

### Step 4: Configure Multiple Authentication Schemes

1. Continue updating `Program.cs`.
2. Replace your existing authentication configuration with this multi-scheme setup:

    > `Program.cs` (authentication configuration)

    ```csharp
    // Configure authentication with multiple schemes
    var authBuilder = builder.Services.AddAuthentication(options =>
    {
        // Set the default scheme to check all authentication types
        options.DefaultScheme = "CustomMultiAuthScheme"; // Default scheme for authentication
        options.DefaultChallengeScheme = CookieAuthenticationDefaults.AuthenticationScheme; // Default challenge to standard cookie login
    })
        .AddPolicyScheme("CustomMultiAuthScheme", "Custom MultiAuth Scheme", options =>
        {
            // This policy scheme will check the name of the cookie and decide which authentication scheme to use
            options.ForwardDefaultSelector = context =>
            {
                // Check if the default auth cookie exists. If it does, use the cookie authentication scheme.
                if (context.Request.Cookies.ContainsKey(".AspNetCore.Cookies"))
                    return CookieAuthenticationDefaults.AuthenticationScheme;
                
                // Check if the Entra External ID cookie exists
                if (context.Request.Cookies.ContainsKey(".AspNetCore.Cookies.External"))
                    return OpenIdConnectDefaults.AuthenticationScheme;
                
                // Otherwise, fall back to the Identity scheme
                return IdentityConstants.ApplicationScheme;
            };
        })
        .AddCookie(CookieAuthenticationDefaults.AuthenticationScheme, options =>
        {
            // Cookie settings
            options.Cookie.HttpOnly = true;
            options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
            options.Cookie.SameSite = SameSiteMode.Lax;
    
            // Expiration settings
            options.ExpireTimeSpan = TimeSpan.FromMinutes(60);
            options.SlidingExpiration = true;
        })
        // Add Microsoft Identity Web with explicit cookie scheme name to avoid conflicts
        .AddMicrosoftIdentityWebApp(options => {
            builder.Configuration.GetSection("AzureAd").Bind(options);
            options.SignInScheme = "Cookies.External"; // Use a different cookie name
    
            // Delete the local cookies when signing out
            options.Events = new OpenIdConnectEvents
            {
                OnRedirectToIdentityProviderForSignOut = context =>
                {
                    // Ensure proper cleanup before redirecting to identity provider
                    context.Response.Cookies.Delete(".AspNetCore.Cookies.External");
                    return Task.CompletedTask;
                }
            };
        }, cookieScheme: "Cookies.External", openIdConnectScheme: OpenIdConnectDefaults.AuthenticationScheme);
    ```

> 💡 **Information**
>
> - **Authentication Schemes**:
>   - `CookieAuthenticationDefaults.AuthenticationScheme`: "Cookies" (default cookie auth)
>   - `IdentityConstants.ApplicationScheme`: "Identity.Application" (Identity auth)
>   - `OpenIdConnectDefaults.AuthenticationScheme`: "OpenIdConnect" (Entra auth)
>
> - **Cookie Names**:
>   - `.AspNetCore.Cookies`: Default cookie authentication
>   - `.AspNetCore.Identity.Application`: ASP.NET Core Identity
>   - `.AspNetCore.Cookies.External`: Entra External ID (custom name to avoid conflicts)
>
> - **Policy Scheme**: Acts as a router to forward requests to the appropriate handler
> - **ForwardDefaultSelector**: Determines which scheme to use based on cookies present
> - **cookieScheme**: Specifies the named cookie scheme for Microsoft Identity
>
> ⚠️ **Common Mistakes**
>
> - Using the same cookie name for different schemes leads to conflicts
> - Not configuring proper sign-out events causes incomplete logouts
> - Forgetting to specify the SignInScheme for OpenID Connect

### Step 5: Add Authentication Logger Middleware

1. Create a new folder called `Middleware` in your project.
2. Create a new file called `AuthenticationLogger.cs`:

    > `Middleware/AuthenticationLogger.cs`

    ```csharp
    namespace MerchStore.Middleware;
    
    public class AuthenticationLogger
    {
        private readonly RequestDelegate _next;
        private readonly ILogger<AuthenticationLogger> _logger;
    
        public AuthenticationLogger(RequestDelegate next, ILogger<AuthenticationLogger> logger)
        {
            _next = next;
            _logger = logger;
        }
    
        public async Task InvokeAsync(HttpContext context)
        {
            // Log authentication state
            if (context.User.Identity is not null && context.User.Identity.IsAuthenticated)
            {
                _logger.LogInformation($"User is authenticated as: {context.User.Identity.Name}");
                _logger.LogInformation($"Authentication Type: {context.User.Identity.AuthenticationType}");
            }
            else
            {
                _logger.LogInformation("User is not authenticated.");
            }
    
            // Go to the next middleware in the pipeline.
            await _next(context);
        }
    }
    ```

3. Update your `Program.cs` to use this middleware:

    > `Program.cs` (middleware registration)

    ```csharp
    app.UseAuthentication(); // Before UseAuthorization()
    app.UseMiddleware<AuthenticationLogger>();
    app.UseAuthorization();
    ```

> 💡 **Information**
>
> - Authentication logger helps debug which schemes and cookies are active
> - It logs the authentication type, user name, claims, and cookies
> - This is invaluable for diagnosing authentication issues with multiple schemes
>
> ⚠️ **Common Mistakes**
>
> - Placing the middleware before UseAuthentication() won't show authentication info
> - Not logging cookies makes it harder to debug policy scheme routing
> - Truncating cookie values too much might hide important debugging information

### Step 6: Update the Login Partial View

1. Open `Views/Shared/_LoginPartial.cshtml`.
2. Replace it with this updated version that supports all three authentication methods:

    > `Views/Shared/_LoginPartial.cshtml`

    ```html
    <ul class="navbar-nav ms-auto">
    @if (User.Identity?.IsAuthenticated == true)
    {
        <li class="nav-item d-flex align-items-center">
            <span class="navbar-text text-dark me-3">
                Hello @User.Identity.Name! 
                @if (User.Identity.AuthenticationType == "Cookies")
                {
                    <span class="badge bg-info">Default</span>
                }
                else if (User.Identity.AuthenticationType == "Identity.Application")
                {
                    <span class="badge bg-success">Identity</span>
                }
                else if (User.Identity.AuthenticationType == "AuthenticationTypes.Federation")
                {
                    <span class="badge bg-primary">Entra ID</span>
                }
            </span>
        </li>
        <li class="nav-item">
            @if (User.Identity.AuthenticationType == "Cookies")
            {
                <form class="form-inline" asp-controller="Account" asp-action="Logout" method="post">
                    <button type="submit" class="btn btn-outline-primary">Logout</button>
                </form>
            }
            else if (User.Identity.AuthenticationType == "Identity.Application")
            {
                <form class="form-inline" asp-area="Identity" asp-page="/Account/Logout" 
                      asp-route-returnUrl="@Url.Action("Index", "Home", new { area = "" })" method="post">
                    <button type="submit" class="btn btn-outline-primary">Logout</button>
                </form>
            }
            else if (User.Identity.AuthenticationType == "AuthenticationTypes.Federation")
            {
                <a class="btn btn-outline-primary" asp-area="MicrosoftIdentity" asp-controller="Account" asp-action="SignOut">Logout</a>
            }
        </li>
    }
    else
    {
        <li class="nav-item dropdown">
            <a class="nav-link dropdown-toggle" href="#" id="navbarDropdown" role="button" 
               data-bs-toggle="dropdown" aria-expanded="false">
                Login
            </a>
            <ul class="dropdown-menu dropdown-menu-end" aria-labelledby="navbarDropdown">
                <li><a class="dropdown-item" asp-controller="Account" asp-action="Login">
                    <i class="bi bi-key"></i> Default Login
                </a></li>
                <li><a class="dropdown-item" asp-area="Identity" asp-page="/Account/Login">
                    <i class="bi bi-person-badge"></i> Identity Login
                </a></li>
                <li><hr class="dropdown-divider"></li>
                <li><a class="dropdown-item" asp-area="MicrosoftIdentity" asp-controller="Account" asp-action="SignIn">
                    <i class="bi bi-microsoft"></i> Entra External ID
                </a></li>
            </ul>
        </li>
    }
    </ul>
    ```

> 💡 **Information**
>
> - **Authentication Types**:
>   - "Cookies": Default cookie authentication
>   - "Identity.Application": ASP.NET Core Identity authentication
>   - "AuthenticationTypes.Federation": Entra External ID authentication
>
> - **Area Routing**:
>   - Default login: No area
>   - Identity login: "Identity" area
>   - Entra login: "MicrosoftIdentity" area
>
> - **Logout Handling**: Each authentication method requires a different logout approach
>
> ⚠️ **Common Mistakes**
>
> - Using incorrect AuthenticationType values for checking the login type
> - Not using the correct areas for Identity and Microsoft Identity actions
> - Using GET instead of POST for logout in cookie and Identity authentications

### Step 7: Create a WhoAmI View to Display Authentication Details

1. Create or update the `Views/Home/WhoAmI.cshtml` view:

    > `Views/Home/WhoAmI.cshtml`

    ```html
    <div class="row justify-content-center">
        <div class="col-md-8">
            <div class="card shadow">
                <div class="card-header bg-primary text-white">
                    <h2 class="fs-4 mb-0">Who Am I?</h2>
                </div>
                <div class="card-body">
                    @if (User.Identity?.IsAuthenticated == true)
                    {
                        <div class="alert @(User.Identity.AuthenticationType == "OpenIdConnect" ? "alert-primary" : 
                                           User.Identity.AuthenticationType == "Identity.Application" ? "alert-success" : "alert-info")">
                            <h4>Authentication Information</h4>
                            <p><strong>Authentication Type:</strong> @User.Identity.AuthenticationType</p>
                            <p><strong>Username:</strong> @User.Identity.Name</p>
                            
                            @if (User.Identity.AuthenticationType == "OpenIdConnect")
                            {
                                <p><strong>Authentication Provider:</strong> Entra External ID</p>
                            }
                            else if (User.Identity.AuthenticationType == "Identity.Application")
                            {
                                <p><strong>Authentication Provider:</strong> ASP.NET Core Identity</p>
                            }
                            else
                            {
                                <p><strong>Authentication Provider:</strong> Default Cookie Authentication</p>
                            }
                        </div>
                        
                        <div class="table-responsive">
                            <table class="table table-striped table-hover">
                                <thead class="table-light">
                                    <tr>
                                        <th>Claim Type</th>
                                        <th>Claim Value</th>
                                    </tr>
                                </thead>
                                <tbody>
                                    @foreach (var claim in User.Claims)
                                    {
                                        <tr>
                                            <td>@claim.Type</td>
                                            <td>@claim.Value</td>
                                        </tr>
                                    }
                                </tbody>
                            </table>
                        </div>
                    }
                    else
                    {
                        <div class="alert alert-warning">
                            <h4>Not Authenticated</h4>
                            <p>You are not currently authenticated. Please log in using one of the available authentication methods.</p>
                            <ul>
                                <li><a asp-controller="Account" asp-action="Login">Custom Cookie Authentication</a></li>
                                <li><a asp-area="Identity" asp-page="/Account/Login">ASP.NET Core Identity</a></li>
                                <li><a asp-area="MicrosoftIdentity" asp-controller="Account" asp-action="SignIn">Entra External ID</a></li>
                            </ul>
                        </div>
                    }
                </div>
                <div class="card-footer text-muted text-center">
                    <small><i class="bi bi-info-circle"></i> These are your identity claims from the authentication system</small>
                </div>
            </div>
        </div>
    </div>
    ```

2. Add a link to the WhoAmI page in your _Layout.cshtml footer:

    > `Views/Shared/_Layout.cshtml` (footer section)

    ```html
    <footer class="border-top footer text-muted">
        <div class="container d-flex justify-content-between align-items-center">
            <div>
                &copy; 2025 - Newsletter - <a asp-area="" asp-controller="Home" asp-action="Privacy">Privacy</a>
            </div>
            <div>
                <a asp-area="" asp-controller="Home" asp-action="WhoAmI" class="text-decoration-none">Who Am I?</a>
            </div>
        </div>
    </footer>
    ```

> 💡 **Information**
>
> - The WhoAmI view displays detailed information about the current authentication
> - It shows the authentication type, username, and all claims
> - Different color schemes help distinguish between authentication methods
> - The view provides login links when the user is not authenticated
>
> ⚠️ **Common Mistakes**
>
> - Not checking if User.Identity is null before accessing its properties
> - Assuming all claims exist for all authentication types
> - Forgetting to add the [Authorize] attribute to the WhoAmI action in HomeController

### Step 8: Add a Home Page with Authentication Options

Create or update the `Views/Home/Index.cshtml` view to showcase all available authentication options:

> `Views/Home/Index.cshtml`

```html
@{
    ViewData["Title"] = "Home Page";
}

<div class="text-center mb-4">
    <h1 class="display-4">Welcome to MerchStore</h1>
    <p class="lead">A demonstration of multiple authentication methods working side by side</p>
</div>

<div class="row">
    <div class="col-md-4 mb-4">
        <div class="card h-100">
            <div class="card-header bg-info text-white">
                <h5 class="mb-0">Default Cookie Authentication</h5>
            </div>
            <div class="card-body">
                <p>Simple cookie-based authentication with default user store.</p>
                <ul>
                    <li>Username: bob.admin</li>
                    <li>Password: admin</li>
                </ul>
                <p>or</p>
                <ul>
                    <li>Username: john.doe</li>
                    <li>Password: pass</li>
                </ul>
            </div>
            <div class="card-footer">
                <a asp-controller="Account" asp-action="Login" class="btn btn-outline-info w-100">Log in with Default Auth</a>
            </div>
        </div>
    </div>
    
    <div class="col-md-4 mb-4">
        <div class="card h-100">
            <div class="card-header bg-success text-white">
                <h5 class="mb-0">ASP.NET Core Identity</h5>
            </div>
            <div class="card-body">
                <p>Entity Framework Core with SQLite database for user management.</p>
                <p>Use the register option to create a new account.</p>
            </div>
            <div class="card-footer">
                <a asp-area="Identity" asp-page="/Account/Login" class="btn btn-outline-success w-100">Log in with Identity</a>
            </div>
        </div>
    </div>
    
    <div class="col-md-4 mb-4">
        <div class="card h-100">
            <div class="card-header bg-primary text-white">
                <h5 class="mb-0">Entra External ID</h5>
            </div>
            <div class="card-body">
                <p>Microsoft's modern authentication service for customer-facing apps.</p>
                <p>Sign in with a Microsoft account or create a new account.</p>
            </div>
            <div class="card-footer">
                <a asp-area="MicrosoftIdentity" asp-controller="Account" asp-action="SignIn" class="btn btn-outline-primary w-100">Log in with Entra External ID</a>
            </div>
        </div>
    </div>
</div>

<div class="text-center mt-4">
    <a asp-controller="Home" asp-action="WhoAmI" class="btn btn-dark">Check Your Identity</a>
</div>
```

> 💡 **Information**
>
> - The Home page provides a clear UI for users to choose between authentication methods
> - Each authentication method is presented with a card showing its features
> - The same protected resources are accessible with any authentication method
> - There are direct links to both the WhoAmI and AuthDebug pages
>
> ⚠️ **Common Mistakes**
>
> - Using incorrect routing for the authentication methods
> - Not providing clear instructions for users on each authentication option
> - Forgetting to include links to all authentication methods

## 🧪 Final Tests

### Run the Application and Validate Your Work

1. Start the application (with https):

   ```bash
   dotnet run --launch-profile https
   ```

2. Open a browser and navigate to:

   ```sh
   https://localhost:[PORT]/
   ```

3. Test each authentication method:

   **Default Cookie Authentication:**
   - Click "Login with Default Auth"
   - Login with bob.admin/admin or john.doe/pass
   - Verify you see the badge "Default" next to your username
   - Access `/Home/WhoAmI` to see your claims
   - Verify authentication type is "Cookies"
   - Use AuthDebug to verify only the `.AspNetCore.Cookies` cookie is present
   - Logout and check that the cookie is removed

   **ASP.NET Core Identity:**
   - Click "Login with Identity"
   - Register a new account or login with existing Identity credentials
   - Verify you see the badge "Identity" next to your username
   - Access `/Home/WhoAmI` to see your claims
   - Verify authentication type is "Identity.Application"
   - Use AuthDebug to verify only the `.AspNetCore.Identity.Application` cookie is present
   - Logout and check that the cookie is removed

   **Entra External ID:**
   - Click "Login with Entra External ID"
   - Complete the Microsoft authentication flow
   - Verify you see the badge "Entra ID" next to your username
   - Access `/Home/WhoAmI` to see your claims
   - Verify authentication type is "AuthenticationTypes.Federation"
   - Use AuthDebug to verify only the `.AspNetCore.Cookies.External` cookie is present
   - Logout and check that the cookie is removed

4. Test authentication debugging:
   - Navigate to `/Home/AuthDebug` after logging in with each method
   - Verify that the correct cookies are shown as present
   - Check that claims are displayed correctly
   - Use the test logout buttons to verify proper sign-out

5. Test the policy scheme routing:
   - Log in with one method, then manually delete the associated cookie
   - Refresh the page and verify you're no longer authenticated
   - Log in with a different method and verify it works correctly
   - Check the logs to see how the policy scheme routes requests

✅ **Expected Results**

- All three authentication methods work independently
- Each method uses its own distinct cookie
- The navigation shows which authentication method is active
- Protected resources are accessible with any valid authentication
- Logout works correctly for each authentication method
- The policy scheme correctly routes to the appropriate handler
- Authentication debugging tools provide helpful insights

## 🔧 Troubleshooting

If you encounter issues:

### Cookie-Related Issues

- **Problem**: Multiple authentication cookies exist simultaneously
  - **Solution**: Ensure proper sign-out for each authentication method
  - Check that you're using different cookie names for each scheme
  - Add code in sign-out events to delete lingering cookies
  
- **Problem**: Authentication cookie not being created
  - **Solution**: Verify your Entra External ID settings in Azure portal
  - Check that redirect URIs match exactly between code and portal
  - Inspect detailed logs to understand the authentication flow
  
- **Problem**: Entra External ID cookie persists after logout
  - **Solution**: Implement the OnRedirectToIdentityProviderForSignOut event
  - Ensure your sign-out handler deletes the cookie explicitly
  - Verify you're using the correct area for Microsoft Identity sign-out

### Policy Scheme Issues

- **Problem**: Wrong authentication scheme being selected
  - **Solution**: Debug cookie existence using AuthDebug view
  - Check the ForwardDefaultSelector logic in your policy scheme
  - Ensure cookie names match exactly what your code is checking for
  
- **Problem**: "No authentication handler is registered for the scheme"
  - **Solution**: Verify all required schemes are registered
  - Check for typos in scheme names
  - Ensure Microsoft Identity Web is properly configured

### Claims-Related Issues

- **Problem**: Missing or different claims between authentication methods
  - **Solution**: Compare claims using WhoAmI view
  - Understand that each provider supplies different claim types
  - Consider adding a ClaimsTransformation to normalize claims
  
- **Problem**: Role claims not being recognized
  - **Solution**: Check how roles are being stored in claims
  - Verify the RoleClaimType is correctly set in token validation parameters
  - Use User.IsInRole() method which checks for the correct claim type

## 🚀 Optional Challenge

Want to take your learning further? Try:

1. **Role-Based Authorization for Entra External ID**:
   - Configure app roles in Azure portal
   - Map claims to ASP.NET Core roles
   - Test admin functionality with all three authentication methods

2. **Enhanced Login UI**:
   - Create a unified login page with all three options
   - Use Bootstrap cards to explain each authentication method
   - Add provider icons for visual identification

3. **Account Linking**:
   - Implement a system to link accounts between authentication methods
   - Allow users to connect their local account with their Microsoft account
   - Maintain a consistent user experience across authentication methods

## 📚 Further Reading

- [Microsoft Identity Web Documentation](https://github.com/AzureAD/microsoft-identity-web/wiki)
- [Multiple Authentication Schemes in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/limitingidentitybyscheme)
- [Authentication Scheme and Cookie Names](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/cookie)
- [Policy Schemes in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/policyschemes)
- [Azure Entra External ID Documentation](https://learn.microsoft.com/en-us/entra/external-id/)

## Done! 🎉

Great job! You've successfully implemented a triple authentication system with custom cookie authentication, ASP.NET Core Identity, and Azure Entra External ID. This sophisticated approach demonstrates your understanding of authentication schemes, cookie management, and policy-based routing in ASP.NET Core! 🚀

## Appendix

### Appendix A: Authentication Scheme Terms Explained

Understanding the difference between authentication terms:

| Term | Definition | Example |
|------|------------|---------|
| **Authentication Scheme** | Named configuration for an authentication handler | "Cookies", "Identity.Application", "OpenIdConnect" |
| **Authentication Handler** | Component that processes authentication | CookieAuthenticationHandler, OpenIdConnectHandler |
| **Cookie Name** | Actual name of the cookie stored in the browser | ".AspNetCore.Cookies", ".AspNetCore.Identity.Application" |
| **Authentication Type** | Value in the AuthenticationType property | "Cookies", "Identity.Application", "AuthenticationTypes.Federation" |
| **Policy Scheme** | Special scheme that routes to other schemes | Our "CustomMultiAuthScheme" |

### Appendix B: Default Scheme and Cookie Names

| Authentication | Default Scheme Name | Default Cookie Name | Default Paths |
|----------------|---------------------|---------------------|---------------|
| Cookie Auth | "Cookies" | ".AspNetCore.Cookies" | Login: "/Account/Login"  Logout: "/Account/Logout"  AccessDenied: "/Account/AccessDenied" |
| Identity | "Identity.Application" | ".AspNetCore.Identity.Application" | Login: "/Identity/Account/Login"  Logout: "/Identity/Account/Logout"  AccessDenied: "/Identity/Account/AccessDenied" |
| OpenID Connect | "OpenIdConnect" | Varies, we set: ".AspNetCore.Cookies.External" | Login: "/MicrosoftIdentity/Account/SignIn"  Logout: "/MicrosoftIdentity/Account/SignOut"  CallbackPath: "/signin-oidc" |

### Appendix C: Understanding the Policy Scheme Router

The policy scheme router works by:

1. **Request arrives** at the application
2. **Authentication middleware** processes the request using the default scheme ("CustomMultiAuthScheme")
3. **Policy scheme handler** examines cookies using ForwardDefaultSelector:

   ```csharp
   options.ForwardDefaultSelector = context => {
       if (context.Request.Cookies.ContainsKey(".AspNetCore.Cookies"))
           return "Cookies";
       else if (context.Request.Cookies.ContainsKey(".AspNetCore.Cookies.External"))
           return "OpenIdConnect";
       else
           return "Identity.Application";
   };
   ```

4. **Authentication continues** with the selected scheme's handler
5. **Result is cached** for the duration of the request

### Appendix D: Microsoft Identity Web Configuration Options

Important options for Microsoft Identity Web:

```csharp
.AddMicrosoftIdentityWebApp(options => {
    // Load from configuration
    builder.Configuration.GetSection("AzureAd").Bind(options);
    
    // The scheme that will issue the cookie after successful authentication
    options.SignInScheme = "Cookies.External";
    
    // Security token validation parameters
    options.TokenValidationParameters = new TokenValidationParameters {
        NameClaimType = "preferred_username",
        RoleClaimType = "roles"
    };
    
    // Handle sign-out to clean up cookies
    options.Events = new OpenIdConnectEvents {
        OnRedirectToIdentityProviderForSignOut = context => {
            context.Response.Cookies.Delete(".AspNetCore.Cookies.External");
            return Task.CompletedTask;
        }
    };
}, 
// Cookie scheme to use for signIn (token validation result)
cookieScheme: "Cookies.External", 
// OpenID Connect scheme name
openIdConnectScheme: OpenIdConnectDefaults.AuthenticationScheme);
```

### Appendix E: Common Authentication Event Handlers

```csharp
options.Events = new OpenIdConnectEvents
{
    // Called when redirecting to the identity provider for sign-in
    OnRedirectToIdentityProvider = context => {
        // Customize the redirect URL or add parameters
        return Task.CompletedTask;
    },
    
    // Called when a successful authorization code was received
    OnAuthorizationCodeReceived = context => {
        // Exchange the code for tokens
        return Task.CompletedTask;
    },
    
    // Called when the user has been successfully authenticated by the identity provider
    OnTokenValidated = context => {
        // Access and modify claims
        var identity = context.Principal.Identity as ClaimsIdentity;
        if (identity != null)
        {
            // Add or modify claims
            identity.AddClaim(new Claim("custom_claim", "custom_value"));
        }
        return Task.CompletedTask;
    },
    
    // Called when redirecting to the identity provider for sign-out
    OnRedirectToIdentityProviderForSignOut = context => {
        // Delete cookies or perform other cleanup
        context.Response.Cookies.Delete(".AspNetCore.Cookies.External");
        return Task.CompletedTask;
    },
    
    // Called when there's an error during the authentication process
    OnRemoteFailure = context => {
        // Handle authentication errors
        _logger.LogError("Remote authentication error: {Error}", context.Failure?.Message);
        context.Response.Redirect("/Error/AuthFailed");
        context.HandleResponse(); // Suppress the default error handling
        return Task.CompletedTask;
    }
};
```

These event handlers give you fine-grained control over the authentication process, allowing you to customize behavior at each step.
