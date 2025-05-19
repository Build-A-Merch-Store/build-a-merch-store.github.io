---
title: "Implementing the Create Product Command with FluentValidation"
date: "2025-05-19"
lastmod: "2025-05-19"
draft: false
weight: 803
toc: true
---

## 🎯 Goal

Implement the Command side of CQRS with a CreateProductCommand that uses FluentValidation to ensure data integrity when creating new products.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 2 (Introducing CQRS with MediatR and AutoMapper)
- Have implemented the Query side of CQRS for retrieving products
- Understand basic MediatR and AutoMapper concepts
- Be familiar with the Product entity and its validation requirements

## 📚 Learning Objectives

By the end of this exercise, you will:

- Implement a **Command** in the CQRS pattern to create a product
- Use **FluentValidation** to validate command inputs
- Create a **Command Handler** to process the creation request
- Integrate **validation with MediatR pipeline** using behavior
- Connect the command to the **Management UI** for product creation
- Understand proper **error handling** patterns with CQRS

## 🔍 Why This Matters

In real-world applications, implementing the Command side of CQRS with validation provides several benefits:

- **Data integrity**: Ensures only valid data enters your domain
- **Separation of concerns**: Validation logic stays out of your domain model and controllers
- **Descriptive errors**: FluentValidation provides clear, user-friendly error messages
- **Maintainability**: Validation rules are centralized and easy to modify
- **Testability**: Commands and validators can be tested in isolation

## 📝 Step-by-Step Instructions

### Step 1: Install FluentValidation Package

**Introduction**: FluentValidation is a powerful library for building strongly-typed validation rules. It provides a fluent interface for defining validation rules for your objects.

1. Add the FluentValidation package to the Application project:

    ```bash
    cd src/MerchStore.Application
    dotnet add package FluentValidation
    dotnet add package FluentValidation.DependencyInjectionExtensions
    ```

> 💡 **Information**
>
> - **FluentValidation**: A popular validation library that allows you to define validation rules in a fluent interface
> - **DependencyInjection**: Extension package for registering validators with the ASP.NET Core DI container
> - **Rule Chaining**: FluentValidation allows you to chain validation rules for cleaner code
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to install the DependencyInjection package
> - Confusing FluentValidation with DataAnnotations (they serve similar purposes but have different APIs)

### Step 2: Create a Command for Creating Products

**Introduction**: In CQRS, Commands represent write operations. A Command object encapsulates the intent to change the system's state along with the data needed for that operation.

1. Create a directory structure for the command:

    ```bash
    mkdir -p src/MerchStore.Application/Products/Commands/CreateProduct
    ```

2. Create the CreateProductCommand class:

    > `src/MerchStore.Application/Products/Commands/CreateProduct/CreateProductCommand.cs`

    ```csharp
    using MediatR;
    using MerchStore.Application.Common.Responses;

    namespace MerchStore.Application.Products.Commands.CreateProduct;

    /// <summary>
    /// Command to create a new product.
    /// </summary>
    public class CreateProductCommand : IRequest<DataResponse<Guid>>
    {
        /// <summary>
        /// Gets or sets the name of the product.
        /// </summary>
        public string Name { get; set; } = string.Empty;
        
        /// <summary>
        /// Gets or sets the description of the product.
        /// </summary>
        public string Description { get; set; } = string.Empty;
        
        /// <summary>
        /// Gets or sets the price amount of the product.
        /// </summary>
        public decimal PriceAmount { get; set; }
        
        /// <summary>
        /// Gets or sets the price currency of the product.
        /// </summary>
        public string PriceCurrency { get; set; } = string.Empty;
        
        /// <summary>
        /// Gets or sets the stock quantity of the product.
        /// </summary>
        public int StockQuantity { get; set; }
        
        /// <summary>
        /// Gets or sets the image URL of the product.
        /// </summary>
        public Uri? ImageUrl { get; set; }
    }
    ```

> 💡 **Information**
>
> - **`IRequest<T>`**: MediatR interface that defines the return type of the command
> - **`DataResponse<Guid>`**: Returns the ID of the newly created product, wrapped in our response type
> - **Properties**: Contain all data needed to create a new product
> - **Immutability**: Consider making this a record type in real-world applications for immutability
>
> ⚠️ **Common Mistakes**
>
> - Adding domain logic to the command (it should only contain data)
> - Using primitive types that don't match your domain model (like string for URL instead of Uri)

### Step 3: Create a Validator for the Command

**Introduction**: Validators ensure that commands contain valid data before they're processed. FluentValidation allows you to define validation rules in a clean, fluent syntax.

1. Create the CreateProductCommandValidator class:

    > `src/MerchStore.Application/Products/Commands/CreateProduct/CreateProductCommandValidator.cs`

    ```csharp
    using FluentValidation;

    namespace MerchStore.Application.Products.Commands.CreateProduct;

    /// <summary>
    /// Validator for the CreateProductCommand.
    /// </summary>
    public class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
    {
        public CreateProductCommandValidator()
        {
            RuleFor(p => p.Name)
                .NotEmpty().WithMessage("Name is required")
                .MaximumLength(100).WithMessage("Name cannot exceed 100 characters");
                
            RuleFor(p => p.Description)
                .NotEmpty().WithMessage("Description is required")
                .MaximumLength(500).WithMessage("Description cannot exceed 500 characters");
                
            RuleFor(p => p.PriceAmount)
                .GreaterThan(0).WithMessage("Price must be greater than zero");
                
            RuleFor(p => p.PriceCurrency)
                .NotEmpty().WithMessage("Currency is required")
                .Length(3).WithMessage("Currency code must be 3 characters (ISO 4217 format)");
                
            RuleFor(p => p.StockQuantity)
                .GreaterThanOrEqualTo(0).WithMessage("Stock quantity cannot be negative");
                
            RuleFor(p => p.ImageUrl)
                .Must(BeAValidUrl).When(p => p != null)
                .WithMessage("Image URL must be a valid HTTP or HTTPS URL");
        }
        
        private bool BeAValidUrl(Uri? uri)
        {
            if (uri == null)
                return true;
                
            return uri.Scheme == Uri.UriSchemeHttp || uri.Scheme == Uri.UriSchemeHttps;
        }
    }
    ```

> 💡 **Information**
>
> - **AbstractValidator**: Base class from FluentValidation for creating validators
> - **RuleFor**: Defines validation rules for a specific property
> - **Chaining**: Multiple validation rules can be chained for a single property
> - **Custom Rules**: Custom validation logic can be defined using methods like `Must`
> - **Conditional Validation**: `When` clause applies rules only under specific conditions
>
> ⚠️ **Common Mistakes**
>
> - Writing inconsistent validation messages
> - Not aligning validation rules with domain entity constraints
> - Neglecting to validate nullable properties conditionally

### Step 4: Create a Validation Behavior for MediatR

**Introduction**: MediatR's pipeline behaviors allow you to intercept and process requests before they reach handlers. We'll create a validation behavior that validates commands before they're processed.

1. Create a directory for behaviors:

    ```bash
    mkdir -p src/MerchStore.Application/Common/Behaviors
    ```

2. Create the ValidationBehavior class:

    > `src/MerchStore.Application/Common/Behaviors/ValidationBehavior.cs`

    ```csharp
    using FluentValidation;
    using MediatR;

    namespace MerchStore.Application.Common.Behaviors;

    /// <summary>
    /// Pipeline behavior that validates requests before they are handled.
    /// </summary>
    /// <typeparam name="TRequest">The type of request being handled.</typeparam>
    /// <typeparam name="TResponse">The type of response from the handler.</typeparam>
    public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
        where TRequest : IRequest<TResponse>
    {
        private readonly IEnumerable<IValidator<TRequest>> _validators;
        
        /// <summary>
        /// Initializes a new instance of the ValidationBehavior class.
        /// </summary>
        /// <param name="validators">The validators for the request.</param>
        public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        {
            _validators = validators;
        }
        
        /// <summary>
        /// Handles the request.
        /// </summary>
        /// <param name="request">The request.</param>
        /// <param name="next">The next handler in the pipeline.</param>
        /// <param name="cancellationToken">The cancellation token.</param>
        /// <returns>The response from the next handler.</returns>
        public async Task<TResponse> Handle(
            TRequest request, 
            RequestHandlerDelegate<TResponse> next, 
            CancellationToken cancellationToken)
        {
            // If there are no validators for this request, continue to the next handler
            if (!_validators.Any())
            {
                return await next();
            }
            
            // Create a validation context
            var context = new ValidationContext<TRequest>(request);
            
            // Run all validators
            var validationResults = await Task.WhenAll(
                _validators.Select(v => v.ValidateAsync(context, cancellationToken)));
            
            // Collect all failures
            var failures = validationResults
                .SelectMany(r => r.Errors)
                .Where(f => f != null)
                .ToList();
            
            // If there are any failures, throw a validation exception
            if (failures.Count != 0)
            {
                throw new ValidationException(failures);
            }
            
            // If validation passes, continue to the next handler
            return await next();
        }
    }
    ```

> 💡 **Information**
>
> - **IPipelineBehavior**: MediatR interface for creating pipeline behaviors
> - **`IValidator<T>`**: FluentValidation interface for validators
> - **Task.WhenAll**: Runs all validators in parallel for better performance
> - **ValidationException**: FluentValidation exception that contains all validation failures
>
> ⚠️ **Common Mistakes**
>
> - Not handling the case where there are no validators
> - Not correctly propagating the cancellation token
> - Forgetting to check if the validators list is empty before using it

### Step 5: Update DependencyInjection to Register Validators and Behaviors

**Introduction**: We need to register our validators and behaviors with the dependency injection container.

1. Update the DependencyInjection class in the Application project:

    > `src/MerchStore.Application/DependencyInjection.cs`

    ```csharp
    using System.Reflection;
    using FluentValidation;
    using MediatR;
    using Microsoft.Extensions.DependencyInjection;
    using MerchStore.Application.Common.Behaviors;

    namespace MerchStore.Application;

    /// <summary>
    /// Extension methods for setting up application services.
    /// </summary>
    public static class DependencyInjection
    {
        /// <summary>
        /// Adds application services to the specified IServiceCollection.
        /// </summary>
        /// <param name="services">The IServiceCollection to add services to.</param>
        /// <returns>The same service collection so that multiple calls can be chained.</returns>
        public static IServiceCollection AddApplication(this IServiceCollection services)
        {
            // Get the assembly where the handlers are located
            var assembly = Assembly.GetExecutingAssembly();
            
            // Register MediatR with validation behavior
            services.AddMediatR(cfg => {
                cfg.RegisterServicesFromAssembly(assembly);
                
                // Add validation behavior to pipeline
                cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
            });
            
            // Register all validators from this assembly
            services.AddValidatorsFromAssembly(assembly);
            
            // Register AutoMapper
            services.AddAutoMapper(assembly);
            
            return services;
        }
    }
    ```

> 💡 **Information**
>
> - **AddValidatorsFromAssembly**: FluentValidation extension for registering all validators in an assembly
> - **AddBehavior**: MediatR extension for adding pipeline behaviors
> - **Type Parameters**: Uses open generic types for registering the behavior
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to register the validation behavior in the MediatR pipeline
> - Not registering all validators from the assembly

### Step 6: Update the AutoMapper Profile for Commands

**Introduction**: We need to update our AutoMapper profile to include mappings for commands. This will simplify the mapping from ViewModel to Command objects.

1. Update the ProductMappingProfile class:

    > `src/MerchStore.Application/Products/Mappings/ProductMappingProfile.cs`

    ```csharp
    using AutoMapper;
    using MerchStore.Application.Products.Commands.CreateProduct;
    using MerchStore.Application.Products.DTOs;
    using MerchStore.Domain.Entities;
    using MerchStore.Domain.ValueObjects;
    
    namespace MerchStore.Application.Products.Mappings;
    
    /// <summary>
    /// AutoMapper profile for mapping between Product entities and DTOs.
    /// </summary>
    public class ProductMappingProfile : Profile
    {
        public ProductMappingProfile()
        {
            // Map from domain entity to DTO
            CreateMap<Product, ProductDto>()
                .ForMember(dest => dest.PriceAmount, opt => opt.MapFrom(src => src.Price.Amount))
                .ForMember(dest => dest.PriceCurrency, opt => opt.MapFrom(src => src.Price.Currency))
                .ForMember(dest => dest.ImageUrl, opt => opt.MapFrom(src => src.ImageUrl != null ? src.ImageUrl.ToString() : null));
       
            // Map from command to Product entity
            CreateMap<CreateProductCommand, Product>()
                .ConstructUsing(src => new Product(
                    src.Name,
                    src.Description,
                    src.ImageUrl,
                    new Money(src.PriceAmount, src.PriceCurrency),
                    src.StockQuantity
                ));
        }
    }
    ```

> 💡 **Information**
>
> - **ConstructUsing**: Creates a new instance using a custom constructor expression
> - **Money Mapping**: Maps from the command to the Money value object
> - **Existing Mappings**: Original domain-to-DTO mappings are preserved
>
> ⚠️ **Common Mistakes**
>
> - Not handling value object creation correctly
> - Overwriting existing mappings that are still needed

### Step 7: Create a Command Handler

**Introduction**: Command handlers contain the logic to process a specific command. They typically modify the system's state and return a result indicating success or failure.

1. Create the CreateProductCommandHandler class:

    > `src/MerchStore.Application/Products/Commands/CreateProduct/CreateProductCommandHandler.cs`

    ```csharp
    using AutoMapper;
    using MediatR;
    using MerchStore.Application.Common.Interfaces;
    using MerchStore.Application.Common.Responses;
    using MerchStore.Domain.Entities;
    using MerchStore.Domain.Interfaces;
    
    namespace MerchStore.Application.Products.Commands.CreateProduct;
    
    /// <summary>
    /// Handler for the CreateProductCommand.
    /// </summary>
    public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand, DataResponse<Guid>>
    {
        private readonly IRepositoryManager _repositoryManager;
        private readonly IMapper _mapper;
    
        /// <summary>
        /// Initializes a new instance of the CreateProductCommandHandler class.
        /// </summary>
        /// <param name="repositoryManager">The repository manager.</param>
        /// <param name="mapper">The AutoMapper instance.</param>
        public CreateProductCommandHandler(IRepositoryManager repositoryManager, IMapper mapper)
        {
            _repositoryManager = repositoryManager;
            _mapper = mapper;
        }
    
        /// <summary>
        /// Handles the CreateProductCommand.
        /// </summary>
        /// <param name="request">The command.</param>
        /// <param name="cancellationToken">The cancellation token.</param>
        /// <returns>A response with the ID of the created product.</returns>
        public async Task<DataResponse<Guid>> Handle(CreateProductCommand request, CancellationToken cancellationToken)
        {
            try
            {
                // Create a Product entity using AutoMapper
                var product = _mapper.Map<Product>(request);
    
                // Add the product to the repository
                await _repositoryManager.ProductRepository.AddAsync(product);
    
                // Save changes
                await _repositoryManager.UnitOfWork.SaveChangesAsync(cancellationToken);
    
                // Return a success response with the new product ID
                return DataResponse<Guid>.Success(product.Id, "Product created successfully");
            }
            catch (ArgumentException ex)
            {
                // Handle validation errors from the domain
                return DataResponse<Guid>.Failure($"Invalid product data: {ex.Message}");
            }
            catch (Exception ex)
            {
                // Handle unexpected errors
                return DataResponse<Guid>.Failure($"Error creating product: {ex.Message}");
            }
        }
    }
    ```

> 💡 **Information**
>
> - **Domain Entity Creation**: The handler creates a new domain entity
> - **Value Objects**: Money is created as a value object
> - **Repository Pattern**: The entity is added to the repository
> - **Unit of Work**: Changes are saved using the unit of work pattern
> - **Error Handling**: Different types of exceptions are handled appropriately
>
> ⚠️ **Common Mistakes**
>
> - Not handling domain exceptions specifically
> - Forgetting to save changes using the unit of work
> - Returning the wrong type of response

### Step 8: Create a Create Product ViewModel

**Introduction**: We need a ViewModel for the Create Product form in the Management area.

1. Create the CreateProductViewModel class:

    > `src/MerchStore.WebUI/Areas/Management/Models/CreateProductViewModel.cs`

    ```csharp
    using System.ComponentModel.DataAnnotations;

    namespace MerchStore.WebUI.Areas.Management.Models;

    /// <summary>
    /// ViewModel for creating a new product.
    /// </summary>
    public class CreateProductViewModel
    {
        /// <summary>
        /// Gets or sets the name of the product.
        /// </summary>
        [Required(ErrorMessage = "Name is required")]
        [StringLength(100, ErrorMessage = "Name cannot exceed 100 characters")]
        [Display(Name = "Product Name")]
        public string Name { get; set; } = string.Empty;
        
        /// <summary>
        /// Gets or sets the description of the product.
        /// </summary>
        [Required(ErrorMessage = "Description is required")]
        [StringLength(500, ErrorMessage = "Description cannot exceed 500 characters")]
        public string Description { get; set; } = string.Empty;
        
        /// <summary>
        /// Gets or sets the price amount of the product.
        /// </summary>
        [Required(ErrorMessage = "Price is required")]
        [Range(0.01, 10000, ErrorMessage = "Price must be greater than 0")]
        [Display(Name = "Price Amount")]
        public decimal PriceAmount { get; set; }
        
        /// <summary>
        /// Gets or sets the price currency of the product.
        /// </summary>
        [Required(ErrorMessage = "Currency is required")]
        [StringLength(3, MinimumLength = 3, ErrorMessage = "Currency code must be 3 characters")]
        [Display(Name = "Currency")]
        public string PriceCurrency { get; set; } = "SEK";
        
        /// <summary>
        /// Gets or sets the stock quantity of the product.
        /// </summary>
        [Required(ErrorMessage = "Stock quantity is required")]
        [Range(0, int.MaxValue, ErrorMessage = "Stock quantity cannot be negative")]
        [Display(Name = "Stock Quantity")]
        public int StockQuantity { get; set; }
        
        /// <summary>
        /// Gets or sets the image URL of the product.
        /// </summary>
        [Display(Name = "Image URL (optional)")]
        [Url(ErrorMessage = "Please enter a valid URL")]
        public string? ImageUrl { get; set; }
    }
    ```

> 💡 **Information**
>
> - **Data Annotations**: Client-side validation attributes for form inputs
> - **Display Names**: Human-friendly field names for UI labels
> - **Default Values**: Currency defaults to "SEK" for convenience
>
> ⚠️ **Common Mistakes**
>
> - Not aligning validation rules with the FluentValidation rules
> - Forgetting to mark required fields with attributes
> - Missing display names which hurts UI readability

### Step 9: Update the WebUI AutoMapper Profile

**Introduction**: We need to update the AutoMapper profile in the WebUI project to map between the CreateProductViewModel and the CreateProductCommand.

1. Update the ProductsViewModelMappingProfile class:

    > `src/MerchStore.WebUI/Infrastructure/Mapping/ProductsViewModelMappingProfile.cs`

    ```csharp
    using AutoMapper;
    using MerchStore.Application.Products.Commands.CreateProduct;
    using MerchStore.Application.Products.DTOs;
    using MerchStore.WebUI.Areas.Management.Models;

    namespace MerchStore.WebUI.Infrastructure.Mapping;

    /// <summary>
    /// AutoMapper profile for mapping between product DTOs and view models.
    /// </summary>
    public class ProductsViewModelMappingProfile : Profile
    {
        public ProductsViewModelMappingProfile()
        {
            // Map from DTO to ViewModel
            CreateMap<ProductDto, ProductViewModel>()
                .ForMember(dest => dest.Price, opt => opt.MapFrom(src => src.PriceAmount))
                .ForMember(dest => dest.Currency, opt => opt.MapFrom(src => src.PriceCurrency));
                
            // Map from CreateProductViewModel to CreateProductCommand
            CreateMap<CreateProductViewModel, CreateProductCommand>()
                .ForMember(dest => dest.ImageUrl, opt => opt.MapFrom(src => 
                    !string.IsNullOrEmpty(src.ImageUrl) ? new Uri(src.ImageUrl) : null));
        }
    }
    ```

> 💡 **Information**
>
> - **String to Uri**: Special handling for mapping ImageUrl from string to Uri
> - **Nullability**: Properly handling nullable ImageUrl
> - **Existing Mappings**: Original DTO-to-ViewModel mappings are preserved
>
> ⚠️ **Common Mistakes**
>
> - Not handling the conversion from string to Uri correctly
> - Not checking for null or empty strings before creating a Uri

### Step 10: Update the Create Product View

**Introduction**: Now that we have our ViewModel ready, let's update the Create Product view in the Management area.

1. Update the Create.cshtml file:

    > `src/MerchStore.WebUI/Areas/Management/Views/Products/Create.cshtml`

    ```cshtml
    @model MerchStore.WebUI.Areas.Management.Models.CreateProductViewModel

    @{
        ViewData["Title"] = "Create Product";
    }

    <div class="card">
        <div class="card-header bg-primary text-white d-flex justify-content-between align-items-center">
            <h1 class="fs-4 m-0">Create New Product</h1>
            <a asp-action="Index" class="btn btn-secondary btn-sm">
                <i class="bi bi-arrow-left"></i> Back to List
            </a>
        </div>
        <div class="card-body">
            <form asp-action="Create" method="post">
                <div asp-validation-summary="ModelOnly" class="text-danger"></div>
                
                <div class="row mb-3">
                    <div class="col-md-6">
                        <div class="form-group mb-3">
                            <label asp-for="Name" class="form-label"></label>
                            <input asp-for="Name" class="form-control" />
                            <span asp-validation-for="Name" class="text-danger"></span>
                        </div>
                        
                        <div class="form-group mb-3">
                            <label asp-for="Description" class="form-label"></label>
                            <textarea asp-for="Description" class="form-control" rows="5"></textarea>
                            <span asp-validation-for="Description" class="text-danger"></span>
                        </div>
                        
                        <div class="form-group mb-3">
                            <label asp-for="ImageUrl" class="form-label"></label>
                            <input asp-for="ImageUrl" class="form-control" />
                            <span asp-validation-for="ImageUrl" class="text-danger"></span>
                            <div class="form-text">Enter a valid URL to an image (optional)</div>
                        </div>
                    </div>
                    
                    <div class="col-md-6">
                        <div class="row">
                            <div class="col-md-6">
                                <div class="form-group mb-3">
                                    <label asp-for="PriceAmount" class="form-label"></label>
                                    <input asp-for="PriceAmount" class="form-control" type="number" step="0.01" min="0.01" />
                                    <span asp-validation-for="PriceAmount" class="text-danger"></span>
                                </div>
                            </div>
                            <div class="col-md-6">
                                <div class="form-group mb-3">
                                    <label asp-for="PriceCurrency" class="form-label"></label>
                                    <input asp-for="PriceCurrency" class="form-control" maxlength="3" />
                                    <span asp-validation-for="PriceCurrency" class="text-danger"></span>
                                </div>
                            </div>
                        </div>
                        
                        <div class="form-group mb-3">
                            <label asp-for="StockQuantity" class="form-label"></label>
                            <input asp-for="StockQuantity" class="form-control" type="number" min="0" />
                            <span asp-validation-for="StockQuantity" class="text-danger"></span>
                        </div>
                    </div>
                </div>
                
                <div class="d-flex gap-2">
                    <button type="submit" class="btn btn-primary">
                        <i class="bi bi-plus-circle"></i> Create Product
                    </button>
                    <a asp-action="Index" class="btn btn-secondary">
                        <i class="bi bi-arrow-left"></i> Cancel
                    </a>
                </div>
            </form>
        </div>
    </div>

    @section Scripts {
        <partial name="_ValidationScriptsPartial" />
    }
    ```

> 💡 **Information**
>
> - **Form Structure**: Clean layout with two columns for better organization
> - **Validation Messages**: Displays validation errors from both client and server
> - **Input Types**: Specific input types for different fields (number, textarea)
> - **Icon Usage**: Bootstrap icons for visual cues
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to include client-side validation scripts
> - Not setting appropriate input attributes like min, max, and step
> - Missing form-text help messages for complex fields

### Step 11: Update the Products Controller to Handle the Create Action

**Introduction**: Now we need to update the ProductsController to handle the create action using our new Command.

1. Update the ProductsController in the Management area to implement the POST method for Create:

    > `src/MerchStore.WebUI/Areas/Management/Controllers/ProductsController.cs`

    ```csharp
    // Add to the existing ProductsController class

    // POST: Management/Products/Create
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(CreateProductViewModel viewModel)
    {
        if (!ModelState.IsValid)
        {
            return View(viewModel);
        }
        
        try
        {
            // Map the view model to a command
            var command = _mapper.Map<CreateProductCommand>(viewModel);
            
            // Send the command to the handler
            var response = await _mediator.Send(command);
            
            // Check if the command was successful
            if (response.IsSuccess)
            {
                // Add a success message to TempData
                TempData["SuccessMessage"] = "Product created successfully";
                
                // Redirect to the product details page
                return RedirectToAction(nameof(Details), new { id = response.Data });
            }
            
            // If the command failed, add the error to ModelState
            ModelState.AddModelError("", response.Message);
            return View(viewModel);
        }
        catch (Exception ex)
        {
            // Handle any unexpected errors
            ModelState.AddModelError("", $"An error occurred: {ex.Message}");
            return View(viewModel);
        }
    }
    ```

> 💡 **Information**
>
> - **Anti-Forgery Token**: Protects against Cross-Site Request Forgery (CSRF) attacks
> - **Model Validation**: Checks if the submitted form data is valid
> - **Command Mapping**: Maps the ViewModel to a Command
> - **Error Handling**: Handles both domain and unexpected errors
> - **Success Notification**: Uses TempData to show a success message
>
> ⚠️ **Common Mistakes**
>
> - Not validating the model state before processing
> - Not handling errors from the command handler
> - Not providing meaningful error messages to the user

### Step 12: Add Success Messages to the Index View

**Introduction**: To provide better feedback to users, let's add a success message display to the Index view.

1. Update the Index.cshtml file to show success messages:

    > `src/MerchStore.WebUI/Areas/Management/Views/Products/Index.cshtml`

    ```cshtml
    @* Add this right after the opening div with d-flex class *@
    
    @if (TempData["SuccessMessage"] != null)
    {
        <div class="alert alert-success alert-dismissible fade show mb-4" role="alert">
            <i class="bi bi-check-circle-fill me-2"></i> @TempData["SuccessMessage"]
            <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
        </div>
    }
    ```

> 💡 **Information**
>
> - **TempData**: ASP.NET Core's mechanism for storing data for a single request
> - **Alert Dismissible**: Bootstrap's component for closable alert messages
> - **Icon**: Visual indicator for success messages
>
> ⚠️ **Common Mistakes**
>
> - Not checking if TempData exists before using it
> - Missing the dismissible functionality for alerts

## 🧪 Final Tests

### Run the Application and Validate Your Work

1. Build and run the application:

   ```bash
   dotnet build
   dotnet run --project src/MerchStore.WebUI
   ```

2. Open a browser and navigate to the Management area.

3. Click on "Manage Products" and then "Add New Product".

4. Fill out the form with valid data and submit it.

5. Verify that the product is created and you are redirected to the details page.

6. Try submitting the form with invalid data (e.g., empty name, negative price) and verify that validation errors are shown.

✅ **Expected Results**

- The Create Product form displays correctly
- Submitting valid data creates a new product successfully
- Validation errors are shown for invalid data
- After successful creation, you're redirected to the product details page
- A success message is shown at the top of the page

## 🔧 Troubleshooting

If you encounter issues:

- Check that all packages are installed correctly
- Verify that FluentValidation is registered in the dependency injection container
- Ensure the validation behavior is registered with MediatR
- Check that your AutoMapper profiles are correctly mapping between types
- Ensure your domain entity's constructor validation aligns with your command validator
- Look for any errors in the browser developer console or server logs

## 🚀 Optional Challenge

Want to take your learning further? Try:

- Implement a custom validator for checking if a product with the same name already exists
- Create a more robust image URL validator that also checks file extensions
- Implement a notification system to show success/error messages in a toast popup
- Add a preview feature that shows what the product will look like before saving it
- Extend the command to support product categories or tags

## 📚 Further Reading

- [FluentValidation Documentation](https://docs.fluentvalidation.net/) - Official documentation
- [CQRS with MediatR and FluentValidation](https://www.codewithmukesh.com/blog/cqrs-in-aspnet-core-3-1/) - An in-depth tutorial
- [Domain-Driven Design Validation](https://enterprisecraftsmanship.com/posts/validation-and-ddd/) - Best practices for validation in DDD
- [Unit of Work Pattern](https://martinfowler.com/eaaCatalog/unitOfWork.html) - Martin Fowler's article
- [AutoMapper Best Practices](https://jimmybogard.com/automapper-usage-guidelines/) - Guidelines from the creator of AutoMapper

## Done! 🎉

Great job! You've now implemented the Command side of CQRS with FluentValidation. You've learned how to:

- Create a command with its validator
- Process it with a handler
- Validate input data before it reaches your domain model
- Connect the command processing to your UI
- Handle errors and provide user feedback

This completes the full CQRS implementation for product creation. You now have both the Query and Command sides of CQRS working in your application, allowing you to read and write data in a clean, maintainable way.

In future exercises, you'll continue this pattern to implement the update and delete operations, further expanding your CQRS skills.
