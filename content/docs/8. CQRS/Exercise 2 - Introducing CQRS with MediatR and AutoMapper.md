---
title: "Introducing CQRS with MediatR and AutoMapper"
date: "2025-05-19"
lastmod: "2025-05-19"
draft: false
weight: 802
toc: true
---

## 🎯 Goal

Implement the CQRS pattern using MediatR and AutoMapper to retrieve and display a list of products in the Management Area.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 1 (Introducing Areas and Preparing for CQRS)
- Have a working Management Area for product administration
- Understand basic service pattern implementation
- Be familiar with dependency injection in ASP.NET Core

## 📚 Learning Objectives

By the end of this exercise, you will:

- Implement the **CQRS pattern** for querying products
- Use **MediatR** to decouple query handlers from controllers
- Configure **AutoMapper** to map between different object types
- Create a **Query** object to represent a data retrieval request
- Implement a **QueryHandler** to process the request
- Understand the benefits of **separation of concerns** in a clean architecture

## 🔍 Why This Matters

In real-world applications, CQRS (Command Query Responsibility Segregation) provides several benefits:

- **Separation of concerns**: Reading data (queries) is separated from modifying data (commands)
- **Scalability**: Read and write operations can be optimized independently
- **Simplicity**: Each operation has a focused purpose, making code easier to understand and maintain
- **Testability**: Individual components can be tested in isolation
- **Performance**: Query models can be optimized for specific read scenarios

## 📝 Step-by-Step Instructions

### Step 1: Install Required Packages

**Introduction**: Let's start by installing the packages we need for CQRS implementation. MediatR is a lightweight mediator pattern implementation, and AutoMapper is used for mapping between different object types.

1. Add the required packages to the Application project:

    ```bash
    cd src/MerchStore.Application
    dotnet add package MediatR
    dotnet add package AutoMapper
    ```

> 💡 **Information**
>
> - **MediatR**: Implements the mediator pattern to decouple request senders from handlers
> - **Extensions.Microsoft.DependencyInjection**: Extensions for registering MediatR in the ASP.NET Core dependency injection container
> - **AutoMapper**: Simplifies object-to-object mapping, reducing boilerplate code
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to install the extension packages needed for dependency injection
> - Installing packages in the wrong project (they should be in the Application layer)

### Step 2: Create a Product Data Transfer Object (DTO)

**Introduction**: Data Transfer Objects (DTOs) are used to transfer data between layers of the application. They contain only the properties needed for a specific operation and help decouple the domain model from the presentation layer.

1. Create a directory structure for DTOs in the Application project:

    ```bash
    mkdir -p src/MerchStore.Application/Products/DTOs
    ```

2. Create a ProductDto class:

    > `src/MerchStore.Application/Products/DTOs/ProductDto.cs`

    ```csharp
    namespace MerchStore.Application.Products.DTOs;

    /// <summary>
    /// Data Transfer Object for returning product information.
    /// </summary>
    public class ProductDto
    {
        public Guid Id { get; set; }
        public string Name { get; set; } = string.Empty;
        public string Description { get; set; } = string.Empty;
        public decimal PriceAmount { get; set; }
        public string PriceCurrency { get; set; } = string.Empty;
        public int StockQuantity { get; set; }
        public string? ImageUrl { get; set; }
    }
    ```

> 💡 **Information**
>
> - **DTOs**: Transfer data between layers, containing only the properties needed by the client
> - **Decoupling**: DTOs decouple the domain model from the presentation layer
> - **Naming Convention**: The "Dto" suffix clearly identifies the purpose of the class
>
> ⚠️ **Common Mistakes**
>
> - Including domain behavior in DTOs (they should be simple data containers)
> - Exposing domain entities directly to the presentation layer

### Step 3: Create a Mapping Profile for AutoMapper

**Introduction**: AutoMapper mapping profiles define how objects are mapped from one type to another. In this case, we'll map from our domain Product entity to the ProductDto.

1. Create a directory for mapping profiles:

    ```bash
    mkdir -p src/MerchStore.Application/Products/Mappings
    ```

2. Create a ProductMappingProfile class:

    > `src/MerchStore.Application/Products/Mappings/ProductMappingProfile.cs`

    ```csharp
    using AutoMapper;
    using MerchStore.Application.Products.DTOs;
    using MerchStore.Domain.Entities;

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
        }
    }
    ```

> 💡 **Information**
>
> - **Profile Class**: Inherits from AutoMapper's Profile class to define mappings
> - **CreateMap**: Specifies the source and destination types for mapping
> - **ForMember**: Customizes mapping for specific properties when the default convention isn't sufficient
> - **Value Objects**: We're extracting the Amount and Currency from the Price value object
>
> ⚠️ **Common Mistakes**
>
> - Not handling nullable properties correctly
> - Not providing custom mappings for complex properties like value objects

### Step 4: Create a Query Object for Retrieving Products

**Introduction**: In CQRS, queries represent read operations. A query object encapsulates the intent to retrieve data and any parameters needed for that operation.

1. Create a directory structure for the query:

    ```bash
    mkdir -p src/MerchStore.Application/Products/Queries/GetProducts
    ```

2. Create the GetProductsQuery class:

    > `src/MerchStore.Application/Products/Queries/GetProducts/GetProductsQuery.cs`

    ```csharp
    using MediatR;
    using MerchStore.Application.Products.DTOs;

    namespace MerchStore.Application.Products.Queries.GetProducts;

    /// <summary>
    /// Query to get all products.
    /// </summary>
    public class GetProductsQuery : IRequest<IEnumerable<ProductDto>>
    {
        // Currently, we don't need any parameters for this query.
        // In the future, we could add parameters for filtering, paging, etc.
    }
    ```

> 💡 **Information**
>
> - **`IRequest<T>`**: MediatR interface that defines the return type of the query
> - **Parameters**: Queries can include properties that represent filter criteria
> - **Future Extensibility**: Comments indicate how this could be extended in the future
>
> ⚠️ **Common Mistakes**
>
> - Adding implementation logic to the query object (it should only contain data, not behavior)
> - Not making the query immutable (consider using a record type in C# 9+)

### Step 5: Create a Query Handler for Processing the Query

**Introduction**: Query handlers contain the logic to process a specific query. They typically retrieve data from a repository and map it to the requested format.

1. Create the GetProductsQueryHandler class:

    > `src/MerchStore.Application/Products/Queries/GetProducts/GetProductsQueryHandler.cs`

    ```csharp
    using AutoMapper;
    using MediatR;
    using MerchStore.Application.Products.DTOs;
    using MerchStore.Domain.Interfaces;

    namespace MerchStore.Application.Products.Queries.GetProducts;

    /// <summary>
    /// Handler for the GetProductsQuery.
    /// </summary>
    public class GetProductsQueryHandler : IRequestHandler<GetProductsQuery, IEnumerable<ProductDto>>
    {
        private readonly IProductRepository _productRepository;
        private readonly IMapper _mapper;
        
        /// <summary>
        /// Initializes a new instance of the GetProductsQueryHandler class.
        /// </summary>
        /// <param name="productRepository">The product repository.</param>
        /// <param name="mapper">The AutoMapper instance.</param>
        public GetProductsQueryHandler(IProductRepository productRepository, IMapper mapper)
        {
            _productRepository = productRepository;
            _mapper = mapper;
        }
        
        /// <summary>
        /// Handles the GetProductsQuery.
        /// </summary>
        /// <param name="request">The query.</param>
        /// <param name="cancellationToken">The cancellation token.</param>
        /// <returns>A collection of product DTOs.</returns>
        public async Task<IEnumerable<ProductDto>> Handle(GetProductsQuery request, CancellationToken cancellationToken)
        {
            // Get all products from the repository
            var products = await _productRepository.GetAllAsync();
            
            // Map the domain entities to DTOs
            return _mapper.Map<IEnumerable<ProductDto>>(products);
        }
    }
    ```

> 💡 **Information**
>
> - **IRequestHandler<TRequest, TResponse>**: MediatR interface for handling a specific request type
> - **Constructor Injection**: Dependencies are injected through the constructor
> - **Repository Pattern**: The handler uses the repository to access data
> - **Mapping**: AutoMapper maps domain entities to DTOs before returning
>
> ⚠️ **Common Mistakes**
>
> - Adding business logic to the handler (it should focus on data retrieval and mapping)
> - Not handling exceptions appropriately
> - Forgetting to use async/await with asynchronous repository methods

### Step 6: Create a Response Wrapper (Optional)

**Introduction**: A response wrapper provides a consistent structure for API responses, including success/failure status and error messages. This is optional but recommended for a more robust implementation.

1. Create a directory for common response types:

    ```bash
    mkdir -p src/MerchStore.Application/Common/Responses
    ```

2. Create a BaseResponse class:

    > `src/MerchStore.Application/Common/Responses/BaseResponse.cs`

    ```csharp
    namespace MerchStore.Application.Common.Responses;

    /// <summary>
    /// Base response class for all operations.
    /// </summary>
    public class BaseResponse
    {
        /// <summary>
        /// Gets or sets a value indicating whether the operation was successful.
        /// </summary>
        public bool IsSuccess { get; set; }
        
        /// <summary>
        /// Gets or sets an error or success message.
        /// </summary>
        public string Message { get; set; } = string.Empty;
        
        /// <summary>
        /// Creates a success response.
        /// </summary>
        /// <param name="message">The success message.</param>
        /// <returns>A success response.</returns>
        public static BaseResponse Success(string message = "Operation completed successfully")
        {
            return new BaseResponse { IsSuccess = true, Message = message };
        }
        
        /// <summary>
        /// Creates a failure response.
        /// </summary>
        /// <param name="message">The error message.</param>
        /// <returns>A failure response.</returns>
        public static BaseResponse Failure(string message)
        {
            return new BaseResponse { IsSuccess = false, Message = message };
        }
    }
    ```

3. Create a `DataResponse<T>` class:

    > `src/MerchStore.Application/Common/Responses/DataResponse.cs`

    ```csharp
    namespace MerchStore.Application.Common.Responses;

    /// <summary>
    /// Generic response class that includes data.
    /// </summary>
    /// <typeparam name="T">The type of data.</typeparam>
    public class DataResponse<T> : BaseResponse
    {
        /// <summary>
        /// Gets or sets the data.
        /// </summary>
        public T? Data { get; set; }
        
        /// <summary>
        /// Creates a success response with data.
        /// </summary>
        /// <param name="data">The data to include in the response.</param>
        /// <param name="message">The success message.</param>
        /// <returns>A success response with data.</returns>
        public static DataResponse<T> Success(T data, string message = "Operation completed successfully")
        {
            return new DataResponse<T> { IsSuccess = true, Message = message, Data = data };
        }
        
        /// <summary>
        /// Creates a failure response.
        /// </summary>
        /// <param name="message">The error message.</param>
        /// <returns>A failure response.</returns>
        public new static DataResponse<T> Failure(string message)
        {
            return new DataResponse<T> { IsSuccess = false, Message = message, Data = default };
        }
    }
    ```

> 💡 **Information**
>
> - **Response Wrapper**: Provides a consistent structure for all responses
> - **Generic Type**: The `DataResponse<T>` class can hold any type of data
> - **Static Factory Methods**: Make it easy to create success or failure responses
>
> ⚠️ **Common Mistakes**
>
> - Overcomplicating response types with too many properties
> - Not handling null data properly

### Step 7: Update the GetProductsQuery to Use the Response Wrapper

**Introduction**: Now that we have a response wrapper, let's update our query and handler to use it for more robust error handling.

1. Update the GetProductsQuery class:

    > `src/MerchStore.Application/Products/Queries/GetProducts/GetProductsQuery.cs`

    ```csharp
    using MediatR;
    using MerchStore.Application.Common.Responses;
    using MerchStore.Application.Products.DTOs;

    namespace MerchStore.Application.Products.Queries.GetProducts;

    /// <summary>
    /// Query to get all products.
    /// </summary>
    public class GetProductsQuery : IRequest<DataResponse<IEnumerable<ProductDto>>>
    {
        // Currently, we don't need any parameters for this query.
        // In the future, we could add parameters for filtering, paging, etc.
    }
    ```

2. Update the GetProductsQueryHandler class:

    > `src/MerchStore.Application/Products/Queries/GetProducts/GetProductsQueryHandler.cs`

    ```csharp
    using AutoMapper;
    using MediatR;
    using MerchStore.Application.Common.Responses;
    using MerchStore.Application.Products.DTOs;
    using MerchStore.Domain.Interfaces;

    namespace MerchStore.Application.Products.Queries.GetProducts;

    /// <summary>
    /// Handler for the GetProductsQuery.
    /// </summary>
    public class GetProductsQueryHandler : IRequestHandler<GetProductsQuery, DataResponse<IEnumerable<ProductDto>>>
    {
        private readonly IProductRepository _productRepository;
        private readonly IMapper _mapper;
        
        /// <summary>
        /// Initializes a new instance of the GetProductsQueryHandler class.
        /// </summary>
        /// <param name="productRepository">The product repository.</param>
        /// <param name="mapper">The AutoMapper instance.</param>
        public GetProductsQueryHandler(IProductRepository productRepository, IMapper mapper)
        {
            _productRepository = productRepository;
            _mapper = mapper;
        }
        
        /// <summary>
        /// Handles the GetProductsQuery.
        /// </summary>
        /// <param name="request">The query.</param>
        /// <param name="cancellationToken">The cancellation token.</param>
        /// <returns>A response containing a collection of product DTOs.</returns>
        public async Task<DataResponse<IEnumerable<ProductDto>>> Handle(GetProductsQuery request, CancellationToken cancellationToken)
        {
            try
            {
                // Get all products from the repository
                var products = await _productRepository.GetAllAsync();
                
                // Map the domain entities to DTOs
                var productDtos = _mapper.Map<IEnumerable<ProductDto>>(products);
                
                // Return a success response with the data
                return DataResponse<IEnumerable<ProductDto>>.Success(productDtos);
            }
            catch (Exception ex)
            {
                // Return a failure response if an error occurs
                return DataResponse<IEnumerable<ProductDto>>.Failure($"Error retrieving products: {ex.Message}");
            }
        }
    }
    ```

> 💡 **Information**
>
> - **Error Handling**: The try-catch block allows handling exceptions and returning a failure response
> - **Success Response**: The data is wrapped in a success response when everything goes well
> - **Cancellation Token**: Passed from the controller to support cancellation of long-running operations
>
> ⚠️ **Common Mistakes**
>
> - Not handling exceptions appropriately
> - Exposing implementation details in error messages (in a production environment, you might want to log the full exception but return a generic message)

### Step 8: Set Up the Dependency Injection

**Introduction**: Now we need to register our services in the dependency injection container.

1. Create a DependencyInjection class in the Application project:

    > `src/MerchStore.Application/DependencyInjection.cs`

    ```csharp
    using System.Reflection;
    using MediatR;
    using Microsoft.Extensions.DependencyInjection;

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
            
            // Register MediatR
            services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(assembly));
            
            // Register AutoMapper
            services.AddAutoMapper(assembly);
            
            return services;
        }
    }
    ```

2. Update the Program.cs file to add the application services:

    > `src/MerchStore.WebUI/Program.cs`

    ```csharp
    // Add this near the beginning of the file, where other using statements are
    using MerchStore.Application;

    // Add this where other services are added
    builder.Services.AddApplication();
    ```

> 💡 **Information**
>
> - **Extension Method**: Makes it easy to register all application services in one call
> - **Assembly Scanning**: MediatR and AutoMapper scan the assembly for handlers and profiles
> - **Chaining**: Returning the service collection allows method chaining
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to register MediatR or AutoMapper
> - Registering services in the wrong assembly

### Step 9: Update the ProductsController to Use MediatR

**Introduction**: Finally, we need to update our ProductsController to use MediatR for retrieving products instead of directly calling the catalog service.

1. Update the ProductsController in the Management area:

    > `src/MerchStore.WebUI/Areas/Management/Controllers/ProductsController.cs`

    ```csharp
    using MediatR;
    using Microsoft.AspNetCore.Mvc;
    using MerchStore.Application.Products.Queries.GetProducts;
    using MerchStore.Application.Services.Interfaces;
    using MerchStore.WebUI.Areas.Management.Models;
    using System;
    using System.Linq;
    using System.Threading.Tasks;
    using AutoMapper;

    namespace MerchStore.WebUI.Areas.Management.Controllers;

    [Area("Management")]
    public class ProductsController : Controller
    {
        private readonly ICatalogService _catalogService;
        private readonly IMediator _mediator;
        private readonly IMapper _mapper;

        public ProductsController(ICatalogService catalogService, IMediator mediator, IMapper mapper)
        {
            _catalogService = catalogService;
            _mediator = mediator;
            _mapper = mapper;
        }

        // GET: Management/Products
        public async Task<IActionResult> Index()
        {
            try
            {
                // Use MediatR to send the query
                var response = await _mediator.Send(new GetProductsQuery());
                
                if (!response.IsSuccess)
                {
                    ViewBag.ErrorMessage = response.Message;
                    return View("Error");
                }
                
                // Map DTOs to ViewModels using AutoMapper
                var viewModels = _mapper.Map<IEnumerable<ProductViewModel>>(response.Data);
                
                return View(viewModels);
            }
            catch (Exception ex)
            {
                // In a real application, log the exception
                ViewBag.ErrorMessage = $"Error loading products: {ex.Message}";
                return View("Error");
            }
        }

        // ... other actions remain the same ...
    }
    ```

> 💡 **Information**
>
> - **Incremental Implementation**: We're only changing the Index action to use CQRS for now
> - **IMediator**: The MediatR interface used to send requests
> - **Mapper**: AutoMapper maps the DTOs to ViewModels
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to inject IMediator and IMapper
> - Not checking the response's IsSuccess property

### Step 10: Create a Mapping Profile for AutoMapper in the Web Project

**Introduction**: We need to create another mapping profile in the Web project to map from DTOs to ViewModels.

1. Create a mapping directory in the Web project:

    ```bash
    mkdir -p src/MerchStore.WebUI/Infrastructure/Mapping
    ```

2. Create a ProductsViewModelMappingProfile class:

    > `src/MerchStore.WebUI/Infrastructure/Mapping/ProductsViewModelMappingProfile.cs`

    ```csharp
    using AutoMapper;
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
        }
    }
    ```

3. Update the Program.cs file to register AutoMapper in the Web project:

    > `src/MerchStore.WebUI/Program.cs`

    ```csharp
    // Update this where you added AddApplication()
    builder.Services.AddApplication();
    
    // Add AutoMapper in the Web project
    builder.Services.AddAutoMapper(typeof(Program).Assembly);
    ```

> 💡 **Information**
>
> - **Separate Profiles**: Each layer has its own mapping profiles
> - **Clear Mappings**: Each mapping is explicitly defined for clarity
> - **Assembly Registration**: The Program.cs assembly is scanned for mapping profiles
>
> ⚠️ **Common Mistakes**
>
> - Conflicting mapping configurations
> - Not registering AutoMapper in both projects

## 🧪 Final Tests

### Run the Application and Validate Your Work

1. Build and run the application:

   ```bash
   dotnet build
   dotnet run --project src/MerchStore.WebUI
   ```

2. Open a browser and navigate to the homepage.

3. Click on the "Management" link in the navigation bar to go to the Management area.

4. Click on "Manage Products" to navigate to the products list.

5. Verify that the products are displayed correctly, now using the CQRS pattern with MediatR.

✅ **Expected Results**

- The Products list displays all products correctly, same as before
- No visible differences for the user, but behind the scenes we're now using CQRS
- The application logs show any MediatR-related operations
- Other product details pages still work using the original service (we'll update these in future exercises)

## 🔧 Troubleshooting

If you encounter issues:

- Check that all packages are installed correctly
- Verify that MediatR and AutoMapper are registered in the dependency injection container
- Look for missing mappings in the profile classes
- Ensure the controller is correctly injecting and using IMediator
- Check that the response wrapper is being used correctly
- Look for any errors in the browser developer console or server logs

## 🚀 Optional Challenge

Want to take your learning further? Try:

- Add a simple validation mechanism for the query using FluentValidation
- Implement pagination for the GetProductsQuery to limit the number of products returned
- Add sorting options to the query (e.g., by name, price, or stock quantity)
- Create a simple logging behavior for MediatR to log all requests and responses
- Add server-side filtering for products based on name or description

## 📚 Further Reading

- [CQRS Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs) - Microsoft's documentation on the CQRS pattern
- [MediatR Documentation](https://github.com/jbogard/MediatR/wiki) - Official MediatR wiki
- [AutoMapper Documentation](https://docs.automapper.org/) - Official AutoMapper documentation
- [Implementing CQRS and Mediator Patterns](https://www.youtube.com/watch?v=YzOBrVlthMk) - A comprehensive video tutorial
- [Clean Architecture with ASP.NET Core](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) - Uncle Bob's article on Clean Architecture

## Done! 🎉

Great job! You've taken the first step towards implementing the CQRS pattern in your application. You've learned how to use MediatR to decouple your controllers from the data access logic, and how to use AutoMapper to map between different object types.

This lays the foundation for implementing the full CQRS pattern in future exercises, where you'll add queries for retrieving individual products, as well as commands for creating, updating, and deleting products.

Remember that CQRS is about separating the responsibility of reading and writing data. In this exercise, we focused on the "Query" side of CQRS, implementing the pattern for retrieving a list of products. In future exercises, we'll explore the "Command" side by implementing commands for modifying data.
