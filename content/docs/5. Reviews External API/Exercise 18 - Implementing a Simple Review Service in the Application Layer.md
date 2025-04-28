---
title: "Implementing a Simple Review Service in the Application Layer"
date: "2025-04-28"
lastmod: "2025-04-28"
draft: false
weight: 505
toc: true
---

## 🎯 Goal

Create a simple Review Service in the application layer that serves as a bridge between your domain logic and infrastructure components, allowing your application to interact with product reviews in a clean, decoupled way.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 14 (Implementing the Domain Layer for Reviews)
- Have completed Exercise 16 (Implementing the Infrastructure Layer for the External Review Service)
- Have completed Exercise 17 (Integration Testing the External Review Service)
- Understand the principles of Clean Architecture and the role of the application layer
- Be familiar with the service pattern in application design

## 📚 Learning Objectives

By the end of this exercise, you will:

- Create an **application service interface** for review operations
- Implement a **review service** that follows clean architecture principles
- Add the service to the **dependency injection container**
- Create a simple **controller** that uses the review service
- Understand the **separation of concerns** between layers
- Learn how to **map domain objects** to presentation models

## 🔍 Why This Matters

In real-world applications, a well-designed application layer is crucial because:

- It provides a clean API for the presentation layer without exposing domain complexities
- It isolates the presentation layer from changes in the infrastructure layer
- It ensures that domain logic stays in the domain layer where it belongs
- It makes your application more testable by allowing easy mocking of services
- It enables you to swap out infrastructure components without affecting the rest of the application
- It serves as a translation layer between pure domain concepts and presentation needs

## 📝 Step-by-Step Instructions

### Step 1: Create the Review Service Interface

**Introduction**: First, we'll define an interface for our review service in the application layer. This interface will define the operations our service will provide to the presentation layer.

1. Make sure the `IReviewService` interface exists in the Application layer:

    > `src/MerchStore.Application/Services/Interfaces/IReviewService.cs`

    ```csharp
    using MerchStore.Domain.Entities;

    namespace MerchStore.Application.Services.Interfaces;

    /// <summary>
    /// Service interface for Review-related operations.
    /// Provides a simple abstraction over the repository layer.
    /// </summary>
    public interface IReviewService
    {
        /// <summary>
        /// Gets all reviews for a specific product
        /// </summary>
        /// <param name="productId">The product ID</param>
        /// <returns>A collection of reviews for the specified product</returns>
        Task<IEnumerable<Review>> GetReviewsByProductIdAsync(Guid productId);
        
        /// <summary>
        /// Gets the average rating for a specific product
        /// </summary>
        /// <param name="productId">The product ID</param>
        /// <returns>The average rating as a double</returns>
        Task<double> GetAverageRatingForProductAsync(Guid productId);
        
        /// <summary>
        /// Gets the total number of reviews for a specific product
        /// </summary>
        /// <param name="productId">The product ID</param>
        /// <returns>The count of reviews for the specified product</returns>
        Task<int> GetReviewCountForProductAsync(Guid productId);
        
        /// <summary>
        /// Adds a new review for a product
        /// </summary>
        /// <param name="review">The review to add</param>
        /// <returns>A task representing the asynchronous operation</returns>
        Task AddReviewAsync(Review review);
    }
    ```

> 💡 **Information**
>
> - **Application Layer Interface**: This interface belongs in the application layer, not the domain layer
> - **Service Pattern**: The service pattern provides a clear API that the presentation layer can use
> - **Abstraction**: The interface abstracts away the details of how reviews are retrieved or stored
> - **Asynchronous Methods**: All methods are asynchronous for better scalability
>
> ⚠️ **Common Mistakes**
>
> - Putting too much domain logic in the service interface
> - Having overly specific methods tied to infrastructure details
> - Missing proper documentation on what each method does
> - Not using asynchronous methods for potentially long-running operations

### Step 2: Implement the Review Service

**Introduction**: Now we'll implement the service interface we just defined. This implementation will use our review repository to interact with the external review service.

1. Create the implementation of the Review Service:

    > `src/MerchStore.Application/Services/Implementations/ReviewService.cs`

    ```csharp
    using MerchStore.Application.Services.Interfaces;
    using MerchStore.Domain.Entities;
    using MerchStore.Domain.Interfaces;

    namespace MerchStore.Application.Services.Implementations;

    /// <summary>
    /// Implementation of the review service.
    /// Acts as a facade over the repository layer.
    /// </summary>
    public class ReviewService : IReviewService
    {
        private readonly IReviewRepository _reviewRepository;
        
        /// <summary>
        /// Constructor with dependency injection
        /// </summary>
        /// <param name="reviewRepository">The review repository</param>
        public ReviewService(IReviewRepository reviewRepository)
        {
            _reviewRepository = reviewRepository;
        }
        
        /// <inheritdoc/>
        public async Task<IEnumerable<Review>> GetReviewsByProductIdAsync(Guid productId)
        {
            var (reviews, _) = await _reviewRepository.GetProductReviewsAsync(productId);
            return reviews;
        }
        
        /// <inheritdoc/>
        public async Task<double> GetAverageRatingForProductAsync(Guid productId)
        {
            var (_, stats) = await _reviewRepository.GetProductReviewsAsync(productId);
            return stats.AverageRating;
        }
        
        /// <inheritdoc/>
        public async Task<int> GetReviewCountForProductAsync(Guid productId)
        {
            var (_, stats) = await _reviewRepository.GetProductReviewsAsync(productId);
            return stats.ReviewCount;
        }
        
        /// <inheritdoc/>
        public Task AddReviewAsync(Review review)
        {
            // Since the IReviewRepository doesn't have an AddAsync method,
            // we need to figure out how to handle this case
            throw new NotImplementedException("Adding reviews is not supported by the current repository interface.");
        }
    }
    ```

> 💡 **Information**
>
> - **Repository Dependency**: The service depends on the repository interface, not its implementation
> - **Tuple Deconstruction**: We use C# tuple deconstruction to get specific parts of the repository response
> - **Not Implemented Method**: We correctly throw NotImplementedException for unimplemented features
> - **InheritDoc**: Using XML documentation inheritance for cleaner code
>
> ⚠️ **Common Mistakes**
>
> - Duplicating repository logic in the service
> - Not handling potential exceptions from the repository
> - Forgetting to properly inject dependencies
> - Silently failing when a method can't be implemented

### Step 3: Register the Service in the DI Container

**Introduction**: To make our service available to the rest of the application, we need to register it with the dependency injection container. We'll update the `DependencyInjection.cs` file in the Application project.

1. Update the DependencyInjection class to register the Review Service:

    > `src/MerchStore.Application/DependencyInjection.cs`

    ```csharp
    using System.Reflection;
    using AutoMapper;
    using FluentValidation;
    using MediatR;
    using Microsoft.Extensions.DependencyInjection;
    using MerchStore.Application.Common.Behaviors;
    using MerchStore.Application.Services.Interfaces;
    using MerchStore.Application.Services.Implementations;

    namespace MerchStore.Application;

    public static class DependencyInjection
    {
        public static IServiceCollection AddApplication(this IServiceCollection services)
        {
            // Register application services
            services.AddScoped<ICatalogService, CatalogService>();
            services.AddScoped<IReviewService, ReviewService>(); // Add this line
            
            return services;
        }
    }
    ```

> 💡 **Information**
>
> - **Service Registration**: Services are registered in the dependency injection container
> - **Service Lifetime**: Using scoped lifetime for services (one instance per HTTP request)
> - **Extension Method Pattern**: Using a clean extension method pattern for registration
> - **Assembly Scanning**: Other components like validators and mappers are registered using assembly scanning
>
> ⚠️ **Common Mistakes**
>
> - Using the wrong service lifetime (singleton vs. scoped vs. transient)
> - Forgetting to register the service in the DI container
> - Registering the service in the wrong layer (should be in Application, not Infrastructure)
> - Not maintaining a consistent pattern for service registration

### Step 4: Create Models for the Presentation Layer

**Introduction**: Before we create a controller that uses our service, we need to define view models that will be used by the presentation layer. These models will contain only the data needed for the views.

1. Create the view models for product reviews:

    > `src/MerchStore.WebUI/Models/ProductReviewViewModel.cs`

    ```csharp
    using MerchStore.Domain.Entities;

    namespace MerchStore.WebUI.Models;

    public class ProductReviewViewModel
    {
        public Product Product { get; set; } = null!;
        public List<Review> Reviews { get; set; } = new List<Review>();
        public double AverageRating { get; set; }
        public int ReviewCount { get; set; }
    }
    ```

2. Create a view model for listing products with their review statistics:

    > `src/MerchStore.WebUI/Models/ProductReviewsViewModel.cs`

    ```csharp
    using MerchStore.Domain.Entities;

    namespace MerchStore.WebUI.Models;

    public class ProductReviewsViewModel
    {
        public List<Product> Products { get; set; } = new List<Product>();
        public Dictionary<Guid, IEnumerable<Review>> ProductReviews { get; set; } = new Dictionary<Guid, IEnumerable<Review>>();
        public Dictionary<Guid, double> AverageRatings { get; set; } = new Dictionary<Guid, double>();
        public Dictionary<Guid, int> ReviewCounts { get; set; } = new Dictionary<Guid, int>();
    }
    ```

> 💡 **Information**
>
> - **View Models**: These models are specifically designed for the presentation layer
> - **Null Reference Protection**: Using null! notation for required references with initialization
> - **Default Initialization**: Collections are initialized with empty collections to avoid null reference exceptions
> - **Dictionary Pattern**: Using dictionaries to efficiently map product IDs to their review data
>
> ⚠️ **Common Mistakes**
>
> - Using domain entities directly in views (breaks separation of concerns)
> - Not initializing collections, which can lead to null reference exceptions
> - Creating overly complex view models with unnecessary properties
> - Mixing presentation concerns with domain concerns

### Step 5: Create a Controller for Reviews

**Introduction**: Now let's create a controller that uses our review service to display product reviews to users. This controller will handle HTTP requests related to reviews.

1. Create the Reviews Controller:

    > `src/MerchStore.WebUI/Controllers/ReviewsController.cs`

    ```csharp
    using Microsoft.AspNetCore.Mvc;
    using MerchStore.Application.Services.Interfaces;

    namespace MerchStore.WebUI.Controllers;

    public class ReviewsController : Controller
    {
        private readonly IReviewService _reviewService;
        private readonly ICatalogService _catalogService;

        public ReviewsController(IReviewService reviewService, ICatalogService catalogService)
        {
            _reviewService = reviewService;
            _catalogService = catalogService;
        }

        // GET: Reviews
        public async Task<IActionResult> Index()
        {
            try
            {
                // Get all products
                var products = await _catalogService.GetAllProductsAsync();
                var viewModel = new ProductReviewsViewModel
                {
                    Products = products.ToList()
                };

                // For each product, get its reviews and calculate the average rating
                foreach (var product in viewModel.Products)
                {
                    viewModel.ProductReviews[product.Id] = await _reviewService.GetReviewsByProductIdAsync(product.Id);
                    viewModel.AverageRatings[product.Id] = await _reviewService.GetAverageRatingForProductAsync(product.Id);
                    viewModel.ReviewCounts[product.Id] = await _reviewService.GetReviewCountForProductAsync(product.Id);
                }

                return View(viewModel);
            }
            catch (Exception ex)
            {
                // Log the error and return an error view
                TempData["ErrorMessage"] = $"Error fetching reviews: {ex.Message}";
                return View("Error");
            }
        }

        // GET: Reviews/Product/{id}
        public async Task<IActionResult> Product(Guid id)
        {
            try
            {
                // Get the product by ID
                var product = await _catalogService.GetProductByIdAsync(id);
                
                if (product is null)
                {
                    return NotFound();
                }

                // Get reviews for the product
                var reviews = await _reviewService.GetReviewsByProductIdAsync(id);
                var averageRating = await _reviewService.GetAverageRatingForProductAsync(id);
                var reviewCount = await _reviewService.GetReviewCountForProductAsync(id);

                var viewModel = new ProductReviewViewModel
                {
                    Product = product,
                    Reviews = reviews.ToList(),
                    AverageRating = averageRating,
                    ReviewCount = reviewCount
                };

                return View(viewModel);
            }
            catch (Exception ex)
            {
                // Log the error and return an error view
                TempData["ErrorMessage"] = $"Error fetching product reviews: {ex.Message}";
                return View("Error");
            }
        }
    }
    ```

> 💡 **Information**
>
> - **Controller Structure**: A standard MVC controller with dependency injection
> - **Service Dependencies**: The controller depends on interfaces, not implementations
> - **Error Handling**: Comprehensive try/catch blocks for robust error handling
> - **Clean Separation**: The controller only handles HTTP concerns, delegating business logic to services
> - **NotFound Results**: Proper HTTP status codes are returned for missing resources
>
> ⚠️ **Common Mistakes**
>
> - Putting business logic in controllers
> - Not handling exceptions properly
> - Making multiple unnecessary service calls
> - Returning inappropriate HTTP status codes

### Step 6: Create Views for Reviews

**Introduction**: Finally, let's create the views that will display the review data to users. For this exercise, we'll create simple views to show the product reviews.

1. Create the Index view for showing all products with their review stats:

    > `src/MerchStore.WebUI/Views/Reviews/Index.cshtml`

    ```html
    @model MerchStore.WebUI.Models.ProductReviewsViewModel

    @{
        ViewData["Title"] = "Product Reviews";
    }

    <div class="container mt-4">
        <h1 class="mb-4">Product Reviews</h1>
        
        @if (!Model.Products.Any())
        {
            <div class="alert alert-info">
                No products available.
            </div>
        }
        else
        {
            <div class="row">
                @foreach (var product in Model.Products)
                {
                    <div class="col-md-6 col-lg-4 mb-4">
                        <div class="card h-100">
                            @if (product.ImageUrl != null)
                            {
                                <img src="@product.ImageUrl" class="card-img-top" alt="@product.Name" style="height: 200px; object-fit: cover;">
                            }
                            else
                            {
                                <div class="card-img-top bg-light d-flex align-items-center justify-content-center" style="height: 200px;">
                                    <span class="text-muted">No image</span>
                                </div>
                            }
                            <div class="card-body">
                                <h5 class="card-title">@product.Name</h5>
                                <p class="card-text">@(product.Description.Length > 100 ? product.Description.Substring(0, 100) + "..." : product.Description)</p>
                                
                                @if (Model.ReviewCounts.TryGetValue(product.Id, out var count) && count > 0)
                                {
                                    <div class="mb-2">
                                        @if (Model.AverageRatings.TryGetValue(product.Id, out var rating))
                                        {
                                            <div class="mb-1">
                                                @for (int i = 1; i <= 5; i++)
                                                {
                                                    if (i <= Math.Floor(rating))
                                                    {
                                                        <i class="bi bi-star-fill text-warning"></i>
                                                    }
                                                    else if (i <= Math.Ceiling(rating) && i > Math.Floor(rating))
                                                    {
                                                        <i class="bi bi-star-half text-warning"></i>
                                                    }
                                                    else
                                                    {
                                                        <i class="bi bi-star text-warning"></i>
                                                    }
                                                }
                                                <span class="ms-1">@rating.ToString("F1")</span>
                                            </div>
                                            <small class="text-muted">@count @(count == 1 ? "review" : "reviews")</small>
                                        }
                                    </div>
                                }
                                else
                                {
                                    <p class="text-muted mb-2">No reviews yet</p>
                                }
                            </div>
                            <div class="card-footer bg-white">
                                <a asp-action="Product" asp-route-id="@product.Id" class="btn btn-outline-primary">View Reviews</a>
                            </div>
                        </div>
                    </div>
                }
            </div>
        }
    </div>
    ```

2. Create the Product view for showing a single product with its reviews:

    > `src/MerchStore.WebUI/Views/Reviews/Product.cshtml`

    ```html
    @model MerchStore.WebUI.Models.ProductReviewViewModel

    @{
        ViewData["Title"] = $"Reviews for {Model.Product.Name}";
    }

    <div class="container mt-4">
        <nav aria-label="breadcrumb">
            <ol class="breadcrumb">
                <li class="breadcrumb-item"><a asp-controller="Home" asp-action="Index">Home</a></li>
                <li class="breadcrumb-item"><a asp-controller="Reviews" asp-action="Index">Reviews</a></li>
                <li class="breadcrumb-item active" aria-current="page">@Model.Product.Name</li>
            </ol>
        </nav>
        
        <div class="row mt-4">
            <div class="col-md-4">
                <div class="card mb-4">
                    @if (Model.Product.ImageUrl != null)
                    {
                        <img src="@Model.Product.ImageUrl" class="card-img-top" alt="@Model.Product.Name">
                    }
                    else
                    {
                        <div class="card-img-top bg-light d-flex align-items-center justify-content-center" style="height: 200px;">
                            <span class="text-muted">No image</span>
                        </div>
                    }
                    <div class="card-body">
                        <h5 class="card-title">@Model.Product.Name</h5>
                        <p class="card-text">@Model.Product.Description</p>
                        <p class="card-text"><strong>Price:</strong> @Model.Product.Price</p>
                        <p class="card-text">
                            <strong>Stock:</strong>
                            @if (Model.Product.StockQuantity > 0)
                            {
                                <span class="text-success">In Stock (@Model.Product.StockQuantity)</span>
                            }
                            else
                            {
                                <span class="text-danger">Out of Stock</span>
                            }
                        </p>
                    </div>
                </div>
            </div>
            
            <div class="col-md-8">
                <h2>Reviews</h2>
                
                <div class="mb-4">
                    <div class="d-flex align-items-center">
                        <div class="me-3">
                            <h1 class="display-4 mb-0">@Model.AverageRating.ToString("F1")</h1>
                            <div>
                                @for (int i = 1; i <= 5; i++)
                                {
                                    if (i <= Math.Floor(Model.AverageRating))
                                    {
                                        <i class="bi bi-star-fill text-warning"></i>
                                    }
                                    else if (i <= Math.Ceiling(Model.AverageRating) && i > Math.Floor(Model.AverageRating))
                                    {
                                        <i class="bi bi-star-half text-warning"></i>
                                    }
                                    else
                                    {
                                        <i class="bi bi-star text-warning"></i>
                                    }
                                }
                            </div>
                        </div>
                        <div>
                            <p class="mb-0">@Model.ReviewCount @(Model.ReviewCount == 1 ? "review" : "reviews")</p>
                        </div>
                    </div>
                </div>
                
                @if (!Model.Reviews.Any())
                {
                    <div class="alert alert-info">
                        No reviews yet. Be the first to review this product!
                    </div>
                }
                else
                {
                    @foreach (var review in Model.Reviews)
                    {
                        <div class="card mb-3">
                            <div class="card-body">
                                <div class="d-flex justify-content-between align-items-center mb-2">
                                    <h5 class="card-title mb-0">@review.Title</h5>
                                    <small class="text-muted">@review.CreatedAt.ToString("MMM dd, yyyy")</small>
                                </div>
                                <div class="mb-2">
                                    @for (int i = 1; i <= 5; i++)
                                    {
                                        if (i <= review.Rating)
                                        {
                                            <i class="bi bi-star-fill text-warning"></i>
                                        }
                                        else
                                        {
                                            <i class="bi bi-star text-warning"></i>
                                        }
                                    }
                                </div>
                                <p class="card-text">@review.Content</p>
                                <p class="card-text"><small class="text-muted">By @review.CustomerName</small></p>
                            </div>
                        </div>
                    }
                }
            </div>
        </div>
    </div>
    ```

3. Add Bootstrap Icons to your layout for the star ratings:

    > `src/MerchStore.WebUI/Views/Shared/_Layout.cshtml`

    Add this line to the head section:

    ```html
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.min.css">
    ```

> 💡 **Information**
>
> - **Bootstrap-based UI**: Using Bootstrap to create a responsive, modern UI
> - **Star Ratings**: Using Bootstrap Icons to display star ratings
> - **Conditional Display**: Different UI for products with and without reviews
> - **Error Handling**: Proper error messages when no products or reviews are available
> - **Breadcrumb Navigation**: Helping users understand where they are in the site hierarchy
>
> ⚠️ **Common Mistakes**
>
> - Not handling edge cases (no products, no reviews)
> - Missing responsive design considerations
> - Overcomplicating the view with unnecessary logic
> - Not providing proper navigation between views

### Step 7: Test Your Implementation

**Introduction**: Let's test our implementation to make sure everything is working correctly. This involves running the application and navigating to the review pages.

1. Build and run the application:

    ```bash
    dotnet build
    dotnet run --project src/MerchStore.WebUI
    ```

2. Open a browser and navigate to:

    ```sh
    https://localhost:7188/Reviews
    ```

3. Check that you can see the list of products with their review statistics.

4. Click on "View Reviews" for a product to see its detailed reviews.

✅ **Expected Results**

- The Reviews index page shows all products with their average ratings and review counts
- Clicking on a product shows its detailed information and all reviews
- Star ratings are displayed correctly based on the average rating
- The UI is responsive and works on different screen sizes
- Error handling is in place for edge cases

## 🔧 Troubleshooting

If you encounter issues:

- **Reviews Not Appearing**:
  - Check the browser console for JavaScript errors
  - Verify the review service is properly registered in the DI container
  - Check that the external review API is accessible

- **Star Ratings Missing**:
  - Make sure Bootstrap Icons is properly included in the layout
  - Verify the star rating calculation in the views

- **404 Not Found Errors**:
  - Check that controller and view names match exactly
  - Verify that the routes are correctly defined

- **Exception Thrown in Service**:
  - Check the implementation of the review repository
  - Verify that the external review API is properly configured

## 🚀 Optional Challenge

Want to take your learning further? Try:

1. **Add a Form to Submit Reviews**: Create a form to allow users to submit new reviews for products

2. **Add Pagination**: Implement pagination for products and reviews when there are many to display

3. **Add Filtering and Sorting**: Allow users to filter and sort reviews by rating, date, etc.

4. **Implement Caching**: Add caching to improve performance by reducing calls to the external review service

5. **Add Unit Tests**: Create comprehensive unit tests for the review service

## 📚 Further Reading

- [ASP.NET Core MVC Documentation](https://docs.microsoft.com/en-us/aspnet/core/mvc/overview) - Microsoft's guide to ASP.NET Core MVC
- [Bootstrap Documentation](https://getbootstrap.com/docs/) - Official Bootstrap documentation
- [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) - Uncle Bob's article on Clean Architecture
- [Service Pattern in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection) - Microsoft's guide to services and DI
- [Bootstrap Icons](https://icons.getbootstrap.com/) - Official Bootstrap Icons documentation

## Done! 🎉

Congratulations! You've successfully implemented a Review Service in the application layer that bridges your domain logic and infrastructure components. This service allows your application to interact with product reviews in a clean, decoupled way.

Your implementation follows the principles of Clean Architecture, with clear separation between layers and proper abstractions. The controller depends on service interfaces rather than concrete implementations, making your code more testable and maintainable.

The views provide a user-friendly interface for viewing product reviews, with star ratings and other visual elements to enhance the user experience. And all of this is achieved without tightly coupling your presentation layer to your domain or infrastructure layers.

In future exercises, we'll build on this foundation to add more advanced features and improve the user experience. 🚀
