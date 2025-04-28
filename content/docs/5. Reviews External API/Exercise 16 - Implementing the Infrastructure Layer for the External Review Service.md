---
title: "Implementing the Infrastructure Layer for the External Review Service"
date: "2025-04-28"
lastmod: "2025-04-28"
draft: false
weight: 503
toc: true
---

## 🎯 Goal

Implement the infrastructure layer components needed to integrate with an external review API while maintaining clean architecture principles and implementing resilience patterns like the Circuit Breaker.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 14 (Implementing the Domain Layer for Reviews)
- Understand basic HTTP client operations in .NET
- Be familiar with dependency injection concepts
- Have basic knowledge of resilience patterns
- Understand configuration management in ASP.NET Core

## 📚 Learning Objectives

By the end of this exercise, you will:

- Implement the **Repository Pattern** for external API integration
- Configure and use **HttpClient** for API communication
- Apply the **Circuit Breaker Pattern** for fault tolerance
- Create **DTOs** for API request/response mapping
- Use **Options Pattern** for external service configuration
- Implement a **Fallback Mechanism** for handling failures
- Understand **Resilient Communication** with external services

## 🔍 Why This Matters

In real-world applications, integrating with external services is crucial, and doing it properly is important because:

- External services can fail, be slow, or become unavailable
- Proper resilience patterns prevent cascading failures in your application
- Clean separation of concerns makes your code more maintainable
- Abstraction of external service details allows for easier testing and changes
- External service configuration often needs to change between environments
- Fallback mechanisms provide graceful degradation of functionality
- Understanding these patterns is essential for building reliable distributed systems

## 📝 Step-by-Step Instructions

### Step 1: Create Data Transfer Objects for the External API

**Introduction**: First, we'll create DTOs that match the structure of the external API's responses. These objects help us map between the external API's data format and our domain models.

1. Create a directory structure for the external reviews service:

    ```bash
    mkdir -p src/MerchStore.Infrastructure/ExternalServices/Reviews/Models
    ```

2. Create the basic DTO for a review:

    > `src/MerchStore.Infrastructure/ExternalServices/Reviews/Models/ReviewDto.cs`

    ```csharp
    namespace MerchStore.Infrastructure.ExternalServices.Reviews.Models;

    public class ReviewDto
    {
        public string? Id { get; set; }
        public string? ProductId { get; set; }
        public string? CustomerName { get; set; }
        public string? Title { get; set; }
        public string? Content { get; set; }
        public int Rating { get; set; }
        public DateTime CreatedAt { get; set; }
        public string? Status { get; set; }
    }
    ```

3. Create the DTO for review statistics:

    > `src/MerchStore.Infrastructure/ExternalServices/Reviews/Models/ReviewStatsDto.cs`

    ```csharp
    namespace MerchStore.Infrastructure.ExternalServices.Reviews.Models;

    public class ReviewStatsDto
    {
        public string? ProductId { get; set; }
        public double AverageRating { get; set; }
        public int ReviewCount { get; set; }
    }
    ```

4. Create the DTO for the combined response:

    > `src/MerchStore.Infrastructure/ExternalServices/Reviews/Models/ReviewResponseDto.cs`

    ```csharp
    namespace MerchStore.Infrastructure.ExternalServices.Reviews.Models;

    public class ReviewResponseDto
    {
        public List<ReviewDto>? Reviews { get; set; }
        public ReviewStatsDto? Stats { get; set; }
    }
    ```

> 💡 **Information**
>
> - **Data Transfer Objects (DTOs)**: Simple classes that match the structure of external API responses
> - **Nullable Properties**: Using nullable reference types to handle potential missing data from the API
> - **Separate Models**: Keeping external API models separate from domain models maintains clean separation
> - **Model Hierarchy**: Matching the API's response structure with nested objects
>
> ⚠️ **Common Mistakes**
>
> - Using domain entities directly for API communication, which violates clean architecture
> - Not handling nullable or missing properties from external APIs
> - Creating overly complex DTOs that don't match the actual API response
> - Exposing external API models to application or domain layers

### Step 2: Create Configuration Options for the External API

**Introduction**: We'll use the Options Pattern to manage the configuration for the external review API. This approach provides strongly-typed access to configuration and allows for different settings in different environments.

1. Create a directory for the configuration:

    ```bash
    mkdir -p src/MerchStore.Infrastructure/ExternalServices/Reviews/Configurations
    ```

2. Create the options class:

    > `src/MerchStore.Infrastructure/ExternalServices/Reviews/Configurations/ReviewApiOptions.cs`

    ```csharp
    namespace MerchStore.Infrastructure.ExternalServices.Reviews.Configurations;

    public class ReviewApiOptions
    {
        public const string SectionName = "ReviewApi";
        
        public string BaseUrl { get; set; } = string.Empty;
        public string ApiKey { get; set; } = string.Empty;
        public string ApiKeyHeaderName { get; set; } = "x-functions-key";
        public int TimeoutSeconds { get; set; } = 30;
        
        // Circuit breaker settings
        public int ExceptionsAllowedBeforeBreaking { get; set; } = 3;
        public int CircuitBreakerDurationSeconds { get; set; } = 30;
    }
    ```

3. Update the application configuration in `appsettings.json`:

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
      },
      "ReviewApi": {
        "BaseUrl": "https://reviewapifunc250420.azurewebsites.net/api/",
        "ApiKey": "campusmolndal",
        "ApiKeyHeaderName": "x-functions-key",
        "TimeoutSeconds": 10,
        "ExceptionsAllowedBeforeBreaking": 3,
        "CircuitBreakerDurationSeconds": 30
      }
    }
    ```

> 💡 **Information**
>
> - **Options Pattern**: A clean way to access configuration values with strong typing
> - **Default Values**: Providing sensible defaults in case configuration is missing
> - **Grouped Settings**: Related settings are grouped under a single section
> - **Circuit Breaker Configuration**: Including settings that control resilience behavior
>
> ⚠️ **Common Mistakes**
>
> - Hardcoding configuration values instead of using the options pattern
> - Not providing default values for optional configuration
> - Storing sensitive information like API keys in source control
> - Not separating configuration for different environments

### Step 3: Implement the HTTP Client for the External API

**Introduction**: Now, we'll create a dedicated HTTP client for communicating with the external review API. This client will handle the low-level HTTP details and provide a clean interface for making API requests.

1. Create the HTTP client class:

    > `src/MerchStore.Infrastructure/ExternalServices/Reviews/ReviewApiClient.cs`

    ```csharp
    using System.Net.Http.Json;
    using Microsoft.Extensions.Logging;
    using Microsoft.Extensions.Options;
    using MerchStore.Infrastructure.ExternalServices.Reviews.Models;
    using MerchStore.Infrastructure.ExternalServices.Reviews.Configurations;
    using System.Text.Json;

    namespace MerchStore.Infrastructure.ExternalServices.Reviews;

    public class ReviewApiClient
    {
        private readonly HttpClient _httpClient;
        private readonly ILogger<ReviewApiClient> _logger;
        private readonly ReviewApiOptions _options;

        // Options for pretty-printing JSON
        private static readonly JsonSerializerOptions _prettyJsonOptions = new() { WriteIndented = true };

        public ReviewApiClient(
            HttpClient httpClient,
            IOptions<ReviewApiOptions> options,
            ILogger<ReviewApiClient> logger)
        {
            _httpClient = httpClient;
            _logger = logger;
            _options = options.Value;
            
            // Configure the HttpClient
            _httpClient.BaseAddress = new Uri(_options.BaseUrl);
            _httpClient.DefaultRequestHeaders.Add(_options.ApiKeyHeaderName, _options.ApiKey);
            _httpClient.Timeout = TimeSpan.FromSeconds(_options.TimeoutSeconds);
        }

        public async Task<ReviewResponseDto?> GetProductReviewsAsync(Guid productId)
        {
            try
            {
                string url = $"products/{productId}/reviews";
                
                _logger.LogInformation("Requesting reviews for product {ProductId} from external API", productId);
                
                var response = await _httpClient.GetAsync(url);
                response.EnsureSuccessStatusCode();
                
                var reviewsResponse = await response.Content.ReadFromJsonAsync<ReviewResponseDto>();
                
                _logger.LogInformation("Successfully retrieved {ReviewCount} reviews for product {ProductId}", 
                    reviewsResponse?.Reviews?.Count ?? 0, productId);

                // Log pretty-printed JSON response in debug mode
                if (_logger.IsEnabled(LogLevel.Debug))
                {
                    var json = JsonSerializer.Serialize(reviewsResponse, _prettyJsonOptions);
                    _logger.LogDebug("Received response: {Json}", json);
                }
                    
                return reviewsResponse;
            }
            catch (HttpRequestException ex)
            {
                _logger.LogError(ex, "HTTP error occurred while fetching reviews for product {ProductId}: {Message}", 
                    productId, ex.Message);
                throw;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Unexpected error occurred while fetching reviews for product {ProductId}: {Message}", 
                    productId, ex.Message);
                throw;
            }
        }
    }
    ```

> 💡 **Information**
>
> - **Typed HttpClient**: A dedicated HttpClient for the review API with specific configuration
> - **Dependency Injection**: The client receives its dependencies via constructor injection
> - **Configuration**: Using the options pattern to configure the HttpClient
> - **Error Handling**: Catching and logging specific exceptions while preserving the exception for the caller
> - **Logging**: Comprehensive logging for debugging and monitoring
>
> ⚠️ **Common Mistakes**
>
> - Creating new HttpClient instances for each request (HttpClient is designed to be reused)
> - Not configuring timeout values, which can lead to hung requests
> - Missing error handling or swallowing exceptions without proper logging
> - Not using typed models for API responses, making deserialization error-prone

### Step 4: Create a Mock Service for Fallback Data

**Introduction**: When the external API is unavailable, we need a fallback mechanism. This mock service will provide synthetic review data that can be used when the real API fails.

1. Create the mock service:

    > `src/MerchStore.Infrastructure/ExternalServices/Reviews/MockReviewService.cs`

    ```csharp
    using MerchStore.Domain.Entities;
    using MerchStore.Domain.Enums;
    using MerchStore.Domain.ValueObjects;

    namespace MerchStore.Infrastructure.ExternalServices.Reviews;

    /// <summary>
    /// Provides mock review data for use as a fallback when the external API is unavailable
    /// </summary>
    public class MockReviewService
    {
        private static readonly Random _random = new Random();
        private static readonly string[] _customerNames = { "John Doe", "Jane Smith", "Bob Johnson", "Alice Brown", "Charlie Davis" };
        private static readonly string[] _reviewContents = {
            "I've been using this for weeks and it's fantastic.",
            "Exactly what I was looking for. High quality.",
            "The product is decent but shipping took too long.",
            "Works as advertised, very happy with my purchase.",
            "Good value for the money, would buy again."
        };

        /// <summary>
        /// Generates mock reviews for a product
        /// </summary>
        public (IEnumerable<Review> Reviews, ReviewStats Stats) GetProductReviews(Guid productId)
        {
            // Generate a consistent number of reviews based on product ID
            // This ensures the same product always gets the same number of reviews
            var productIdHash = productId.GetHashCode();
            var reviewCount = Math.Abs(productIdHash % 6); // 0-5 reviews

            var reviews = GenerateMockReviews(productId, reviewCount);
            
            // Calculate average rating
            double averageRating = reviews.Any() 
                ? Math.Round(reviews.Average(r => r.Rating), 1) 
                : 0;
                
            var stats = new ReviewStats(productId, averageRating, reviewCount);
            
            return (reviews, stats);
        }

        private IEnumerable<Review> GenerateMockReviews(Guid productId, int count)
        {
            var reviews = new List<Review>();
            var productSeed = productId.GetHashCode();
            var random = new Random(productSeed); // Use product ID as seed for consistency

            for (int i = 0; i < count; i++)
            {
                // Use a deterministic approach to generate review data
                int dayOffset = random.Next(1, 31);
                var createdAt = DateTime.UtcNow.AddDays(-dayOffset);
                
                int ratingBase = random.Next(1, 101);
                int rating = ratingBase switch
                {
                    var n when n <= 10 => 1, // 10% chance of 1-star
                    var n when n <= 25 => 2, // 15% chance of 2-stars
                    var n when n <= 50 => 3, // 25% chance of 3-stars
                    var n when n <= 80 => 4, // 30% chance of 4-stars
                    _ => 5                   // 20% chance of 5-stars
                };

                string title = $"Sample Review: {i+1} for Product";
                string customerName = _customerNames[random.Next(_customerNames.Length)];
                string content = _reviewContents[random.Next(_reviewContents.Length)];

                reviews.Add(new Review(
                    Guid.NewGuid(),
                    productId,
                    customerName,
                    title,
                    content,
                    rating,
                    createdAt,
                    ReviewStatus.Approved
                ));
            }

            // Sort by date descending (newest first)
            return reviews.OrderByDescending(r => r.CreatedAt).ToList();
        }
    }
    ```

> 💡 **Information**
>
> - **Fallback Mechanism**: Creates synthetic but realistic-looking review data when the real API is unavailable
> - **Deterministic Generation**: Using the product ID as a seed ensures consistent results for the same product
> - **Realistic Distribution**: Generating ratings with a realistic distribution (not just random numbers)
> - **Domain Model Usage**: Returns data directly in the domain model format
>
> ⚠️ **Common Mistakes**
>
> - Generating completely random data each time, which would create inconsistent user experiences
> - Not matching the domain model's validation rules in the mock data
> - Creating unrealistic test data that doesn't represent real-world scenarios
> - Over-complicating the mock implementation with unnecessary features

### Step 5: Implement the Repository with Circuit Breaker Pattern

**Introduction**: Now, we'll implement the `IReviewRepository` interface using the Circuit Breaker pattern. This pattern prevents cascading failures by stopping requests to a failing service temporarily and providing a fallback.

1. Install the Polly package for implementing the Circuit Breaker pattern:

    ```bash
    dotnet add src/MerchStore.Infrastructure package Polly
    ```

2. Create the repository implementation with Circuit Breaker:

    > `src/MerchStore.Infrastructure/ExternalServices/Reviews/ExternalReviewRepository.cs`

    ```csharp
    using MerchStore.Domain.Entities;
    using MerchStore.Domain.Enums;
    using MerchStore.Domain.Interfaces;
    using MerchStore.Domain.ValueObjects;
    using MerchStore.Infrastructure.ExternalServices.Reviews.Configurations;
    using Microsoft.Extensions.Logging;
    using Microsoft.Extensions.Options;
    using Polly;
    using Polly.CircuitBreaker;

    namespace MerchStore.Infrastructure.ExternalServices.Reviews;

    /// <summary>
    /// Repository implementation that integrates with the external review API
    /// and implements circuit breaker pattern for resilience
    /// </summary>
    public class ExternalReviewRepository : IReviewRepository
    {
        private readonly ReviewApiClient _apiClient;
        private readonly MockReviewService _mockReviewService;
        private readonly ILogger<ExternalReviewRepository> _logger;
        private readonly AsyncCircuitBreakerPolicy _circuitBreakerPolicy;

        public ExternalReviewRepository(
            ReviewApiClient apiClient,
            MockReviewService mockReviewService,
            IOptions<ReviewApiOptions> options,
            ILogger<ExternalReviewRepository> logger)
        {
            _apiClient = apiClient;
            _mockReviewService = mockReviewService;
            _logger = logger;
            
            var circuitOptions = options.Value;
            
            // Configure the circuit breaker policy
            _circuitBreakerPolicy = Policy
                .Handle<HttpRequestException>()
                .Or<TimeoutException>()
                .CircuitBreakerAsync(
                    exceptionsAllowedBeforeBreaking: circuitOptions.ExceptionsAllowedBeforeBreaking,
                    durationOfBreak: TimeSpan.FromSeconds(circuitOptions.CircuitBreakerDurationSeconds),
                    onBreak: (ex, breakDuration) => 
                    {
                        _logger.LogWarning(
                            "Circuit breaker opened for {BreakDuration} after {ExceptionType}: {ExceptionMessage}",
                            breakDuration, ex.GetType().Name, ex.Message);
                    },
                    onReset: () => 
                    {
                        _logger.LogInformation("Circuit breaker reset, external API calls resumed");
                    },
                    onHalfOpen: () => 
                    {
                        _logger.LogInformation("Circuit breaker half-open, testing external API");
                    }
                );
        }

        public async Task<(IEnumerable<Review> Reviews, ReviewStats Stats)> GetProductReviewsAsync(Guid productId)
        {
            try
            {
                // Use circuit breaker to call the external API
                return await _circuitBreakerPolicy.ExecuteAsync(async () => 
                {
                    var response = await _apiClient.GetProductReviewsAsync(productId);
                    
                    if (response?.Reviews == null || response.Stats == null)
                    {
                        throw new InvalidOperationException("External API returned incomplete data");
                    }
                    
                    // Map the external DTOs to domain entities
                    var reviews = response.Reviews.Select(r => new Review(
                        Guid.Parse(r.Id ?? Guid.NewGuid().ToString()),
                        Guid.Parse(r.ProductId ?? productId.ToString()),
                        r.CustomerName ?? "Unknown",
                        r.Title ?? "No Title",
                        r.Content ?? "No Content",
                        r.Rating,
                        r.CreatedAt,
                        ParseReviewStatus(r.Status)
                    )).ToList();
                    
                    var stats = new ReviewStats(
                        productId, 
                        response.Stats.AverageRating,
                        response.Stats.ReviewCount
                    );
                    
                    return (reviews, stats);
                });
            }
            catch (BrokenCircuitException)
            {
                _logger.LogWarning("Circuit is open, using mock review service for product {ProductId}", productId);
                return _mockReviewService.GetProductReviews(productId);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error getting reviews for product {ProductId}, falling back to mock data", productId);
                return _mockReviewService.GetProductReviews(productId);
            }
        }

        private static ReviewStatus ParseReviewStatus(string? status)
        {
            return status?.ToLowerInvariant() switch
            {
                "approved" => ReviewStatus.Approved,
                "rejected" => ReviewStatus.Rejected,
                "pending" => ReviewStatus.Pending,
                _ => ReviewStatus.Pending
            };
        }
    }
    ```

> 💡 **Information**
>
> - **Circuit Breaker Pattern**: Prevents cascading failures by "opening the circuit" after consecutive failures
> - **Three Circuit States**:
>   - **Closed**: Normal operation, requests pass through
>   - **Open**: After too many failures, no requests are attempted for a period
>   - **Half-Open**: After the break duration, allows a test request to see if the service has recovered
> - **Fallback Strategy**: Falls back to the mock service when the API is unavailable
> - **Exception Handling**: Different handling for circuit-broken state vs. other exceptions
> - **DTO to Domain Mapping**: Transforms API DTOs to domain entities with proper validation
>
> ⚠️ **Common Mistakes**
>
> - Not handling missing or null data from the external API
> - Configuring circuit breaker thresholds that are too high or too low
> - Missing fallback strategy when the circuit is open
> - Not logging circuit state changes for monitoring

### Step 6: Register the Infrastructure Services

**Introduction**: Finally, we'll register all the services we've created with the dependency injection container to make them available to the application.

1. Install required packages:

    ```bash
    dotnet add src/MerchStore.Infrastructure/MerchStore.Infrastructure.csproj package Microsoft.Extensions.Http
    ```

2. Update the Infrastructure layer's dependency injection extension methods:

    > `src/MerchStore.Infrastructure/DependencyInjection.cs`

    ```csharp
   using Microsoft.EntityFrameworkCore;
    using Microsoft.Extensions.Configuration;
    using Microsoft.Extensions.DependencyInjection;
    using MerchStore.Application.Common.Interfaces;
    using MerchStore.Domain.Interfaces;
    using MerchStore.Infrastructure.Persistence;
    using MerchStore.Infrastructure.Persistence.Repositories;
    using MerchStore.Infrastructure.ExternalServices.Reviews.Configurations;
    using MerchStore.Infrastructure.ExternalServices.Reviews;
    
    namespace MerchStore.Infrastructure;
    
    /// <summary>
    /// Contains extension methods for registering Infrastructure layer services with the dependency injection container.
    /// This keeps all registration logic in one place and makes it reusable.
    /// </summary>
    public static class DependencyInjection
    {
        /// <summary>
        /// Adds Infrastructure layer services to the DI container
        /// </summary>
        /// <param name="services">The service collection to add services to</param>
        /// <param name="configuration">The configuration for database connection strings</param>
        /// <returns>The service collection for chaining</returns>
        public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration configuration)
        {
            // Call specific registration methods
            services.AddPersistenceServices(configuration);
            services.AddReviewServices(configuration);
            // Add calls to other infrastructure registration methods here if needed (e.g., file storage, email service)
    
            return services;
        }
    
        /// <summary>
        /// Registers services related to data persistence (EF Core, Repositories, UnitOfWork).
        /// </summary>
        /// <param name="services">The service collection.</param>
        /// <param name="configuration">The application configuration.</param>
        /// <returns>The service collection for chaining.</returns>
        public static IServiceCollection AddPersistenceServices(this IServiceCollection services, IConfiguration configuration)
        {
            // Register DbContext with in-memory database
            // In a real application, you'd use a real database
            services.AddDbContext<AppDbContext>(options =>
                options.UseInMemoryDatabase("MerchStoreDb"));
    
            // Register repositories
            services.AddScoped<IProductRepository, ProductRepository>();
    
            // Register Unit of Work
            services.AddScoped<IUnitOfWork, UnitOfWork>();
    
            // Register Repository Manager
            services.AddScoped<IRepositoryManager, RepositoryManager>();
    
            // Add logging services if not already added
            services.AddLogging();
    
            // Register DbContext seeder
            services.AddScoped<AppDbContextSeeder>();
    
            return services;
        }
    
        /// <summary>
        /// Registers services related to the External Review API integration.
        /// </summary>
        /// <param name="services">The service collection.</param>
        /// <param name="configuration">The application configuration.</param>
        /// <returns>The service collection for chaining.</returns>
        public static IServiceCollection AddReviewServices(this IServiceCollection services, IConfiguration configuration)
        {
            // Register External Api options
            services.Configure<ReviewApiOptions>(configuration.GetSection(ReviewApiOptions.SectionName));
    
            // Register HttpClient for ReviewApiClient
            services.AddHttpClient<ReviewApiClient>()
                .SetHandlerLifetime(TimeSpan.FromMinutes(5)); // Set a lifetime for the handler
    
            // Register the mock service
            services.AddSingleton<MockReviewService>();
    
            // Register the repository with the circuit breaker
            services.AddScoped<IReviewRepository, ExternalReviewRepository>();
    
            return services;
        }
    
        /// <summary>
        /// Seeds the database with initial data.
        /// This is an extension method on IServiceProvider to allow it to be called from Program.cs.
        /// </summary>
        /// <param name="serviceProvider">The service provider to resolve dependencies</param>
        /// <returns>A task representing the asynchronous operation</returns>
        public static async Task SeedDatabaseAsync(this IServiceProvider serviceProvider)
        {
            using var scope = serviceProvider.CreateScope();
            var seeder = scope.ServiceProvider.GetRequiredService<AppDbContextSeeder>();
            await seeder.SeedAsync();
        }
    }
    ```

> 💡 **Information**
>
> - **Separation of Concerns**: Using separate methods for different types of services makes the code more maintainable
> - **Options Configuration**: Registering the options pattern with the appropriate configuration section
> - **HttpClient Factory**: Using the factory pattern for HttpClient registration, which handles proper lifecycle management
> - **Service Lifetimes**: Using appropriate lifetimes for different types of services
>   - **Singleton**: Used for the mock service since it's stateless and can be shared
>   - **Scoped**: Used for repositories and other services that should be per-request
>
> ⚠️ **Common Mistakes**
>
> - Using the wrong service lifetime for services, which can lead to memory leaks or unexpected behavior
> - Forgetting to register required services or dependencies
> - Not configuring typed HttpClient correctly
> - Mixing concerns between different types of services

## 🔍 Understanding the Circuit Breaker Pattern

The Circuit Breaker pattern is a resilience pattern that prevents cascading failures in distributed systems. It's named after electrical circuit breakers that protect electrical circuits from damage due to excess current.

### How the Circuit Breaker Works

The circuit breaker has three states:

1. **Closed State (Normal Operation)**
   - Initial state where all calls pass through normally
   - Each failure is counted
   - When failures reach a threshold, the circuit transitions to Open state

2. **Open State (Failure Operation)**
   - All calls immediately return with an error (BrokenCircuitException in Polly)
   - No actual calls are made to the failing service
   - After a specified timeout period, the circuit transitions to Half-Open state

3. **Half-Open State (Testing Recovery)**
   - Allows a limited number of calls through to test if the service has recovered
   - If these calls succeed, the circuit returns to Closed state
   - If these calls fail, the circuit returns to Open state

### Circuit Breaker Implementation with Polly

In our implementation, we're using Polly's `CircuitBreakerAsync` method with several parameters:

```csharp
_circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .Or<TimeoutException>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: circuitOptions.ExceptionsAllowedBeforeBreaking,
        durationOfBreak: TimeSpan.FromSeconds(circuitOptions.CircuitBreakerDurationSeconds),
        onBreak: (ex, breakDuration) => { /* Log circuit opening */ },
        onReset: () => { /* Log circuit closing */ },
        onHalfOpen: () => { /* Log circuit half-open */ }
    );
```

Let's break this down:

- **`Handle<HttpRequestException>().Or<TimeoutException>()`**: Specifies which exceptions the circuit breaker should monitor
- **exceptionsAllowedBeforeBreaking**: Number of consecutive failures before opening the circuit
- **durationOfBreak**: How long the circuit stays open before trying again
- **onBreak, onReset, onHalfOpen**: Event handlers for state changes

### Using the Circuit Breaker with ExecuteAsync

To use the circuit breaker, we wrap our API call with the policy:

```csharp
return await _circuitBreakerPolicy.ExecuteAsync(async () => 
{
    // API call logic here
});
```

When the circuit is open, this will throw a `BrokenCircuitException` which we catch and handle by falling back to the mock service.

### Benefits of the Circuit Breaker Pattern

1. **Fail Fast**: Prevents cascading failures by quickly rejecting calls to a failing service
2. **Self-Healing**: Automatically tests if the service has recovered
3. **Resilience**: Makes your application robust against external service failures
4. **Graceful Degradation**: Allows for fallback strategies when services are down
5. **Monitoring**: State changes provide valuable metrics for system health

## 🧪 Final Tests

### Verify Your Infrastructure Implementation

1. Ensure all required packages are installed:

    ```bash
    dotnet restore
    ```

2. Build the solution to check for compilation errors:

    ```bash
    dotnet build
    ```

3. Verify the configuration in appsettings.json has the correct values for the review API.

✅ **Expected Results**

- The application should build successfully
- All services should be properly registered with dependency injection
- The infrastructure layer should be ready to connect to the external review API

## 🔧 Troubleshooting

If you encounter issues:

- Check that all required packages are properly installed
- Verify the ReviewApi configuration in appsettings.json
- Ensure the external API URL is accessible from your development environment
- Check for proper exception handling and fallback mechanisms
- Verify the circuit breaker configuration values are reasonable for your scenario

## 🚀 Optional Challenge

Want to take your learning further? Try:

1. Implement caching for review data to reduce API calls
2. Add metrics collection to track API performance and circuit breaker state
3. Create a more sophisticated fallback strategy that combines cached data with mock data
4. Implement retry policies for transient failures before the circuit breaker trips
5. Create a health check endpoint that reports the status of the external API connection

## 📚 Further Reading

- [Circuit Breaker Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker) - Microsoft's explanation of the pattern
- [Polly Documentation](https://github.com/App-vNext/Polly) - Comprehensive guide to Polly, the resilience framework
- [HttpClientFactory in ASP.NET Core](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests) - Best practices for HTTP clients
- [Resilience Patterns](https://docs.microsoft.com/en-us/azure/architecture/patterns/category/resiliency) - Additional patterns for building resilient systems
- [Options Pattern in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options) - Microsoft's guide to the options pattern

## Done! 🎉

Great job! You've successfully implemented the infrastructure layer for the external review service integration. Through this exercise, you've learned how to:

- Create DTOs to map between external API data and your domain models
- Use the Options Pattern for strongly-typed configuration management
- Implement a dedicated HTTP client for the external API
- Apply the Circuit Breaker Pattern to handle failures gracefully
- Provide a fallback mechanism with mock data when the external service is unavailable
- Register all components properly in the dependency injection container

This implementation maintains clean architecture principles by keeping the external service integration details isolated in the infrastructure layer, while still exposing the functionality through the domain interfaces. The resilience patterns you've implemented will help ensure your application remains responsive even when external services fail.

In a real-world application, integrating with external services is common, and knowing how to do this properly with appropriate error handling and resilience patterns is an essential skill. The concepts you've learned here—especially the Circuit Breaker pattern—can be applied to many different types of external service integrations beyond just reviews.

In the next exercise, we'll build on this foundation by implementing the application layer service that will use this infrastructure to provide review functionality to the presentation layer of our application. 🚀
