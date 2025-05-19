---
title: "Introducing Areas and Preparing for CQRS"
date: "2025-05-19"
lastmod: "2025-05-19"
draft: false
weight: 801
toc: true
---

## 🎯 Goal

Create a dedicated Management Area for the application with placeholder views for product management, to prepare for implementing CQRS pattern in subsequent exercises.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 6 (Building a Simple Product Catalog)
- Understand basic ASP.NET Core MVC concepts
- Have a working product catalog with service-based architecture

## 📚 Learning Objectives

By the end of this exercise, you will:

- Implement **ASP.NET Core Areas** to separate different functional sections of your application
- Create a basic **Management Area** for product administration
- Set up **placeholder views** that will integrate with CQRS in future exercises
- Understand how to **organize your application** by functional responsibilities
- Learn how to **navigate between different areas** in your application

## 🔍 Why This Matters

In real-world applications, organizing code by functional areas is crucial because:

- It improves maintainability by grouping related functionality together
- It enables better separation of concerns between different parts of the application
- It provides a foundation for implementing more advanced patterns like CQRS
- It's a standard practice in professional ASP.NET Core development
- It makes your application more modular and easier to understand

## 📝 Step-by-Step Instructions

### Step 1: Create the Management Area Structure

**Introduction**: Areas in ASP.NET Core MVC allow you to separate parts of your application into isolated sections. We'll start by creating the folder structure for our Management area.

1. Create the Management area folder structure:

    ```bash
    mkdir -p src/MerchStore.WebUI/Areas/Management/Controllers
    mkdir -p src/MerchStore.WebUI/Areas/Management/Views/Products
    mkdir -p src/MerchStore.WebUI/Areas/Management/Views/Shared
    mkdir -p src/MerchStore.WebUI/Areas/Management/Models
    ```

2. Create the _ViewImports.cshtml file for the Management area:

    > `src/MerchStore.WebUI/Areas/Management/Views/_ViewImports.cshtml`

    ```cshtml
    @using MerchStore.WebUI
    @using MerchStore.WebUI.Models
    @using MerchStore.WebUI.Areas.Management.Models
    @addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
    ```

3. Create the _ViewStart.cshtml file for the Management area:

    > `src/MerchStore.WebUI/Areas/Management/Views/_ViewStart.cshtml`

    ```cshtml
    @{
        Layout = "_Layout";
    }
    ```

> 💡 **Information**
>
> - **Areas**: Areas in ASP.NET Core provide a way to organize related functionality into a separate namespace and folder structure
> - **ViewImports**: This file imports common namespaces for all views in the area
> - **ViewStart**: This file sets the default layout for all views in the area
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to create the _ViewImports and `_ViewStart` files, which results in broken views
> - Placing area files in the wrong folder structure, which may cause routing issues

### Step 2: Create a Home Controller for the Management Area

**Introduction**: Every area should have a home controller to handle the default route for that area. This gives users a landing page when they navigate to the area.

1. Create a basic HomeController for the Management area:

    > `src/MerchStore.WebUI/Areas/Management/Controllers/HomeController.cs`

    ```csharp
    using Microsoft.AspNetCore.Mvc;

    namespace MerchStore.WebUI.Areas.Management.Controllers;

    [Area("Management")]
    public class HomeController : Controller
    {
        public IActionResult Index()
        {
            return View();
        }
    }
    ```

2. Create the Index view for the Management Home controller:

    > `src/MerchStore.WebUI/Areas/Management/Views/Home/Index.cshtml`

    ```cshtml
    @{
        ViewData["Title"] = "Management Dashboard";
    }

    <div class="text-center">
        <h1 class="display-4">Management Dashboard</h1>
        <p class="lead">Welcome to the product management area.</p>
    </div>

    <div class="row mt-4">
        <div class="col-md-4">
            <div class="card mb-4">
                <div class="card-body">
                    <h5 class="card-title">Products</h5>
                    <p class="card-text">Manage product catalog items</p>
                    <a asp-area="Management" asp-controller="Products" asp-action="Index" class="btn btn-primary">
                        Manage Products
                    </a>
                </div>
            </div>
        </div>
        <div class="col-md-4">
            <div class="card mb-4">
                <div class="card-body">
                    <h5 class="card-title">Orders</h5>
                    <p class="card-text">View and manage customer orders</p>
                    <a href="#" class="btn btn-secondary">Coming Soon</a>
                </div>
            </div>
        </div>
        <div class="col-md-4">
            <div class="card mb-4">
                <div class="card-body">
                    <h5 class="card-title">Customers</h5>
                    <p class="card-text">Manage customer accounts and data</p>
                    <a href="#" class="btn btn-secondary">Coming Soon</a>
                </div>
            </div>
        </div>
    </div>
    ```

> 💡 **Information**
>
> - **Area Attribute**: The `[Area("Management")]` attribute is required on all controllers in an area
> - **Dashboard Design**: This simple dashboard provides navigation to different management functions
> - **Coming Soon**: We're only implementing Products management now, but the dashboard shows the potential for expansion

### Step 3: Create a Products Controller for the Management Area

**Introduction**: Now we'll create a ProductsController in the Management area to handle product administrative tasks. This controller will use the existing CatalogService but will be prepared for future CQRS implementation.

1. Create the ProductsController for the Management area:

    > `src/MerchStore.WebUI/Areas/Management/Controllers/ProductsController.cs`

    ```csharp
    using Microsoft.AspNetCore.Mvc;
    using MerchStore.Application.Services.Interfaces;
    using MerchStore.Domain.Entities;
    using MerchStore.WebUI.Areas.Management.Models;
    using System;
    using System.Linq;
    using System.Threading.Tasks;

    namespace MerchStore.WebUI.Areas.Management.Controllers;

    [Area("Management")]
    public class ProductsController : Controller
    {
        private readonly ICatalogService _catalogService;

        public ProductsController(ICatalogService catalogService)
        {
            _catalogService = catalogService;
        }

        // GET: Management/Products
        public async Task<IActionResult> Index()
        {
            try
            {
                var products = await _catalogService.GetAllProductsAsync();
                
                var viewModels = products.Select(p => new ProductViewModel
                {
                    Id = p.Id,
                    Name = p.Name,
                    Description = p.Description.Length > 100 
                        ? p.Description.Substring(0, 97) + "..." 
                        : p.Description,
                    Price = p.Price.Amount,
                    Currency = p.Price.Currency,
                    StockQuantity = p.StockQuantity,
                    ImageUrl = p.ImageUrl?.ToString()
                }).ToList();
                
                return View(viewModels);
            }
            catch (Exception ex)
            {
                // In a real application, log the exception
                ViewBag.ErrorMessage = $"Error loading products: {ex.Message}";
                return View("Error");
            }
        }

        // GET: Management/Products/Details/5
        public async Task<IActionResult> Details(Guid id)
        {
            try
            {
                var product = await _catalogService.GetProductByIdAsync(id);
                
                if (product == null)
                {
                    return NotFound();
                }
                
                var viewModel = new ProductViewModel
                {
                    Id = product.Id,
                    Name = product.Name,
                    Description = product.Description,
                    Price = product.Price.Amount,
                    Currency = product.Price.Currency,
                    StockQuantity = product.StockQuantity,
                    ImageUrl = product.ImageUrl?.ToString()
                };
                
                return View(viewModel);
            }
            catch (Exception ex)
            {
                ViewBag.ErrorMessage = $"Error loading product details: {ex.Message}";
                return View("Error");
            }
        }

        // GET: Management/Products/Create
        public IActionResult Create()
        {
            return View();
        }

        // GET: Management/Products/Edit/5
        public async Task<IActionResult> Edit(Guid id)
        {
            try
            {
                var product = await _catalogService.GetProductByIdAsync(id);
                
                if (product == null)
                {
                    return NotFound();
                }
                
                var viewModel = new ProductViewModel
                {
                    Id = product.Id,
                    Name = product.Name,
                    Description = product.Description,
                    Price = product.Price.Amount,
                    Currency = product.Price.Currency,
                    StockQuantity = product.StockQuantity,
                    ImageUrl = product.ImageUrl?.ToString()
                };
                
                return View(viewModel);
            }
            catch (Exception ex)
            {
                ViewBag.ErrorMessage = $"Error loading product for editing: {ex.Message}";
                return View("Error");
            }
        }

        // GET: Management/Products/Delete/5
        public async Task<IActionResult> Delete(Guid id)
        {
            try
            {
                var product = await _catalogService.GetProductByIdAsync(id);
                
                if (product == null)
                {
                    return NotFound();
                }
                
                var viewModel = new ProductViewModel
                {
                    Id = product.Id,
                    Name = product.Name,
                    Description = product.Description,
                    Price = product.Price.Amount,
                    Currency = product.Price.Currency,
                    StockQuantity = product.StockQuantity,
                    ImageUrl = product.ImageUrl?.ToString()
                };
                
                return View(viewModel);
            }
            catch (Exception ex)
            {
                ViewBag.ErrorMessage = $"Error loading product for deletion: {ex.Message}";
                return View("Error");
            }
        }
    }
    ```

> 💡 **Information**
>
> - **Read-Only Controller**: This controller only implements GET actions for now as placeholders
> - **Service-Based Architecture**: We're still using the CatalogService, but will replace it with CQRS in future exercises
> - **Error Handling**: Basic error handling is implemented to catch and display exceptions
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to add the [Area] attribute to the controller
> - Implementing too much functionality before establishing the pattern, which makes future refactoring harder

### Step 4: Create a ViewModel for Products

**Introduction**: View models help separate your domain models from your views. They contain only the data needed for a specific view and can include display formatting logic.

1. Create a ProductViewModel for the Management area:

    > `src/MerchStore.WebUI/Areas/Management/Models/ProductViewModel.cs`

    ```csharp
    using System;

    namespace MerchStore.WebUI.Areas.Management.Models;

    public class ProductViewModel
    {
        public Guid Id { get; set; }
        public string Name { get; set; } = string.Empty;
        public string Description { get; set; } = string.Empty;
        public decimal Price { get; set; }
        public string Currency { get; set; } = string.Empty;
        public int StockQuantity { get; set; }
        public string? ImageUrl { get; set; }
        
        // Helper properties for the view
        public string FormattedPrice => $"{Price} {Currency}";
        public bool HasImage => !string.IsNullOrEmpty(ImageUrl);
        public bool InStock => StockQuantity > 0;
    }
    ```

> 💡 **Information**
>
> - **View Models**: These are specific models tailored for view rendering needs
> - **Computed Properties**: The model includes computed properties like FormattedPrice for display purposes
> - **Future Compatibility**: This structure is designed to be easily adapted for AutoMapper in future exercises

### Step 5: Create Placeholder Views for Products

**Introduction**: Let's create the basic views for product management. These will be simple placeholders that we'll enhance in future exercises.

1. Create the Index view for Products:

    > `src/MerchStore.WebUI/Areas/Management/Views/Products/Index.cshtml`

    ```cshtml
    @model IEnumerable<MerchStore.WebUI.Areas.Management.Models.ProductViewModel>

    @{
        ViewData["Title"] = "Product Management";
    }

    <div class="d-flex justify-content-between align-items-center mb-4">
        <h1>Product Management</h1>
        <a asp-action="Create" class="btn btn-primary">
            <i class="bi bi-plus-circle"></i> Add New Product
        </a>
    </div>

    @if (!Model.Any())
    {
        <div class="alert alert-info">
            No products available. Click "Add New Product" to create one.
        </div>
    }
    else
    {
        <div class="table-responsive">
            <table class="table table-striped table-hover">
                <thead class="table-dark">
                    <tr>
                        <th>Name</th>
                        <th>Description</th>
                        <th>Price</th>
                        <th>Stock</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    @foreach (var item in Model)
                    {
                        <tr>
                            <td>@item.Name</td>
                            <td>@item.Description</td>
                            <td>@item.FormattedPrice</td>
                            <td>
                                @if (item.InStock)
                                {
                                    <span class="badge bg-success">@item.StockQuantity in stock</span>
                                }
                                else
                                {
                                    <span class="badge bg-danger">Out of stock</span>
                                }
                            </td>
                            <td>
                                <div class="btn-group" role="group">
                                    <a asp-action="Details" asp-route-id="@item.Id" class="btn btn-sm btn-info">
                                        <i class="bi bi-eye"></i>
                                    </a>
                                    <a asp-action="Edit" asp-route-id="@item.Id" class="btn btn-sm btn-primary">
                                        <i class="bi bi-pencil"></i>
                                    </a>
                                    <a asp-action="Delete" asp-route-id="@item.Id" class="btn btn-sm btn-danger">
                                        <i class="bi bi-trash"></i>
                                    </a>
                                </div>
                            </td>
                        </tr>
                    }
                </tbody>
            </table>
        </div>
    }
    ```

2. Create the Details view for Products:

    > `src/MerchStore.WebUI/Areas/Management/Views/Products/Details.cshtml`

    ```cshtml
    @model MerchStore.WebUI.Areas.Management.Models.ProductViewModel

    @{
        ViewData["Title"] = "Product Details";
    }

    <div class="d-flex justify-content-between align-items-center mb-4">
        <h1>Product Details</h1>
        <div>
            <a asp-action="Edit" asp-route-id="@Model.Id" class="btn btn-primary">Edit</a>
            <a asp-action="Index" class="btn btn-secondary">Back to List</a>
        </div>
    </div>

    <div class="row">
        <div class="col-md-4">
            @if (Model.HasImage)
            {
                <img src="@Model.ImageUrl" alt="@Model.Name" class="img-fluid rounded" />
            }
            else
            {
                <div class="bg-light rounded d-flex align-items-center justify-content-center" style="height: 200px;">
                    <i class="bi bi-image text-secondary" style="font-size: 5rem;"></i>
                </div>
            }
        </div>
        <div class="col-md-8">
            <dl class="row">
                <dt class="col-sm-3">ID</dt>
                <dd class="col-sm-9"><code>@Model.Id</code></dd>
                
                <dt class="col-sm-3">Name</dt>
                <dd class="col-sm-9">@Model.Name</dd>
                
                <dt class="col-sm-3">Description</dt>
                <dd class="col-sm-9">@Model.Description</dd>
                
                <dt class="col-sm-3">Price</dt>
                <dd class="col-sm-9">@Model.FormattedPrice</dd>
                
                <dt class="col-sm-3">Stock</dt>
                <dd class="col-sm-9">
                    @if (Model.InStock)
                    {
                        <span class="badge bg-success">@Model.StockQuantity in stock</span>
                    }
                    else
                    {
                        <span class="badge bg-danger">Out of stock</span>
                    }
                </dd>
            </dl>
        </div>
    </div>

    <div class="mt-3">
        <a asp-action="Edit" asp-route-id="@Model.Id" class="btn btn-primary">Edit</a>
        <a asp-action="Delete" asp-route-id="@Model.Id" class="btn btn-danger">Delete</a>
        <a asp-action="Index" class="btn btn-secondary">Back to List</a>
    </div>
    ```

3. Create placeholder views for Create, Edit, and Delete:

    > `src/MerchStore.WebUI/Areas/Management/Views/Products/Create.cshtml`

    ```cshtml
    @model MerchStore.WebUI.Areas.Management.Models.ProductViewModel

    @{
        ViewData["Title"] = "Create Product";
    }

    <h1>Create Product</h1>

    <div class="alert alert-info">
        <h4>Coming Soon</h4>
        <p>This functionality will be implemented in future exercises with CQRS pattern.</p>
    </div>

    <div>
        <a asp-action="Index" class="btn btn-secondary">Back to List</a>
    </div>
    ```

    > `src/MerchStore.WebUI/Areas/Management/Views/Products/Edit.cshtml`

    ```cshtml
    @model MerchStore.WebUI.Areas.Management.Models.ProductViewModel

    @{
        ViewData["Title"] = "Edit Product";
    }

    <h1>Edit Product</h1>

    <div class="alert alert-info">
        <h4>Coming Soon</h4>
        <p>This functionality will be implemented in future exercises with CQRS pattern.</p>
    </div>

    <div>
        <a asp-action="Index" class="btn btn-secondary">Back to List</a>
    </div>
    ```

    > `src/MerchStore.WebUI/Areas/Management/Views/Products/Delete.cshtml`

    ```cshtml
    @model MerchStore.WebUI.Areas.Management.Models.ProductViewModel

    @{
        ViewData["Title"] = "Delete Product";
    }

    <h1>Delete Product</h1>

    <div class="alert alert-info">
        <h4>Coming Soon</h4>
        <p>This functionality will be implemented in future exercises with CQRS pattern.</p>
    </div>

    <div>
        <a asp-action="Index" class="btn btn-secondary">Back to List</a>
    </div>
    ```

> 💡 **Information**
>
> - **Placeholder Views**: These views set up the UI structure but some don't have functionality yet
> - **Bootstrap Styling**: The views use Bootstrap components for a clean, responsive layout
> - **Icon Library**: Bootstrap Icons are used for visual cues in buttons and empty states
>
> ⚠️ **Common Mistakes**
>
> - Creating too much functionality that will be refactored later
> - Not providing clear visual cues for functionality that's not yet implemented

### Step 6: Update the Program.cs File to Support Areas

**Introduction**: We need to configure ASP.NET Core to support areas by adding the appropriate route mapping.

1. Update the Program.cs file to add area support:

    > `src/MerchStore.WebUI/Program.cs`

    ```csharp
    // Add area route - this must come before the default route
    app.MapControllerRoute(
        name: "areas",
        pattern: "{area:exists}/{controller=Home}/{action=Index}/{id?}");
    ```

> 💡 **Information**
>
> - **Area Routing**: The `{area:exists}` constraint in the route pattern tells ASP.NET Core to use this route only when the area exists
> - **Route Order**: The area route must come before the default route, or the default route might match first
>
> ⚠️ **Common Mistakes**
>
> - Placing the area route after the default route, which can prevent area routes from matching
> - Forgetting to add the area route entirely, which makes area controllers unreachable

### Step 7: Add Navigation to the Management Area

**Introduction**: Users need a way to access the Management area from the main navigation menu.

1. Update the navigation menu in the main layout file:

    > `src/MerchStore.WebUI/Views/Shared/_Layout.cshtml`

    ```cshtml
    <!-- Find the navigation section that looks something like this: -->
    <div class="navbar-collapse collapse d-sm-inline-flex justify-content-between">
        <ul class="navbar-nav flex-grow-1">
            <li class="nav-item">
                <a class="nav-link text-dark" asp-area="" asp-controller="Home" asp-action="Index">Home</a>
            </li>
            <li class="nav-item">
                <a class="nav-link text-dark" asp-area="" asp-controller="Catalog" asp-action="Index">Store</a>
            </li>
            <!-- Add the following new item to the menu -->
            <li class="nav-item">
                <a class="nav-link text-dark" asp-area="Management" asp-controller="Home" asp-action="Index">
                    <i class="bi bi-gear"></i> Management
                </a>
            </li>
        </ul>
    </div>
    ```

> 💡 **Information**
>
> - **Area Navigation**: The `asp-area` attribute tells ASP.NET Core which area to navigate to
> - **Icon Usage**: Bootstrap icons add visual cues to navigation items
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to include the `asp-area` attribute, which prevents proper navigation to the area
> - Not providing a clear visual distinction for different application sections

## 🧪 Final Tests

### Run the Application and Validate Your Work

1. Build and run the application:

   ```bash
   dotnet build
   dotnet run --project src/MerchStore.WebUI
   ```

2. Open a browser and navigate to the homepage.

3. Click on the "Management" link in the navigation bar.

4. You should see the Management Dashboard with a card for Products.

5. Click on "Manage Products" to navigate to the products list.

6. View the details of a product by clicking on the eye icon.

7. Try the placeholder views for Create, Edit, and Delete.

✅ **Expected Results**

- The Management Area is accessible from the main navigation menu
- The Dashboard shows available management functions
- The Products list displays all products from your database
- Product Details shows complete information about the selected product
- Create, Edit, and Delete views show "Coming Soon" placeholders
- Navigation between views works correctly

## 🔧 Troubleshooting

If you encounter issues:

- Check that your area folders follow the correct structure
- Ensure all controllers have the `[Area("Management")]` attribute
- Verify that you've added the area route to Program.cs
- Check that _ViewImports.cshtml and `_ViewStart.cshtml` exist in the area folder
- Look for any errors in the browser developer console

## 🚀 Optional Challenge

Want to take your learning further? Try:

- Add a simple authentication system to restrict access to the Management area
- Create a custom layout for the Management area that's different from the main application
- Add sorting and filtering options to the Products list view
- Implement breadcrumb navigation to improve user experience in the Management area

## 📚 Further Reading

- [ASP.NET Core Areas](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/areas) - Microsoft's documentation on Areas
- [Bootstrap Icons](https://icons.getbootstrap.com/) - Documentation for Bootstrap Icons
- [ASP.NET Core Tag Helpers](https://docs.microsoft.com/en-us/aspnet/core/mvc/views/tag-helpers/intro) - Guide to using Tag Helpers in views
- [Route Order in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing) - Details on how route order affects matching

## Done! 🎉

Great job! You've successfully created a Management Area for your application with placeholder views for product management. This lays the groundwork for implementing the CQRS pattern in upcoming exercises.

In the next exercise, we'll introduce MediatR and implement the CQRS pattern to handle product management operations.
