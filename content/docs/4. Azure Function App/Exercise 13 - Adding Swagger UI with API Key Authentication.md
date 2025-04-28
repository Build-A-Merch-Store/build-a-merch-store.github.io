---
title: "Adding Swagger UI with API Key Authentication"
date: "2025-04-28"
lastmod: "2025-04-28"
draft: false
weight: 402
toc: true
---

## 🎯 Goal

Enhance the mock Reviews API by adding interactive API documentation using Swagger UI, including support for testing endpoints secured with an API Key directly from the browser.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed **Exercise 12** or have a working Azure Function App project.
- Have the Azure Functions Core Tools installed.
- Have basic knowledge of C#, HTTP APIs, and OpenAPI (Swagger).
- Have Visual Studio Code installed with the Azure Functions extension and C# Dev Kit.

## 📚 Learning Objectives

By the end of this exercise, you will:

- Integrate the **Swagger UI** into an Azure Functions project.
- Configure **OpenAPI metadata** including security definitions for API Keys.
- Implement custom configuration classes for **OpenAPI settings**.
- Create a custom **document filter** to add security requirements to the entire API.
- Test **authenticated API endpoints** using the Swagger UI interface.

## 🔍 Why This Matters

Adding Swagger UI to your API provides significant benefits:

- **Interactive Documentation**: Allows users (and developers) to explore and understand the API without reading code.
- **Simplified Testing**: Enables easy testing of API endpoints directly from the browser, including handling authentication.
- **Clear Contract**: Defines the API's capabilities, parameters, responses, and security requirements explicitly.
- **Improved Discoverability**: Makes the API easier to find and use.
- **Client Code Generation**: OpenAPI definitions can be used to automatically generate client SDKs.

## 📝 Step-by-Step Instructions

### Step 1: Install Necessary NuGet Package

**Introduction**: First, ensure the necessary NuGet package for OpenAPI support in isolated Azure Functions is installed.

1. Open a terminal in the `ReviewApiFunction` project directory (`MerchStoreDemo/infra/ReviewApiFunction/`).
2. Run the following command to add the package (if not already present):

   ```bash
   dotnet add package Microsoft.Azure.Functions.Worker.Extensions.OpenApi
   ```

> 💡 **Information**
>
> - This package provides the necessary attributes and services to generate an OpenAPI specification document and host the Swagger UI.
> - This builds on the Azure Function created in Exercise 12 by adding documentation capabilities to it.

### Step 2: Create an API Key Authentication Document Filter

**Introduction**: First, we'll create a document filter that will apply API key authentication globally to all endpoints in the Swagger documentation.

1. Create a new file named `ApiKeyAuthDocumentFilter.cs` in the `ReviewApiFunction` project.
2. Add the following code:

   ```csharp
   // filepath: MerchStoreDemo/infra/ReviewApiFunction/ApiKeyAuthDocumentFilter.cs
   using Microsoft.Azure.WebJobs.Extensions.OpenApi.Core.Abstractions;
   using Microsoft.OpenApi.Models;
   using System.Collections.Generic;

   namespace ReviewApiFunction
   {
       // Custom document filter to add API key authentication to the Swagger documentation
       public class ApiKeyAuthDocumentFilter : IDocumentFilter
       {
           public void Apply(IHttpRequestDataObject request, OpenApiDocument document)
           {
               // Initialize components if it's null
               if (document.Components == null)
               {
                   document.Components = new OpenApiComponents();
               }
               
               if (document.Components.SecuritySchemes == null)
               {
                   document.Components.SecuritySchemes = new Dictionary<string, OpenApiSecurityScheme>();
               }
               
               // Add API key security scheme
               document.Components.SecuritySchemes.Add("function_key", new OpenApiSecurityScheme
               {
                   Name = "x-functions-key",
                   Type = SecuritySchemeType.ApiKey,
                   In = ParameterLocation.Header,
                   Description = "Azure Function API Key authentication"
               });
               
               // Initialize security requirements if null
               document.SecurityRequirements ??= new List<OpenApiSecurityRequirement>();
               
               // Add global security requirement
               document.SecurityRequirements.Add(new OpenApiSecurityRequirement
               {
                   {
                       new OpenApiSecurityScheme
                       {
                           Reference = new OpenApiReference
                           {
                               Type = ReferenceType.SecurityScheme,
                               Id = "function_key"
                           }
                       },
                       new List<string>()
                   }
               });
           }
       }
   }
   ```

> 💡 **Information**
>
> - This class implements `IDocumentFilter` which enables modification of the OpenAPI document.
> - The `Apply` method adds an API key security scheme to the document and applies it globally to all API endpoints.
> - The security scheme uses the `x-functions-key` header which is the standard header used for Azure Functions authentication, matching the authentication we set up in Exercise 12.
> - By adding this as a global security requirement, all API operations will show the lock icon in Swagger UI.

### Step 3: Create Custom Swagger Configuration Classes

**Introduction**: Now, we'll create classes to customize the OpenAPI configuration and Swagger UI appearance.

1. Create a new file named `SwaggerConfiguration.cs` in the `ReviewApiFunction` project.
2. Add the following code:

   ```csharp
   // filepath: MerchStoreDemo/infra/ReviewApiFunction/SwaggerConfiguration.cs
   using Microsoft.Azure.WebJobs.Extensions.OpenApi.Core.Abstractions;
   using Microsoft.Azure.WebJobs.Extensions.OpenApi.Core.Enums;
   using Microsoft.OpenApi.Models;
   using System.Collections.Generic;
   using System.Reflection;
   using System.Threading.Tasks;

   namespace ReviewApiFunction
   {
       public class SwaggerConfiguration : IOpenApiConfigurationOptions
       {
           public OpenApiInfo Info { get; set; } = new OpenApiInfo()
           {
               Version = "1.0.0",
               Title = "Product Reviews API",
               Description = "API for product reviews in the MerchStore application",
               Contact = new OpenApiContact()
               {
                   Name = "MerchStore Team"
               }
           };

           public List<OpenApiServer> Servers { get; set; } = new List<OpenApiServer>();
           public OpenApiVersionType OpenApiVersion { get; set; } = OpenApiVersionType.V3;
           public bool IncludeRequestingHostName { get; set; } = true;
           public bool ForceHttps { get; set; } = false;
           public bool ForceHttp { get; set; } = false;
           // Add the filter here
           public List<IDocumentFilter> DocumentFilters { get; set; } = new List<IDocumentFilter>
           {
               new ApiKeyAuthDocumentFilter()
           };
       }

       public class SwaggerUIConfiguration : IOpenApiCustomUIOptions
       {
           public string? CustomStylesheetPath { get; set; }
           public string? CustomJavaScriptPath { get; set; }

           public Task<string?> GetJavaScriptAsync()
           {
               return Task.FromResult<string?>(null);
           }

           public Task<string?> GetStylesheetAsync()
           {
               return Task.FromResult<string?>(null);
           }
       }
   }
   ```

> 💡 **Information**
>
> - `SwaggerConfiguration` implements `IOpenApiConfigurationOptions` to define the API information and settings.
> - The `DocumentFilters` property includes our custom `ApiKeyAuthDocumentFilter` to apply API key authentication.
> - `SwaggerUIConfiguration` implements `IOpenApiCustomUIOptions` to allow customization of the Swagger UI.
> - In this implementation, we're returning null from the stylesheet and JavaScript methods to use default styles.
> - Both classes are defined in the same file for simplicity, though they could be separated.

### Step 4: Register Services in `Program.cs`

**Introduction**: Now we need to tell the Azure Functions host to use our custom configuration classes for OpenAPI.

1. Open the `Program.cs` file.
2. Add the necessary using statements to the top of the file:

   ```csharp
    using Microsoft.Azure.WebJobs.Extensions.OpenApi.Core.Abstractions;
    using ReviewApiFunction;
   ```

3. Locate the `.ConfigureServices` section within the `HostBuilder` configuration.
4. Add the following lines to register your custom configuration classes as singletons:

   ```csharp
   // Add OpenAPI configuration using our custom classes
   services.AddSingleton<IOpenApiConfigurationOptions, SwaggerConfiguration>();

   // Add custom UI options
   services.AddSingleton<IOpenApiCustomUIOptions, SwaggerUIConfiguration>();
   ```

> 💡 **Information**
>
> - `AddSingleton<IOpenApiConfigurationOptions, SwaggerConfiguration>()`: Tells the OpenAPI extension to use your `SwaggerConfiguration` class for generating the OpenAPI spec.
> - `AddSingleton<IOpenApiCustomUIOptions, SwaggerUIConfiguration>()`: Tells the extension to use your `SwaggerUIConfiguration` for customizing the UI.
> - This step integrates with the existing Function App setup from Exercise 12, extending its capabilities with Swagger documentation.

### Step 5: Enhance the Function with OpenAPI Attributes

**Introduction**: Now, we'll enhance our HTTP Trigger function from Exercise 12 with OpenAPI attributes to document it.

1. Open the `GetProductReviews.cs` file we created in Exercise 12.
2. Add the necessary using statements at the top of the file:

   ```csharp
    using Microsoft.Azure.WebJobs.Extensions.OpenApi.Core.Attributes;
    using Microsoft.OpenApi.Models;
    using System.Net;
   ```

3. Update the `[Function("GetProductReviews")]` method by adding OpenAPI attributes before it:

   ```csharp
   [Function("GetProductReviews")]
   [OpenApiOperation(operationId: "GetProductReviews", tags: new[] { "Reviews" }, Summary = "Get product reviews", Description = "This retrieves all reviews for a specific product.")]
   [OpenApiParameter(name: "productId", In = ParameterLocation.Path, Required = true, Type = typeof(string), Summary = "The ID of the product to get reviews for", Description = "The product ID must be a valid GUID")]
   [OpenApiResponseWithBody(statusCode: HttpStatusCode.OK, contentType: "application/json", bodyType: typeof(ReviewResponse), Summary = "Successful operation", Description = "The product reviews were successfully retrieved")]
   [OpenApiResponseWithoutBody(statusCode: HttpStatusCode.BadRequest, Summary = "Invalid product ID", Description = "The product ID format was invalid")]
   [OpenApiResponseWithoutBody(statusCode: HttpStatusCode.InternalServerError, Summary = "Internal server error", Description = "An unexpected error occurred processing the request")]
   public async Task<IActionResult> Run(
       [HttpTrigger(AuthorizationLevel.Function, "get", Route = "products/{productId}/reviews")] HttpRequest req,
       string productId)
   {
       // Existing function code remains the same
   ```

> 💡 **Information**
>
> - The OpenAPI attributes add metadata to your function that will be used to generate the Swagger documentation.
> - We're documenting the same function created in Exercise 12, so the core functionality doesn't change.
> - Note that we don't need to add an explicit `[OpenApiSecurity]` attribute because our document filter applies security globally.
> - These attributes enhance the API documentation without changing the behavior of the API from Exercise 12.

### Step 6: Test Locally with Swagger UI

**Introduction**: Now let's run the function locally and test the secured endpoint using the Swagger UI.

1. Start the function host locally from the `ReviewApiFunction` directory:

   ```bash
   func start
   ```

2. Wait for the host to start. It will output the URLs for your functions and also the Swagger UI endpoints. Look for lines similar to:

   ```text
    Functions:
    
            GetProductReviews: [GET] http://localhost:7071/api/products/{productId}/reviews
    
            RenderOAuth2Redirect: [GET] http://localhost:7071/api/oauth2-redirect.html
    
            RenderOpenApiDocument: [GET] http://localhost:7071/api/openapi/{version}.{extension}
    
            RenderSwaggerDocument: [GET] http://localhost:7071/api/swagger.{extension}
    
            RenderSwaggerUI: [GET] http://localhost:7071/api/swagger/ui
   ```

3. Open the Swagger UI URL (e.g., `http://localhost:7071/api/swagger/ui`) in your web browser.
4. You should see the "Product Reviews API" documentation with the endpoint description. Notice the **Authorize** button (with a lock icon) near the top right.
5. Click the **Authorize** button. A dialog will appear asking for the API Key (`x-functions-key`).
6. Enter the function key value into the "Value" field in the Swagger UI authorization dialog and click "Authorize", then "Close". The lock icon should now appear closed.
7. Expand the `GET /api/products/{productId}/reviews` operation.
8. Click "Try it out".
9. Enter a valid GUID for the `productId` (e.g., `00000000-0000-0000-0000-000000000000`).
10. Click "Execute".

    - **Expected Result**: You should receive a `200 OK` response with the mock review data including a list of reviews and statistics. The "Curl" command shown will include the `-H 'x-functions-key: YOUR_KEY_VALUE'` header.
    - **If Unauthorized (401)**: Double-check that you entered the correct API key in the "Authorize" dialog and that the `AuthorizationLevel` in the `HttpTrigger` attribute is set to `Function`.
    - **If Bad Request (400)**: Make sure you entered a valid GUID format for the product ID.

> 💡 **Information**
>
> - The Swagger UI provides an interactive way to test the same API you created in Exercise 12.
> - This testing approach complements the curl and Postman testing methods shown in Exercise 12.
> - The API key authorization works the same way as in Exercise 12, but now it's integrated with the Swagger UI for easier testing.

### Step 7: Deploy the Updated Function to Azure

**Introduction**: Let's update our Azure Function App with the Swagger UI enhancements.

1. Set the function app name variable to match what was created in Exercise 12:

   ```bash
   FUNCTION_APP_NAME="ReviewApiFunc$(date +"%y%m%d")"
   ```

2. Publish the updated function to Azure:

   ```bash
   func azure functionapp publish $FUNCTION_APP_NAME
   ```

3. After successful deployment, verify the Swagger UI endpoint:

   ```bash
   echo "Swagger UI: https://$FUNCTION_APP_NAME.azurewebsites.net/api/swagger/ui"
   echo "OpenAPI Spec: https://$FUNCTION_APP_NAME.azurewebsites.net/api/openapi/v3.json"
   ```

> 💡 **Information**
>
> - This deployment updates your existing function app from Exercise 12 with the new Swagger capabilities.
> - The API functionality remains the same, but now it includes interactive documentation.
> - You can share the Swagger UI URL with other developers to help them understand your API.

## 🧪 Testing the Swagger UI in Production

To verify that your Swagger UI is working properly in your deployed Azure Function App:

1. Open the Swagger UI URL in your browser:

   ```sh
   https://<your-function-app-name>.azurewebsites.net/api/swagger/ui
   ```

2. You should see the same Swagger UI interface that you tested locally.

3. Try authenticating and making requests to confirm that the authentication flow works in production:
   - Click the "Authorize" button
   - Enter the API key value (`campusmolndal` if you followed Exercise 12)
   - Try calling the API with a valid product ID (3fa85f64-5717-4562-b3fc-2c963f66afa6)

4. Compare the documentation experience with the manual testing approaches from Exercise 12:
   - Swagger UI provides interactive documentation and testing
   - The client application from Exercise 12 provides a more programmatic way to access the API
   - Both approaches use the same underlying API and authentication mechanism

> 💡 **Information**
>
> - The Swagger UI can be especially useful for other team members who need to understand how to use your API.
> - The OpenAPI specification can be used to generate client code in various languages.

## 🔧 Troubleshooting

If you encounter issues with the Swagger UI:

- **Swagger UI Not Found**:
  - Make sure the OpenAPI package is properly installed and referenced in the project
  - Verify that the OpenAPI services are registered in `Program.cs`
  - Check Azure Function logs for any errors related to OpenAPI initialization

- **Authentication Not Working in Swagger UI**:
  - Ensure the API key is entered correctly in the Authorize dialog
  - Check that the `function_key` security scheme name in `ApiKeyAuthDocumentFilter.cs` is consistent
  - Verify that the document filter is properly registered in the `SwaggerConfiguration` class

- **OpenAPI Metadata Not Showing**:
  - Ensure that the OpenAPI attributes are correctly applied to your function
  - Check that the model classes have the appropriate documentation comments
  - Verify that the `SwaggerConfiguration` class is properly configured

- **Deployment Issues**:
  - Make sure all NuGet packages are properly restored before deployment
  - Check deployment logs for any errors related to the OpenAPI extension
  - Verify that the target runtime is compatible with the OpenAPI extension

## 🚀 Optional Challenge

Want to extend your Swagger implementation? Try these challenges:

1. **Add XML Documentation**: Enhance your models with XML documentation comments that will be displayed in the Swagger UI.
2. **Custom Swagger Theme**: Create a custom theme for your Swagger UI by implementing the `GetStylesheetAsync` method in `SwaggerUIConfiguration`.
3. **Multiple Authentication Methods**: Add support for both API key and OAuth authentication in your Swagger UI.
4. **Additional Endpoints**: Create a new endpoint with different input/output models and document it using OpenAPI attributes.
5. **Generate Client SDK**: Use the OpenAPI specification to generate a client SDK in TypeScript, C#, or another language of your choice.

## 📚 Further Reading

- [OpenAPI Extension for Azure Functions](https://github.com/Azure/azure-functions-openapi-extension)
- [OpenAPI Specification](https://swagger.io/specification/)
- [Swagger UI Configuration](https://swagger.io/docs/open-source-tools/swagger-ui/usage/configuration/)
- [API Design Best Practices](https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design)
- [OpenAPI Document Filters](https://github.com/domaindrivendev/Swashbuckle.AspNetCore#document-filters)

## 🎉 Done

Congratulations! You've successfully enhanced your Azure Function App with Swagger UI and API Key authentication. This improvement makes your API more developer-friendly by providing interactive documentation and testing capabilities, while still maintaining the security of your endpoints.

Remember that this is building directly on top of the mock Reviews API you created in Exercise 12, enhancing it with documentation and interactive testing capabilities. The combination of a mock API (Exercise 12) with proper documentation (Exercise 13) provides an excellent development experience for your team.
