---
title: "Security Enhancements - Roles, Hashing, and Cookie Security"
date: "2025-05-05"
lastmod: "2025-05-05"
draft: false
weight: 602
toc: true
---

## 🎯 Goal

Enhance your application security by implementing role-based authorization, password hashing, and secure cookie configuration to create a more robust authentication system.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 1 (Implementing Cookie Authentication)
- Understand basic authentication and authorization concepts
- Be familiar with ASP.NET Core middleware configuration

## 📚 Learning Objectives

By the end of this exercise, you will:

- Implement **role-based authorization** to control access to different parts of your application
- Use **BCrypt** for secure password hashing
- Configure **enhanced cookie security** settings to protect against common attacks
- Create **authorization policies** for complex access control scenarios
- Be **redirected** to the page you tried to access after successful login

## 🔍 Why This Matters

In real-world applications, enhanced security is crucial because:

- Role-based access control enables different user privileges and responsibilities
- Password hashing protects user credentials even if the database is compromised
- Secure cookie configuration prevents common web vulnerabilities like XSS and CSRF
- It demonstrates professional security practices expected in production applications

## 📝 Step-by-Step Instructions

### Step 1: Install BCrypt.Net-Next Package

1. Open a terminal in your project directory.
2. Install the BCrypt.Net-Next NuGet package:

   ```bash
   dotnet add package BCrypt.Net-Next
   ```

3. Verify the package was added to your project file:

    > `MerchStore.csproj`

    ```xml
    <ItemGroup>
      <PackageReference Include="BCrypt.Net-Next" Version="4.0.3" />
    </ItemGroup>
    ```

> 💡 **Information**
>
> - **BCrypt**: A password hashing function designed to be computationally expensive
> - **Work Factor**: BCrypt allows you to adjust the computational cost as hardware improves
> - **Salting**: BCrypt automatically generates and stores a unique salt for each password
>
> ⚠️ **Common Mistakes**
>
> - Using outdated hashing algorithms like MD5 or SHA1 for passwords
> - Not using a sufficiently high work factor (minimum 10-12 recommended)
> - Trying to implement your own hashing algorithm

### Step 2: Create User Roles Class

1. Navigate to the `Models` folder.
2. Create a new file named `UserRoles.cs`.
3. Add the following code:

    > `Models/UserRoles.cs`

    ```csharp
    namespace MerchStore.Models;

    public static class UserRoles
    {
        public const string Administrator = "Administrator";
        public const string Customer = "Customer";
        
        public static IEnumerable<string> AllRoles => new[] { Administrator, Customer };
    }
    ```

> 💡 **Information**
>
> - **Static Class**: Provides a centralized place for role definitions
> - **Const Strings**: Prevents typos and enables compile-time checking
> - **AllRoles Property**: Useful for seeding roles or validation

### Step 3: Update AccountController with Roles and Password Hashing

1. Open `Controllers/AccountController.cs`.
2. Replace the existing code with the following enhanced version:

    > `Controllers/AccountController.cs`

    ```csharp
    using System.Security.Claims;
    using MerchStore.Models;
    using Microsoft.AspNetCore.Authentication;
    using Microsoft.AspNetCore.Authentication.Cookies;
    using Microsoft.AspNetCore.Authorization;
    using Microsoft.AspNetCore.Mvc;

    namespace MerchStore.Controllers;

    public class AccountController : Controller
    {
        // Simulated user database - in production, this would be in a database
        private static readonly Dictionary<string, (string PasswordHash, string Role)> Users = new()
        {
            // Password: "admin123" (hashed with BCrypt)
            ["admin"] = ("$2a$11$rBNxyD7V1aPNtsqTM5hAj.kxd67q7wTBVRUPnnLU9OYbTpNx8xfQm", UserRoles.Administrator),
            
            // Password: "password123" (hashed with BCrypt)
            ["john.doe"] = ("$2a$11$J7IZK4jZMYJGCCKU/NUEBOxWts7eAGWmrjbYbzchuaa.bXNBGKrDS", UserRoles.Customer)
        };

        [HttpGet]
        public IActionResult Login(string? returnUrl = null)
        {
            ViewData["ReturnUrl"] = returnUrl;
            return View();
        }

        [HttpPost]
        [ValidateAntiForgeryToken]
        public async Task<IActionResult> LoginAsync(LoginViewModel model, string? returnUrl = null)
        {
            ViewData["ReturnUrl"] = returnUrl;

            if (!ModelState.IsValid)
            {
                return View(model);
            }

            // Check if user exists and verify password
            if (Users.TryGetValue(model.Username ?? "", out var userData) && 
                BCrypt.Net.BCrypt.Verify(model.Password, userData.PasswordHash))
            {
                // Create claims including role
                var claims = new List<Claim>
                {
                    new Claim(ClaimTypes.Name, model.Username!),
                    new Claim(ClaimTypes.Role, userData.Role)
                };

                var identity = new ClaimsIdentity(claims, CookieAuthenticationDefaults.AuthenticationScheme);
                var principal = new ClaimsPrincipal(identity);

                // Sign in with enhanced security options
                await HttpContext.SignInAsync(
                    CookieAuthenticationDefaults.AuthenticationScheme,
                    principal);

                // Redirect to return URL if valid, otherwise to home
                if (!string.IsNullOrEmpty(returnUrl) && Url.IsLocalUrl(returnUrl))
                {
                    return Redirect(returnUrl);
                }

                return RedirectToAction("Index", "Home");
            }

            ModelState.AddModelError(string.Empty, "Invalid login attempt.");
            return View(model);
        }

        [HttpPost]
        [ValidateAntiForgeryToken]
        public async Task<IActionResult> Logout()
        {
            await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
            return RedirectToAction("Index", "Home");
        }

        [HttpGet]
        public IActionResult AccessDenied()
        {
            return View();
        }

        // Utility method to hash passwords (for demonstration)
        [HttpGet]
        [Authorize(Roles = UserRoles.Administrator)]
        public IActionResult HashPassword(string password)
        {
            if (string.IsNullOrEmpty(password))
            {
                return BadRequest("Password is required");
            }

            var hash = BCrypt.Net.BCrypt.HashPassword(password);
            return Ok(new { password, hash });
        }
    }
    ```

> 💡 **Information**
>
> - **BCrypt.Verify**: Compares a plain text password with a hash
> - **Role Claims**: Added to the user's identity for authorization
> - **ReturnUrl**: Allows redirecting back to the originally requested page
> - **Url.IsLocalUrl**: Prevents open redirect vulnerabilities
> - **HashPassword Action**: Admin-only utility for generating password hashes
>
> ⚠️ **Common Mistakes**
>
> - Not validating returnUrl can lead to open redirect attacks
> - Storing plain text passwords instead of hashes
> - Not including roles in claims for authorization

### Step 4: Configure Enhanced Cookie Security

1. Open `Program.cs`.
2. Update the authentication configuration with enhanced security options:

    > `Program.cs`

    ```csharp
    using Microsoft.AspNetCore.Authentication.Cookies;
    using MerchStore.Models;

    var builder = WebApplication.CreateBuilder(args);

    // Add services to the container.
    builder.Services.AddControllersWithViews();

    // Configure cookie authentication with enhanced security
    builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
        .AddCookie(options =>
        {
            // Cookie settings
            options.Cookie.HttpOnly = true;
            options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
            options.Cookie.SameSite = SameSiteMode.Lax;
            options.Cookie.Name = "MerchStore.Auth";
            
            // Expiration settings
            options.ExpireTimeSpan = TimeSpan.FromMinutes(60);
            options.SlidingExpiration = true;
            
            // Authentication paths
            options.LoginPath = "/Account/Login";
            options.LogoutPath = "/Account/Logout";
            options.AccessDeniedPath = "/Account/AccessDenied";
        });

    // Configure authorization policies
    builder.Services.AddAuthorization(options =>
    {
        options.AddPolicy("AdminOnly", policy => 
            policy.RequireRole(UserRoles.Administrator));
        
        options.AddPolicy("AdminOrCustomer", policy => 
            policy.RequireRole(UserRoles.Administrator, UserRoles.Customer));
    });

    var app = builder.Build();
    ```

> 💡 **Information**
>
> - **HttpOnly**: Prevents JavaScript access to cookies (XSS protection)
> - **SecurePolicy**: Ensures cookies are only sent over HTTPS
> - **SameSite**: Provides CSRF protection by controlling when cookies are sent
> - **SlidingExpiration**: Extends the session on user activity
> - **Authorization Policies**: Define reusable access control rules
>
> ⚠️ **Common Mistakes**
>
> - Using SameSiteMode.None without understanding the security implications
> - Not setting SecurePolicy for production applications
> - Creating overly permissive authorization policies

### Step 5: Create Admin Dashboard

1. Create a new controller for the admin area.
2. Create a new file `Controllers/AdminController.cs`:

    > `Controllers/AdminController.cs`

    ```csharp
    using Microsoft.AspNetCore.Authorization;
    using Microsoft.AspNetCore.Mvc;
    using MerchStore.Models;

    namespace MerchStore.Controllers;

    [Authorize(Policy = "AdminOnly")]
    public class AdminController : Controller
    {
        public IActionResult Dashboard()
        {
            return View();
        }

        public IActionResult Users()
        {
            // In a real application, this would fetch from a database
            var users = new List<string> { "admin", "john.doe" };
            return View(users);
        }
    }
    ```

3. Create the dashboard view in `Views/Admin/Dashboard.cshtml`:

    > `Views/Admin/Dashboard.cshtml`

    ```html
    @{
        ViewData["Title"] = "Admin Dashboard";
    }
    
    <div class="row">
        <div class="col-md-12">
            <h1 class="display-4">Admin Dashboard</h1>
            <p class="lead">Welcome to the administration area.</p>
        </div>
    </div>
    
    <div class="row mt-4">
        <div class="col-md-12">
            <div class="alert alert-info">
                <h5>Security Information</h5>
                <p>You are logged in as: <strong>@User.Identity?.Name</strong></p>
                <p>Your role: <strong>@User.FindFirst(System.Security.Claims.ClaimTypes.Role)?.Value</strong></p>
            </div>
        </div>
    </div>
    ```

### Step 6: Create Access Denied View

1. Create a view for handling unauthorized access attempts.
2. Create `Views/Account/AccessDenied.cshtml`:

    > `Views/Account/AccessDenied.cshtml`

    ```html
    @{
        ViewData["Title"] = "Access Denied";
    }

    <div class="row justify-content-center">
        <div class="col-md-8">
            <div class="card border-danger">
                <div class="card-header bg-danger text-white">
                    <h2 class="fs-4 mb-0">Access Denied</h2>
                </div>
                <div class="card-body">
                    <div class="alert alert-danger">
                        <h4 class="alert-heading">Unauthorized Access</h4>
                        <p>You do not have permission to access this resource.</p>
                        <hr>
                        <p class="mb-0">Please contact your administrator if you believe this is an error.</p>
                    </div>
                    <div class="mt-3">
                        <a href="@Url.Action("Index", "Home")" class="btn btn-primary">Return to Home</a>
                    </div>
                </div>
            </div>
        </div>
    </div>
    ```

### Step 7: Update Navigation to Include Admin Link

1. Open `Views/Shared/_Layout.cshtml`.
2. Add an admin navigation link that only shows for administrators:

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
            @if (User.IsInRole(MerchStore.Models.UserRoles.Administrator))
            {
                <li class="nav-item">
                    <a class="nav-link text-dark" asp-area="" asp-controller="Admin" asp-action="Dashboard">Admin</a>
                </li>
            }
        </ul>
        <partial name="_LoginPartial" />
    </div>
    ```

### Step 8: Update Login View to Show Available Credentials

1. Open `Views/Account/Login.cshtml`.
2. Update the footer to show the new test credentials:

    > `Views/Account/Login.cshtml`

    ```html
    <div class="card-footer text-muted text-center">
        <small>
            <i class="bi bi-info-circle"></i> Test Credentials:<br>
            Admin: admin / admin123<br>
            Customer: john.doe / password123
        </small>
    </div>
    ```

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

3. Test role-based authorization:

   - Login as a customer (john.doe/pass)
   - Try to access /Admin/Dashboard (should see Access Denied)
   - Logout and login as admin (bob.admin/admin)
   - Access /Admin/Dashboard (should see the dashboard)

4. Test password hashing:

   - As admin, navigate to /Account/HashPassword?password=test123
   - Observe the BCrypt hash output
   - Try logging in with wrong passwords

5. Verify enhanced cookie security:

   - Open browser developer tools
   - Check the cookies section
   - Verify the auth cookie has Secure, HttpOnly flags

✅ **Expected Results**

- Admin users can access the admin dashboard
- Non-admin users see an access denied page
- Passwords are verified against BCrypt hashes
- Authentication cookies have proper security flags
- Navigation shows admin link only for administrators

## 🔧 Troubleshooting

If you encounter issues:

- Check that role claims are properly added during login
- Ensure authorization policies match the roles defined
- Verify BCrypt password hashes are correct
- Check browser console for cookie security warnings

## 🚀 Optional Challenge

Want to take your learning further? Try:

- Implementing a user registration system with password hashing
- Adding more granular permissions with custom policies
- Creating a role management interface for administrators
- Implementing password reset functionality

## 📚 Further Reading

- [ASP.NET Core Role-based Authorization](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/roles) - Microsoft documentation
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) - Security best practices
- [ASP.NET Core Cookie Authentication Security](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/cookie) - Cookie security options

## Done! 🎉

Great job! You've successfully implemented **role-based authorization**, **password hashing**, and **enhanced cookie security**! This creates a more secure and professional authentication system for your web applications. 🚀

## Appendix

### Appendix A: Understanding BCrypt

BCrypt is a password hashing function designed specifically for password storage:

1. **Work Factor**: BCrypt allows you to specify a cost factor that determines how computationally expensive the hashing process is. This helps future-proof against improvements in hardware.

   ```csharp
   // Work factor of 11 (2^11 iterations)
   var hash = BCrypt.Net.BCrypt.HashPassword("password", 11);
   ```

2. **Salt Generation**: BCrypt automatically generates a random salt for each password, which is stored as part of the hash.

3. **Hash Format**: BCrypt hashes look like: `$2a$11$salt22characters...hash31characters...`
   - `$2a$`: Algorithm identifier
   - `$11$`: Work factor
   - Next 22 characters: Salt
   - Remaining characters: Hash

### Appendix B: Cookie Security Options

Understanding cookie security options is crucial for web application security:

| Option | Purpose | Recommended Setting |
|--------|---------|-------------------|
| HttpOnly | Prevents JavaScript access | `true` |
| Secure | HTTPS only | `CookieSecurePolicy.Always` |
| SameSite | CSRF protection | `SameSiteMode.Lax` or `Strict` |
| Domain | Cookie scope | Specific domain |
| Path | URL path scope | As restrictive as possible |
| Expires | Absolute expiration | Based on security needs |

### Appendix C: Role-Based vs Claims-Based Authorization

ASP.NET Core supports both role-based and claims-based authorization:

1. **Role-Based Authorization**:
   - Simple, hierarchical access control
   - Good for basic permission systems
   - Example: `[Authorize(Roles = "Admin,Manager")]`

2. **Claims-Based Authorization**:
   - More flexible and granular
   - Can include any type of user information
   - Example: `[Authorize(Policy = "MinimumAge")]`

3. **Policy-Based Authorization**:
   - Combines multiple requirements
   - Enables complex authorization logic
   - Example:

     ```csharp
     options.AddPolicy("ContentEditor", policy =>
         policy.RequireClaim("Department", "Marketing")
               .RequireRole("Employee"));
     ```

Understanding these different approaches helps in designing appropriate authorization systems for your applications.
