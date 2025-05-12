---
title: "Custom Telemetry and Business Metrics"
date: "2025-05-12"
lastmod: "2025-05-12"
draft: false
weight: 703
toc: true
---

## 🎯 Goal

Enhance your MerchStore application's monitoring by implementing custom telemetry tracking for business events and metrics, enabling you to monitor user behavior and key performance indicators beyond standard technical telemetry.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 2 (Structured and Scoped Logging with Session Tracking)
- Have a functioning MerchStore application with Application Insights and logging configured
- Understand basic telemetry concepts in ASP.NET Core
- Be familiar with structured logging patterns
- Have basic knowledge of Kusto Query Language (KQL)

## 📚 Learning Objectives

By the end of this exercise, you will:

- Implement **custom events** tracking for user actions
- Create **custom metrics** for business KPIs
- Use **TelemetryClient** to send custom telemetry
- Query custom events and metrics using **KQL**
- Understand the difference between events and metrics

## 🔍 Why This Matters

In real-world applications, custom telemetry is essential because:

- Technical metrics alone don't tell the complete business story
- Custom events help understand user behavior and feature adoption
- Business metrics enable data-driven decision making
- KPIs can be monitored and alerted on just like technical metrics
- Custom telemetry bridges the gap between technical and business teams

## 📝 Step-by-Step Instructions

### Step 1: Configure TelemetryClient in Controllers

First, let's add TelemetryClient to our controllers to enable custom telemetry tracking.

1. Update `Controllers/HomeController.cs` to inject TelemetryClient:

   > `Controllers/HomeController.cs`

   ```csharp
   using System.Diagnostics;
   using Microsoft.ApplicationInsights;
   using Microsoft.AspNetCore.Mvc;
   using MerchStore.Models;
   using Microsoft.AspNetCore.Authorization;

   namespace MerchStore.Controllers;

   public class HomeController : Controller
   {
       private readonly ILogger<HomeController> _logger;
       private readonly TelemetryClient _telemetryClient;

       public HomeController(ILogger<HomeController> logger, TelemetryClient telemetryClient)
       {
           _logger = logger;
           _telemetryClient = telemetryClient;
       }

       public IActionResult Index()
       {
           var user = User.Identity?.Name ?? "Anonymous";
           
           // Existing structured logging
           _logger.LogInformation("Index action called by user: {User}", user);
           
           // Track a custom event for page views
           _telemetryClient.TrackEvent("PageViewed", new Dictionary<string, string>
           {
               { "PageName", "Home" },
               { "User", user }
           });
           
           return View();
       }

       public IActionResult Privacy()
       {
           var user = User.Identity?.Name ?? "Anonymous";
           
           _logger.LogInformation("Privacy page accessed by user: {User}", user);
           
           // Track privacy page view with additional properties
           _telemetryClient.TrackEvent("PageViewed", new Dictionary<string, string>
           {
               { "PageName", "Privacy" },
               { "User", user },
               { "Timestamp", DateTime.UtcNow.ToString("o") }
           });
           
           return View();
       }

       [ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, NoStore = true)]
       public IActionResult Error()
       {
           var requestId = Activity.Current?.Id ?? HttpContext.TraceIdentifier;
           
           _logger.LogError("Error page displayed for request: {RequestId}", requestId);
           
           // Track error page view as a custom event
           _telemetryClient.TrackEvent("ErrorPageViewed", new Dictionary<string, string>
           {
               { "RequestId", requestId },
               { "User", User.Identity?.Name ?? "Anonymous" }
           });
           
           return View(new ErrorViewModel { RequestId = requestId });
       }

       // This action accepts authentication from either scheme
       [Authorize]
       public IActionResult WhoAmI()
       {
           _logger.LogInformation("WhoAmI accessed by authenticated user: {User}", User.Identity?.Name);
           
           // Track authenticated page access
           _telemetryClient.TrackEvent("AuthenticatedPageAccessed", new Dictionary<string, string>
           {
               { "PageName", "WhoAmI" },
               { "User", User.Identity.Name }
           });
           
           return View();
       }
   }
   ```

> 💡 **Information**
>
> - **TelemetryClient**: The main class for sending custom telemetry to Application Insights
> - **TrackEvent**: Used for discrete events (things that happen at a point in time)
> - **Properties**: Key-value pairs that make events searchable and filterable
> - **Event Names**: Should be consistent and descriptive
>
> ⚠️ **Common Mistakes**
>
> - Not injecting TelemetryClient in the constructor
> - Using inconsistent event names across the application
> - Including too many properties that create noise
> - Not considering the cost of high-volume events

### Step 2: Track Business Events in AccountController

Now let's track authentication-related business events.

1. Update `Controllers/AccountController.cs` to track login and logout events:

   > `Controllers/AccountController.cs`

   ```csharp
   using System.Security.Claims;
   using MerchStore.Models;
   using Microsoft.ApplicationInsights;
   using Microsoft.AspNetCore.Authentication;
   using Microsoft.AspNetCore.Authentication.Cookies;
   using Microsoft.AspNetCore.Authorization;
   using Microsoft.AspNetCore.Mvc;

   namespace MerchStore.Controllers;

   public class AccountController : Controller
   {
       // ... existing code ...

       private readonly ILogger<AccountController> _logger;
       private readonly TelemetryClient _telemetryClient;

       public AccountController(ILogger<AccountController> logger, TelemetryClient telemetryClient)
       {
           _logger = logger;
           _telemetryClient = telemetryClient;
       }

       // ... existing code ...

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
               _logger.LogInformation("Login attempt started");

               // Track login attempt as custom event
               _telemetryClient.TrackEvent("LoginAttempted", new Dictionary<string, string>
               {
                   { "Username", model.Username ?? "Anonymous" },
                   { "ReturnUrl", returnUrl ?? "none" }
               });

               if (!ModelState.IsValid)
               {
                   _logger.LogWarning("Login attempt failed - invalid model state");
                   return View(model);
               }

               // Verify the user's credentials against the mocked database.
               if (Users.TryGetValue(model.Username ?? "", out var userData) && 
                   BCrypt.Net.BCrypt.Verify(model.Password, userData.PasswordHash))
               {
                   // ... existing authentication code ...

                   // Sign in the user with the cookie authentication scheme.
                   await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, principal);

                   // Log the successful login
                   _logger.LogInformation("Login successful for user {Username} with role {Role}", 
                       model.Username, userData.Role);

                   // Track successful login with custom metrics
                   _telemetryClient.TrackEvent("LoginSucceeded", new Dictionary<string, string>
                   {
                       { "Username", model.Username },
                       { "Role", userData.Role }
                   });

                   // Track a metric for successful logins
                   _telemetryClient.TrackMetric("LoginSuccess", 1);

                   // Redirect logic...
                   if (!string.IsNullOrEmpty(returnUrl) && Url.IsLocalUrl(returnUrl))
                   {
                       return Redirect(returnUrl);
                   }

                   return RedirectToAction("Index", "Home");
               }

               _logger.LogWarning("Login failed - invalid credentials");

               // Track failed login
               _telemetryClient.TrackEvent("LoginFailed", new Dictionary<string, string>
               {
                   { "Username", model.Username ?? "Anonymous" },
                   { "Reason", "InvalidCredentials" }
               });

               // Track a metric for failed logins
               _telemetryClient.TrackMetric("LoginFailure", 1);

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

           // Track logout event
           _telemetryClient.TrackEvent("UserLoggedOut", new Dictionary<string, string>
           {
               { "Username", username }
           });

           // Track session duration metric (if session data is available)
           if (HttpContext.Session != null)
           {
               var sessionStartKey = "SessionStartTime";
               var sessionStartTime = HttpContext.Session.GetString(sessionStartKey);
               if (!string.IsNullOrEmpty(sessionStartTime))
               {
                   var sessionDuration = DateTime.UtcNow - DateTime.Parse(sessionStartTime);
                   _telemetryClient.TrackMetric("SessionDuration", sessionDuration.TotalMinutes);
               }
           }

           // Redirect to a public area of your application
           return RedirectToAction("Index", "Home");
       }

       // ... rest of the code ...
   }
   ```

> 💡 **Information**
>
> - **TrackEvent vs TrackMetric**: Events are for things that happen, metrics are for measurements
> - **Authentication Events**: Critical for understanding user engagement and security patterns
> - **Session Metrics**: Help understand user engagement duration
> - **Event Properties**: Enable filtering and grouping in analysis
>
> ⚠️ **Common Mistakes**
>
> - Tracking too many metrics that increase costs
> - Not differentiating between events and metrics properly
> - Forgetting to handle null values in properties
> - Not considering GDPR/privacy when tracking user data

### Step 3: Run the Application and Generate Telemetry

1. Run your application and perform various actions:

   ```bash
   dotnet run
   ```

2. Perform these actions to generate telemetry:
   - Visit the home page
   - Visit the privacy page
   - Try to access the WhoAmI page (should redirect to login)
   - Log in with valid credentials (bob.admin/admin or john.doe/pass)
   - Visit the WhoAmI page again
   - Log out
   - Try a failed login attempt

3. Wait 2-5 minutes for telemetry to appear in Application Insights.

### Step 4: Query Custom Events in Application Insights

1. Navigate to your Application Insights resource in the Azure Portal.

2. Go to the "Logs" section and run these KQL queries:

   **Query 1: Page View Analysis**

   ```kusto
   customEvents
   | where timestamp > ago(1h)
   | where name == "PageViewed"
   | summarize ViewCount = count() by PageName = tostring(customDimensions.PageName)
   | order by ViewCount desc
   | render barchart
   ```

   **Query 2: Authentication Events Timeline**

   ```kusto
   customEvents
   | where timestamp > ago(1h)
   | where name in ("LoginAttempted", "LoginSucceeded", "LoginFailed", "UserLoggedOut")
   | project timestamp, EventName = name, Username = tostring(customDimensions.Username)
   | order by timestamp desc
   ```

   **Query 3: Login Success Rate**

   ```kusto
   customEvents
   | where timestamp > ago(1h)
   | where name in ("LoginSucceeded", "LoginFailed")
   | summarize 
       SuccessCount = countif(name == "LoginSucceeded"),
       FailureCount = countif(name == "LoginFailed")
   | extend TotalAttempts = SuccessCount + FailureCount
   | extend SuccessRate = round(100.0 * SuccessCount / TotalAttempts, 2)
   | project TotalAttempts, SuccessCount, FailureCount, SuccessRate
   ```

✅ **Expected Results**

- Page view events showing which pages are most visited
- Authentication events showing the user journey
- Login success rate metrics

### Step 5: Query Custom Metrics

1. Run these KQL queries to analyze metrics:

   **Query 1: Login Metrics Over Time**

   ```kusto
   customMetrics
   | where timestamp > ago(1h)
   | where name in ("LoginSuccess", "LoginFailure")
   | summarize 
       SuccessCount = sumif(value, name == "LoginSuccess"),
       FailureCount = sumif(value, name == "LoginFailure")
       by bin(timestamp, 5m)
   | render timechart
   ```

## 🧪 Final Tests

### Comprehensive Testing

1. Generate more telemetry by using the application for 5-10 minutes:
   - Multiple login/logout cycles
   - Visit different pages
   - Try some failed logins

2. Create a trend analysis query:

   ```kusto
   customEvents
   | where timestamp > ago(1h)
   | where name in ("PageViewed", "LoginSucceeded", "LoginFailed")
   | summarize EventCount = count() by bin(timestamp, 5m), EventType = name
   | render timechart
   ```

✅ **Expected Results**

- Custom events appearing in Application Insights within minutes
- Metrics showing aggregated values over time
- Dashboard displaying business KPIs
- Trend analysis showing user behavior patterns

## 🔧 Troubleshooting

If you encounter issues:

- **No custom events appearing**:
  - Verify TelemetryClient is properly injected
  - Check that Application Insights connection string is configured
  - Wait 2-5 minutes for data to appear
  - Check developer tools for any JavaScript errors

- **Metrics not showing up**:
  - Ensure TrackMetric is called with numeric values
  - Verify metric names are consistent
  - Check the customMetrics table specifically

- **KQL queries returning no data**:
  - Verify event names match exactly
  - Check time ranges in queries
  - Start with simpler queries to verify data exists

## 🚀 Optional Challenge

Want to take your learning further? Try:

- Create custom metrics for page load times
- Track user preferences and feature usage
- Build a conversion funnel for a multi-step process
- Create alerts based on business metrics thresholds
- Implement A/B testing tracking with custom properties

## 📚 Further Reading

- [Application Insights API for custom events and metrics](https://docs.microsoft.com/en-us/azure/azure-monitor/app/api-custom-events-metrics)
- [TelemetryClient Class](https://docs.microsoft.com/en-us/dotnet/api/microsoft.applicationinsights.telemetryclient)
- [KQL Query Examples](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/samples)
- [Creating dashboards in Azure Portal](https://docs.microsoft.com/en-us/azure/azure-portal/azure-portal-dashboards)

## Done! 🎉

Great job! You've successfully implemented custom telemetry tracking in your MerchStore application. You now have:

1. **Custom Events** tracking user actions and behaviors
2. **Custom Metrics** measuring business KPIs
3. **KQL Queries** to analyze your custom telemetry
4. **Business Insights** from your application data

This custom telemetry provides valuable business insights beyond technical monitoring, helping you understand user behavior and make data-driven decisions! 🚀

## Appendix: Events vs Metrics Quick Reference

### When to Use Events

- User actions (clicked button, viewed page)
- System occurrences (order placed, payment processed)
- Errors or warnings (validation failed, timeout occurred)
- State changes (user logged in, feature enabled)

### When to Use Metrics

- Measurements (response time, queue length)
- Counts (items in cart, active users)
- Rates (requests per second, conversion rate)
- Business values (revenue, cost, margin)

### Event Properties Best Practices

```csharp
// Good - Clear, consistent properties
_telemetryClient.TrackEvent("OrderCompleted", new Dictionary<string, string>
{
    { "OrderId", orderId },
    { "CustomerId", customerId },
    { "PaymentMethod", paymentMethod }
});

// Bad - Inconsistent, too many properties
_telemetryClient.TrackEvent("order_done", new Dictionary<string, string>
{
    { "order", orderId },
    { "cust_id", customerId },
    { "timestamp", DateTime.Now.ToString() }, // Unnecessary - already tracked
    { "user_agent", Request.Headers["User-Agent"] } // Too technical
});
```

### Metric Naming Conventions

```csharp
// Good - Clear, measurable metrics
_telemetryClient.TrackMetric("CartValue", cartTotal);
_telemetryClient.TrackMetric("LoginDuration", duration.TotalSeconds);
_telemetryClient.TrackMetric("SearchResultCount", results.Count);

// Bad - Unclear or non-numeric
_telemetryClient.TrackMetric("UserHappy", 1); // Too vague
_telemetryClient.TrackMetric("Error", 1); // Should be an event
_telemetryClient.TrackMetric("Status", isActive ? 1 : 0); // Should be a property
```
