---
title: "Implementing Cookie Authentication"
date: "2025-05-05"
lastmod: "2025-05-05"
draft: false
weight: 601
toc: true
---

## 🎯 Goal

Enhance your application by implementing cookie-based authentication to improve security and provide user-specific experiences in your MerchStore application.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 2 (Creating a Simple Product Entity)
- Understand basic authentication concepts
- Be familiar with ASP.NET Core MVC patterns

## 📚 Learning Objectives

By the end of this exercise, you will:

- Implement **cookie-based authentication** to secure your application
- Use **ASP.NET Core Identity** middleware for authentication
- Configure **authorization policies** to handle protected resources
- Understand how **claims-based identity** works within the ASP.NET Core framework

## 🔍 Why This Matters

In real-world applications, authentication is crucial because:

- It enables secure access to user-specific features and data
- It's an industry standard approach for web application security
- It will be foundational for building user accounts, shopping carts, and order history features later

## 📝 Step-by-Step Instructions

### Step 1: Configure Cookie Authentication in Program.cs

1. Open `Program.cs` in the project root.
2. Add the cookie authentication services before the `builder.Build()` call.

    > `Program.cs`

    ```csharp
    using Microsoft.AspNetCore.Authentication.Cookies;

    var builder = WebApplication.CreateBuilder(args);

    // Add services to the container.
    builder.Services.AddControllersWithViews();

    // Add cookie authentication services
    builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
        .AddCookie();

    var app = builder.Build();
    ```

3. Configure the authentication middleware in the request pipeline:

    > `Program.cs`

    ```csharp
    app.UseHttpsRedirection();
    app.UseRouting();

    app.UseAuthentication(); // Before UseAuthorization()
    app.UseAuthorization();

    app.MapStaticAssets();
    ```

> 💡 **Information**
>
> - **Authentication Services**: `AddAuthentication()` registers the authentication services with the default scheme
> - **Cookie Authentication**: `AddCookie()` configures cookie-based authentication with default settings
> - **Middleware Order**: Authentication middleware must come before authorization middleware
> - **Default Settings**: The default cookie configuration uses reasonably secure settings, but can be customized for production
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to add `UseAuthentication()` middleware will prevent authentication from working
> - Placing authentication middleware after authorization middleware will cause authentication failures

### Step 2: Create the Login View Model

1. Navigate to the `Models` folder.
2. Create a new file named `LoginViewModel.cs`.
3. Add the following code:

    > `Models/LoginViewModel.cs`

    ```csharp
    using System.ComponentModel.DataAnnotations;

    namespace MerchStore.Models;

    public class LoginViewModel
    {
        [Required]
        public string? Username { get; set; }

        [Required]
        [DataType(DataType.Password)]
        public string? Password { get; set; }
    }
    ```

> 💡 **Information**
>
> - **Data Annotations**: Provide both server-side and client-side validation
> - **Password DataType**: Tells ASP.NET Core to render this as a password input field
> - **Nullable Properties**: Properties are nullable to allow model binding to work correctly with validation

### Step 3: Create the Account Controller

1. Navigate to the `Controllers` folder.
2. Create a new file named `AccountController.cs`.
3. Add the following code:

    > `Controllers/AccountController.cs`

    ```csharp
    using System.Security.Claims;
    using MerchStore.Models;
    using Microsoft.AspNetCore.Authentication;
    using Microsoft.AspNetCore.Authentication.Cookies;
    using Microsoft.AspNetCore.Mvc;

    namespace MerchStore.Controllers;

    public class AccountController : Controller
    {
        // Mocked user "database" for demonstration purposes.
        private const string MockedUsername = "john.doe";
        private const string MockedPassword = "pass"; // Note: NEVER hard-code passwords in real applications.

        // This is a simple login page that allows users to enter their credentials.
        [HttpGet]
        public IActionResult Login()
        {
            return View();
        }

        // This action handles the login form submission.
        [HttpPost]
        [ValidateAntiForgeryToken] // This ensures that the form is submitted with a valid anti-forgery token to prevent CSRF attacks.
        public async Task<IActionResult> LoginAsync(LoginViewModel model)
        {
            // Check model validators
            if (!ModelState.IsValid)
            {
                return View(model);
            }

            // Verify the user's credentials against the mocked database.
            if (model.Username == MockedUsername && model.Password == MockedPassword)
            {
                // Set up the session/cookie for the authenticated user.
                var claims = new[] { new Claim(ClaimTypes.Name, model.Username) };
                var identity = new ClaimsIdentity(claims, CookieAuthenticationDefaults.AuthenticationScheme);
                var principal = new ClaimsPrincipal(identity);

                // Sign in the user with the cookie authentication scheme.
                await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, principal);

                // Redirect to a secure area of your application
                return RedirectToAction("Index", "Home");
            }

            ModelState.AddModelError(string.Empty, "Invalid login attempt."); // Generic error message for security reasons.
            return View(model);
        }

        [HttpPost]
        [ValidateAntiForgeryToken]
        public async Task<IActionResult> Logout()
        {
            // Sign out the user by removing the authentication cookie.
            await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);

            // Redirect to a public area of your application
            return RedirectToAction("Index", "Home");
        }
    }
    ```

> 💡 **Information**
>
> - **Claims Identity**: Represents the user's identity with a collection of claims (like username)
> - **Principal**: Encapsulates the identity and can contain multiple identities
> - **Anti-forgery Token**: Prevents Cross-Site Request Forgery (CSRF) attacks
> - **Generic Error Message**: Avoid revealing whether username or password was incorrect for security
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to use `async` with authentication operations will cause compilation errors
> - Not validating model state can lead to security vulnerabilities
> - Revealing specific error messages can help attackers enumerate valid usernames

### Step 4: Create the Login View

1. Navigate to the `Views/Account` folder (create it if it doesn't exist).
2. Create a new file named `Login.cshtml`.
3. Add the following code:

    > `Views/Account/Login.cshtml`

    ```html
    @model MerchStore.Models.LoginViewModel

    @{
        ViewData["Title"] = "Login";
    }

    <div class="row justify-content-center">
        <div class="col-md-6">
            <div class="card shadow">
                <div class="card-header bg-primary text-white">
                    <h2 class="fs-4 mb-0">@ViewData["Title"]</h2>
                </div>
                <div class="card-body">
                    <form method="post" asp-antiforgery="true">
                        <div asp-validation-summary="All" class="text-danger"></div>
                        
                        <div class="mb-3">
                            <label asp-for="Username" class="form-label"></label>
                            <input asp-for="Username" class="form-control" autocomplete="username" aria-required="true" />
                            <span asp-validation-for="Username" class="text-danger"></span>
                        </div>
                        
                        <div class="mb-3">
                            <label asp-for="Password" class="form-label"></label>
                            <input asp-for="Password" class="form-control" autocomplete="current-password" aria-required="true" />
                            <span asp-validation-for="Password" class="text-danger"></span>
                        </div>
                        
                        <div>
                            <button type="submit" class="btn btn-primary w-100">Log in</button>
                        </div>
                    </form>
                </div>
                <div class="card-footer text-muted text-center">
                    <small><i class="bi bi-info-circle"></i> For testing, use username: john.doe, password: pass</small>
                </div>
            </div>
        </div>
    </div>

    @section Scripts {
        <partial name="_ValidationScriptsPartial" />
    }
    ```

> 💡 **Information**
>
> - **Anti-forgery Token**: The `asp-antiforgery="true"` attribute automatically includes the token
> - **Validation Summary**: Displays all validation errors in one place
> - **Autocomplete Attributes**: Help password managers work correctly
> - **Bootstrap Styling**: Uses Bootstrap classes for responsive layout and styling

### Step 5: Create the Login Partial View

1. Navigate to the `Views/Shared` folder.
2. Create a new file named `_LoginPartial.cshtml`.
3. Add the following code:

    > `Views/Shared/_LoginPartial.cshtml`

    ```html
    @using Microsoft.AspNetCore.Authentication.Cookies;

    <ul class="navbar-nav ms-auto">
        @if (User.Identity != null && User.Identity.IsAuthenticated && User.Identity.AuthenticationType == CookieAuthenticationDefaults.AuthenticationScheme)
        {
            @* The user is authenticated and the authentication type is CookieAuthenticationDefaults.AuthenticationScheme *@
            <li class="nav-item d-flex align-items-center">
                <span class="navbar-text text-dark me-3">Hello @User.Identity.Name!</span>
            </li>
            <li class="nav-item">
                <form class="form-inline" asp-controller="Account" asp-action="Logout" method="post">
                     @* Add Antiforgery token for POST logout *@
                     @Html.AntiForgeryToken() 
                    <button type="submit" class="btn btn-primary">Logout</button>
                </form>
            </li>
        }
        else
        {
            @* The user is not authenticated *@
            <li class="nav-item">
                <a class="btn btn-primary" asp-controller="Account" asp-action="Login">Login</a>
            </li>
        }
    </ul>
    ```

> 💡 **Information**
>
> - **Identity Checking**: Verifies the user is authenticated before showing logout button
> - **Authentication Type Check**: Ensures we're using the cookie authentication scheme
> - **POST for Logout**: Uses a form with POST method for logout to prevent CSRF attacks
> - **Conditional Rendering**: Shows different UI based on authentication state

### Step 6: Update the Layout to Include Login Partial

1. Open `Views/Shared/_Layout.cshtml`.
2. Add the login partial just before closing the navbar-collapse div:

    > `Views/Shared/_Layout.cshtml`

    ```html
    <div class="navbar-collapse collapse d-sm-inline-flex justify-content-between">
        <ul class="navbar-nav flex-grow-1">
            <li class="nav-item">
                <a class="nav-link text-dark" asp-area="" asp-controller="Home" asp-action="Index">Home</a>
            </li>
            <li class="nav-item">
                <a class="nav-link text-dark" asp-area="" asp-controller="Home" asp-action="Privacy">Privacy</a>
            </li>
        </ul>
        <partial name="_LoginPartial" />
    </div>
    ```

### Step 7: Secure a Controller Action

1. Open `Controllers/HomeController.cs`.
2. Add the `[Authorize]` attribute to protect a specific action:

    > `Controllers/HomeController.cs`

    ```csharp
    using Microsoft.AspNetCore.Authorization;

    // ... existing code ...

    [Authorize] // This attribute ensures that only authenticated users can access this action.
    public IActionResult WhoAmI()
    {
        return View();
    }
    ```

3. Create the corresponding view in `Views/Home/WhoAmI.cshtml`:

    > `Views/Home/WhoAmI.cshtml`

    ```html
    <div class="row justify-content-center">
        <div class="col-md-8">
            <div class="card shadow">
                <div class="card-header bg-primary text-white">
                    <h2 class="fs-4 mb-0">Who Am I?</h2>
                </div>
                <div class="card-body">
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
                </div>
                <div class="card-footer text-muted text-center">
                    <small><i class="bi bi-info-circle"></i> These are your identity claims from the authentication system</small>
                </div>
            </div>
        </div>
    </div>
    ```

> 💡 **Information**
>
> - **[Authorize] Attribute**: Restricts access to authenticated users only
> - **Claims Display**: Shows all claims associated with the authenticated user
> - **User.Claims**: Accessible in views and controllers for the current authenticated user
> - **Automatic Redirection**: Unauthenticated users are automatically redirected to the login page

## 🧪 Final Tests

### Run the Application and Validate Your Work

1. Start the application:

   ```bash
   dotnet run
   ```

2. Open a browser and navigate to:

   ```sh
   http://localhost:[PORT]/
   ```

3. Test the authentication by:

   - Clicking the "Login" button
   - Entering invalid credentials (should see error message)
   - Entering valid credentials (john.doe/pass)
   - Verifying you're redirected and see "Hello john.doe!"
   - Accessing the "Who Am I?" page (should work when authenticated)

4. Test authorization:

   - Click "Logout"
   - Try to access `/Home/WhoAmI` directly (should redirect to login)

✅ **Expected Results**

- The login page should display with proper styling
- Invalid credentials should show a generic error message
- Valid credentials should authenticate and redirect to home page
- The navigation bar should show "Hello [username]!" when authenticated
- Protected pages should only be accessible when authenticated
- Logout should clear the authentication cookie

## 🔧 Troubleshooting

If you encounter issues:

- Check that authentication middleware is configured before authorization
- Ensure anti-forgery tokens are included in POST forms
- Verify the correct authentication scheme is used consistently
- Check for proper async/await usage in authentication actions

## 🚀 Optional Challenge

Want to take your learning further? Try:

- Implementing "Remember Me" functionality with persistent cookies
- Adding role-based authorization with multiple user types
- Creating a registration page for new users
- Implementing password hashing for secure credential storage

## 📚 Further Reading

- [ASP.NET Core Authentication Documentation](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/) - Official Microsoft documentation
- [Cookie Authentication in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/cookie) - Detailed cookie authentication guide
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) - Security best practices

## Done! 🎉

Great job! You've successfully implemented **cookie-based authentication** and learned how to use **claims-based identity** in ASP.NET Core! This knowledge will help you build more secure and user-specific features in your web applications. 🚀

## Appendix

### Appendix A: Understanding Cookie Authentication Flow

Cookie authentication in ASP.NET Core follows a specific flow that's important to understand:

1. **Authentication Process**:

   ```sh
   User submits credentials → Controller validates → Creates claims identity → 
   Signs in user → Cookie is created → User is redirected
   ```

2. **Subsequent Requests**:

   ```sh
   Browser sends cookie → Middleware validates cookie → 
   Creates ClaimsPrincipal → Request continues with authenticated context
   ```

3. **Cookie Contents**:

   - The authentication cookie contains encrypted user claims
   - It's HttpOnly by default (not accessible via JavaScript)
   - Can be configured with various security options

4. **Sign Out Process**:

   ```sh
   User clicks logout → Controller calls SignOutAsync → 
   Cookie is invalidated → User loses authenticated status
   ```

Understanding this flow helps in debugging authentication issues and implementing more complex authentication scenarios.

### Appendix B: Cookie Security Best Practices

When implementing cookie authentication, consider these security best practices:

1. **Secure Cookie Settings**:

   ```csharp
   builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
       .AddCookie(options =>
       {
           options.Cookie.HttpOnly = true;
           options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
           options.Cookie.SameSite = SameSiteMode.Strict;
           options.ExpireTimeSpan = TimeSpan.FromMinutes(30);
           options.SlidingExpiration = true;
       });
   ```

2. **Password Storage**:
   - Never store plain-text passwords
   - Use strong hashing algorithms (like BCrypt or Argon2)
   - Implement proper salt generation

3. **Session Management**:
   - Set appropriate session timeouts
   - Implement sliding expiration for active users
   - Provide secure logout functionality

4. **CSRF Protection**:
   - Always use anti-forgery tokens for POST actions
   - Validate tokens on the server side
   - Use proper HTTP methods (POST for state-changing operations)

These practices help protect against common web security vulnerabilities like session hijacking, cross-site scripting (XSS), and cross-site request forgery (CSRF).

### Appendix C: Claims-Based Identity

Claims-based identity is central to ASP.NET Core authentication:

1. **What are Claims?**
   - Claims are statements about a user (name, email, role, etc.)
   - They form the basis of authorization decisions
   - Claims are key-value pairs stored in the authentication cookie

2. **Common Claim Types**:

   ```csharp
   var claims = new List<Claim>
   {
       new Claim(ClaimTypes.Name, username),
       new Claim(ClaimTypes.Email, email),
       new Claim(ClaimTypes.Role, "Administrator"),
       new Claim("Department", "Sales"),
       new Claim("EmployeeId", "12345")
   };
   ```

3. **Using Claims for Authorization**:

   ```csharp
   [Authorize(Roles = "Administrator")]
   public IActionResult AdminPanel()
   {
       return View();
   }
   ```

4. **Accessing Claims in Code**:

   ```csharp
   // In a controller
   var userName = User.FindFirst(ClaimTypes.Name)?.Value;
   var isAdmin = User.IsInRole("Administrator");
   
   // In a view
   @if (User.HasClaim("Department", "Sales"))
   {
       <p>Sales Department Content</p>
   }
   ```

Understanding claims helps in building flexible authorization systems that can adapt to complex business requirements.
