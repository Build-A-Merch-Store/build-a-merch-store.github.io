---
title: "Adding API Key Authentication to Your Basic API"
date: "2025-04-20"
lastmod: "2025-04-24"
draft: false
weight: 303
toc: true
---

## 🎯 Goal

Secure your Basic Products API by implementing API Key authentication. This simple but effective authentication mechanism will ensure that only clients with a valid API key can access your endpoints.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 7 (Implementing a Basic Product Catalog API)
- Have completed Exercise 8 (Testing Your API with Various Tools)
- Have your Basic Products API running and accessible
- Understand basic HTTP concepts including headers
- Be familiar with ASP.NET Core middleware and authentication concepts

## 📚 Learning Objectives

By the end of this exercise, you will:

- Implement a **custom authentication handler** for API Key validation
- Create a **middleware pipeline** for authentication and authorization
- Apply **attribute-based security** to your API controllers
- Configure **API key storage** using application settings
- Test authenticated endpoints using **different API testing tools**
- Understand the **security implications** of API Key authentication

## 🔍 Why This Matters

In real-world applications, API security is crucial because:

- Unprotected APIs can expose sensitive data and operations
- Authentication confirms the identity of API clients
- API Keys provide a simple yet effective access control mechanism
- Multiple authentication schemes may be necessary for different clients
- Security is a fundamental requirement for production-ready APIs
- Understanding authentication fundamentals prepares you for more advanced security patterns

## 📝 Step-by-Step Instructions

### Step 1: Define the API Key Authentication Scheme

**Introduction**: First, we'll define the constants and options for our API Key authentication scheme. This establishes the foundation for our authentication implementation.

1. Create a folder structure for authentication components:

    ```bash
    mkdir -p src/MerchStore.WebUI/Authentication/ApiKey
    ```

2. Create a file for the authentication scheme constants:

    > `src/MerchStore.WebUI/Authentication/ApiKey/ApiKeyAuthenticationDefaults.cs`

    ```csharp
    namespace MerchStore.WebUI.Authentication.ApiKey;

    /// <summary>
    /// Default values used by API key authentication.
    /// </summary>
    public static class ApiKeyAuthenticationDefaults
    {
        /// <summary>
        /// Default value for AuthenticationScheme property in the ApiKeyAuthenticationOptions
        /// </summary>
        public const string AuthenticationScheme = "ApiKey";
        
        /// <summary>
        /// The default header name where the API key is expected to be transmitted
        /// </summary>
        public const string HeaderName = "X-API-Key";
    }
    ```

3. Create a class for the authentication options:

    > `src/MerchStore.WebUI/Authentication/ApiKey/ApiKeyAuthenticationOptions.cs`

    ```csharp
    using Microsoft.AspNetCore.Authentication;

    namespace MerchStore.WebUI.Authentication.ApiKey;

    /// <summary>
    /// Options for API key authentication.
    /// </summary>
    public class ApiKeyAuthenticationOptions : AuthenticationSchemeOptions
    {
        /// <summary>
        /// The header name where the API key is expected to be transmitted.
        /// Defaults to "X-API-Key".
        /// </summary>
        public string HeaderName { get; set; } = ApiKeyAuthenticationDefaults.HeaderName;
        
        /// <summary>
        /// The API key that clients must provide to be authenticated.
        /// </summary>
        public string ApiKey { get; set; } = string.Empty;
    }
    ```

> 💡 **Information**
>
> - **Authentication Scheme**: A named configuration that defines how authentication works
> - **Authentication Options**: Configuration settings for the authentication process
> - **Header-based Authentication**: API Keys are commonly transmitted in an HTTP header
> - **X-API-Key Convention**: The "X-" prefix denotes a custom (non-standard) HTTP header
>
> ⚠️ **Common Mistakes**
>
> - Using inconsistent scheme names across the application
> - Not documenting the expected header name for API consumers
> - Using spaces or special characters in header names (avoid this)

### Step 2: Implement the API Key Authentication Handler

**Introduction**: Next, we'll implement the authentication handler that validates the API key. This handler will check if the incoming request contains a valid API key in the specified header.

1. Create the API Key authentication handler class:

    > `src/MerchStore.WebUI/Authentication/ApiKey/ApiKeyAuthenticationHandler.cs`

    ```csharp
    using System.Security.Claims;
    using System.Text.Encodings.Web;
    using Microsoft.AspNetCore.Authentication;
    using Microsoft.Extensions.Options;

    namespace MerchStore.WebUI.Authentication.ApiKey;

    /// <summary>
    /// Authentication handler for API key authentication.
    /// </summary>
    public class ApiKeyAuthenticationHandler : AuthenticationHandler<ApiKeyAuthenticationOptions>
    {
        /// <summary>
        /// Initializes a new instance of the <see cref="ApiKeyAuthenticationHandler"/> class.
        /// </summary>
        public ApiKeyAuthenticationHandler(
            IOptionsMonitor<ApiKeyAuthenticationOptions> options,
            ILoggerFactory logger,
            UrlEncoder encoder) : base(options, logger, encoder)
        {
        }

        /// <summary>
        /// Verifies that the request contains a valid API key in the header.
        /// </summary>
        protected override Task<AuthenticateResult> HandleAuthenticateAsync()
        {
            // Check if the request header contains the API key
            if (!Request.Headers.TryGetValue(Options.HeaderName, out var apiKeyHeaderValues))
            {
                Logger.LogWarning("API key missing. Header '{HeaderName}' not found in the request.", Options.HeaderName);
                return Task.FromResult(AuthenticateResult.Fail($"API key header '{Options.HeaderName}' not found."));
            }

            // Check if the header value is empty
            var providedApiKey = apiKeyHeaderValues.FirstOrDefault();
            if (string.IsNullOrWhiteSpace(providedApiKey))
            {
                Logger.LogWarning("API key is empty. Header '{HeaderName}' has no value.", Options.HeaderName);
                return Task.FromResult(AuthenticateResult.Fail("API key is empty."));
            }

            // Validate the API key against the configured value
            if (providedApiKey != Options.ApiKey)
            {
                Logger.LogWarning("Invalid API key provided: {ProvidedKey}", providedApiKey);
                return Task.FromResult(AuthenticateResult.Fail("Invalid API key."));
            }

            // If the API key is valid, create a claims identity and authentication ticket
            var claims = new[] { new Claim(ClaimTypes.Name, "API User") };
            var identity = new ClaimsIdentity(claims, Scheme.Name);
            var principal = new ClaimsPrincipal(identity);
            var ticket = new AuthenticationTicket(principal, Scheme.Name);

            Logger.LogInformation("API key authentication successful");
            return Task.FromResult(AuthenticateResult.Success(ticket));
        }
    }
    ```

> 💡 **Information**
>
> - **Authentication Handler**: Implements the logic that validates credentials and creates identity
> - **HandleAuthenticateAsync**: The main method called during the authentication process
> - **AuthenticateResult**: Represents the outcome of an authentication attempt (Success/Fail)
> - **Claims-based Identity**: ASP.NET Core uses claims to represent information about the authenticated user
> - **Authentication Ticket**: Contains the principal (user identity) and properties for the authentication session
>
> ⚠️ **Common Mistakes**
>
> - Not handling missing or empty headers properly
> - Hardcoding API keys in the handler class rather than using configuration
> - Not providing useful error messages in failure cases
> - Insufficient logging for troubleshooting authentication issues

### Step 3: Create Authentication Extensions

**Introduction**: To make our authentication scheme easier to register, we'll create extension methods for the authentication builder. This follows the common pattern used in ASP.NET Core for registering services.

1. Create the extension methods class:

    > `src/MerchStore.WebUI/Authentication/ApiKey/ApiKeyAuthenticationExtensions.cs`

    ```csharp
    using Microsoft.AspNetCore.Authentication;

    namespace MerchStore.WebUI.Authentication.ApiKey;

    /// <summary>
    /// Extension methods for API key authentication.
    /// </summary>
    public static class ApiKeyAuthenticationExtensions
    {
        /// <summary>
        /// Adds API key authentication to the authentication builder.
        /// </summary>
        /// <param name="builder">The authentication builder.</param>
        /// <param name="configureOptions">A delegate to configure the options.</param>
        /// <returns>The authentication builder for method chaining.</returns>
        public static AuthenticationBuilder AddApiKey(
            this AuthenticationBuilder builder,
            Action<ApiKeyAuthenticationOptions>? configureOptions = null)
        {
            return builder.AddScheme<ApiKeyAuthenticationOptions, ApiKeyAuthenticationHandler>(
                ApiKeyAuthenticationDefaults.AuthenticationScheme,
                configureOptions);
        }

        /// <summary>
        /// Adds API key authentication to the authentication builder with a specific API key.
        /// </summary>
        /// <param name="builder">The authentication builder.</param>
        /// <param name="apiKey">The API key that clients must provide.</param>
        /// <returns>The authentication builder for method chaining.</returns>
        public static AuthenticationBuilder AddApiKey(
            this AuthenticationBuilder builder,
            string apiKey)
        {
            return builder.AddApiKey(options => options.ApiKey = apiKey);
        }
    }
    ```

> 💡 **Information**
>
> - **Extension Methods**: Allow adding methods to existing types without modifying them
> - **Fluent API**: Method chaining makes configuration more readable
> - **Authentication Builder**: Used to configure authentication services in ASP.NET Core
> - **Delegate Options**: Allow flexible configuration of authentication options
>
> ⚠️ **Common Mistakes**
>
> - Not providing both simple and configurable extension methods
> - Using method names that conflict with existing ASP.NET Core methods
> - Not documenting extension methods properly with XML comments

### Step 4: Create Swagger Support for API Key Authentication

**Introduction**: To make our API more developer-friendly, we'll configure Swagger to support API Key authentication. This allows developers to test the API directly from the Swagger UI.

1. Create a Security Requirements Operation Filter class to apply API key requirements to Swagger operations:

    > `src/MerchStore.WebUI/Infrastructure/SecurityRequirementsOperationFilter.cs`

    ```csharp
    using Microsoft.AspNetCore.Authorization;
    using Microsoft.OpenApi.Models;
    using Swashbuckle.AspNetCore.SwaggerGen;
    using System.Reflection;
    using MerchStore.WebUI.Authentication.ApiKey;

    namespace MerchStore.WebUI.Infrastructure;

    /// <summary>
    /// Operation filter to add security requirements for controller-based endpoints
    /// </summary>
    public class SecurityRequirementsOperationFilter : IOperationFilter
    {
        public void Apply(OpenApiOperation operation, OperationFilterContext context)
        {
            // Only add security requirements to controller-based endpoints
            // This excludes minimal API endpoints
            if (context.ApiDescription.ActionDescriptor.GetType().Name.Contains("ControllerActionDescriptor"))
            {
                // Check if the endpoint requires authorization
                var methodInfo = context.MethodInfo;
                var controllerType = methodInfo?.DeclaringType;
                
                if (methodInfo != null)
                {
                    var hasAuthorizeAttribute = methodInfo.GetCustomAttribute<AuthorizeAttribute>() != null
                                             || controllerType?.GetCustomAttribute<AuthorizeAttribute>() != null;
                    
                    if (hasAuthorizeAttribute)
                    {
                        // Add API key security requirement
                        operation.Security = new List<OpenApiSecurityRequirement>
                        {
                            new OpenApiSecurityRequirement
                            {
                                {
                                    new OpenApiSecurityScheme
                                    {
                                        Reference = new OpenApiReference
                                        {
                                            Type = ReferenceType.SecurityScheme,
                                            Id = ApiKeyAuthenticationDefaults.AuthenticationScheme
                                        }
                                    },
                                    Array.Empty<string>()
                                }
                            }
                        };
                    }
                }
            }
        }
    }
    ```

> 💡 **Information**
>
> - **Operation Filter**: Enhances Swagger operations with additional metadata
> - **Authorization Detection**: Checks controllers and actions for the `[Authorize]` attribute
> - **Security Requirements**: Adds API Key security requirements to the Swagger UI
> - **Cross-Cutting Concern**: Applies security requirements without modifying controller code
>
> ⚠️ **Common Mistakes**
>
> - Not checking both controller and action method for authorization attributes
> - Applying security requirements to all endpoints regardless of their authorization needs
> - Using inconsistent security scheme names

### Step 5: Configure API Keys in Application Settings

**Introduction**: For security and flexibility, we'll store the API key in application settings rather than hardcoding it. This allows for different keys in different environments without changing code.

1. Update the `appsettings.json` file to include the API key:

    > `src/MerchStore.WebUI/appsettings.json`

    ```json
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft.AspNetCore": "Warning"
        }
      },
      "AllowedHosts": "*",
      "ApiKey": {
        "Value": "API_KEY"
      }
    }
    ```

2. Create a separate configuration in `appsettings.Development.json` for development:

    > `src/MerchStore.WebUI/appsettings.Development.json`

    ```json
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft.AspNetCore": "Warning"
        }
      },
      "ApiKey": {
        "Value": "API_KEY"
      }
    }
    ```

> 💡 **Information**
>
> - **Environment-specific Settings**: Different files for different environments (Development, Production, etc.)
> - **Configuration Hierarchy**: Settings in environment-specific files override the base settings
> - **Sensitive Data**: In a real application, you would use a secure secret manager for production keys
> - **Nested Configuration**: Using nested objects (ApiKey.Value) organizes related settings
>
> ⚠️ **Common Mistakes**
>
> - Storing production API keys in source control
> - Using predictable or simple API keys
> - Not having different keys for different environments
> - Not documenting the configuration requirements

### Step 6: Register Authentication in Program.cs

**Introduction**: Now we need to register our authentication scheme with the ASP.NET Core dependency injection system and configure Swagger to support API Key authentication. This makes our authentication handler available to the application.

1. Update the `Program.cs` file to register the API Key authentication and configure Swagger:

    > `src/MerchStore.WebUI/Program.cs`

    ```csharp
    using MerchStore.Application;
    using MerchStore.Infrastructure;
    using MerchStore.WebUI.Authentication.ApiKey;

    var builder = WebApplication.CreateBuilder(args);

    // Add services to the container.
    builder.Services.AddControllersWithViews();

    // Add API Key authentication
    builder.Services.AddAuthentication()
       .AddApiKey(builder.Configuration["ApiKey:Value"] ?? throw new InvalidOperationException("API Key is not configured in the application settings."));

    // Add API Key authorization
    builder.Services.AddAuthorization(options =>
    {
        options.AddPolicy("ApiKeyPolicy", policy =>
            policy.AddAuthenticationSchemes(ApiKeyAuthenticationDefaults.AuthenticationScheme)
                  .RequireAuthenticatedUser());
    });

    // Add Application services - this includes Services, Interfaces, etc.
    builder.Services.AddApplication();

    // Add Infrastructure services - this includes DbContext, Repositories, etc.
    builder.Services.AddInfrastructure(builder.Configuration);

    // Add Swagger for API documentation
    builder.Services.AddEndpointsApiExplorer();
    builder.Services.AddSwaggerGen(options =>
    {
        options.SwaggerDoc("v1", new OpenApiInfo 
        { 
            Title = "MerchStore API", 
            Version = "v1",
            Description = "API for MerchStore product catalog",
            Contact = new OpenApiContact
            {
                Name = "MerchStore Support",
                Email = "support@merchstore.example.com"
            }
        });

        // Include XML comments if you've enabled XML documentation in your project
        var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
        var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
        if (File.Exists(xmlPath))
        {
            options.IncludeXmlComments(xmlPath);
        }
        
        // Add API Key authentication support to Swagger UI
        options.AddSecurityDefinition(ApiKeyAuthenticationDefaults.AuthenticationScheme, new OpenApiSecurityScheme
        {
            Description = "API Key Authentication. Enter your API key in the field below.",
            Name = ApiKeyAuthenticationDefaults.HeaderName,
            In = ParameterLocation.Header,
            Type = SecuritySchemeType.ApiKey,
            Scheme = ApiKeyAuthenticationDefaults.AuthenticationScheme
        });
        
        // Apply API key requirement only to controller-based endpoints
        options.OperationFilter<SecurityRequirementsOperationFilter>();
    });

    var app = builder.Build();

    // Configure the HTTP request pipeline.
    if (!app.Environment.IsDevelopment())
    {
        app.UseExceptionHandler("/Home/Error");
        // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
        app.UseHsts();
    }
    else
    {
        // In development, seed the database with test data using the extension method
        app.Services.SeedDatabaseAsync().Wait();
    }

    app.UseHttpsRedirection();
    app.UseStaticFiles();
    app.UseRouting();

    // Add authentication middleware
    app.UseAuthentication();
    
    // Add authorization middleware
    app.UseAuthorization();

    app.MapControllerRoute(
        name: "default",
        pattern: "{controller=Home}/{action=Index}/{id?}");

    app.Run();
    ```

> 💡 **Information**
>
> - **Authentication Registration**: Sets up the authentication services with our custom scheme
> - **Default Schemes**: Defines which authentication scheme to use by default
> - **Middleware Order**: Authentication and authorization middleware must be added in the right order
> - **Authorization Policy**: Defines a named policy that requires the API key authentication scheme
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to call `UseAuthentication()` in the middleware pipeline
> - Placing authentication middleware in the wrong order (after routing but before endpoints)
> - Not configuring default authentication and challenge schemes
> - Forgetting to add authorization services and middleware

### Step 6: Apply Authentication to the API Controller

**Introduction**: Finally, we'll apply the authentication requirement to our API controller. This will ensure that only requests with a valid API key can access our endpoints.

1. Update the BasicProductsApiController to require authentication:

    > `src/MerchStore.WebUI/Controllers/Api/Products/BasicProductsApiController.cs`

    ```csharp
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.AspNetCore.Authorization;
    using MerchStore.WebUI.Models.Api.Basic;
    using MerchStore.Application.Services.Interfaces;

    namespace MerchStore.WebUI.Controllers.Api.Products;

    /// <summary>
    /// Basic API controller for read-only product operations.
    /// Requires API Key authentication.
    /// </summary>
    [Route("api/basic/products")]
    [ApiController]
    [Authorize(Policy = "ApiKeyPolicy")]
    public class BasicProductsApiController : ControllerBase
    {
        private readonly ICatalogService _catalogService;
        
        /// <summary>
        /// Constructor with dependency injection
        /// </summary>
        /// <param name="catalogService">The catalog service for accessing product data</param>
        public BasicProductsApiController(ICatalogService catalogService)
        {
            _catalogService = catalogService;
        }
        
        // The rest of the controller remains unchanged...
    }
    ```

> 💡 **Information**
>
> - **Authorize Attribute**: Indicates that the controller requires authentication
> - **Policy-based Authorization**: Uses the policy we defined earlier
> - **Controller-level Attribute**: Applies to all actions in the controller
> - **Action-level Override**: You could override this at the action level if needed
>
> ⚠️ **Common Mistakes**
>
> - Using `[Authorize]` without specifying the correct authentication scheme or policy
> - Forgetting to register the authorization policy
> - Applying authentication inconsistently across API endpoints

### Step 7: Test the Secured API

**Introduction**: Now that we've implemented API Key authentication, let's test our API to ensure it requires a valid API key and rejects unauthorized requests.

1. Run your application:

    ```bash
    dotnet run --project src/MerchStore.WebUI
    ```

2. Test with curl without providing an API key:

    ```bash
    curl -X GET https://localhost:7188/api/basic/products -k -v
    ```

    You should receive a 401 Unauthorized response.

3. Test with curl providing a valid API key:

    ```bash
    curl -X GET https://localhost:7188/api/basic/products -k -v -H "X-API-Key: API_KEY"
    ```

    You should receive a 200 OK response with the list of products.

4. Test with REST Client in VS Code:

    > `api-tests.rest`

    ```sh
    ### Get all products (without API key - should fail)
    GET https://localhost:7188/api/basic/products

    ### Get all products (with valid API key)
    GET https://localhost:7188/api/basic/products
    X-API-Key: API_KEY
    ```

5. Test with Postman:
   - Create a new request to `https://localhost:7188/api/basic/products`
   - Add a header with key `X-API-Key` and value `API_KEY`
   - Send the request and verify you receive a 200 OK response

> 💡 **Information**
>
> - **401 Unauthorized**: The correct response for missing or invalid authentication
> - **HTTP Headers**: API keys are sent in request headers for security
> - **Verbose Output**: Using `-v` with curl shows detailed request/response information
> - **Testing Tools**: Testing with multiple tools ensures the authentication works consistently
>
> ⚠️ **Common Mistakes**
>
> - Using the wrong header name
> - Case sensitivity issues in the API key
> - Not checking for 401 vs 403 status codes (authentication vs. authorization)
> - Confusing API keys between different environments

## 🧪 Final Tests

### Verify Authentication is Working Correctly

1. Test the API without authentication, expecting a 401 Unauthorized response.
2. Test the API with the wrong API key, expecting a 401 Unauthorized response.
3. Test the API with the correct API key, expecting a 200 OK response with product data.
4. Verify that API key validation messages appear in the logs.

✅ **Expected Results**

- The API endpoints now require a valid API key
- Requests without an API key are rejected with a 401 Unauthorized response
- Requests with an invalid API key are rejected with a 401 Unauthorized response
- Requests with a valid API key receive a 200 OK response with the expected data
- Authentication-related log messages are generated to help with troubleshooting

## 🔧 Troubleshooting

If you encounter issues:

- **401 Unauthorized when using the correct API key**:
  - Check that the API key in your request exactly matches the one in settings
  - Verify the header name is correct (X-API-Key)
  - Ensure your authorization policy is correctly defined
  - Check ASP.NET Core logs for authentication failure details

- **No 401 response when omitting the API key**:
  - Confirm that you've added the `[Authorize]` attribute to your controller
  - Verify that `UseAuthentication()` and `UseAuthorization()` are in the middleware pipeline
  - Check the order of middleware registration in Program.cs

- **API key configuration issues**:
  - Use `IOptions<ApiKeyOptions>` to debug configuration values at runtime
  - Check environment-specific settings files for overrides
  - Inspect configuration using the Configuration page in Dev Tools

## 🚀 Optional Challenge

Want to take your learning further? Try:

1. **Multiple API Keys**: Modify the authentication handler to support multiple valid API keys from configuration
2. **Rate Limiting**: Add rate limiting based on the API key to limit how many requests each client can make
3. **API Key Management UI**: Create a simple admin UI for generating, viewing, and revoking API keys
4. **Key Scope**: Extend the authentication handler to support different permission scopes for different API keys

## 📚 Further Reading

- [ASP.NET Core Authentication](https://docs.microsoft.com/en-us/aspnet/core/security/authentication) - Microsoft's documentation on authentication
- [Custom Authentication in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/customize-authentication) - Detailed guide on custom authentication
- [OWASP API Security](https://owasp.org/www-project-api-security/) - Best practices for API security
- [API Key Best Practices](https://cloud.google.com/endpoints/docs/openapi/when-why-api-key) - Google's guide on API key usage
- [Security Headers](https://securityheaders.com/) - Information on security-related HTTP headers

## Done! 🎉

Congratulations! You've successfully implemented API Key authentication for your Basic Products API. This approach provides a simple yet effective layer of security that ensures only authorized clients can access your endpoints.

API Key authentication is a great starting point for API security and is commonly used for server-to-server authentication, internal APIs, and developer access. As your API evolves, you might need more sophisticated authentication methods like OAuth 2.0 or JWT for user-specific authentication and authorization.

The skills you've learned in this exercise form the foundation for implementing more advanced authentication schemes in future exercises. 🚀
