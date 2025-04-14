---
title: "Build a Simple Product Catalog"
date: "2025-04-14"
lastmod: "2025-04-14"
draft: false
weight: 205
toc: true
---

## 🎯 Goal

Create a simple product catalog to display products from the database using a traditional service approach.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 1 (Setting Up a Clean Architecture Solution)
- Have completed Exercise 2 (Creating a Simple Product Entity)
- Have completed Exercise 4 (Implementing the Infrastructure Layer for Products)
- Understand basic ASP.NET Core MVC concepts
- Be familiar with basic HTML and CSS

## 📚 Learning Objectives

By the end of this exercise, you will:

- Create a **simple service layer** in the application project
- Implement **service interfaces** for dependency injection
- Build a **CatalogController** in the WebUI project
- Create **View Models** to transfer data to the views
- Develop **Razor views** to display products in a grid layout
- Implement a **product details page** for individual products
- Connect all layers from database to user interface

## 🔍 Why This Matters

In real-world applications, quickly visualizing data is crucial because:

- It provides immediate feedback on your development progress
- It allows stakeholders to see tangible results early in the development process
- It helps identify design and usability issues early
- It bridges the gap between back-end and front-end development
- It motivates developers by showing concrete results of their work

## 📝 Step-by-Step Instructions

### Step 1: Create a Simple Catalog Service Interface

**Introduction**: First, we'll create a simple service interface in the Application layer. This interface will define the operations our service will provide, following the interface segregation principle from SOLID.

1. Create a directory structure for services:

    ```bash
    mkdir -p src/MerchStore.Application/Services
    mkdir -p src/MerchStore.Application/Services/Interfaces
    ```

2. Create the Catalog service interface:

    > `src/MerchStore.Application/Services/Interfaces/ICatalogService.cs`

    ```csharp
    using MerchStore.Domain.Entities;

    namespace MerchStore.Application.Services.Interfaces;

    /// <summary>
    /// Service interface for Catalog-related operations.
    /// Provides a simple abstraction over the repository layer.
    /// </summary>
    public interface ICatalogService
    {
        /// <summary>
        /// Gets all available products
        /// </summary>
        /// <returns>A collection of all products</returns>
        Task<IEnumerable<Product>> GetAllProductsAsync();
        
        /// <summary>
        /// Gets a product by its unique identifier
        /// </summary>
        /// <param name="id">The product ID</param>
        /// <returns>The product if found, null otherwise</returns>
        Task<Product?> GetProductByIdAsync(Guid id);
    }
    ```

> 💡 **Information**
>
> - **Interface Segregation**: By keeping the interface focused on specific operations, we follow the Interface Segregation Principle
> - **Task-based Asynchronous Pattern**: Using async/await for potentially long-running operations
> - **Domain Entities**: The service returns domain entities, which will be mapped to view models in the controller
> - **Clean Separation**: This interface belongs in the Application layer as it's an abstraction over infrastructure

### Step 2: Implement the Catalog Service

**Introduction**: Now let's implement the Catalog service interface. This implementation will depend on our repository but provide a slightly higher-level abstraction for the controllers to use.

1. Create a directory for service implementations:

    ```bash
    mkdir -p src/MerchStore.Application/Services/Implementations
    ```

2. Create the catalog service implementation:

    > `src/MerchStore.Application/Services/Implementations/CatalogService.cs`

    ```csharp
    using MerchStore.Application.Services.Interfaces;
    using MerchStore.Domain.Entities;
    using MerchStore.Domain.Interfaces;

    namespace MerchStore.Application.Services.Implementations;

    /// <summary>
    /// Implementation of the catalog service.
    /// Acts as a facade over the repository layer.
    /// </summary>
    public class CatalogService : ICatalogService
    {
        private readonly IProductRepository _productRepository;
        
        /// <summary>
        /// Constructor with dependency injection
        /// </summary>
        /// <param name="productRepository">The product repository</param>
        public CatalogService(IProductRepository productRepository)
        {
            _productRepository = productRepository;
        }
        
        /// <inheritdoc/>
        public async Task<IEnumerable<Product>> GetAllProductsAsync()
        {
            return await _productRepository.GetAllAsync();
        }
        
        /// <inheritdoc/>
        public async Task<Product?> GetProductByIdAsync(Guid id)
        {
            return await _productRepository.GetByIdAsync(id);
        }
    }
    ```

> 💡 **Information**
>
> - **Dependency Injection**: The service takes the repository as a constructor parameter
> - **Thin Service Layer**: For now, the service simply forwards calls to the repository
> - **Future Extensibility**: As the application grows, you could add business logic here (such as filtering, caching, etc.)
> - **Repository Abstraction**: The service depends on the repository interface from the Domain layer, not the implementation

### Step 3: Register the Service in DI Container

**Introduction**: To make our service available throughout the application, we need to register it with the Dependency Injection container. We'll create an extension method in the Application project to keep this configuration organized.

1. Install the Nuget package for dependency injection extensions:

    ```bash
    dotnet add /Users/lasse/Developer/CLO_Development/CLO24/CommonProject/MerchStoreDemo/src/MerchStore.Application/ package Microsoft.Extensions.DependencyInjection
    ```

2. Create a DependencyInjection class in the Application project:

    > `src/MerchStore.Application/DependencyInjection.cs`

    ```csharp
    using Microsoft.Extensions.DependencyInjection;
    using MerchStore.Application.Services.Implementations;
    using MerchStore.Application.Services.Interfaces;

    namespace MerchStore.Application;

    /// <summary>
    /// Contains extension methods for registering Application layer services with the dependency injection container.
    /// </summary>
    public static class DependencyInjection
    {
        /// <summary>
        /// Adds Application layer services to the DI container
        /// </summary>
        /// <param name="services">The service collection to add services to</param>
        /// <returns>The service collection for chaining</returns>
        public static IServiceCollection AddApplication(this IServiceCollection services)
        {
            // Register application services
            services.AddScoped<ICatalogService, CatalogService>();
            
            return services;
        }
    }
    ```

> 💡 **Information**
>
> - **Extension Method**: Using an extension method makes the code more readable in Program.cs
> - **Scoped Lifetime**: Services are created once per HTTP request
> - **Service Collection Chaining**: Returning the service collection allows method chaining in Program.cs
> - **Organization**: Keeping registration code in the same project as the services helps with maintainability

### Step 4: Create View Models for Catalogs

**Introduction**: View models are specialized models designed for the presentation layer. They contain only the data and formatting needed by the views, decoupling the views from the domain model.

1. Create a directory for catalog-related view models:

    ```bash
    mkdir -p src/MerchStore.WebUI/Models/Catalog
    ```

2. Create a ProductCardViewModel for displaying products in a list:

    > `src/MerchStore.WebUI/Models/Catalog/ProductCardViewModel.cs`

    ```csharp
    namespace MerchStore.WebUI.Models.Catalog;

    public class ProductCardViewModel
    {
        public Guid Id { get; set; }
        public string Name { get; set; } = string.Empty;
        public string TruncatedDescription { get; set; } = string.Empty;
        public string FormattedPrice { get; set; } = string.Empty;
        public decimal PriceAmount { get; set; }
        public string? ImageUrl { get; set; }
        public bool HasImage => !string.IsNullOrEmpty(ImageUrl);
        public bool InStock => StockQuantity > 0;
        public int StockQuantity { get; set; }
    }
    ```

3. Create a ProductCatalogViewModel as a container for the product list:

    > `src/MerchStore.WebUI/Models/Catalog/ProductCatalogViewModel.cs`

    ```csharp
    namespace MerchStore.WebUI.Models.Catalog;

    public class ProductCatalogViewModel
    {
        public List<ProductCardViewModel> FeaturedProducts { get; set; } = new List<ProductCardViewModel>();
    }
    ```

4. Create a ProductDetailsViewModel for displaying detailed product information:

    > `src/MerchStore.WebUI/Models/Catalog/ProductDetailsViewModel.cs`

    ```csharp
    namespace MerchStore.WebUI.Models.Catalog;

    public class ProductDetailsViewModel
    {
        public Guid Id { get; set; }
        public string Name { get; set; } = string.Empty;
        public string Description { get; set; } = string.Empty;
        public string FormattedPrice { get; set; } = string.Empty;
        public decimal PriceAmount { get; set; }
        public string? ImageUrl { get; set; }
        public bool HasImage => !string.IsNullOrEmpty(ImageUrl);
        public bool InStock => StockQuantity > 0;
        public int StockQuantity { get; set; }
    }
    ```

> 💡 **Information**
>
> - **View-Specific Properties**: View models include computed properties like `HasImage` and `InStock` to simplify view logic
> - **Presentation Logic**: Formatting display values like `FormattedPrice` and `TruncatedDescription` keeps the views cleaner
> - **Separation of Concerns**: View models prevent domain entities from being exposed directly to views
> - **Data Transfer**: View models only include the data needed for their specific view
>
> ⚠️ **Common Mistakes**
>
> - Using domain entities directly in views, which creates tight coupling
> - Not including presentation-focused properties in view models, which pushes formatting logic into views
> - Creating a single view model for multiple views with different needs

### Step 5: Create the Catalog Controller

**Introduction**: The controller ties together our service layer and our views, handling HTTP requests and coordinating the flow of data between the application and presentation layers.

1. Create a CatalogController to handle product display functionality:

    > `src/MerchStore.WebUI/Controllers/CatalogController.cs`

    ```csharp
    using Microsoft.AspNetCore.Mvc;
    using MerchStore.Application.Services.Interfaces;
    using MerchStore.WebUI.Models.Catalog;

    namespace MerchStore.WebUI.Controllers;

    public class CatalogController : Controller
    {
        private readonly ICatalogService _catalogService;
        
        public CatalogController(ICatalogService catalogService)
        {
            _catalogService = catalogService;
        }
        
        // GET: Catalog
        public async Task<IActionResult> Index()
        {
            try
            {
                // Get all products from the service
                var products = await _catalogService.GetAllProductsAsync();
                
                // Map domain entities to view models
                var productViewModels = products.Select(p => new ProductCardViewModel
                {
                    Id = p.Id,
                    Name = p.Name,
                    TruncatedDescription = p.Description.Length > 100 
                        ? p.Description.Substring(0, 97) + "..." 
                        : p.Description,
                    FormattedPrice = p.Price.ToString(),
                    PriceAmount = p.Price.Amount,
                    ImageUrl = p.ImageUrl?.ToString(),
                    StockQuantity = p.StockQuantity
                }).ToList();
                
                // Create the product catalog view model
                var viewModel = new ProductCatalogViewModel
                {
                    FeaturedProducts = productViewModels
                };
                
                return View(viewModel);
            }
            catch (Exception ex)
            {
                // Log the exception
                // In a real application, you should use a proper logging framework
                Console.WriteLine($"Error in ProductCatalog: {ex.Message}");
                
                // Show an error message to the user
                ViewBag.ErrorMessage = "An error occurred while loading products. Please try again later.";
                return View("Error");
            }
        }
        
        // GET: Store/Details/5
        public async Task<IActionResult> Details(Guid id)
        {
            try
            {
                // Get the specific product from the service
                var product = await _catalogService.GetProductByIdAsync(id);
                
                // Return 404 if product not found
                if (product is null)
                {
                    return NotFound();
                }
                
                // Map domain entity to view model
                var viewModel = new ProductDetailsViewModel
                {
                    Id = product.Id,
                    Name = product.Name,
                    Description = product.Description,
                    FormattedPrice = product.Price.ToString(),
                    PriceAmount = product.Price.Amount,
                    ImageUrl = product.ImageUrl?.ToString(),
                    StockQuantity = product.StockQuantity
                };
                
                return View(viewModel);
            }
            catch (Exception ex)
            {
                // Log the exception
                Console.WriteLine($"Error in ProductDetails: {ex.Message}");
                
                // Show an error message to the user
                ViewBag.ErrorMessage = "An error occurred while loading the product. Please try again later.";
                return View("Error");
            }
        }
    }
    ```

> 💡 **Information**
>
> - **Controller Responsibilities**: Controllers handle HTTP requests, call services, map data to view models, and return views
> - **Dependency Injection**: The controller receives the product service via constructor injection
> - **Error Handling**: Try-catch blocks handle exceptions and return appropriate error views
> - **Action Methods**: Each public method corresponds to a different endpoint/view
> - **Manual Mapping**: For simplicity, we're manually mapping entities to view models
>
> ⚠️ **Common Mistakes**
>
> - Not handling exceptions, which can result in unhandled errors reaching the user
> - Putting too much business logic in controllers rather than the service layer
> - Missing model validation for inputs (not needed in this read-only example)

### Step 6: Create the Product Catalog View

**Introduction**: The product catalog view will display a grid of products, making it easy for users to browse the available merchandise. We'll use a simple and clean design focused on showcasing the products.

1. Create the directory for the view:

    ```bash
    mkdir -p src/MerchStore.WebUI/Views/Catalog
    ```

2. Create the Index view for the catalog:

    > `src/MerchStore.WebUI/Views/Catalog/Index.cshtml`

    ```html
    @model MerchStore.WebUI.Models.Catalog.ProductCatalogViewModel
    
    @{
        ViewData["Title"] = "MerchStore - Products";
    }
    
    <style>
        .product-image {
            object-fit: cover;
            height: 200px;
            width: 100%;
        }
    </style>
    
    <div class="text-center">
        <h1 class="display-4 mb-4">Product Catalog</h1>
        <p class="lead mb-5">Browse our awesome merchandise below!</p>
    </div>
    
    @if (Model.FeaturedProducts.Any())
    {
        <div class="row row-cols-1 row-cols-md-2 row-cols-lg-3 g-4">
            @foreach (var product in Model.FeaturedProducts)
            {
                <div class="col">
                    <div class="card h-100 shadow-sm">
                        @if (product.HasImage)
                        {
                            <img src="@product.ImageUrl" class="card-img-top product-image" alt="@product.Name">
                        }
                        else
                        {
                            <div class="card-img-top bg-light text-center p-5">
                                <span class="text-muted">No image available</span>
                            </div>
                        }
                        <div class="card-body">
                            <h5 class="card-title">@product.Name</h5>
                            <p class="card-text">@product.TruncatedDescription</p>
                        </div>
                        <div class="card-footer bg-white d-flex justify-content-between align-items-center">
                            <span class="text-primary fw-bold">@product.FormattedPrice</span>
                            <div>
                                @if (product.InStock)
                                {
                                    <span class="badge bg-success me-2">In Stock</span>
                                }
                                else
                                {
                                    <span class="badge bg-danger me-2">Out of Stock</span>
                                }
                                <a asp-action="Details" asp-route-id="@product.Id" class="btn btn-outline-primary btn-sm">
                                    View Details
                                </a>
                            </div>
                        </div>
                    </div>
                </div>
            }
        </div>
    }
    else
    {
        <div class="alert alert-info text-center">
            <h2>No products available</h2>
            <p>Check back soon for our latest merchandise!</p>
        </div>
    }
    ```

> 💡 **Information**
>
> - **Bootstrap Grid**: Using Bootstrap's responsive grid system to display products
> - **Card Component**: Each product is displayed in a card with image, title, description, and price
> - **Conditional Rendering**: Different content is shown if there are no products or no image
> - **CSS Section**: Additional styles are added in a section that can be placed in the layout file
> - **Tag Helpers**: Using ASP.NET Core tag helpers for generating links to the Details action

### Step 7: Create the Product Details View

**Introduction**: The product details view will display more detailed information about a single product. This view is displayed when a user clicks on a product in the catalog.

1. Create the Details view:

    > `src/MerchStore.WebUI/Views/Catalog/Details.cshtml`

    ```html
    @model MerchStore.WebUI.Models.Catalog.ProductDetailsViewModel
    
    @{
        ViewData["Title"] = $"MerchStore - {Model.Name}";
    }
    
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.10.5/font/bootstrap-icons.css">
    <style>
        .product-detail-image {
            max-height: 500px;
            object-fit: cover;
            width: 100%;
        }
    </style>
    
    <nav aria-label="breadcrumb" class="mt-3">
        <ol class="breadcrumb">
            <li class="breadcrumb-item"><a asp-controller="Home" asp-action="Index">Home</a></li>
            <li class="breadcrumb-item"><a asp-controller="Catalog" asp-action="Index">Products</a></li>
            <li class="breadcrumb-item active" aria-current="page">@Model.Name</li>
        </ol>
    </nav>
    
    <div class="row mt-4">
        <div class="col-md-6">
            @if (Model.HasImage)
            {
                <img src="@Model.ImageUrl" class="img-fluid product-detail-image rounded shadow" alt="@Model.Name">
            }
            else
            {
                <div class="bg-light text-center p-5 rounded shadow">
                    <h3 class="text-muted">No image available</h3>
                </div>
            }
        </div>
        <div class="col-md-6">
            <h1 class="mb-3">@Model.Name</h1>
            <h4 class="text-primary mb-4">@Model.FormattedPrice</h4>
    
            @if (Model.InStock)
            {
                <div class="alert alert-success d-inline-block">
                    <i class="bi bi-check-circle"></i> In Stock (@Model.StockQuantity available)
                </div>
            }
            else
            {
                <div class="alert alert-danger d-inline-block">
                    <i class="bi bi-x-circle"></i> Out of Stock
                </div>
            }
    
            <div class="mt-4">
                <h5>Description</h5>
                <p class="lead">@Model.Description</p>
            </div>
    
            <div class="mt-4">
                <a asp-action="Index" class="btn btn-outline-secondary">
                    <i class="bi bi-arrow-left"></i> Back to Products
                </a>
            </div>
        </div>
    </div>
    ```

> 💡 **Information**
>
> - **Breadcrumb Navigation**: Helps users understand where they are in the site hierarchy
> - **Two-Column Layout**: Image on the left, product details on the right
> - **Conditional Display**: Different UI for in-stock vs. out-of-stock products
> - **Back Button**: Easy navigation back to the products list
> - **Bootstrap Icons**: Using Bootstrap Icons for visual indicators

### Step 8: Update the Program.cs File

**Introduction**: Finally, we need to update the Program.cs file to register our services with the dependency injection container and configure our application.

1. Update the Program.cs file:

    > `src/MerchStore.WebUI/Program.cs`

    ```csharp
    using MerchStore.Application;
    using MerchStore.Infrastructure;
    
    var builder = WebApplication.CreateBuilder(args);
    
    // Add services to the container.
    builder.Services.AddControllersWithViews();
    
    // Add Application services - this includes Services, Interfaces, etc.
    builder.Services.AddApplication();
    
    // Add Infrastructure services - this includes DbContext, Repositories, etc.
    builder.Services.AddInfrastructure(builder.Configuration);
    
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
    app.UseRouting();
    
    app.UseAuthorization();
    
    app.MapStaticAssets();
    
    app.MapControllerRoute(
        name: "default",
        pattern: "{controller=Home}/{action=Index}/{id?}")
        .WithStaticAssets();
    
    
    app.Run();
    ```

> 💡 **Information**
>
> - **Service Registration**: Adding Application and Infrastructure services to the DI container
> - **Database Seeding**: Seeding the database in development mode
> - **Middleware Pipeline**: Configuring the HTTP request processing pipeline
> - **Route Registration**: Defining the default route pattern

### Step 9: Add a Link to the Product Catalog

**Introduction**: To make our product catalog accessible, we'll add a link to it in the main navigation menu.

1. Update the _Layout.cshtml file:

    > `src/MerchStore.WebUI/Views/Shared/_Layout.cshtml`

    ```html
    <!-- Find the navigation section that looks something like this: -->
    <div class="navbar-collapse collapse d-sm-inline-flex justify-content-between">
        <ul class="navbar-nav flex-grow-1">
            <li class="nav-item">
                <a class="nav-link text-dark" asp-area="" asp-controller="Home" asp-action="Index">Home</a>
            </li>
            <!-- Add this new item below the Home link -->
            <li class="nav-item">
                <a class="nav-link text-dark" asp-area="" asp-controller="Catalog" asp-action="Index">Products</a>
            </li>
            <li class="nav-item">
                <a class="nav-link text-dark" asp-area="" asp-controller="Home" asp-action="Privacy">Privacy</a>
            </li>
        </ul>
    </div>
    ```

> 💡 **Information**
>
> - **Navigation Link**: Adding a link to the Products page in the main navigation
> - **Tag Helpers**: Using ASP.NET Core tag helpers to generate the URL
> - **No Area**: The Store controller is in the main area, not in an admin area

## 🧪 Final Tests

### Run the Application and Validate Your Work

1. Build and run the application:

   ```bash
   dotnet build
   dotnet run --project src/MerchStore.WebUI
   ```

2. Open a browser and navigate to the homepage.

3. Click on the "Products" link in the navigation menu.

4. Verify that the catalog displays the sample products.

5. Click on a product to view its details.

✅ **Expected Results**

- The Products catalog page shows a grid of products
- Each product card displays name, price, description, and availability
- Clicking on a product takes you to a detailed view
- The breadcrumb navigation helps you navigate back to the product catalog
- The layout is responsive and looks good on different screen sizes

## 🔧 Troubleshooting

If you encounter issues:

- Check that your service registrations are correctly set up in Program.cs
- Verify that the database seeder is running and adding products
- Make sure all your view models match the properties used in the views
- Check for correct namespace imports in all files
- Look for any errors in the browser developer console or server logs

## 🚀 Optional Challenge

Want to take your learning further? Try:

- Adding search functionality to filter products by name
- Implementing product categories and filtering by category
- Adding sorting options (by price, name, etc.)
- Creating a simple shopping cart using session storage
- Improving the UI with additional styling and animations

## 📚 Further Reading

- [ASP.NET Core MVC Documentation](https://docs.microsoft.com/en-us/aspnet/core/mvc/overview) - Microsoft's documentation on ASP.NET Core MVC
- [Introduction to Razor Pages](https://docs.microsoft.com/en-us/aspnet/core/razor-pages/) - An alternative approach to building web UIs in ASP.NET Core
- [Bootstrap Documentation](https://getbootstrap.com/docs/) - Documentation for the Bootstrap CSS framework
- [Clean Architecture in ASP.NET Core](https://docs.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures) - Microsoft's guidance on application architecture

## Done! 🎉

Great job! You've successfully created a simple product catalog for your merchandise application. This gives you a quick visual result of your work and demonstrates how all the layers work together, from the database all the way to the user interface.

While this implementation uses a simpler service-based approach instead of CQRS with MediatR, it still maintains the clean architecture principles by keeping your domain entities separate from your presentation layer and using services as an abstraction over your repositories.

The naming convention used in this exercise (using "Catalog" instead of "Store") ensures that this implementation will not interfere with the real store implementation you'll build later in the course.

In future exercises, we'll expand on this foundation to add more advanced features and eventually transition to the CQRS pattern for a more scalable architecture. 🚀
