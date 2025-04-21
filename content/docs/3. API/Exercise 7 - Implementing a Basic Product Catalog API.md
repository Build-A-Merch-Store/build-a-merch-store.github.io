---
title: "Implementing a Basic Product Catalog API"
date: "2025-04-19"
lastmod: "2025-04-19"
draft: false
weight: 301
toc: true
---

## 🎯 Goal

Create a simple read-only API for the product catalog that exposes listing and details endpoints using a traditional controller-based approach.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 1 (Setting Up a Clean Architecture Solution)
- Have completed Exercise 2 (Creating a Simple Product Entity)
- Have completed Exercise 4 (Implementing the Infrastructure Layer for Products)
- Have completed Exercise 6 (Building a Simple Product Catalog)
- Understand basic ASP.NET Core MVC concepts
- Be familiar with HTTP and REST concepts

## 📚 Learning Objectives

By the end of this exercise, you will:

- Create a **dedicated API controller** for product catalog endpoints
- Implement **REST-compliant endpoints** for listing and retrieving products
- Define **Data Transfer Objects** (DTOs) specifically for API responses
- Follow **proper naming conventions and file organization** for APIs
- Understand **status codes** and their appropriate usage
- Test your API using **HTTP requests**

## 🔍 Why This Matters

In real-world applications, APIs are crucial because:

- They enable integration with other systems and applications
- They allow for building various client applications (web, mobile, desktop)
- They promote separation between frontend and backend
- They represent industry standard patterns for system communication
- They provide a structured way to expose your domain functionality

## 📝 Step-by-Step Instructions

### Step 1: Create a Basic Product DTO

**Introduction**: First, we'll create a Data Transfer Object (DTO) that will be returned from our API. This separates our domain model from our API response model, providing better encapsulation and flexibility.

1. Create a directory structure for API models:

    ```bash
    mkdir -p src/MerchStore.WebUI/Models/Api/Basic
    ```

2. Create the BasicProductDto class:

    > `src/MerchStore.WebUI/Models/Api/Basic/BasicProductDto.cs`

    ```csharp
    namespace MerchStore.WebUI.Models.Api.Basic;

    /// <summary>
    /// Simple Data Transfer Object for product information in the Basic API.
    /// </summary>
    /// <remarks>
    /// This DTO contains only the essential product information needed for the Basic API response.
    /// It's intentionally kept separate from DTOs used in other API implementations to allow
    /// independent evolution.
    /// </remarks>
    public class BasicProductDto
    {
        /// <summary>
        /// The unique identifier of the product
        /// </summary>
        public Guid Id { get; set; }
        
        /// <summary>
        /// The name of the product
        /// </summary>
        public string Name { get; set; } = string.Empty;
        
        /// <summary>
        /// A description of the product
        /// </summary>
        public string Description { get; set; } = string.Empty;
        
        /// <summary>
        /// The price amount of the product
        /// </summary>
        public decimal Price { get; set; }
        
        /// <summary>
        /// The currency code for the price (e.g., "SEK", "USD")
        /// </summary>
        public string Currency { get; set; } = string.Empty;
        
        /// <summary>
        /// The URL to the product image, if available
        /// </summary>
        public string? ImageUrl { get; set; }
        
        /// <summary>
        /// The current stock quantity of the product
        /// </summary>
        public int StockQuantity { get; set; }
        
        /// <summary>
        /// Indicates whether the product is currently in stock
        /// </summary>
        public bool InStock => StockQuantity > 0;
    }
    ```

> 💡 **Information**
>
> - **DTOs as Presentation Concerns**: DTOs belong in the presentation layer (WebUI) since they shape data for external consumption, similar to ViewModels
> - **Clean Architecture Separation**: This keeps the Application layer focused on business rules without mixing presentation concerns
> - **Models/Api Organization**: Placing DTOs in a dedicated Api folder distinguishes them from MVC ViewModels while keeping them in the presentation layer
> - **XML Documentation**: Adding XML comments helps with API documentation and may be used later by Swagger
> - **Computed Properties**: Properties like `InStock` make the API more useful without exposing internal domain logic

### Step 2: Create the Basic Products API Controller

**Introduction**: Now we'll create a dedicated API controller specifically for the basic product catalog functionality. This controller will follow RESTful principles and use our existing CatalogService for data access.

1. Create a directory structure for API controllers:

    ```bash
    mkdir -p src/MerchStore.WebUI/Controllers/Api/Products
    ```

2. Create the BasicProductsApiController class:

    > `src/MerchStore.WebUI/Controllers/Api/Products/BasicProductsApiController.cs`

    ```csharp
    using Microsoft.AspNetCore.Mvc;
    using MerchStore.WebUI.Models.Api.Basic;
    using MerchStore.Application.Services.Interfaces;
    
    namespace MerchStore.WebUI.Controllers.Api.Products;
    
    /// <summary>
    /// Basic API controller for read-only product operations.
    /// Provides simple endpoints for listing and retrieving products.
    /// </summary>
    [Route("api/basic/products")]
    [ApiController]
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
    
        /// <summary>
        /// Gets all products
        /// </summary>
        /// <returns>A list of all products</returns>
        /// <response code="200">Returns the list of products</response>
        [HttpGet]
        [ProducesResponseType(typeof(IEnumerable<BasicProductDto>), StatusCodes.Status200OK)]
        public async Task<IActionResult> GetAll()
        {
            try
            {
                // Get all products from the service
                var products = await _catalogService.GetAllProductsAsync();
    
                // Map domain entities to DTOs
                var productDtos = products.Select(p => new BasicProductDto
                {
                    Id = p.Id,
                    Name = p.Name,
                    Description = p.Description,
                    Price = p.Price.Amount,
                    Currency = p.Price.Currency,
                    ImageUrl = p.ImageUrl?.ToString(),
                    StockQuantity = p.StockQuantity
                });
    
                // Return 200 OK with the list of products
                return Ok(productDtos);
            }
            catch
            {
                // Log the exception in a real application
    
                // Return 500 Internal Server Error
                return StatusCode(StatusCodes.Status500InternalServerError, 
                    new { message = "An error occurred while retrieving products" });
            }
        }
    
        /// <summary>
        /// Gets a specific product by ID
        /// </summary>
        /// <param name="id">The ID of the product to retrieve</param>
        /// <returns>The requested product</returns>
        /// <response code="200">Returns the requested product</response>
        /// <response code="404">If the product is not found</response>
        [HttpGet("{id}")]
        [ProducesResponseType(typeof(BasicProductDto), StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        public async Task<IActionResult> GetById(Guid id)
        {
            try
            {
                // Get the specific product from the service
                var product = await _catalogService.GetProductByIdAsync(id);
    
                // Return 404 Not Found if the product doesn't exist
                if (product is null)
                {
                    return NotFound(new { message = $"Product with ID {id} not found" });
                }
    
                // Map domain entity to DTO
                var productDto = new BasicProductDto
                {
                    Id = product.Id,
                    Name = product.Name,
                    Description = product.Description,
                    Price = product.Price.Amount,
                    Currency = product.Price.Currency,
                    ImageUrl = product.ImageUrl?.ToString(),
                    StockQuantity = product.StockQuantity
                };
    
                // Return 200 OK with the product
                return Ok(productDto);
            }
            catch
            {
                // Log the exception in a real application
    
                // Return 500 Internal Server Error
                return StatusCode(StatusCodes.Status500InternalServerError, 
                    new { message = "An error occurred while retrieving the product" });
            }
        }
    }
    ```

> 💡 **Information**
>
> - **Route Prefix**: The `[Route("api/basic/products")]` attribute defines the base URL for all endpoints in this controller
> - **ApiController Attribute**: The `[ApiController]` attribute enables API-specific behaviors like automatic model validation
> - **Response Types**: Using `[ProducesResponseType]` helps document the possible responses for each endpoint
> - **HTTP Status Codes**: Using appropriate status codes (200 OK, 404 Not Found, 500 Internal Server Error) follows REST best practices
> - **Exception Handling**: Try-catch blocks provide basic error handling with appropriate status codes
> - **Resource-based Routing**: The URL structure follows REST conventions with resources (products) as the main concept

### Step 3: Test the API Endpoints

**Introduction**: Now that we've implemented our API endpoints, let's test them using HTTP requests. This can be done with tools like Postman, curl, or even a web browser for GET requests.

1. Run your application:

    ```bash
    dotnet run --project src/MerchStore.WebUI
    ```

2. Test the "Get All Products" endpoint:

   **Request:**

   ```sh
   GET https://localhost:7188/api/basic/products
   ```

   You should receive a JSON response with all products:

   **Response:**

   ```json
   [
     {
       "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
       "name": "Conference T-Shirt",
       "description": "A comfortable cotton t-shirt with the conference logo.",
       "price": 249.99,
       "currency": "SEK",
       "imageUrl": "https://merchstore202503311226.blob.core.windows.net/images/tshirt.png",
       "stockQuantity": 50,
       "inStock": true
     },
     {
       "id": "6fb15d3e-c0a1-4f8f-9f88-4c139417bf3d",
       "name": "Developer Mug",
       "description": "A ceramic mug with a funny programming joke.",
       "price": 149.5,
       "currency": "SEK",
       "imageUrl": "https://merchstore202503311226.blob.core.windows.net/images/mug.png",
       "stockQuantity": 100,
       "inStock": true
     }
     // More products...
   ]
   ```

3. Test the "Get Product by ID" endpoint:

   **Request:**

   ```sh
   GET https://localhost:7188/api/basic/products/3fa85f64-5717-4562-b3fc-2c963f66afa6
   ```

   Replace the ID with an actual product ID from your database.

   You should receive a JSON response with the specific product:

   **Response:**

   ```json
   {
     "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
     "name": "Conference T-Shirt",
     "description": "A comfortable cotton t-shirt with the conference logo.",
     "price": 249.99,
     "currency": "SEK",
     "imageUrl": "https://merchstore202503311226.blob.core.windows.net/images/tshirt.png",
     "stockQuantity": 50,
     "inStock": true
   }
   ```

4. Test with an invalid product ID:

   **Request:**

   ```sh
   GET https://localhost:7188/api/basic/products/00000000-0000-0000-0000-000000000000
   ```

   You should receive a 404 Not Found response:

   **Response:**

   ```json
   {
     "message": "Product with ID 00000000-0000-0000-0000-000000000000 not found"
   }
   ```

> 💡 **Information**
>
> - **Content Type**: The API automatically returns responses with `Content-Type: application/json`
> - **Status Codes**: The response includes the appropriate HTTP status code (200, 404, etc.)
> - **Error Messages**: Error responses include a descriptive message to help clients understand the problem
> - **Testing Tools**: While you can use a browser for GET requests, tools like Postman or curl provide more capabilities for API testing

### Step 4: Add API Documentation with Swagger (Optional)

**Introduction**: Swagger (OpenAPI) provides interactive documentation for your API. It's especially useful for developers who need to understand and use your API.

1. Add the Swagger NuGet package to your WebUI project:

    ```bash
    dotnet add src/MerchStore.WebUI/MerchStore.WebUI.csproj package Swashbuckle.AspNetCore
    ```

2. Configure Swagger in Program.cs:

    > `src/MerchStore.WebUI/Program.cs`

    ```csharp
    using System.Reflection;
    using MerchStore.Application;
    using MerchStore.Infrastructure;
    using Microsoft.OpenApi.Models;
    
    var builder = WebApplication.CreateBuilder(args);
    
    // Add services to the container.
    builder.Services.AddControllersWithViews();
    
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
    
        // Enable Swagger UI in development
        app.UseSwagger();
        app.UseSwaggerUI(options =>
        {
            options.SwaggerEndpoint("/swagger/v1/swagger.json", "MerchStore API V1");
        });
    }
    
    app.UseHttpsRedirection();
    app.UseRouting();
    
    app.UseAuthorization();
    
    app.MapStaticAssets();
    
    app.MapControllerRoute(
        name: "default",
        pattern: "{controller=Home}/{action=Index}/{id?}")
        .WithStaticAssets();
    
    
    app.Run();
    ```

3. Enable XML documentation in your WebUI project file:

    > `src/MerchStore.WebUI/MerchStore.WebUI.csproj`

    ```xml
    <PropertyGroup>
      <TargetFramework>net9.0</TargetFramework>
      <Nullable>enable</Nullable>
      <ImplicitUsings>enable</ImplicitUsings>
      <!-- Add these lines: -->
      <GenerateDocumentationFile>true</GenerateDocumentationFile>
      <NoWarn>$(NoWarn);1591</NoWarn>
    </PropertyGroup>
    ```

4. Run your application and navigate to the Swagger UI:

    ```sh
    https://localhost:7188/swagger
    ```

> 💡 **Information**
>
> - **OpenAPI Specification**: Swagger implements the OpenAPI specification, which is an industry standard for REST API documentation
> - **Interactive Documentation**: Swagger UI allows users to try out API endpoints directly from the documentation
> - **XML Comments**: The XML comments you added to your controller and DTO are used to generate documentation
> - **Development Only**: Swagger UI is typically only enabled in development environments for security reasons
>
> ⚠️ **Common Mistakes**
>
> - Not enabling XML documentation generation in the project file
> - Not including complete XML documentation for endpoints and parameters
> - Exposing Swagger UI in production environments

## 🧪 Final Tests

### Run the Application and Test the API

1. Build and run the application:

   ```bash
   dotnet build
   dotnet run --project src/MerchStore.WebUI
   ```

2. Test the API endpoints as described in Step 3.

3. If you implemented Swagger, explore the API documentation.

✅ **Expected Results**

- The "Get All Products" endpoint returns a list of all products in JSON format
- The "Get Product by ID" endpoint returns a specific product when given a valid ID
- The "Get Product by ID" endpoint returns a 404 Not Found response for invalid IDs
- Error cases are handled appropriately with correct status codes and messages
- If implemented, Swagger provides accurate documentation of your API

## 🔧 Troubleshooting

If you encounter issues:

- Check that your controller route is correctly defined as `api/basic/products`
- Verify that the CatalogService is correctly registered in the DI container
- Ensure the database seeder is running and adding products
- Check for any exceptions in the application logs
- Verify JSON serialization settings if properties are missing in the response

## 🚀 Optional Challenge

Want to take your learning further? Try:

- Adding filtering by name (search functionality) to the "Get All Products" endpoint
- Implementing pagination for the "Get All Products" endpoint
- Adding a "Get Products by Category" endpoint if you have categories implemented
- Creating a more sophisticated DTO mapper using a library like AutoMapper
- Adding caching headers to improve API performance

## 📚 Further Reading

- [ASP.NET Core Web API Documentation](https://docs.microsoft.com/en-us/aspnet/core/web-api/) - Microsoft's official documentation
- [REST API Design Best Practices](https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design) - Microsoft's guidelines for designing RESTful APIs
- [Status Codes in REST APIs](https://www.restapitutorial.com/httpstatuscodes.html) - Reference for HTTP status codes
- [DTOs vs Domain Entities](https://ardalis.com/dto-or-not-the-data-transfer-object-pattern/) - An article discussing when to use DTOs
- [Swagger/OpenAPI with ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/tutorials/web-api-help-pages-using-swagger) - Microsoft's guide to API documentation

## Done! 🎉

Congratulations! You've successfully implemented a basic read-only API for your product catalog. This API follows RESTful principles and provides a solid foundation for more advanced API implementations you'll build later in the course.

This exercise introduced important API concepts like resource-based routing, appropriate HTTP status codes, DTOs for data transfer, and error handling. These concepts will be valuable as you progress to more sophisticated API patterns in future exercises. 🚀
