---
title: "Adding ASP.NET Core Identity with EF Core"
date: "2025-05-05"
lastmod: "2025-05-05"
draft: false
weight: 603
toc: true
---

## 🎯 Goal

Enhance your application by implementing ASP.NET Core Identity with Entity Framework Core and SQLite alongside your existing cookie authentication system, allowing users to choose between two authentication methods.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 2 (Security Enhancements - Roles, Hashing, and Cookie Security)
- Understand basic authentication concepts in ASP.NET Core
- Have basic knowledge of Entity Framework Core
- Have the .NET SDK installed with EF Core tools

## 📚 Learning Objectives

By the end of this exercise, you will:

- Implement **ASP.NET Core Identity** with Entity Framework Core
- Configure **SQLite** as the database provider
- Set up **multiple authentication schemes** working in parallel using a Policy Scheme
- Use **Identity's default UI** with scaffolded Razor Pages
- Understand how to perform **database migrations** with EF Core
- Create a system where users can choose between authentication methods
- Debug authentication issues with custom middleware

## 🔍 Why This Matters

In real-world applications, understanding multiple authentication approaches is crucial because:

- It provides flexibility in authentication strategies
- It enables gradual migration from custom authentication to Identity
- It demonstrates how ASP.NET Core handles multiple authentication schemes
- It showcases the integration of Entity Framework Core with Identity
- It teaches debugging techniques for authentication issues

## 📝 Step-by-Step Instructions

### Step 1: Install Required NuGet Packages

1. Open a terminal in your project directory.
2. Install the following NuGet packages:

   ```bash
   dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
   dotnet add package Microsoft.EntityFrameworkCore.Sqlite
   dotnet add package Microsoft.EntityFrameworkCore.Design
   dotnet add package Microsoft.AspNetCore.Identity.UI
   ```

3. Verify the packages were added to your project file:

    > `MerchStore.csproj`

    ```xml
    <ItemGroup>
      <PackageReference Include="BCrypt.Net-Next" Version="4.0.3" />
      <PackageReference Include="Microsoft.AspNetCore.Identity.EntityFrameworkCore" Version="9.0.0" />
      <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="9.0.0" />
      <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="9.0.0">
        <PrivateAssets>all</PrivateAssets>
        <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      </PackageReference>
      <PackageReference Include="Microsoft.AspNetCore.Identity.UI" Version="9.0.0" />
    </ItemGroup>
    ```

> 💡 **Information**
>
> - **Identity.EntityFrameworkCore**: Provides Identity stores that use Entity Framework Core
> - **EntityFrameworkCore.Sqlite**: SQLite database provider for EF Core
> - **EntityFrameworkCore.Design**: Design-time components for EF Core (needed for migrations)
> - **Identity.UI**: Provides default UI pages for Identity (login, register, etc.)
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to install the Design package will prevent migrations from working
> - Version mismatches between packages can cause compatibility issues

### Step 2: Create the Database Context

1. Create a new folder called `Data` in your project root.
2. Create a new file named `ApplicationDbContext.cs` in the `Data` folder:

    > `Data/ApplicationDbContext.cs`

    ```csharp
    using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
    using Microsoft.EntityFrameworkCore;

    namespace MerchStore.Data;

    public class ApplicationDbContext : IdentityDbContext
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
            : base(options)
        {
        }
    }
    ```

> 💡 **Information**
>
> - **IdentityDbContext**: Base class that includes all the DbSet properties needed for Identity
> - **DbContextOptions**: Configuration options for the database context
> - This minimal implementation is sufficient for basic Identity functionality

### Step 3: Configure Connection String

1. Open `appsettings.json`.
2. Add a connection string for SQLite:

    > `appsettings.json`

    ```json
    {
      "ConnectionStrings": {
        "DefaultConnection": "Data Source=merchstore.db"
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
> - **Data Source**: Specifies the SQLite database file name
> - The database file will be created in the project root directory
> - SQLite is a file-based database, perfect for development and small applications

### Step 4: Update Program.cs for Dual Authentication

1. Open `Program.cs`.
2. Update the configuration to support both authentication systems:

    > `Program.cs`

    ```csharp
    using MerchStore.Data;
    using MerchStore.Models;
    using Microsoft.AspNetCore.Authentication.Cookies;
    using Microsoft.AspNetCore.Identity;
    using Microsoft.EntityFrameworkCore;

    var builder = WebApplication.CreateBuilder(args);

    // Add services to the container.
    builder.Services.AddControllersWithViews();

    // Add Razor Pages (required for Identity UI)
    builder.Services.AddRazorPages();

    // Configure Entity Framework Core with SQLite
    builder.Services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));

    // Add ASP.NET Core Identity
    builder.Services.AddDefaultIdentity<IdentityUser>(options => 
        options.SignIn.RequireConfirmedAccount = false)
        .AddEntityFrameworkStores<ApplicationDbContext>();

    // Configure authentication with multiple schemes
    builder.Services.AddAuthentication(options =>
    {
        // Set the default scheme to check both authentication types
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
                
                // Otherwise, fall back to the Identity scheme.
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

    // Configure the HTTP request pipeline.
    if (!app.Environment.IsDevelopment())
    {
        app.UseExceptionHandler("/Home/Error");
        // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
        app.UseHsts();
    }

    app.UseHttpsRedirection();
    app.UseRouting();

    app.UseAuthentication(); // Before UseAuthorization()
    app.UseAuthorization();

    app.MapStaticAssets();

    app.MapControllerRoute(
        name: "default",
        pattern: "{controller=Home}/{action=Index}/{id?}")
        .WithStaticAssets();

    // Add Razor Pages for Identity UI
    app.MapRazorPages();

    app.Run();
    ```

> 💡 **Information**
>
> - **AddRazorPages()**: Required for Identity's default UI to work
> - **AddDefaultIdentity**: Configures Identity with a default user type
> - **Policy Scheme**: Acts as a router between different authentication schemes
> - **ForwardDefaultSelector**: Checks which authentication cookie exists and routes accordingly
> - **MapRazorPages()**: Enables routing for Identity's Razor Pages
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to add `AddRazorPages()` and `MapRazorPages()` will cause Identity UI to not work
> - Not configuring the policy scheme correctly will cause authentication to fail
> - The order of middleware is crucial - authentication must come before authorization

### Step 5: Update Controllers for Authentication Compatibility

1. Open `Controllers/AccountController.cs`.
2. Update the controller to ensure it works with the default cookie scheme:

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
            // Password: "admin" (hashed with BCrypt)
            ["bob.admin"] = ("$2a$11$Je1CiT.kfqqbD9gJgHZ43O0pYF67N.VfAen6eM.Vppf8y/wmrreiG", UserRoles.Administrator),
            
            // Password: "pass" (hashed with BCrypt)
            ["john.doe"] = ("$2a$11$M4afRoHaNiKucxLAhWXwHeEUvVEeg2VBbpN1gRtvZpgfAiXF7GcIq", UserRoles.Customer)
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

                // Sign in with the default cookie scheme
                await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, principal);

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

### Step 6: Update HomeController for Multiple Authentication Schemes

1. Open `Controllers/HomeController.cs`.
2. Modify the controller to accept both authentication types:

    > `Controllers/HomeController.cs`

    ```csharp
    using System.Diagnostics;
    using Microsoft.AspNetCore.Mvc;
    using MerchStore.Models;
    using Microsoft.AspNetCore.Authorization;
    using Microsoft.AspNetCore.Identity;
    using Microsoft.AspNetCore.Authentication.Cookies;

    namespace MerchStore.Controllers;

    public class HomeController : Controller
    {
        private readonly ILogger<HomeController> _logger;

        public HomeController(ILogger<HomeController> logger)
        {
            _logger = logger;
        }

        public IActionResult Index()
        {
            return View();
        }

        public IActionResult Privacy()
        {
            return View();
        }

        [ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, NoStore = true)]
        public IActionResult Error()
        {
            return View(new ErrorViewModel { RequestId = Activity.Current?.Id ?? HttpContext.TraceIdentifier });
        }

        // This action accepts authentication from either scheme
        [Authorize]
        public IActionResult WhoAmI()
        {
            return View();
        }
    }
    ```

> 💡 **Information**
>
> - The `[Authorize]` attribute without scheme specification works with our policy scheme
> - The policy scheme automatically determines which authentication to use
> - This allows both authentication methods to access protected resources

### Step 7: Update Navigation for Dual Authentication

1. Open `Views/Shared/_LoginPartial.cshtml`.
2. Update it to handle both authentication types:

    > `Views/Shared/_LoginPartial.cshtml`

    ```html
    @using Microsoft.AspNetCore.Authentication.Cookies
    @using Microsoft.AspNetCore.Identity
    @inject SignInManager<IdentityUser> SignInManager
    @inject UserManager<IdentityUser> UserManager

    <ul class="navbar-nav ms-auto">
    @if (User.Identity?.IsAuthenticated == true)
    {
        <li class="nav-item d-flex align-items-center">
            <span class="navbar-text text-dark me-3">
                Hello @User.Identity.Name! 
                @(User.Identity.AuthenticationType == CookieAuthenticationDefaults.AuthenticationScheme ? "(Default)" : "(Identity)")
            </span>
        </li>
        <li class="nav-item">
            @if (User.Identity.AuthenticationType == CookieAuthenticationDefaults.AuthenticationScheme)
            {
                <form class="form-inline" asp-controller="Account" asp-action="Logout" method="post">
                    <button type="submit" class="btn btn-primary">Logout</button>
                </form>
            }
            else
            {
                <form class="form-inline" asp-area="Identity" asp-page="/Account/Logout" 
                      asp-route-returnUrl="@Url.Action("Index", "Home", new { area = "" })" method="post">
                    <button type="submit" class="btn btn-primary">Logout</button>
                </form>
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
                <li><a class="dropdown-item" asp-controller="Account" asp-action="Login">Default Login</a></li>
                <li><a class="dropdown-item" asp-area="Identity" asp-page="/Account/Login">Identity Login</a></li>
            </ul>
        </li>
    }
    </ul>
    ```

### Step 8: Create Database Migration

1. Open a terminal in your project directory.
2. Install the Entity Framework Core tools if you haven't already:

   ```bash
   dotnet tool install --global dotnet-ef
   ```

3. Create the initial migration:

   ```bash
   dotnet ef migrations add InitialIdentitySchema
   ```

4. Apply the migration to create the database:

   ```bash
   dotnet ef database update
   ```

> 💡 **Information**
>
> - **dotnet-ef**: Command-line tools for Entity Framework Core
> - **migrations add**: Creates a new migration with the specified name
> - **database update**: Applies pending migrations to the database
> - The SQLite database file will be created automatically
>
> ⚠️ **Common Mistakes**
>
> - Running migrations before installing the EF Core tools
> - Forgetting to add the Design package will cause migration commands to fail
> - Not having the correct connection string will prevent database creation

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

3. Test the dual authentication system:

   **Default Cookie Authentication:**
   - Click "Login" dropdown and choose "Default Login"
   - Login with existing credentials (bob.admin/admin or john.doe/pass)
   - Verify you see "(Default)" in the navigation
   - Access /Home/WhoAmI
   - Logout

   **Identity Authentication:**
   - Click "Login" dropdown and choose "Identity Login"
   - Register a new account using Identity
   - Login with the new account
   - Verify you see "(Identity)" in the navigation
   - Access /Home/WhoAmI
   - Logout

4. Test authorization:
   - Try accessing admin pages with both authentication types
   - Verify that only admin users can access admin pages
   - Test that authorization policies work with both systems

✅ **Expected Results**

- Both authentication systems work independently
- Users can choose between login methods
- Protected pages are accessible with either authentication
- The database is created with Identity tables
- User registration works through Identity UI
- Navigation shows which authentication system is in use
- Admin pages are only accessible to users with admin role

## 🔧 Troubleshooting

If you encounter issues:

### Authentication Issues

- Check that the policy scheme is configured correctly
- Verify cookies are being set in browser dev tools
- Ensure the cookie names match what the policy scheme expects
- Make sure authentication middleware comes before authorization

### Database Issues

- Ensure SQLite database file has write permissions
- Delete the .db file and re-run migrations if corrupted
- Check connection string in appsettings.json

### Identity UI Not Working

- Verify AddRazorPages() and MapRazorPages() are configured
- Check that Identity.UI package is installed
- Ensure the correct authentication scheme is specified

## 🚀 Optional Challenge: Add Authentication Logger

Want to debug authentication issues more effectively? Add an authentication logger middleware:

### Step 1: Create the Middleware

1. Create a new folder called `Middleware` in your project root.
2. Create a new file named `AuthenticationLogger.cs`:

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
            var user = context.User;
            _logger.LogInformation($"Path: {context.Request.Path}");
            _logger.LogInformation($"Is Authenticated: {user.Identity?.IsAuthenticated}");
            _logger.LogInformation($"Authentication Type: {user.Identity?.AuthenticationType}");
            _logger.LogInformation($"Name: {user.Identity?.Name}");
            
            if (user.Claims.Any())
            {
                _logger.LogInformation("Claims:");
                foreach (var claim in user.Claims)
                {
                    _logger.LogInformation($"  {claim.Type}: {claim.Value}");
                }
            }
            
            // Log cookies
            if (context.Request.Cookies.Any())
            {
                _logger.LogInformation("Cookies:");
                foreach (var cookie in context.Request.Cookies)
                {
                    _logger.LogInformation($"  {cookie.Key}: {cookie.Value.Substring(0, Math.Min(20, cookie.Value.Length))}...");
                }
            }

            await _next(context);
        }
    }
    ```

### Step 2: Add the Middleware to the Pipeline

1. Open `Program.cs`.
2. Add the middleware after authentication:

    ```csharp
    app.UseAuthentication();
    app.UseMiddleware<AuthenticationLogger>(); // Add this line
    app.UseAuthorization();
    ```

### Step 3: Test with Logging

Run the application and watch the console output as you:

- Navigate between pages
- Login with different authentication methods
- Access protected resources
- Logout

The logger will help you understand the authentication flow and debug any issues.

## 📚 Further Reading

- [ASP.NET Core Authentication Documentation](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/)
- [Multiple Authentication Schemes](https://docs.microsoft.com/en-us/aspnet/core/security/authorization/limitingidentitybyscheme)
- [Entity Framework Core with SQLite](https://docs.microsoft.com/en-us/ef/core/providers/sqlite/)
- [Identity UI Scaffolding](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/scaffold-identity)
- [Policy Schemes in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/policyschemes)

## Done! 🎉

Great job! You've successfully implemented a **dual authentication system** with both custom cookie authentication and ASP.NET Core Identity. You now understand how to work with **multiple authentication schemes**, **Entity Framework Core**, and **Identity's default UI**. The policy scheme approach provides a clean way to route between different authentication methods! 🚀

## Appendix

### Appendix A: Understanding Policy Schemes

Policy schemes in ASP.NET Core provide a way to dynamically select authentication schemes at runtime:

1. **ForwardDefaultSelector**: Determines which scheme to use for authentication
2. **Cookie Checking**: Looks for specific cookies to identify the authentication method
3. **Scheme Routing**: Routes requests to the appropriate authentication handler

The policy scheme acts as a router, checking for authentication cookies and forwarding to the appropriate handler.

### Appendix B: Authentication Flow

The authentication flow with policy schemes:

1. Request arrives at the application
2. Policy scheme checks for authentication cookies
3. If default cookie is found, routes to cookie authentication
4. If not found, routes to Identity authentication
5. Authentication handler validates the user
6. User principal is created with claims
7. Authorization policies are evaluated

### Appendix C: Entity Framework Core Migrations

Key migration commands:

| Command | Purpose |
|---------|---------|
| `dotnet ef migrations add [name]` | Create a new migration |
| `dotnet ef database update` | Apply pending migrations |
| `dotnet ef migrations remove` | Remove the last migration |
| `dotnet ef database drop` | Drop the database |
| `dotnet ef migrations list` | List all migrations |

### Appendix D: Identity Default UI

ASP.NET Core Identity provides these default pages:

- `/Identity/Account/Login` - User login
- `/Identity/Account/Register` - New user registration
- `/Identity/Account/ForgotPassword` - Password recovery
- `/Identity/Account/Manage` - User profile management
- `/Identity/Account/Logout` - User logout
- `/Identity/Account/ConfirmEmail` - Email confirmation

These can be customized by scaffolding individual pages.

### Appendix E: Cookie Authentication Configuration

Important cookie settings:

| Setting | Purpose | Default |
|---------|---------|---------|
| `HttpOnly` | Prevents JavaScript access | `true` |
| `SecurePolicy` | HTTPS enforcement | `None` |
| `SameSite` | CSRF protection | `Lax` |
| `ExpireTimeSpan` | Cookie lifetime | 14 days |
| `SlidingExpiration` | Extends expiration on activity | `true` |

The default cookie name is `.AspNetCore.Cookies` when using `CookieAuthenticationDefaults.AuthenticationScheme`.
