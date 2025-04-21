---
title: "Implementing a Minimal API Approach"
date: "2025-04-20"
lastmod: "2025-04-20"
draft: false
weight: 305
toc: true
---

## 🎯 Goal

Implement a product catalog API using ASP.NET Core's Minimal API approach, running in parallel with your existing controller-based API. This approach offers a more concise and lightweight alternative for building APIs.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 7 (Implementing a Basic Product Catalog API)
- Have your existing API running and accessible
- Understand basic ASP.NET Core concepts
- Be familiar with C# lambda expressions and extension methods

## 📚 Learning Objectives

By the end of this exercise, you will:

- Create **endpoint-focused APIs** using the Minimal API pattern
- Organize Minimal APIs using **extension methods** for better maintainability
- Understand the **differences between traditional controllers and Minimal APIs**
- Implement **proper route naming** to avoid conflicts with existing APIs
- Return **appropriate HTTP responses** from Minimal API endpoints
- Explore **mapping techniques** between domain entities and response objects

## 🔍 Why This Matters

In real-world applications, Minimal APIs are valuable because:

- They reduce boilerplate code for simple endpoints
- They offer better performance for lightweight microservices
- They provide a more focused approach to API development
- They're easier to understand for simple CRUD operations
- They enable rapid API development while maintaining good practices
- They represent the modern direction of ASP.NET Core development

## 📝 Step-by-Step Instructions

### Step 1: Create a Response Model for Minimal API

**Introduction**: First, we'll create a dedicated response model for our Minimal API. This helps avoid any naming conflicts with existing DTOs and clearly separates our different API implementations.

1. Install OpenAPI nuget package:

    ```bash
    dotnet add src/MerchStore.WebUI package Microsoft.AspNetCore.OpenApi
    ```

2. Create a directory structure for Minimal API models:

    ```bash
    mkdir -p src/MerchStore.WebUI/Models/Api/Minimal
    ```

3. Create the MinimalProductResponse class:

    > `src/MerchStore.WebUI/Models/Api/Minimal/MinimalProductResponse.cs`

    ```csharp
    namespace MerchStore.WebUI.Models.Api.Minimal;

    /// <summary>
    /// Response model for the Minimal API product endpoints.
    /// </summary>
    public class MinimalProductResponse
    {
        /// <summary>
        /// The unique identifier of the product.
        /// </summary>
        public Guid Id { get; set; }
        
        /// <summary>
        /// The name of the product.
        /// </summary>
        public string Name { get; set; } = string.Empty;
        
        /// <summary>
        /// A description of the product.
        /// </summary>
        public string Description { get; set; } = string.Empty;
        
        /// <summary>
        /// The price amount of the product.
        /// </summary>
        public decimal Price { get; set; }
        
        /// <summary>
        /// The currency code for the price (e.g., "SEK", "USD").
        /// </summary>
        public string Currency { get; set; } = string.Empty;
        
        /// <summary>
        /// The URL to the product image, if available.
        /// </summary>
        public string? ImageUrl { get; set; }
        
        /// <summary>
        /// The current stock quantity of the product.
        /// </summary>
        public int StockQuantity { get; set; }
        
        /// <summary>
        /// Indicates whether the product is currently in stock.
        /// </summary>
        public bool InStock => StockQuantity > 0;
    }
    ```

> 💡 **Information**
>
> - **Separate Response Models**: Using dedicated response models for each API implementation prevents conflicts and allows for independent evolution
> - **XML Documentation**: Adding XML comments helps with API documentation and Swagger integration
> - **Property Design**: The model properties match those on our domain entity, but without domain behavior or validation
> - **Calculated Properties**: Properties like `InStock` provide derived information for API consumers
>
> ⚠️ **Common Mistakes**
>
> - Reusing DTOs across different API implementations, which creates tight coupling
> - Not providing XML documentation for Swagger/OpenAPI generation
> - Adding domain logic to response models, which should only be for data transfer

### Step 2: Create a Minimal API Endpoints Class

**Introduction**: Next, we'll create a dedicated class for our Minimal API endpoints. This promotes clean organization and separation of concerns, while making our Program.cs file more manageable.

1. Create a directory structure for Minimal API endpoints:

    ```bash
    mkdir -p src/MerchStore.WebUI/Endpoints
    ```

2. Create the MinimalProductEndpoints class:

    > `src/MerchStore.WebUI/Endpoints/MinimalProductEndpoints.cs`

    ```csharp
    using MerchStore.Application.Services.Interfaces;
    using MerchStore.WebUI.Models.Api.Minimal;

    namespace MerchStore.WebUI.Endpoints;

    /// <summary>
    /// Extension methods for registering minimal API product endpoints.
    /// </summary>
    public static class MinimalProductEndpoints
    {
        /// <summary>
        /// Maps all product-related endpoints for the minimal API.
        /// </summary>
        /// <param name="app">The web application.</param>
        /// <returns>The web application for method chaining.</returns>
        public static WebApplication MapMinimalProductEndpoints(this WebApplication app)
        {
            // Define a route group for the minimal product API
            var group = app.MapGroup("/api/minimal/products")
                .WithTags("Minimal Products API")
                .WithOpenApi();

            // Get all products endpoint
            group.MapGet("/", GetAllProducts)
                .WithName("GetAllProductsMinimal")
                .WithDescription("Gets all available products")
                .Produces<List<MinimalProductResponse>>(StatusCodes.Status200OK)
                .Produces(StatusCodes.Status500InternalServerError);

            // Get product by ID endpoint
            group.MapGet("/{id}", GetProductById)
                .WithName("GetProductByIdMinimal")
                .WithDescription("Gets a specific product by its unique identifier")
                .Produces<MinimalProductResponse>(StatusCodes.Status200OK)
                .Produces(StatusCodes.Status404NotFound)
                .Produces(StatusCodes.Status500InternalServerError);

            return app;
        }

        /// <summary>
        /// Gets all products.
        /// </summary>
        private static async Task<IResult> GetAllProducts(ICatalogService catalogService)
        {
            try
            {
                // Get all products from the service
                var products = await catalogService.GetAllProductsAsync();
                
                // Map domain entities to response objects
                var response = products.Select(p => new MinimalProductResponse
                {
                    Id = p.Id,
                    Name = p.Name,
                    Description = p.Description,
                    Price = p.Price.Amount,
                    Currency = p.Price.Currency,
                    ImageUrl = p.ImageUrl?.ToString(),
                    StockQuantity = p.StockQuantity
                }).ToList();
                
                return Results.Ok(response);
            }
            catch (Exception ex)
            {
                // Log the exception in a real application
                return Results.Problem($"An error occurred while retrieving products: {ex.Message}");
            }
        }

        /// <summary>
        /// Gets a product by ID.
        /// </summary>
        private static async Task<IResult> GetProductById(Guid id, ICatalogService catalogService)
        {
            try
            {
                // Get the specific product from the service
                var product = await catalogService.GetProductByIdAsync(id);
                
                // Return 404 Not Found if the product doesn't exist
                if (product is null)
                {
                    return Results.NotFound($"Product with ID {id} not found");
                }
                
                // Map domain entity to response object
                var response = new MinimalProductResponse
                {
                    Id = product.Id,
                    Name = product.Name,
                    Description = product.Description,
                    Price = product.Price.Amount,
                    Currency = product.Price.Currency,
                    ImageUrl = product.ImageUrl?.ToString(),
                    StockQuantity = product.StockQuantity
                };
                
                return Results.Ok(response);
            }
            catch (Exception ex)
            {
                // Log the exception in a real application
                return Results.Problem($"An error occurred while retrieving the product: {ex.Message}");
            }
        }
    }
    ```

> 💡 **Information**
>
> - **Extension Method Pattern**: Using extension methods for registering endpoints keeps code organized
> - **Route Group**: Grouping related endpoints under a common prefix improves organization
> - **OpenAPI Integration**: The `WithTags`, `WithName`, and `WithDescription` methods enhance Swagger documentation
> - **Status Code Documentation**: The `Produces` method documents possible response types
> - **Dependency Injection**: Services are injected directly into endpoint handlers
> - **Results Factory**: The `Results` class provides factory methods for creating HTTP responses
>
> ⚠️ **Common Mistakes**
>
> - Not using route groups, which makes route management more difficult
> - Forgetting to document endpoints for OpenAPI/Swagger
> - Placing handler logic directly in Program.cs, making it hard to maintain
> - Using inconsistent naming for routes and endpoint handlers

### Step 3: Register the Minimal API Endpoints

**Introduction**: Now we'll update the Program.cs file to register our Minimal API endpoints. This makes them available alongside our existing controller-based API.

1. Update the Program.cs file to register the Minimal API endpoints:

    > `src/MerchStore.WebUI/Program.cs`

    ```csharp
    // Add this using statement at the top
    using MerchStore.WebUI.Endpoints;

    // Add this after app.MapControllerRoute(...)
    app.MapMinimalProductEndpoints();
    ```

> 💡 **Information**
>
> - **Extension Method Usage**: The extension method makes registering endpoints clean and concise
> - **Coexistence**: Minimal APIs can run alongside controller-based APIs in the same application
> - **Endpoint Registration Order**: The order of endpoint registration doesn't affect routing
> - **Method Chaining**: Extension methods support fluent API patterns with method chaining
>
> ⚠️ **Common Mistakes**
>
> - Registering endpoints before adding necessary middleware like routing
> - Not adding OpenAPI/Swagger support for Minimal API endpoints
> - Forgetting to register services required by the endpoints

### Step 4: Configure OpenAPI Support for Minimal APIs (Optional)

**Introduction**: To ensure our Minimal API endpoints are properly documented in Swagger, we'll update the Swagger configuration. This provides a cohesive API documentation experience across all API implementations.

1. Update the Swagger configuration in Program.cs:

    > `src/MerchStore.WebUI/Program.cs`

    ```csharp
    // Update the Swagger configuration to include operation IDs for minimal APIs
    builder.Services.AddEndpointsApiExplorer();
    builder.Services.AddSwaggerGen(options =>
    {
        options.SwaggerDoc("v1", new OpenApiInfo 
        { 
            Title = "MerchStore API", 
            Version = "v1",
            Description = "API for MerchStore product catalog"
        });
        
        // Include XML comments if you've enabled XML documentation
        var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
        var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
        if (File.Exists(xmlPath))
        {
            options.IncludeXmlComments(xmlPath);
        }
        
        // Configure operation IDs for minimal APIs to avoid conflicts
        options.CustomOperationIds(apiDesc =>
        {
            return apiDesc.TryGetMethodInfo(out var methodInfo) ? methodInfo.Name : null;
        });
    });
    ```

> 💡 **Information**
>
> - **Operation IDs**: Unique identifiers for API operations in OpenAPI documentation
> - **XML Comments**: Documentation comments are included in the OpenAPI specification
> - **CustomOperationIds**: Helps avoid conflicts in operation IDs between different API styles
> - **Swagger UI**: Properly configured Swagger provides interactive documentation for all API endpoints
>
> ⚠️ **Common Mistakes**
>
> - Not configuring proper operation IDs, leading to conflicts in Swagger
> - Forgetting to include XML comments in the OpenAPI specification
> - Not organizing API endpoints with tags, making documentation harder to navigate

### Step 5: Test the Minimal API Endpoints

**Introduction**: Finally, let's test our Minimal API endpoints to ensure they work correctly alongside our existing API. We'll use different tools to verify that both implementations can coexist peacefully.

1. Run your application:

    ```bash
    dotnet run --project src/MerchStore.WebUI
    ```

2. Test the "Get All Products" endpoint:

    ```sh
    GET https://localhost:7188/api/minimal/products
    ```

3. Test the "Get Product by ID" endpoint:

    ```sh
    GET https://localhost:7188/api/minimal/products/3fa85f64-5717-4562-b3fc-2c963f66afa6
    ```

    (Replace the ID with an actual product ID from your database)

4. Compare the responses with your existing API endpoints:

    ```sh
    GET https://localhost:7188/api/basic/products
    GET https://localhost:7188/api/basic/products/3fa85f64-5717-4562-b3fc-2c963f66afa6
    ```

5. If you've configured Swagger, check that both API implementations are properly documented:

    ```sh
    https://localhost:7188/swagger
    ```

> 💡 **Information**
>
> - **Parallel APIs**: Both API implementations should function independently
> - **Route Separation**: Different API prefixes prevent routing conflicts
> - **Response Format**: Both APIs should return consistent data structures
> - **Documentation**: Swagger should show both sets of endpoints with proper descriptions
>
> ⚠️ **Common Mistakes**
>
> - Using the same route prefixes for different API implementations
> - Not checking for errors in the console or logs
> - Assuming that both APIs will behave identically in all scenarios

## 🧪 Final Tests

### Verify Your Minimal API Implementation

1. Test both the Minimal API and Controller-based API endpoints to ensure they both work correctly.

2. Compare the response payloads to ensure consistency between API implementations.

3. Check the Swagger documentation to verify that both sets of endpoints are properly documented.

4. Use the browser's Network tab to compare request and response times between the two implementations.

✅ **Expected Results**

- The Minimal API endpoints return the same data as the Controller-based API
- Both API implementations work independently without conflicts
- Swagger documentation includes both API implementations with proper descriptions
- Error handling works correctly in the Minimal API implementation

## 🔧 Troubleshooting

If you encounter issues:

- **Endpoint not found (404)**:
  - Check that you've registered the endpoints correctly
  - Verify the route prefixes are correct
  - Ensure the endpoint mapping comes after UseRouting()

- **Service resolution errors**:
  - Confirm that all required services are registered in the DI container
  - Check for proper service lifetime (scoped, singleton, etc.)

- **Documentation issues**:
  - Verify XML documentation is enabled in the project file
  - Check that you've configured Swagger to include XML comments
  - Ensure operation IDs are unique across all API endpoints

- **Response formatting issues**:
  - If snake_case is not applied, verify that JSON options are configured correctly
  - Check that the response model properties match the expected format

## 🚀 Optional Challenge

Want to take your learning further? Try:

1. **Additional Endpoints**: Add POST, PUT, and DELETE endpoints to the Minimal API implementation
2. **Endpoint Grouping**: Experiment with more complex endpoint grouping and route patterns
3. **Response Mapping**: Use a mapping library like AutoMapper to map entities to response objects
4. **Request Validation**: Implement request validation using extension methods
5. **Documentation Enhancement**: Add more detailed OpenAPI documentation with examples

## 📚 Further Reading

- [Minimal APIs in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis) - Microsoft's official documentation
- [Organizing Minimal API Endpoints](https://benfoster.io/blog/mvc-to-minimal-apis-aspnet-6/) - Best practices for organizing endpoints
- [Route Groups in .NET 7](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/route-groups) - Using route groups for better organization
- [Validation in Minimal APIs](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/validation) - Adding validation to Minimal API requests
- [OpenAPI Support in Minimal APIs](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/openapi) - Documenting Minimal APIs with OpenAPI

## Done! 🎉

Congratulations! You've successfully implemented a product catalog API using the Minimal API approach. This implementation runs alongside your existing controller-based API, demonstrating how different API styles can coexist in a single application.

Minimal APIs offer a more concise and focused approach to API development, with less boilerplate code and a more direct connection between routes and handlers. This can be especially valuable for microservices or simpler API scenarios where the full MVC pattern might be overkill.

By implementing both approaches, you now have firsthand experience with the tradeoffs between traditional controllers and Minimal APIs, allowing you to choose the right tool for each situation in your future API development. 🚀

## What's Next?

In future exercises, you'll explore:

1. **Versioned API with CQRS**: Building more sophisticated APIs using CQRS pattern with API versioning
2. **Advanced Authentication**: Implementing more robust authentication like JWT tokens
3. **API Documentation**: Creating comprehensive API documentation with examples and scenarios
4. **Performance Optimization**: Measuring and improving API performance across different implementations
