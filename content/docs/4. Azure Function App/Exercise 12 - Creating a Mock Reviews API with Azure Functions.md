---
title: "Creating a Mock Reviews API with Azure Functions"
date: "2025-04-28"
lastmod: "2025-04-28"
draft: false
weight: 401
toc: true
---

## 🎯 Goal

Create a simplified mock external Reviews API using Azure Functions that will generate random reviews for products during development and testing. This will allow you to work on your application without relying on the actual third-party review service.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have an Azure subscription (a free trial account works fine)
- Have the Azure CLI installed on your machine
- Have the [Azure Functions Core Tools](https://learn.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=windows%2Cisolated-process%2Cnode-v4%2Cpython-v2%2Chttp-trigger%2Ccontainer-apps&pivots=programming-language-csharp) installed
- Have basic knowledge of C# and HTTP APIs
- Have .NET 6 SDK or later installed
- Have Visual Studio Code installed with the Azure Functions extension

## 📚 Learning Objectives

By the end of this exercise, you will:

- Create and deploy an **Azure Function App** using the Azure CLI
- Configure **API Key authentication** for secure access
- Implement a **mock HTTP API endpoint** that generates random data
- Apply **Infrastructure as Code** principles with shell scripting
- Understand the **Azure Functions programming model**
- Test an API endpoint using **HTTP requests**

## 🔍 Why This Matters

In real-world development, creating mock external APIs is valuable because:

- It enables development when the actual third-party service is unavailable
- It eliminates dependencies on external services during testing
- It provides consistent test data that doesn't change between test runs
- It prevents hitting rate limits or incurring costs from the real API
- It gives you full control over response scenarios
- It helps demonstrate the correct usage of service abstractions in a Clean Architecture

## 📝 Step-by-Step Instructions

### Step 1: Set Up Your Azure Environment

**Introduction**: First, we'll create a shell script to provision all the necessary Azure resources for our Function App. This script will handle creating the resource group, storage account, and function app with the proper configuration.

1. Open a terminal or command prompt and log in to Azure:

    ```bash
    az login
    ```

2. Create a new file named `create_review_api.sh` with the following content:

    ```bash
    #!/bin/bash

    # Variables
    RESOURCE_GROUP="ReviewApiMockRG" # Change to your preferred resource group name
    LOCATION="northeurope" # Change to your preferred Azure region
    TODAY=$(date +"%y%m%d") # Get current date in YYMMDD format
    STORAGE_ACCOUNT_NAME="reviewapifunc$TODAY" # Must be unique
    FUNCTION_APP_NAME="ReviewApiFunc$TODAY" # Must be unique
    RUNTIME="dotnet-isolated"
    API_KEY="campusmolndal"

    echo "Using the following configuration:"
    echo "Resource Group: $RESOURCE_GROUP"
    echo "Location: $LOCATION"
    echo "Storage Account: $STORAGE_ACCOUNT_NAME"
    echo "Function App: $FUNCTION_APP_NAME"
    echo "API Key: $API_KEY"

    # Create Resource Group
    echo "Creating Resource Group: $RESOURCE_GROUP..."
    az group create --name $RESOURCE_GROUP --location $LOCATION

    # Create Storage Account for Function App
    echo "Creating Storage Account: $STORAGE_ACCOUNT_NAME..."
    az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION \
                            --resource-group $RESOURCE_GROUP \
                            --sku Standard_LRS \
                            --allow-blob-public-access false

    # Create Function App and link to Storage Account
    echo "Creating Function App: $FUNCTION_APP_NAME..."
    az functionapp create --name $FUNCTION_APP_NAME \
                        --resource-group $RESOURCE_GROUP \
                        --consumption-plan-location $LOCATION\
                        --runtime $RUNTIME \
                        --functions-version 4 \
                        --storage-account $STORAGE_ACCOUNT_NAME

    # Enable CORS for Function App
    echo "Enabling CORS for Function App..."
    az functionapp cors add --name $FUNCTION_APP_NAME \
                          --resource-group $RESOURCE_GROUP \
                          --allowed-origins "*"

    # Create function API key
    echo "Setting up API key..."
    az functionapp keys set --name $FUNCTION_APP_NAME \
                          --resource-group $RESOURCE_GROUP \
                          --key-name reviewApiKey \
                          --key-type functionKeys \
                          --key-value $API_KEY

    # Wait until the Function App is running
    function wait_until_functionapp_is_running {
        STATUS=$(az functionapp show --name $FUNCTION_APP_NAME --resource-group $RESOURCE_GROUP --query "state" --output tsv)

        while [[ "$STATUS" != "Running" ]]; do
            echo "Waiting for Function App: $FUNCTION_APP_NAME to be ready (Current status: $STATUS)..."
            sleep 10  # Wait for 10 seconds before checking again
            STATUS=$(az functionapp show --name $FUNCTION_APP_NAME --resource-group $RESOURCE_GROUP --query "state" --output tsv)
        done

        echo "Function App: $FUNCTION_APP_NAME is now running!"
    }

    wait_until_functionapp_is_running

    echo "Environment setup complete! Make note of these values for later use:"
    echo "Function App Name: $FUNCTION_APP_NAME"
    echo "API Key: $API_KEY"
    ```

3. Make the script executable and run it:

    ```bash
    chmod +x create_review_api.sh
    ./create_review_api.sh
    ```

4. The script will create all necessary resources and wait until the function app is running. Make note of the function app name that's displayed at the end of the script execution.

> 💡 **Information**
>
> - **Date-based Naming**: Using the current date in resource names makes them unique but still meaningful
> - **Resource Group**: A logical container for related Azure resources
> - **Storage Account**: Required by Functions for state management
> - **Consumption Plan**: Serverless model that scales automatically and bills only for actual usage
> - **CORS Configuration**: Allows web applications to call your API from different domains
> - **Predefined API Key**: Using a known value makes development easier than random keys
> - **Wait Function**: Ensures the resources are fully deployed before proceeding
>
> ⚠️ **Common Mistakes**
>
> - Not granting execute permission to the shell script with `chmod +x`
> - Storage account names must be all lowercase with no special characters
> - Chosen regions must support all the services you're deploying
> - Not waiting for resources to fully provision before attempting to use them

### Step 2: Create a Local Function Project

**Introduction**: Now we'll create a local Azure Functions project to implement our mock Reviews API.

1. Create a new directory for your project and navigate to it:

    ```bash
    mkdir ReviewApiFunction
    cd ReviewApiFunction
    ```

2. Initialize a new Azure Functions project:

    ```bash
    func init --worker-runtime dotnet-isolated
    ```

3. Create a new HTTP-triggered function for product reviews:

    ```bash
    func new --template "HTTP trigger" --name GetProductReviews
    ```

4. Add the project to the solution (run the command from the solution root directory):

    ```bash
    dotnet sln add infra/ReviewApiFunction/ReviewApiFunction.csproj
    ```

5. Create a models file to define our review data structures. Create `Models.cs` in the project root:

    ```csharp
    // Models.cs
    using System;
    using System.Collections.Generic;

    namespace ReviewApiFunction.Models
    {
        public class Review
        {
            public required string Id { get; set; }
            public required string ProductId { get; set; }
            public required string CustomerName { get; set; }
            public required string Title { get; set; }
            public required string Content { get; set; }
            public int Rating { get; set; }
            public DateTime CreatedAt { get; set; }
            public required string Status { get; set; }
            
            // Parameterless constructor for serialization
            public Review() { }
        }

        public class ReviewResponse
        {
            public required List<Review> Reviews { get; set; }
            public required ReviewStats Stats { get; set; }
            
            // Parameterless constructor for serialization
            public ReviewResponse() { }
            
            // Constructor with required parameters
            public ReviewResponse(List<Review> reviews, ReviewStats stats)
            {
                Reviews = reviews;
                Stats = stats;
            }
        }

        public class ReviewStats
        {
            public required string ProductId { get; set; }
            public double AverageRating { get; set; }
            public int ReviewCount { get; set; }
            
            // Parameterless constructor for serialization
            public ReviewStats() { }
        }
    }
    ```

6. Update the `GetProductReviews.cs` file with our mock review generation logic:

    ```csharp
    using Microsoft.Azure.Functions.Worker;
    using Microsoft.Extensions.Logging;
    using Microsoft.AspNetCore.Http;
    using Microsoft.AspNetCore.Mvc;
    using ReviewApiFunction.Models;

    namespace ReviewApiFunction
    {
        public class GetProductReviews
        {
            private readonly ILogger<GetProductReviews> _logger;

            private static readonly Random _random = new Random();
            private static readonly string[] _customerNames = { "John Doe", "Jane Smith", "Bob Johnson", "Alice Brown", "Charlie Davis" };
            private static readonly string[] _reviewTitles = { "Great product!", "Highly recommended", "Exceeded expectations", "Not bad", "Could be better" };
            private static readonly string[] _reviewContents = {
                "I've been using this for weeks and it's fantastic.",
                "Exactly what I was looking for. High quality.",
                "The product is decent but shipping took too long.",
                "Works as advertised, very happy with my purchase.",
                "Good value for the money, would buy again."
            };

            public GetProductReviews(ILogger<GetProductReviews> logger)
            {
                _logger = logger;
            }

            [Function("GetProductReviews")]
            public async Task<IActionResult> Run(
                [HttpTrigger(AuthorizationLevel.Function, "get", Route = "products/{productId}/reviews")] HttpRequest req,
                string productId)
            {
                _logger.LogInformation("Processing request for product reviews.");

                try
                {
                    // Validate product ID is a valid GUID
                    if (!Guid.TryParse(productId, out Guid productGuid))
                    {
                        return new BadRequestObjectResult(new { error = "Invalid product ID format. Must be a valid GUID." });
                    }

                    // Generate random number of reviews (0-5)
                    int reviewCount = _random.Next(0, 6);
                    List<Review> reviews = GenerateRandomReviews(productId, reviewCount);

                    // Calculate average rating
                    double averageRating = reviews.Any() 
                        ? Math.Round(reviews.Average(r => r.Rating), 1) 
                        : 0;

                    // Create the response object
                    var response = new ReviewResponse
                    {
                        Reviews = reviews,
                        Stats = new ReviewStats
                        {
                            ProductId = productId,
                            AverageRating = averageRating,
                            ReviewCount = reviewCount
                        }
                    };

                    // Add a small delay to simulate network latency
                    await Task.Delay(300);

                    return new OkObjectResult(response);
                }
                catch (Exception ex)
                {
                    _logger.LogError($"Error processing request: {ex.Message}");
                    return new StatusCodeResult(StatusCodes.Status500InternalServerError);
                }
            }

            private static List<Review> GenerateRandomReviews(string productId, int count)
            {
                var reviews = new List<Review>();

                for (int i = 0; i < count; i++)
                {
                    // Create a random date within the last 30 days
                    var createdAt = DateTime.UtcNow.AddDays(-_random.Next(1, 31));
                    
                    // Generate a random rating, weighted toward positive reviews
                    int rating = _random.Next(1, 101) switch
                    {
                        var n when n <= 10 => 1, // 10% chance of 1-star
                        var n when n <= 25 => 2, // 15% chance of 2-stars
                        var n when n <= 50 => 3, // 25% chance of 3-stars
                        var n when n <= 80 => 4, // 30% chance of 4-stars
                        _ => 5                   // 20% chance of 5-stars
                    };

                    reviews.Add(new Review
                    {
                        Id = Guid.NewGuid().ToString(),
                        ProductId = productId,
                        CustomerName = _customerNames[_random.Next(_customerNames.Length)],
                        Title = _reviewTitles[_random.Next(_reviewTitles.Length)],
                        Content = _reviewContents[_random.Next(_reviewContents.Length)],
                        Rating = rating,
                        CreatedAt = createdAt,
                        Status = "approved"
                    });
                }

                // Sort by date descending (newest first)
                return reviews.OrderByDescending(r => r.CreatedAt).ToList();
            }
        }
    }
    ```

> 💡 **Information**
>
> - **HTTP Trigger**: Responds to HTTP requests with the specified methods and route
> - **Authorization Level**: "Function" requires an API key to access the endpoint
> - **Route Parameter**: The `{productId}` is a route parameter extracted from the URL
> - **Random Data Generation**: Creates realistic-looking reviews with varied ratings
> - **Response Structure**: Includes both reviews and stats in a single response
>
> ⚠️ **Common Mistakes**
>
> - Not handling empty review scenarios (when count is 0)
> - Missing proper error handling for unexpected exceptions
> - Using hard-coded values instead of randomization
> - Not validating input parameters

### Step 3: Test Locally

**Introduction**: Before deploying to Azure, it's a good practice to test your function locally to ensure it works as expected.

1. Start the function app locally:

    ```bash
    func start
    ```

2. Note the URL output in the console, which will look similar to:

    ```sh
    Http Functions:
            GetProductReviews: [GET] http://localhost:7071/api/products/{productId}/reviews
    ```

3. Open a new terminal window and use `curl` to test the function:

    ```bash
    # Replace with a valid GUID
    curl "http://localhost:7071/api/products/3fa85f64-5717-4562-b3fc-2c963f66afa6/reviews"
    ```

4. Verify that the response contains a list of reviews and the statistics object with an average rating.

> 💡 **Information**
>
> - **Local Testing**: Functions can run locally before deploying to Azure
> - **Function Keys**: Even in local development, function-level authorization requires keys
> - **Multiple Runs**: Each run should generate different random reviews
>
> ⚠️ **Common Mistakes**
>
> - Testing with an invalid GUID format
> - Forgetting to include the function key in the request
> - Not stopping the local function runtime before deploying

### Step 4: Deploy to Azure

**Introduction**: Now that we've tested our function locally, let's deploy it to Azure.

1. Set the function app name variable to match what was created by the script:

    ```bash
    FUNCTION_APP_NAME="ReviewApiFunc$(date +"%y%m%d")"
    echo "Using function app: $FUNCTION_APP_NAME"
    ```

2. Publish the function to Azure:

    ```bash
    func azure functionapp publish $FUNCTION_APP_NAME
    ```

3. After successful deployment, verify the function endpoint and API key in a browser:

    ```bash
    https://reviewapifunc250420.azurewebsites.net/api/products/00000000-0000-0000-0000-000000000000/reviews?code=campusmolndal
    ```

   You can also view the function in the Azure Portal:
   - Navigate to your function app
   - Go to "Functions" > "GetProductReviews"
   - Under "Function Keys" you should see the "reviewApiKey" with your configured value

> 💡 **Information**
>
> - **Deployment**: The `func azure functionapp publish` command packages and deploys your code
> - **Function URL**: Each function has a unique URL that includes your function app name
> - **Function Keys**: Different from the host keys and provide access to specific functions
>
> ⚠️ **Common Mistakes**
>
> - Deploying before testing locally
> - Not checking deployment logs for errors
> - Sharing function keys in code repositories

### Step 6: Test the Deployed API

**Introduction**: Finally, let's test our deployed API to ensure it works as expected in Azure.

1. Test the deployed function using curl (replace the date part in the function app name with your actual values):

    ```bash
    FUNCTION_APP_NAME="ReviewApiFunc$(date +"%y%m%d")"
    API_KEY="campusmolndal"
    
    curl "https://$FUNCTION_APP_NAME.azurewebsites.net/api/products/3fa85f64-5717-4562-b3fc-2c963f66afa6/reviews?code=$API_KEY"
    ```

    or in the header

    ```bash
    FUNCTION_APP_NAME="ReviewApiFunc$(date +"%y%m%d")"
    API_KEY="campusmolndal"
    
    curl -H "x-functions-key: $API_KEY" "https://$FUNCTION_APP_NAME.azurewebsites.net/api/products/3fa85f64-5717-4562-b3fc-2c963f66afa6/reviews"
    ```

2. You can also test it using a tool like Postman:
   - Set the HTTP method to GET
   - Enter the URL: `https://ReviewApiFunc230420.azurewebsites.net/api/products/3fa85f64-5717-4562-b3fc-2c963f66afa6/reviews`
   - Add a query parameter: `code` with the value `campusmolndal`
   - Send the request and verify the response

   Note: Replace `ReviewApiFunc230420` with your actual function app name based on the current date.

3. Try different product IDs to see different randomly generated reviews.

> 💡 **Information**
>
> - **API Testing**: Always test your deployed APIs to ensure they work in the cloud
> - **Query Parameters**: Function keys can be provided via the `code` query parameter
> - **Function App URLs**: Follow the pattern `https://{function-app-name}.azurewebsites.net/api/{route}`
>
> ⚠️ **Common Mistakes**
>
> - Using an incorrect function key
> - Not using HTTPS for the deployed function URL
> - Not including the /api/ prefix in the URL

## 🧪 Final Tests

### Make a Production-Like Test

To create a simple client application for testing the Reviews API, follow these steps:

1. First, create a new console application project within the infrastructure directory:

    ```bash
    cd infra
    dotnet new console -n ReviewApiClient
    ```

2. Add the project to the solution:

    ```bash
    dotnet sln add infra/ReviewApiClient/ReviewApiClient.csproj
    ```

3. Replace the content of the Program.cs file in the ReviewApiClient project with the following code:

    ```csharp
    using System;
    using System.Net.Http;
    using System.Net.Http.Json;
    using System.Text.Json;
    using System.Threading.Tasks;

    class Program
    {
        static async Task Main(string[] args)
        {
            try
            {
                // Use today's date to construct the Function App name
                string today = DateTime.Now.ToString("yyMMdd");
                string functionAppName = $"ReviewApiFunc{today}";
                string functionKey = "campusmolndal";
                
                // You can either use a random GUID or a specific product ID
                // For a random GUID:
                string productId = Guid.NewGuid().ToString();
                
                // Or use a specific product ID to get consistent results:
                // string productId = "3fa85f64-5717-4562-b3fc-2c963f66afa6";

                // Create HttpClient
                using var client = new HttpClient();
                
                Console.WriteLine("Review API Client Test");
                Console.WriteLine("=====================");
                Console.WriteLine($"Product ID: {productId}");
                Console.WriteLine();
                
                // METHOD 1: API Key in Query String
                string urlWithQueryParam = $"https://{functionAppName}.azurewebsites.net/api/products/{productId}/reviews?code={functionKey}";
                Console.WriteLine("METHOD 1: API Key in Query String");
                Console.WriteLine($"Requesting from: {urlWithQueryParam}");
                Console.WriteLine();
                
                // Make the first request with query string
                var responseWithQueryParam = await client.GetStringAsync(urlWithQueryParam);
                
                // Output the raw response
                Console.WriteLine("Response from query param method:");
                Console.WriteLine(responseWithQueryParam);
                Console.WriteLine();
                
                // Parse and display the response in a more readable format
                var options = new JsonSerializerOptions { WriteIndented = true };
                var formattedJsonQueryParam = JsonSerializer.Serialize(
                    JsonSerializer.Deserialize<JsonElement>(responseWithQueryParam),
                    options);
                
                Console.WriteLine("Formatted response (query param):");
                Console.WriteLine(formattedJsonQueryParam);
                Console.WriteLine();
                Console.WriteLine("===========================================");
                Console.WriteLine();
                
                // METHOD 2: API Key in Header
                string urlWithHeader = $"https://{functionAppName}.azurewebsites.net/api/products/{productId}/reviews";
                Console.WriteLine("METHOD 2: API Key in Header");
                Console.WriteLine($"Requesting from: {urlWithHeader}");
                Console.WriteLine($"With header: x-functions-key: {functionKey}");
                Console.WriteLine();
                
                // Create a new request message with the header
                var request = new HttpRequestMessage(HttpMethod.Get, urlWithHeader);
                request.Headers.Add("x-functions-key", functionKey);
                
                // Send the request and get the response
                var headerResponse = await client.SendAsync(request);
                headerResponse.EnsureSuccessStatusCode();
                
                var responseWithHeader = await headerResponse.Content.ReadAsStringAsync();
                
                // Output the raw response
                Console.WriteLine("Response from header method:");
                Console.WriteLine(responseWithHeader);
                Console.WriteLine();
                
                // Parse and display the response in a more readable format
                var formattedJsonHeader = JsonSerializer.Serialize(
                    JsonSerializer.Deserialize<JsonElement>(responseWithHeader),
                    options);

                Console.WriteLine("Formatted response (header):");
                Console.WriteLine(formattedJsonHeader);
                Console.WriteLine();
                Console.WriteLine("===========================================");
                Console.WriteLine();
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error: {ex.Message}");
                if (ex.InnerException != null)
                {
                    Console.WriteLine($"Inner error: {ex.InnerException.Message}");
                }
            }
        }
    }
    ```

4. Build and run the client application to test your deployed Azure Function:

    ```bash
    cd infra/ExternalApiMock/ReviewApiClient
    dotnet build
    dotnet run
    ```

5. Try running the client multiple times to see different random reviews generated.

> 💡 **Information**
>
> - The client application uses `DateTime.Now.ToString("yyMMdd")` to generate the function app name based on the current date
> - If you run this on a different day than you created the function app, you'll need to modify this line in the code:
>
>   ```csharp
>   string today = DateTime.Now.ToString("yyMMdd");
>   ```
>
> - You could replace it with a hardcoded value matching your function app name:
>
>   ```csharp
>   string today = "240420"; // Replace with your actual function app date
>   ```
>
> - Alternatively, you could pass the function app name as a command line argument to make the client more flexible

✅ **Expected Results**

- The function returns a JSON response with a list of reviews and stats
- The number of reviews varies between 0 and 5
- The review content, titles, and ratings are randomly generated
- The average rating matches the actual average of the ratings
- Each run produces different results

## 🔧 Troubleshooting

If you encounter issues:

- **Function Not Found**:
  - Ensure the function name matches exactly in the deployment
  - Check function logs in the Azure Portal

- **Authentication Errors**:
  - Verify you're using the correct function key
  - Check that you're passing the key correctly in the request

- **Deployment Failures**:
  - Look at the deployment logs for specific errors
  - Ensure your function app settings are correct
  - Check if the storage account exists and is accessible

- **Local Testing Issues**:
  - Verify your `local.settings.json` file has the correct values
  - Check if required ports are available on your machine
  - Ensure the Azure Functions Core Tools are up to date

## 🚀 Optional Challenge

Want to take your learning further? Try:

1. **Add More Endpoints**: Create additional endpoints like `GET /products/{productId}/review-stats` that only returns statistics
2. **Implement Error Scenarios**: Add functionality to simulate errors and timeouts for testing error handling
3. **Database Integration**: Store mock reviews in Azure Cosmos DB instead of generating them on each request
4. **Authentication Mechanisms**: Explore different authentication options like JWT tokens instead of API keys
5. **GitHub Actions**: Set up a CI/CD pipeline to automatically deploy your function when code changes

## 📚 Further Reading

- [Azure Functions Documentation](https://docs.microsoft.com/en-us/azure/azure-functions/)
- [HTTP Trigger in Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-http-webhook)
- [Azure Bicep Documentation](https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Function App Authentication](https://docs.microsoft.com/en-us/azure/app-service/configure-authentication-provider-aad)
- [API Design Best Practices](https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design)

## Done! 🎉

Congratulations! You've successfully created a mock Reviews API using Azure Functions with API Key authentication. This mock API will help you continue development and testing of your MerchStore application without depending on the actual external review service.

Remember that this mock is intentionally simplified but can be extended as needed. By using Azure Functions, you get a scalable, cost-effective solution that only charges you for actual usage, making it perfect for development and testing scenarios. The Infrastructure as Code approach with Bicep allows for consistent, repeatable deployments. 🚀
