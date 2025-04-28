---
title: "Integration Testing the External Review Service"
date: "2025-04-28"
lastmod: "2025-04-28"
draft: false
weight: 504
toc: true
---

## 🎯 Goal

Create comprehensive integration tests for the External Review Service infrastructure to verify it works properly with the real API and handles failure scenarios gracefully, without requiring a UI.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 16 (Implementing the Infrastructure Layer for the External Review Service)
- Understand basic xUnit testing concepts
- Be familiar with integration testing principles
- Know how to work with mock objects and dependency injection
- Understand logging and configuration in .NET

## 📚 Learning Objectives

By the end of this exercise, you will:

- Create an **integration test project** for infrastructure components
- Configure **test-specific settings** in appsettings.json
- Build a **test fixture** for shared setup and teardown logic
- Write tests that verify **successful API connectivity**
- Implement tests for the **Circuit Breaker pattern**
- Use **test categorization** with xUnit traits
- Configure **detailed logging** for test diagnostics
- Learn how to run **conditional test execution** with filters

## 🔍 Why This Matters

In real-world applications, integration testing external services is crucial because:

- It verifies that your integration code works with the actual external service
- It helps catch configuration issues before deployment
- It validates your resilience patterns (like Circuit Breaker) work as expected
- It provides confidence without needing a UI or manual testing
- It serves as living documentation of how the external service behaves
- It gives you an early warning when the external API changes
- It helps you understand performance characteristics and timeouts

## 📝 Step-by-Step Instructions

### Step 1: Create and Configure the Integration Test Project

**Introduction**: First, we'll create a dedicated test project for our infrastructure integration tests and configure it with the necessary packages and settings.

1. Create a directory for your test project:

    ```bash
    mkdir -p tests/MerchStore.Infrastructure.IntegrationTests
    ```

2. Create a new test project:

    ```bash
    dotnet new xunit -o tests/MerchStore.Infrastructure.IntegrationTests
    ```

3. Add test project to solution:

    ```bash
    dotnet sln add tests/MerchStore.Infrastructure.IntegrationTests/MerchStore.Infrastructure.IntegrationTests.csproj
    ```

4. Add necessary package references and project references:

    ```bash
    dotnet add tests/MerchStore.Infrastructure.IntegrationTests/MerchStore.Infrastructure.IntegrationTests.csproj reference src/MerchStore.Infrastructure/MerchStore.Infrastructure.csproj
    dotnet add tests/MerchStore.Infrastructure.IntegrationTests/MerchStore.Infrastructure.IntegrationTests.csproj package Microsoft.Extensions.Configuration
    dotnet add tests/MerchStore.Infrastructure.IntegrationTests/MerchStore.Infrastructure.IntegrationTests.csproj package Microsoft.Extensions.Configuration.Binder
    dotnet add tests/MerchStore.Infrastructure.IntegrationTests/MerchStore.Infrastructure.IntegrationTests.csproj package Microsoft.Extensions.Configuration.Json
    dotnet add tests/MerchStore.Infrastructure.IntegrationTests/MerchStore.Infrastructure.IntegrationTests.csproj package Microsoft.Extensions.DependencyInjection
    dotnet add tests/MerchStore.Infrastructure.IntegrationTests/MerchStore.Infrastructure.IntegrationTests.csproj package Microsoft.Extensions.Logging
    dotnet add tests/MerchStore.Infrastructure.IntegrationTests/MerchStore.Infrastructure.IntegrationTests.csproj package Microsoft.Extensions.Logging.Console
    dotnet add tests/MerchStore.Infrastructure.IntegrationTests/MerchStore.Infrastructure.IntegrationTests.csproj package Microsoft.Extensions.Logging.Debug
    ```

5. Create an appsettings.json file in the test project:

    > `tests/MerchStore.Infrastructure.IntegrationTests/appsettings.json`

    ```json
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft.AspNetCore": "Warning",
          "MerchStore.Infrastructure": "Debug",
          "System.Net.Http.HttpClient": "Warning"
        }
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

6. Update your test project file to ensure appsettings.json is copied to the output directory:

    > `tests/MerchStore.Infrastructure.IntegrationTests/MerchStore.Infrastructure.IntegrationTests.csproj`

    ```xml
    <Project Sdk="Microsoft.NET.Sdk">

      <PropertyGroup>
        <TargetFramework>net9.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <Nullable>enable</Nullable>
        <IsPackable>false</IsPackable>
      </PropertyGroup>

      <ItemGroup>
        <PackageReference Include="coverlet.collector" Version="6.0.2" />
        <PackageReference Include="Microsoft.Extensions.Configuration" Version="9.0.4" />
        <PackageReference Include="Microsoft.Extensions.Configuration.Binder" Version="9.0.4" />
        <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="9.0.4" />
        <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="9.0.4" />
        <PackageReference Include="Microsoft.Extensions.Logging" Version="9.0.4" />
        <PackageReference Include="Microsoft.Extensions.Logging.Console" Version="9.0.4" />
        <PackageReference Include="Microsoft.Extensions.Logging.Debug" Version="9.0.4" />
        <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.11.1" />
        <PackageReference Include="xunit" Version="2.9.2" />
        <PackageReference Include="xunit.runner.visualstudio" Version="2.8.2" />
      </ItemGroup>

      <ItemGroup>
        <Using Include="Xunit" />
      </ItemGroup>
      
      <ItemGroup>
        <ProjectReference Include="..\..\src\MerchStore.Infrastructure\MerchStore.Infrastructure.csproj" />
      </ItemGroup>

      <ItemGroup>
        <None Update="appsettings.json">
          <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
        </None>
      </ItemGroup>

    </Project>
    ```

> 💡 **Information**
>
> - **CopyToOutputDirectory**: The special "PreserveNewest" setting ensures your appsettings.json file is copied to the build output directory
> - **Debug Log Level**: Setting the MerchStore.Infrastructure logger to Debug enables JSON response logging
> - **Project Reference**: Directly referencing the Infrastructure project gives access to all its classes, even non-public ones
> - **Logging Packages**: Including Console and Debug logging helps with test diagnostics
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to copy appsettings.json to the output directory
> - Missing necessary package references for configuration and logging
> - Using different configuration values than the main application

### Step 2: Create a Test Fixture for Shared Setup

**Introduction**: A test fixture provides shared setup code that can be reused across multiple test classes. This is especially useful for integration tests where setup is often complex.

1. Create a test fixture for the ReviewApi integration tests:

    > `tests/MerchStore.Infrastructure.IntegrationTests/ReviewApiIntegrationTestFixture.cs`

    ```csharp
    using Microsoft.Extensions.Configuration;
    using Microsoft.Extensions.DependencyInjection;
    using Microsoft.Extensions.Logging;

    namespace MerchStore.Infrastructure.IntegrationTests;

    /// <summary>
    /// Sets up the necessary configuration and dependency injection container
    /// for running integration tests against the real External Review API.
    /// This fixture is created once per test class.
    /// </summary>
    public class ReviewApiIntegrationTestFixture : IDisposable
    {
        public IServiceProvider ServiceProvider { get; private set; }
        public IConfiguration Configuration { get; private set; }

        public ReviewApiIntegrationTestFixture()
        {
            // Build configuration
            Configuration = new ConfigurationBuilder()
                // Set base path to the directory where the test assembly is running
                .SetBasePath(Directory.GetCurrentDirectory())
                // Add the appsettings.json file we created for tests
                .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
                // Optionally add environment variables or other sources
                .Build();

            // Set up dependency injection
            var services = new ServiceCollection();

            // Add logging (optional, but helpful for debugging)
            // This configures logging providers based on the "Logging" section in appsettings.json
            services.AddLogging(builder =>
            {
                builder.AddConfiguration(Configuration.GetSection("Logging"));
                // Add common providers - output will show in test runner console/debug output
                builder.AddConsole();
                builder.AddDebug();
            });

            // --- Reuse your existing DI setup ---
            // This registers ReviewApiOptions, HttpClient for ReviewApiClient,
            // MockReviewService, and ExternalReviewRepository as IReviewRepository etc.
            // It reads the "ReviewApi" section from the Configuration built above.
            services.AddReviewServices(Configuration);
            // --- --- --- --- --- --- --- --- ---

            // Build the service provider
            ServiceProvider = services.BuildServiceProvider();
        }

        /// <summary>
        /// Cleans up resources used by the ServiceProvider if necessary.
        /// Called automatically by xUnit after all tests in the class using this fixture have run.
        /// </summary>
        public void Dispose()
        {
            // Dispose the service provider if it implements IDisposable (which ServiceProvider does)
            // This ensures resources like HttpClient handlers might be cleaned up.
            if (ServiceProvider is IDisposable disposable)
            {
                disposable.Dispose();
            }
            GC.SuppressFinalize(this);
        }
    }
    ```

> 💡 **Information**
>
> - **xUnit's IClassFixture**: This pattern ensures the fixture is created once per test class
> - **IDisposable**: Implementing this interface ensures resources are properly cleaned up
> - **Configuration Building**: Loading configuration from appsettings.json just like a real application
> - **Service Registration**: Reusing the same DI setup as the main application
> - **Logging Configuration**: Setting up logging to aid in test diagnostics
>
> ⚠️ **Common Mistakes**
>
> - Not disposing of resources properly
> - Creating a new test fixture instance for each test
> - Not sharing the ServiceProvider across tests in the same class

### Step 3: Create Integration Tests for the ExternalReviewRepository

**Introduction**: Now we'll create the main test class that verifies our External Review Repository works with the actual API.

1. Create the ExternalReviewRepository integration test class:

    > `tests/MerchStore.Infrastructure.IntegrationTests/ExternalReviewRepositoryIntegrationTests.cs`

    ```csharp
    using MerchStore.Domain.Interfaces; // For IReviewRepository
    using MerchStore.Domain.Entities; // For Review
    using MerchStore.Domain.ValueObjects; // For ReviewStats
    using Microsoft.Extensions.DependencyInjection; // For GetRequiredService
    using Microsoft.Extensions.Logging; // For ILogger (optional)
    
    namespace MerchStore.Infrastructure.IntegrationTests;
    
    /// <summary>
    /// Contains integration tests for the ExternalReviewRepository,
    /// specifically testing its interaction with the live external review API.
    /// Uses the ReviewApiIntegrationTestFixture to get configured services.
    /// </summary>
    public class ExternalReviewRepositoryIntegrationTests : IClassFixture<ReviewApiIntegrationTestFixture>
    {
        // Service provider instance from the fixture
        private readonly IServiceProvider _serviceProvider;
        // The repository instance we will test, resolved via DI
        private readonly IReviewRepository _reviewRepository;
        // Optional logger for writing output during tests
        private readonly ILogger<ExternalReviewRepositoryIntegrationTests> _logger;
    
        // The fixture instance is injected by xUnit via the constructor
        public ExternalReviewRepositoryIntegrationTests(ReviewApiIntegrationTestFixture fixture)
        {
            _serviceProvider = fixture.ServiceProvider;
            // Resolve the IReviewRepository from the DI container set up by the fixture.
            // This gives us an instance of ExternalReviewRepository configured with ReviewApiClient etc.
            _reviewRepository = _serviceProvider.GetRequiredService<IReviewRepository>();
            // Resolve a logger instance (optional)
            _logger = _serviceProvider.GetRequiredService<ILogger<ExternalReviewRepositoryIntegrationTests>>();
        }
    
        /// <summary>
        /// Tests retrieving reviews for a product known to exist in the external API.
        /// This test WILL FAIL if the external API is down, the product ID is invalid,
        /// or the API key/URL in appsettings.json is incorrect.
        /// </summary>
        [Fact] // Marks this as a test method
        [Trait("Category", "Integration")] // Mark as an Integration test
        [Trait("Category", "ExternalAPI")] // Mark specifically as hitting an external API
        public async Task GetProductReviewsAsync_WhenApiIsAvailable_ReturnsRealDataForKnownProduct()
        {
            // Arrange
            _logger.LogInformation("Starting test: GetProductReviewsAsync_WhenApiIsAvailable_ReturnsRealDataForKnownProduct");
    
            // --- IMPORTANT ---
            // Replace this Guid with an ACTUAL Product ID that EXISTS in your external review API.
            // Find one via the API's documentation or by calling it directly (e.g., using Postman).
            // Using a known, existing ID is crucial for a valid "happy path" test.
            var knownProductId = Guid.Parse("f77132b5-0482-4e05-9d1b-13507c53f64b"); // <<< --- CHANGE THIS GUID! ---
            _logger.LogInformation("Testing with known Product ID: {ProductId}", knownProductId);
            // --- --- --- ---
    
            // Act
            _logger.LogInformation("Calling _reviewRepository.GetProductReviewsAsync...");
            (IEnumerable<Review> Reviews, ReviewStats Stats) result;
            try
            {
                 result = await _reviewRepository.GetProductReviewsAsync(knownProductId);
                 _logger.LogInformation("Call completed. Received {ReviewCount} reviews.", result.Reviews?.Count() ?? 0);
            }
            catch (Exception ex)
            {
                // Log the exception if the call fails unexpectedly
                _logger.LogError(ex, "Call to GetProductReviewsAsync failed unexpectedly: {ErrorMessage}", ex.Message);
                // Rethrow or Assert.Fail to ensure the test fails clearly
                Assert.Fail($"GetProductReviewsAsync threw an unexpected exception: {ex.Message}");
                return; // Keep compiler happy, Assert.Fail prevents reaching here
            }
    
            // Assert
            _logger.LogInformation("Performing assertions...");
            // Basic checks to ensure data structure is as expected
            Assert.NotNull(result.Reviews); // Should be an empty list, not null, even if no reviews
            Assert.NotNull(result.Stats);   // Stats object should always be returned
    
            // Check if the ProductId in the stats matches the requested one
            Assert.Equal(knownProductId, result.Stats.ProductId);
    
            // More specific assertions (adapt based on expected data for your knownProductId):
            // - Is the review count non-negative?
            Assert.True(result.Stats.ReviewCount >= 0, $"Review count should be >= 0, but was {result.Stats.ReviewCount}");
    
            // - If you expect reviews for this specific product, check they are not empty
            //   (Comment out if the known product might legitimately have zero reviews)
            // Assert.NotEmpty(result.Reviews);
    
            // - Check if the average rating is within the valid range (0 to 5)
            Assert.InRange(result.Stats.AverageRating, 0.0, 5.0);
    
            // - If reviews exist, check some basic properties of the first review
            if (result.Reviews.Any())
            {
                var firstReview = result.Reviews.First();
                Assert.NotNull(firstReview.CustomerName);
                Assert.NotNull(firstReview.Title);
                Assert.NotNull(firstReview.Content);
                Assert.InRange(firstReview.Rating, 1, 5);
                Assert.Equal(knownProductId, firstReview.ProductId); // Ensure review belongs to the product
            }
    
            _logger.LogInformation("SUCCESS: Test passed for Product {ProductId}. Retrieved {ReviewCount} reviews. Avg Rating: {AverageRating}",
                knownProductId, result.Stats.ReviewCount, result.Stats.AverageRating);
        }
    }
    ```

> 💡 **Information**
>
> - **`IClassFixture<T>`**: A pattern in xUnit that ensures the test fixture is created once per test class
> - **Traits**: Special attributes that categorize tests for conditional execution
> - **Comprehensive Logging**: Detailed logging helps diagnose test failures
> - **Try/Catch in Tests**: Catches exceptions and provides better error messages
> - **External API Dependency**: These tests depend on the actual external service being available
>
> ⚠️ **Common Mistakes**
>
> - Using a non-existent product ID for the "happy path" test
> - Not handling exceptions properly in tests
> - Missing important assertions about the returned data
> - Not using traits to categorize tests that depend on external services

### Step 4: Create Tests for the Circuit Breaker Pattern

**Introduction**: Now let's create specialized tests that verify the Circuit Breaker pattern works correctly. These tests are especially important because they check that your application is resilient against external service failures.

1. Create a test class for the Circuit Breaker:

    > `tests/MerchStore.Infrastructure.IntegrationTests/ExternalReviewRepositoryCircuitBreakerTests.cs`

    ```csharp
    using MerchStore.Infrastructure.ExternalServices.Reviews; // For ReviewApiClient, ExternalReviewRepository etc.
    using MerchStore.Infrastructure.ExternalServices.Reviews.Configurations; // For ReviewApiOptions
    using Microsoft.Extensions.DependencyInjection; // For GetRequiredService, IServiceScopeFactory
    using Microsoft.Extensions.Logging; // For ILogger, ILoggerFactory
    using Microsoft.Extensions.Options; // For IOptions, Options.Create

    namespace MerchStore.Infrastructure.IntegrationTests;

    /// <summary>
    /// Contains integration tests specifically for verifying the circuit breaker
    /// behavior of the ExternalReviewRepository.
    /// </summary>
    public class ExternalReviewRepositoryCircuitBreakerTests : IClassFixture<ReviewApiIntegrationTestFixture>
    {
        // Create a scope factory to manage the lifetime of services
        private readonly IServiceScopeFactory _scopeFactory; 

        // Constructor receives the shared fixture
        public ExternalReviewRepositoryCircuitBreakerTests(ReviewApiIntegrationTestFixture fixture)
        {
            // Use the fixture to get the service provider
            _scopeFactory = fixture.ServiceProvider.GetRequiredService<IServiceScopeFactory>();
        }

        /// <summary>
        /// Tests that the circuit breaker opens after the configured number of exceptions
        /// and that subsequent calls trigger the fallback mechanism (MockReviewService),
        /// logging the details of the fallback reviews.
        /// </summary>
        [Fact]
        [Trait("Category", "Integration")]
        [Trait("Category", "CircuitBreaker")]
        public async Task CircuitBreaker_OpensAfterThresholdAndFallsBack()
        {
            // Arrange
            using var scope = _scopeFactory.CreateScope();
            var serviceProvider = scope.ServiceProvider;
            var loggerFactory = serviceProvider.GetRequiredService<ILoggerFactory>();
            var mockReviewService = serviceProvider.GetRequiredService<MockReviewService>();
            var originalOptions = serviceProvider.GetRequiredService<IOptions<ReviewApiOptions>>().Value;

            // Create a new ReviewApiOptions instance with a non-existent endpoint to trigger failures
            // This simulates the external API being down or unreachable
            var testOptions = new ReviewApiOptions
            {
                BaseUrl = "http://localhost:9999/invalid-path/", // Non-existent endpoint
                ApiKey = "test-key",
                ApiKeyHeaderName = originalOptions.ApiKeyHeaderName,
                TimeoutSeconds = 5,
                ExceptionsAllowedBeforeBreaking = originalOptions.ExceptionsAllowedBeforeBreaking > 0 ? originalOptions.ExceptionsAllowedBeforeBreaking : 2, // Use 2 for test
                CircuitBreakerDurationSeconds = 5
            };
            var optionsWrapper = Options.Create(testOptions);

            // Create a new HttpClient with the test options and configure the repository
            var httpClient = serviceProvider.GetRequiredService<IHttpClientFactory>().CreateClient();
            var apiClient = new ReviewApiClient(httpClient, optionsWrapper, loggerFactory.CreateLogger<ReviewApiClient>());
            var repository = new ExternalReviewRepository(apiClient, mockReviewService, optionsWrapper, loggerFactory.CreateLogger<ExternalReviewRepository>());

            var productId = Guid.NewGuid(); // Generate a new product ID for testing. Does not matter. 
            int exceptionsAllowed = testOptions.ExceptionsAllowedBeforeBreaking;
            var logger = loggerFactory.CreateLogger<ExternalReviewRepositoryCircuitBreakerTests>(); // Get logger instance

            logger.LogInformation("Circuit Breaker Test: ExceptionsAllowedBeforeBreaking = {Count}", exceptionsAllowed);

            // Act - Trigger initial failures
            // We expect these calls to fail internally, be caught by the repository's
            // outer catch block, and return mock data.
            for (int i = 0; i < exceptionsAllowed; i++)
            {
                int callNum = i + 1;
                logger.LogInformation("Circuit Breaker Test: Making failing call #{CallNum}", callNum);
                try
                {
                    // Call the method - it should complete successfully by returning mock data
                    var (initialFallbackReviews, initialFallbackStats) = await repository.GetProductReviewsAsync(productId);

                    // Verify it returned mock data even here
                    if (initialFallbackReviews.Any())
                    {
                         logger.LogInformation("Circuit Breaker Test: Call #{CallNum} completed and returned mock data (Title: '{Title}') as expected (due to repo catch block).",
                            callNum, initialFallbackReviews.First().Title);
                         Assert.StartsWith("Sample Review:", initialFallbackReviews.First().Title, StringComparison.OrdinalIgnoreCase);

                         // --- Log details of initial fallback reviews ---
                         logger.LogInformation("--- Details of Initial Fallback Reviews (Call #{CallNum}) ---", callNum);
                         int reviewIndex = 0;
                         foreach (var review in initialFallbackReviews)
                         {
                             // Log each review's details in a more readable format
                             logger.LogInformation(
                                 "Initial Fallback Review [{Index}]:\n" +
                                 "  ID      : {ReviewId}\n" +
                                 "  Rating  : {Rating}\n" +
                                 "  Customer: {CustomerName}\n" +
                                 "  Title   : {ReviewTitle}\n" +
                                 "  Content : {ReviewContent}",
                                 ++reviewIndex, // Increment index for readability
                                 review.Id,
                                 review.Rating,
                                 review.CustomerName,
                                 review.Title,
                                 review.Content // Added Content
                             );
                         }
                         logger.LogInformation("--- End Initial Fallback Review Details ---");
                         // --- End Logging ---
                    }
                    else
                    {
                         logger.LogInformation("Circuit Breaker Test: Call #{CallNum} completed and returned 0 mock reviews as expected (due to repo catch block).", callNum);
                         Assert.Equal(0, initialFallbackStats.ReviewCount);
                    }
                }
                catch (Exception ex)
                {
                    // If any *other* exception occurs here, fail the test as setup might be wrong
                    logger.LogError(ex, "Circuit Breaker Test: Call #{CallNum} failed unexpectedly during initial failure loop.", callNum);
                    Assert.Fail($"Call #{callNum} threw an unexpected exception: {ex.GetType().Name} - {ex.Message}");
                }
            }

            // Act & Assert - Trigger the circuit opening and verify fallback
            int finalCallNum = exceptionsAllowed + 1;
            logger.LogInformation("Circuit Breaker Test: Making call #{CallNum} (circuit should be open, should fallback)", finalCallNum);

            // This call should now trigger the fallback because the circuit is open.
            // The repository catches the BrokenCircuitException internally and returns mock data.
            var (fallbackReviews, fallbackStats) = await repository.GetProductReviewsAsync(productId);

            // Assert: Check if the returned data comes from the MockReviewService
            Assert.NotNull(fallbackReviews);
            Assert.NotNull(fallbackStats);

            // MockReviewService adds "Sample Review:" prefix to titles
            if (fallbackReviews.Any())
            {
                // Check the first review's title prefix to confirm it's likely from the mock service
                Assert.StartsWith("Sample Review:", fallbackReviews.First().Title, StringComparison.OrdinalIgnoreCase);
                logger.LogInformation("Circuit Breaker Test: Call #{CallNum} correctly returned mock data (Title: '{Title}'). Circuit successfully opened and fallback occurred.",
                    finalCallNum, fallbackReviews.First().Title);

                // --- Log details of final fallback reviews ---
                logger.LogInformation("--- Details of Final Fallback Reviews (Call #{CallNum}) ---", finalCallNum);
                int reviewIndex = 0;
                foreach (var review in fallbackReviews)
                {
                     // Log each review's details in a more readable format
                     logger.LogInformation(
                         "Final Fallback Review [{Index}]:\n" +
                         "  ID      : {ReviewId}\n" +
                         "  Rating  : {Rating}\n" +
                         "  Customer: {CustomerName}\n" +
                         "  Title   : {ReviewTitle}\n" +
                         "  Content : {ReviewContent}",
                         ++reviewIndex, // Increment index for readability
                         review.Id,
                         review.Rating,
                         review.CustomerName,
                         review.Title,
                         review.Content // Added Content
                     );
                }
                logger.LogInformation("--- End Final Fallback Review Details ---");
                // --- End Logging ---
            }
            else
            {
                // It's possible the mock service generated 0 reviews
                Assert.Equal(0, fallbackStats.ReviewCount);
                 logger.LogInformation("Circuit Breaker Test: Call #{CallNum} correctly returned 0 mock reviews. Circuit successfully opened and fallback occurred.", finalCallNum);
            }
        }
    }
    ```

> 💡 **Information**
>
> - **Circuit Breaker Testing**: Creating a controlled environment to test circuit breaker behavior
> - **Artificial Failures**: Using a non-existent endpoint to trigger predictable failures
> - **Service Scope Factory**: Creating scoped services to isolate tests
> - **Detailed Logging**: Comprehensive logging to explain circuit breaker state
> - **Structured Log Messages**: Using multi-line formatted log entries for better readability
>
> ⚠️ **Common Mistakes**
>
> - Not isolating test services properly
> - Setting unrealistic circuit breaker thresholds
> - Missing assertions for the mock service fallback
> - Not handling exceptions properly in tests

### Step 5: Running Tests with Filters

**Introduction**: Since our integration tests depend on external services that might not always be available, we need to be able to run specific categories of tests selectively.

1. Running all tests:

    ```bash
    dotnet test tests/MerchStore.Infrastructure.IntegrationTests
    ```

2. Running only integration tests:

    ```bash
    dotnet test tests/MerchStore.Infrastructure.IntegrationTests --filter "Category=Integration"
    ```

3. Running only circuit breaker tests:

    ```bash
    dotnet test tests/MerchStore.Infrastructure.IntegrationTests --filter "Category=CircuitBreaker"
    ```

4. Excluding tests that hit the external API:

    ```bash
    dotnet test tests/MerchStore.Infrastructure.IntegrationTests --filter "Category!=ExternalAPI"
    ```

> 💡 **Information**
>
> - **Test Filtering**: The `--filter` parameter allows selection of tests based on their traits or names
> - **Category Traits**: Using traits categorizes tests and makes them selectively runnable
> - **Logical Operators**: You can use `!=` to exclude tests from a specific category
> - **Script Automation**: Using scripts makes it easy to run standard test suites consistently
>
> ⚠️ **Common Mistakes**
>
> - Running external API tests in CI/CD pipelines without proper handling for failures
> - Using wrong filter syntax (e.g., forgetting quotes around filter expressions)
> - Not documenting which tests depend on external services
> - Running all tests when the external service is known to be down

### Step 6: Analyzing Test Results and Logs

**Introduction**: Understanding test results and logs is crucial for diagnosing issues with integration tests, especially when they interact with external services.

1. Running tests with increased verbosity:

   ```bash
   dotnet test tests/MerchStore.Infrastructure.IntegrationTests -v n
   ```

2. Understanding the logging output:

   When tests run, you'll see detailed logs in the test output, particularly when the log level is set to Debug. For example:

   ```sh
   info: MerchStore.Infrastructure.IntegrationTests.ExternalReviewRepositoryIntegrationTests[0]
         Starting test: GetProductReviewsAsync_WhenApiIsAvailable_ReturnsRealDataForKnownProduct
   info: MerchStore.Infrastructure.IntegrationTests.ExternalReviewRepositoryIntegrationTests[0]
         Testing with known Product ID: f77132b5-0482-4e05-9d1b-13507c53f64b
   info: MerchStore.Infrastructure.ExternalServices.Reviews.ReviewApiClient[0]
         Requesting reviews for product f77132b5-0482-4e05-9d1b-13507c53f64b from external API
   dbug: MerchStore.Infrastructure.ExternalServices.Reviews.ReviewApiClient[0]
         Received response: {
           "Reviews": [
             {
               "Id": "review-123",
               "ProductId": "f77132b5-0482-4e05-9d1b-13507c53f64b",
               "CustomerName": "John Doe",
               "Title": "Great product!",
               "Content": "I love this product. It exceeded my expectations.",
               "Rating": 5,
               "CreatedAt": "2025-04-15T14:30:45Z",
               "Status": "Approved"
             }
           ],
           "Stats": {
             "ProductId": "f77132b5-0482-4e05-9d1b-13507c53f64b",
             "AverageRating": 4.5,
             "ReviewCount": 3
           }
         }
   ```

3. Analyzing failed tests:

   When tests fail, the output will include stack traces and error messages:

   ```sh
   Failed MerchStore.Infrastructure.IntegrationTests.ExternalReviewRepositoryIntegrationTests.GetProductReviewsAsync_WhenApiIsAvailable_ReturnsRealDataForKnownProduct [52 ms]
     Error Message:
      GetProductReviewsAsync threw an unexpected exception: Connection refused
     Stack Trace:
        at MerchStore.Infrastructure.IntegrationTests.ExternalReviewRepositoryIntegrationTests.GetProductReviewsAsync_WhenApiIsAvailable_ReturnsRealDataForKnownProduct() in /Path/to/MerchStore.Infrastructure.IntegrationTests/ExternalReviewRepositoryIntegrationTests.cs:line 56
   ```

4. Viewing JSON responses:

   When the log level for `MerchStore.Infrastructure` is set to `Debug`, you'll see the actual JSON responses from the external API, which can be invaluable for debugging mapping issues:

   ```sh
   dbug: MerchStore.Infrastructure.ExternalServices.Reviews.ReviewApiClient[0]
         Received response: {
           "Reviews": [...],
           "Stats": {...}
         }
   ```

> 💡 **Information**
>
> - **Verbosity Levels**: `-v n` sets normal verbosity, `-v d` for detailed, `-v diag` for diagnostic
> - **JSON Response Logging**: Debug log level triggers JSON response logging in ReviewApiClient
> - **Structured Logging**: Using structured log formats helps parse and analyze logs programmatically
> - **Log Categories**: Each class has its own log category for filtering logs
>
> ⚠️ **Common Mistakes**
>
> - Not setting appropriate log levels for different components
> - Missing important log information by using too restrictive log filters
> - Exposing sensitive information (like API keys) in logs
> - Not capturing logs in CI/CD environments when tests fail

## 🧪 Final Tests

### Running All Tests

1. Build the test project:

   ```bash
   dotnet build tests/MerchStore.Infrastructure.IntegrationTests
   ```

2. Run all tests:

   ```bash
   dotnet test tests/MerchStore.Infrastructure.IntegrationTests
   ```

3. Run only the tests that don't hit the external API:

   ```bash
   dotnet test tests/MerchStore.Infrastructure.IntegrationTests --filter "Category!=ExternalAPI"
   ```

### Expected Results

✅ **Happy Path Tests**

- The `GetProductReviewsAsync_WhenApiIsAvailable_ReturnsRealDataForKnownProduct` test passes when:
  - The external API is available
  - The API key is valid
  - The product ID exists in the external system

- The `GetProductReviewsAsync_ForNonExistentProduct_ShouldReturnDataFromExternalMock` test passes when:
  - The external mock API handles non-existent product IDs correctly

✅ **Circuit Breaker Tests**

- The `CircuitBreaker_OpensAfterThresholdAndFallsBack` test passes when:
  - The circuit breaker correctly opens after the threshold of failures
  - The fallback mechanism (MockReviewService) kicks in when the circuit is open
  - The log output confirms the expected sequence of events

## 🔧 Troubleshooting

If you encounter issues:

- **API Connectivity Issues**:
  - Check if the API URL in appsettings.json is correct
  - Verify the API key is valid and hasn't expired
  - Ensure there are no network issues (e.g., firewalls blocking the connection)
  - Try testing the API directly using Postman or curl

- **Missing Dependencies**:
  - Ensure all required NuGet packages are installed
  - Verify that the Infrastructure project is correctly referenced

- **Configuration Issues**:
  - Make sure appsettings.json is being copied to the output directory
  - Check that the ReviewApi section has all required settings

- **Inconsistent Test Results**:
  - External services can be unreliable, giving different results each run
  - For consistent results, focus on the circuit breaker tests which don't depend on the actual API
  - Use filters to exclude tests that hit the external API during local development

## 🚀 Optional Challenge

Want to take your learning further? Try:

1. **Mock HTTP Handler**: Create a test double for HttpClient that doesn't hit the real API but returns predictable responses

2. **Improved Circuit Breaker**: Enhance the circuit breaker tests to verify the half-open state works correctly

3. **Performance Testing**: Add tests that measure the response time of the external API and verify it meets acceptable thresholds

4. **Resilience Testing**: Create a test that simulates different types of failures (timeouts, bad requests, server errors) and verifies the system handles them all correctly

5. **Setup CI Pipeline**: Configure a GitHub Actions workflow that runs only the non-external tests on each PR

## 📚 Further Reading

- [xUnit Docs: Shared Context](https://xunit.net/docs/shared-context) - More about test fixtures in xUnit
- [Polly Documentation](https://github.com/App-vNext/Polly) - Details on Circuit Breaker and other resilience patterns
- [Structured Logging with Serilog](https://serilog.net/) - Advanced logging techniques
- [Integration Testing in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/test/integration-tests) - Microsoft's guide to integration testing
- [Testing with Dependency Injection](https://docs.microsoft.com/en-us/dotnet/core/testing/dependency-injection) - How to use DI in tests

## Done! 🎉

Congratulations! You've successfully created comprehensive integration tests for the External Review Service infrastructure. These tests verify that your code works with the real API, handles failures gracefully, and implements the Circuit Breaker pattern correctly for resilience.

Integration tests like these are a vital part of ensuring that your application works correctly with external dependencies without requiring manual testing or a UI. They serve as both documentation and validation of your external service integration.

By categorizing your tests with traits, you've made it possible to run different subsets of tests depending on the situation, which is essential when dealing with external services that might not always be available.

In future exercises, we'll explore more advanced testing techniques and expand our test coverage to other parts of the application. 🚀
