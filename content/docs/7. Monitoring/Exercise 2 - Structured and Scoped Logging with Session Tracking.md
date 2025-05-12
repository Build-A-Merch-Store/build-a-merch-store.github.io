---
title: "Structured and Scoped Logging with Session Tracking"
date: "2025-05-12"
lastmod: "2025-05-12"
draft: false
weight: 702
toc: true
---

## 🎯 Goal

Enhance your MerchStore application's logging capabilities by implementing structured logging patterns, scoped logging contexts, and session-based tracking to create searchable and analyzable logs in Azure Application Insights.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 1 (Getting Started with Azure Application Insights)
- Have a functioning MerchStore application with Application Insights configured
- Understand basic logging concepts in ASP.NET Core
- Be familiar with middleware concepts in ASP.NET Core
- Have basic knowledge of Kusto Query Language (KQL)

## 📚 Learning Objectives

By the end of this exercise, you will:

- Implement **structured logging** patterns for better searchability
- Create **logging scopes** to add context to related log entries
- Track user sessions using **middleware** for better user journey analysis
- Query structured logs using **Kusto Query Language (KQL)**
- Understand the difference between structured and unstructured logging

## 🔍 Why This Matters

In real-world applications, proper logging patterns are essential because:

- Structured logging makes logs searchable and analyzable at scale
- Logging scopes provide context that helps troubleshoot complex operations
- Session tracking enables understanding of user behavior patterns
- Well-structured logs reduce mean time to resolution (MTTR) for incidents
- KQL queries transform raw logs into actionable insights

## 📝 Step-by-Step Instructions

### Step 1: Implement Structured Logging

Let's start by updating the HomeController to use structured logging patterns.

1. Open `Controllers/HomeController.cs` and update it to demonstrate both unstructured and structured logging:

   > `Controllers/HomeController.cs`

   ```csharp
   using System.Diagnostics;
   using Microsoft.AspNetCore.Mvc;
   using MerchStore.Models;
   using Microsoft.AspNetCore.Authorization;

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
           // Add a log message to indicate the action was called
           _logger.LogInformation("Index action called");
           
           var user = User.Identity?.Name ?? "Anonymous";
           
           // Use unstructured logging to log the user name
           _logger.LogInformation($"Index action called by user: {user}");
           
           // Use structured logging to log the user name
           _logger.LogInformation("Index action called by user: {User}", user);
           
           return View();
       }

       public IActionResult Privacy()
       {
           // Add structured logging for the Privacy page
           _logger.LogInformation("Privacy page accessed by user: {User}", 
               User.Identity?.Name ?? "Anonymous");
           
           return View();
       }

       [ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, NoStore = true)]
       public IActionResult Error()
       {
           var requestId = Activity.Current?.Id ?? HttpContext.TraceIdentifier;
           
           // Log the error page access with structured data
           _logger.LogError("Error page displayed for request: {RequestId}", requestId);
           
           return View(new ErrorViewModel { RequestId = requestId });
       }

       // This action accepts authentication from either scheme
       [Authorize]
       public IActionResult WhoAmI()
       {
           // Log authorized access with user information
           _logger.LogInformation("WhoAmI accessed by authenticated user: {User}", 
               User.Identity?.Name);
           
           return View();
       }
   }
   ```

> 💡 **Information**
>
> - **Unstructured logging**: Uses string interpolation (`$""`) which creates unique log messages
> - **Structured logging**: Uses placeholders (`{}`) which create searchable properties
> - **Property names**: The placeholder name becomes a searchable property in Application Insights
> - **Consistency**: Always use the same property name for the same type of data (e.g., "User")
>
> ⚠️ **Common Mistakes**
>
> - Mixing string interpolation with log message templates
> - Inconsistent property naming across the application
> - Logging sensitive information without considering security implications

### Step 2: Test Structured Logging and Run KQL Queries

1. Run your application and visit the Home page multiple times to generate log data:

   ```bash
   dotnet run
   ```

2. Navigate to your Application Insights resource in the Azure Portal.

3. Go to the "Logs" section and run this KQL query to analyze structured logs:

   ```kusto
   traces
   | where timestamp > ago(1h)
   | where customDimensions.CategoryName contains "MerchStore"
   | where isnotempty(customDimensions.User) // Make sure the User dimension exists
   | summarize LogCount = count() by User = tostring(customDimensions.User) // Count logs per user
   | order by LogCount desc // Optional: Show most active users first
   | render barchart // Or piechart, columnchart, etc.
   ```

4. Compare structured vs unstructured logs with this query:

   ```kusto
   traces
   | where timestamp > ago(1h)
   | where customDimensions.CategoryName contains "HomeController"
   | project timestamp, message, customDimensions
   | order by timestamp desc
   | take 20
   ```

✅ **Expected Results**

- The KQL query should show a chart with log counts per user
- Structured logs will have the "User" property in customDimensions
- Unstructured logs will only have the information in the message field

### Step 3: Implement Scoped Logging in AccountController

Now let's implement logging scopes to add context to login operations.

1. Update `Controllers/AccountController.cs` to use logging scopes:

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
       // Mocked user "database" for demonstration purposes.
       private static readonly Dictionary<string, (string PasswordHash, string Role)> Users = new()
       {
           // Password: "admin" (hashed with BCrypt)
           ["bob.admin"] = ("$2a$11$Je1CiT.kfqqbD9gJgHZ43O0pYF67N.VfAen6eM.Vppf8y/wmrreiG", UserRoles.Administrator),

           // Password: "pass" (hashed with BCrypt)
           ["john.doe"] = ("$2a$11$M4afRoHaNiKucxLAhWXwHeEUvVEeg2VBbpN1gRtvZpgfAiXF7GcIq", UserRoles.Customer)
       };

       private readonly ILogger<AccountController> _logger;

       public AccountController(ILogger<AccountController> logger)
       {
           _logger = logger;
       }

       // This is a simple login page that allows users to enter their credentials.
       [HttpGet]
       public IActionResult Login(string? returnUrl = null)
       {
           ViewData["ReturnUrl"] = returnUrl;
           return View();
       }

       // This action handles the login form submission.
       [HttpPost]
       [ValidateAntiForgeryToken]
       public async Task<IActionResult> LoginAsync(LoginViewModel model, string? returnUrl = null)
       {
           ViewData["ReturnUrl"] = returnUrl;

           // Create a logging scope for the login attempt
           using (_logger.BeginScope(new Dictionary<string, object>
           {
               ["LoginAttempt"] = true,
               ["Username"] = model.Username ?? "Anonymous",
               ["ReturnUrl"] = returnUrl ?? "none"
           }))
           {
               // Log the login attempt
               _logger.LogInformation("Login attempt started");

               if (!ModelState.IsValid)
               {
                   _logger.LogWarning("Login attempt failed - invalid model state");
                   return View(model);
               }

               // Verify the user's credentials against the mocked database.
               if (Users.TryGetValue(model.Username ?? "", out var userData) && 
                   BCrypt.Net.BCrypt.Verify(model.Password, userData.PasswordHash))
               {
                   // Set up the session/cookie for the authenticated user.
                   var claims = new List<Claim>
                   {
                       new Claim(ClaimTypes.Name, model.Username!),
                       new Claim(ClaimTypes.Role, userData.Role)
                   };

                   var identity = new ClaimsIdentity(claims, CookieAuthenticationDefaults.AuthenticationScheme);
                   var principal = new ClaimsPrincipal(identity);

                   // Sign in the user with the cookie authentication scheme.
                   await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, principal);

                   // Log the successful login
                   _logger.LogInformation("Login successful for user {Username} with role {Role}", 
                       model.Username, userData.Role);

                   // Redirect to return URL if valid, otherwise to home
                   if (!string.IsNullOrEmpty(returnUrl) && Url.IsLocalUrl(returnUrl))
                   {
                       return Redirect(returnUrl);
                   }

                   return RedirectToAction("Index", "Home");
               }

               _logger.LogWarning("Login failed - invalid credentials");
               ModelState.AddModelError(string.Empty, "Invalid login attempt.");
               return View(model);
           }
       }

       [HttpPost]
       [ValidateAntiForgeryToken]
       public async Task<IActionResult> Logout()
       {
           var username = User.Identity?.Name ?? "Unknown";
           
           // Sign out the user by removing the authentication cookie.
           await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);

           _logger.LogInformation("User {Username} logged out", username);

           // Redirect to a public area of your application
           return RedirectToAction("Index", "Home");
       }

       [HttpGet]
       public IActionResult AccessDenied()
       {
           _logger.LogWarning("Access denied for user {User} on path {Path}", 
               User.Identity?.Name ?? "Anonymous",
               HttpContext.Request.Path);
               
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

> 💡 **Information**
>
> - **BeginScope**: Creates a logging scope that adds properties to all logs within the using block
> - **Scope Properties**: Available in customDimensions for all logs within the scope
> - **Nested Operations**: All logs within the scope automatically include the scope properties
> - **Disposal**: The scope is automatically cleaned up when exiting the using block
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to dispose the scope (always use `using` statement)
> - Creating overly broad scopes that span too many operations
> - Including sensitive data in scope properties

### Step 4: Test Scoped Logging and Run KQL Queries

1. Test the login functionality with both successful and failed attempts.

2. Run this KQL query to see the scoped login logs:

   ```kusto
   traces
   | where timestamp > ago(30m) // Adjust time window if needed
   // Filter for the specific logs from your scope
   | where message contains "Login"
   | where customDimensions.LoginAttempt == true
   | order by timestamp desc
   | project timestamp, message, customDimensions
   ```

3. Analyze login patterns with this query:

   ```kusto
   traces
   | where timestamp > ago(1h)
   | where customDimensions.LoginAttempt == true
   | summarize 
       TotalAttempts = count(),
       SuccessfulLogins = countif(message contains "successful"),
       FailedLogins = countif(message contains "failed")
       by Username = tostring(customDimensions.Username)
   | project Username, TotalAttempts, SuccessfulLogins, FailedLogins, 
       SuccessRate = round(100.0 * SuccessfulLogins / TotalAttempts, 2)
   | order by TotalAttempts desc
   ```

✅ **Expected Results**

- All logs within the login scope should have LoginAttempt, Username, and ReturnUrl properties
- The queries should show login attempts grouped by user with success rates

### Step 5: Implement Session-Based Logging Middleware

Now let's add session tracking to understand user behavior across requests.

1. Create a new folder called `Middleware` in your project if it doesn't exist.

2. Create `Middleware/SessionLoggingMiddleware.cs`:

   > `Middleware/SessionLoggingMiddleware.cs`

   ```csharp
   namespace MerchStore.Middleware;

   public class SessionLoggingMiddleware
   {
       private readonly RequestDelegate _next;
       private readonly ILogger<SessionLoggingMiddleware> _logger;

       public SessionLoggingMiddleware(RequestDelegate next, ILogger<SessionLoggingMiddleware> logger)
       {
           _next = next;
           _logger = logger;
       }

       public async Task InvokeAsync(HttpContext context)
       {
           // Ensure session is available
           if (context.Session != null)
           {
               // Get or create session ID
               var sessionId = context.Session.Id;

               // Get or set session start time
               const string sessionStartKey = "SessionStartTime";
               if (!context.Session.Keys.Contains(sessionStartKey))
               {
                   context.Session.SetString(sessionStartKey, DateTime.UtcNow.ToString("o"));
                   _logger.LogInformation("New session started. SessionId: {SessionId}", sessionId);
               }

               var sessionStartTime = context.Session.GetString(sessionStartKey);
               var sessionDuration = DateTime.UtcNow - DateTime.Parse(sessionStartTime!);

               // Create session scope
               using (_logger.BeginScope(new Dictionary<string, object>
               {
                   ["SessionId"] = sessionId,
                   ["SessionDuration"] = sessionDuration.TotalMinutes,
                   ["PageViewsInSession"] = context.Session.GetInt32("PageViews") ?? 0,
                   ["User"] = context.User.Identity?.Name ?? "Anonymous"
               }))
               {
                   // Increment page view counter
                   var pageViews = context.Session.GetInt32("PageViews") ?? 0;
                   context.Session.SetInt32("PageViews", pageViews + 1);

                   // Log the page view
                   _logger.LogInformation("Page view: {Path} - View #{PageView} in session", 
                       context.Request.Path, pageViews + 1);

                   await _next(context);
               }
           }
           else
           {
               await _next(context);
           }
       }
   }

   public static class SessionLoggingMiddlewareExtensions
   {
       public static IApplicationBuilder UseSessionLogging(this IApplicationBuilder builder)
       {
           return builder.UseMiddleware<SessionLoggingMiddleware>();
       }
   }
   ```

3. Configure session services and middleware in `Program.cs`:

   > `Program.cs`

   ```csharp
   // Add this to your using statements
   using MerchStore.Middleware;

   // Add session services (add this with other services)
   builder.Services.AddDistributedMemoryCache();
   builder.Services.AddSession(options =>
   {
       options.IdleTimeout = TimeSpan.FromMinutes(30);
       options.Cookie.HttpOnly = true;
       options.Cookie.IsEssential = true;
   });

   // ... other services ...

   var app = builder.Build();

   // ... other middleware ...

   app.UseSession(); // Must come before our custom middleware
   app.UseSessionLogging(); // Add session logging

   // ... rest of middleware pipeline ...
   ```

> 💡 **Information**
>
> - **Session Middleware**: Provides session state functionality
> - **Session ID**: Unique identifier for each user session
> - **Session Scope**: Adds session context to all logs within a request
> - **Page Views**: Simple metric to track user engagement
>
> ⚠️ **Common Mistakes**
>
> - Adding session middleware in wrong order (must be before custom middleware)
> - Not configuring session services before using session middleware
> - Storing too much data in session (impacts performance)

### Step 6: Test Session Logging and Run KQL Queries

1. Run your application and navigate through multiple pages to generate session data:
   - Visit the home page
   - Log in as a user
   - Visit different pages multiple times
   - Log out and log in as a different user

2. Run this KQL query to analyze session statistics:

   ```kusto
   // Find average session stats per user
   traces
   | where timestamp > ago(7d)
   | where isnotempty(customDimensions.SessionId) and isnotempty(customDimensions.User) and isnotempty(customDimensions.SessionDuration) and isnotempty(customDimensions.PageViewsInSession)
   | summarize MaxPageViews = max(toint(customDimensions.PageViewsInSession)) by SessionId = tostring(customDimensions.SessionId), User = tostring(customDimensions.User)
   | summarize AvgPageViews=avg(MaxPageViews), SessionCount=count() by User
   | order by SessionCount desc
   ```

✅ **Expected Results**

- Session statistics showing average page views and session counts per user
- Duration patterns showing how long users spend in the application
- Page visit patterns showing most popular pages per user

## 🧪 Final Tests

### Comprehensive Testing

1. Run your application and perform these actions:
   - Visit pages as an anonymous user
   - Log in and visit multiple pages
   - Log out and log in as a different user
   - Trigger some failed login attempts

2. Wait 2-5 minutes for data to appear in Application Insights.

3. Run a comprehensive analysis query:

   ```kusto
   traces
   | where timestamp > ago(1h)
   | where customDimensions.CategoryName contains "MerchStore"
   | summarize 
       TotalLogs = count(),
       UniqueUsers = dcount(tostring(customDimensions.User)),
       UniqueSessions = dcount(tostring(customDimensions.SessionId)),
       LoginAttempts = countif(customDimensions.LoginAttempt == true),
       PageViews = countif(message contains "Page view")
   | project 
       TotalLogs,
       UniqueUsers,
       UniqueSessions,
       LoginAttempts,
       PageViews,
       AvgLogsPerUser = round(1.0 * TotalLogs / UniqueUsers, 2),
       AvgPageViewsPerSession = round(1.0 * PageViews / UniqueSessions, 2)
   ```

✅ **Expected Results**

- Structured logs with searchable properties in customDimensions
- Scoped logs showing context for related operations
- Session tracking providing insights into user behavior
- KQL queries successfully analyzing the structured data

## 🔧 Troubleshooting

If you encounter issues:

- **No customDimensions in logs**:
  - Verify you're using structured logging syntax with placeholders
  - Check that logging scopes are properly configured
  - Wait 2-5 minutes for data to appear in Application Insights

- **Session data not appearing**:
  - Verify session services are configured before middleware
  - Check that session middleware is in the correct order
  - Ensure cookies are enabled in your browser

- **KQL queries returning no data**:
  - Check the time range in your queries
  - Verify property names match exactly (case-sensitive)
  - Use simpler queries first to verify data exists

## 🚀 Optional Challenge

Want to take your learning further? Try:

- Create a dashboard in Azure that visualizes user journeys through your application
- Implement custom KQL functions for frequently-used queries
- Add geographic location tracking to sessions
- Create alerts based on unusual session patterns

## 📚 Further Reading

- [Structured Logging in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging)
- [Log Analytics Query Language](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/get-started-queries)
- [Session State in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/app-state)
- [Logging Best Practices](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/loggermessage)

## Done! 🎉

Great job! You've successfully implemented three essential logging patterns in your MerchStore application:

1. **Structured logging** for searchable log properties
2. **Scoped logging** for operational context
3. **Session logging** for user behavior tracking

These patterns provide the foundation for effective application monitoring and troubleshooting. Your logs are now structured, contextual, and analyzable using powerful KQL queries in Application Insights! 🚀

## Appendix: Quick Reference for Logging Patterns

### Structured Logging Syntax

```csharp
// Good - Creates searchable properties
_logger.LogInformation("User {Username} logged in at {LoginTime}", username, DateTime.UtcNow);

// Bad - Creates unique messages that are hard to search
_logger.LogInformation($"User {username} logged in at {DateTime.UtcNow}");
```

### Scoped Logging Pattern

```csharp
using (_logger.BeginScope(new Dictionary<string, object>
{
    ["OperationId"] = operationId,
    ["UserId"] = userId
}))
{
    // All logs here will include OperationId and UserId
    _logger.LogInformation("Starting operation");
    // ... operation code ...
    _logger.LogInformation("Operation completed");
}
```

### KQL Query Structure

```kusto
traces
| where timestamp > ago(1h)                    // Time filter
| where customDimensions.PropertyName == value // Property filter
| summarize count() by PropertyName            // Aggregation
| order by count_ desc                         // Sorting
| render barchart                              // Visualization
```
