---
title: "Testing Your API with Various Tools"
date: "2025-04-19"
lastmod: "2025-04-19"
draft: false
weight: 302
toc: true
---

## 🎯 Goal

Learn how to test your Basic Products API using different tools: curl, REST Client extension for VS Code, and Postman. This will give you a variety of approaches for API testing and development.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 7 (Implementing a Basic Product Catalog API)
- Have your API running and accessible at `https://localhost:7188` (or your specific port)
- Have basic familiarity with command-line operations
- Be using VS Code as your editor
- Have administrative access to install software on your computer

## 📚 Learning Objectives

By the end of this exercise, you will:

- Use **curl** to make HTTP requests from the command line
- Install and use the **REST Client extension** in VS Code to test APIs directly from your editor
- Install and use **Postman** to create a collection of API requests
- Understand the **advantages and limitations** of each approach
- Compare **HTTP requests and responses** across different tools
- Learn how to **troubleshoot common API testing issues**

## 🔍 Why This Matters

In real-world development, using API testing tools is crucial because:

- They help you verify your API works correctly from a client perspective
- They allow you to test and debug APIs without building a full client application
- Different tools offer various capabilities for different testing scenarios
- Familiarity with multiple tools gives you flexibility in different environments
- Understanding HTTP requests/responses at a deeper level makes you a better API developer
- These are industry-standard tools used by professional developers daily

## 📝 Step-by-Step Instructions

### Step 1: Using curl for API Testing

**Introduction**: curl is a command-line tool for transferring data with URLs. It's available on virtually all operating systems and is one of the most widely used tools for quick API testing and debugging.

#### 1.1: Installing and Verifying curl

1. **For Windows users**:
   - curl is included with Windows 10 (version 1803 and later)
   - Open Command Prompt or PowerShell to verify:

     ```bash
     curl --version
     ```

   - If not available, download from [curl's official site](https://curl.se/windows/)

2. **For macOS users**:
   - curl is pre-installed on macOS
   - Open Terminal and verify:

     ```bash
     curl --version
     ```

3. **For Linux users**:
   - Most distributions include curl by default
   - If not, install with your package manager:

     ```bash
     # Ubuntu/Debian
     sudo apt-get install curl
     
     # Fedora/RHEL
     sudo dnf install curl
     ```

#### 1.2: Basic curl Commands

Make sure your API is running before proceeding. Start it with:

```bash
dotnet run --project src/MerchStore.WebUI
```

1. **Get all products**:

   ```bash
   curl -X GET https://localhost:7188/api/basic/products -k
   ```

   > The `-k` flag tells curl to skip SSL certificate validation, which is useful for local development with self-signed certificates.

2. **Get a specific product** (replace the ID with an actual product ID from your database):

   ```bash
   curl -X GET https://localhost:7188/api/basic/products/3fa85f64-5717-4562-b3fc-2c963f66afa6 -k
   ```

3. **Format the JSON output for better readability** (PowerShell):

   ```powershell
   curl -X GET https://localhost:7188/api/basic/products -k | ConvertFrom-Json | ConvertTo-Json -Depth 10
   ```

   For macOS/Linux with Python installed:

   ```bash
   curl -X GET https://localhost:7188/api/basic/products -k | python3 -m json.tool
   ```

> 💡 **Information**
>
> - **Using curl**: The `-X` flag specifies the HTTP method, and the URL is the endpoint you're requesting
> - **SSL Certificates**: The `-k` flag is important for testing local HTTPS endpoints with self-signed certificates
> - **Formatting Output**: By default, curl outputs raw JSON without formatting, so using additional tools to format the JSON makes it more readable
> - **Headers**: You can add headers with `-H "Header-Name: value"` (useful when we add authentication later)
>
> ⚠️ **Common Mistakes**
>
> - Forgetting the `-k` flag when calling HTTPS endpoints locally leads to SSL verification errors
> - Not checking if your API is actually running before making requests
> - Forgetting to replace placeholder IDs with actual valid IDs from your database

### Step 2: Using REST Client Extension in VS Code

**Introduction**: REST Client is a VS Code extension that allows you to send HTTP requests and view responses directly within VS Code. It's a convenient way to test APIs without leaving your development environment.

#### 2.1: Installing REST Client Extension

1. Open VS Code
2. Click on the Extensions icon in the Activity Bar (or press `Ctrl+Shift+X` / `Cmd+Shift+X`)
3. Search for "REST Client"
4. Click Install on the extension by Huachao Mao
5. After installation, VS Code may prompt you to reload - click "Reload" if prompted

#### 2.2: Creating a .rest or .http File

1. Create a new file in your project called `api-tests.rest` or `api-tests.http` (either extension works)

   > `basic-api-tests.http`

   ```sh
   ### Get all products
   GET https://localhost:7188/api/basic/products
   
   ### Get product by ID (replace with a valid ID)
   GET https://localhost:7188/api/basic/products/3fa85f64-5717-4562-b3fc-2c963f66afa6
   
   ### Get product with invalid ID (404 response)
   GET https://localhost:7188/api/basic/products/00000000-0000-0000-0000-000000000000
   ```

2. Above each request, you'll see a "Send Request" link. Click it to execute that particular request.

3. When you send a request, a new editor tab opens with the response, including:
   - Status code
   - Response headers
   - Response body (formatted JSON)
   - Response time

### Step 3: Using Postman for API Testing

**Introduction**: Postman is a powerful API platform for building and using APIs. It's more feature-rich than curl or REST Client and offers capabilities like collections, automated testing, and team collaboration.

#### 3.1: Installing Postman

1. Visit [Postman's website](https://www.postman.com/downloads/) and download the appropriate version for your operating system
2. Install the application following the standard installation process for your OS
3. Launch Postman after installation
4. You can choose to create an account or skip sign-in for now (an account is required for some features like syncing and collaboration)

#### 3.2: Creating Basic Requests

1. Click the "+" button to create a new request tab

2. **Get all products**:
   - Set the method to GET
   - Enter the URL: `https://localhost:7188/api/basic/products`
   - Click "Send"
   - You'll see the response in the lower section of the window

3. **Get a specific product**:
   - Create another request with method GET
   - Enter URL: `https://localhost:7188/api/basic/products/3fa85f64-5717-4562-b3fc-2c963f66afa6` (use a valid ID)
   - Click "Send"

4. **Test a 404 response**:
   - Create another request with method GET
   - Enter URL: `https://localhost:7188/api/basic/products/00000000-0000-0000-0000-000000000000`
   - Click "Send"
   - You should see a 404 response

#### 3.3: Creating a Collection

1. Click the "Collections" tab in the sidebar (if not already selected)
2. Click the "+" button to create a new collection
3. Name it "MerchStore API Tests"
4. Click "Create"

5. **Save your existing requests to the collection**:
   - Click the "Save" button on each request tab
   - Select the "MerchStore API Tests" collection
   - Give each request a descriptive name (e.g., "Get All Products", "Get Product by ID", "Get Non-existent Product")
   - Click "Save"

6. Your requests are now organized in a collection and can be easily run again later

#### 3.4: Setting Up Environment Variables

1. Click the "Environments" tab in the sidebar
2. Click the "+" button to create a new environment
3. Name it "Local Development"
4. Add these variables:
   - `baseUrl` with value `https://localhost:7188`
   - `productId` with a valid product ID value

5. Click "Save"
6. Select "Local Development" from the environment dropdown in the upper-right corner

7. **Update your requests to use environment variables**:
   - Edit your existing requests
   - Change the URLs to use the variables: `{{baseUrl}}/api/basic/products` and `{{baseUrl}}/api/basic/products/{{productId}}`
   - Click "Save" for each request

### Step 4: Comparing the Tools

**Introduction**: Each tool has its strengths and use cases. Understanding when to use each one will make you more productive.

1. **Comparison Matrix**:

   | Feature | curl | REST Client | Postman |
   |---------|------|-------------|---------|
   | **Setup complexity** | Low (built-in) | Medium (extension) | High (separate app) |
   | **Learning curve** | Steep (command-line) | Gentle | Moderate |
   | **UI friendliness** | Command-line only | Simple, integrated | Full-featured GUI |
   | **Request organization** | None built-in | File-based | Collections |
   | **Environment management** | Manual | Basic support | Comprehensive |
   | **Testing capabilities** | None built-in | None built-in | Advanced |
   | **Automation potential** | High (scripts) | Limited | High (runners, CLI) |
   | **Integration with workflow** | Command-line/scripts | Directly in editor | Separate application |
   | **Team collaboration** | Limited | Via file sharing | Built-in features |

2. **When to use each tool**:

   - **curl**: Great for quick tests, CI/CD pipelines, and when working in terminal/command line
   - **REST Client**: Ideal during development when you're already in VS Code and want to quickly test endpoints
   - **Postman**: Best for comprehensive API testing, documentation, automation, and when working in teams

> 💡 **Information**
>
> - **Multiple Tools**: Professional developers often use multiple tools depending on the specific task
> - **Scripting**: curl commands can be incorporated into bash/PowerShell scripts for automation
> - **Version Control**: REST Client files can be checked into version control, documenting API usage
> - **Collaboration**: Postman collections can be shared with team members for consistent testing
>
> ⚠️ **Common Mistakes**
>
> - Using a complex tool for simple tasks (e.g., Postman for a quick one-off request)
> - Not leveraging the full capabilities of advanced tools (e.g., not using Postman's testing features)
> - Forgetting to update test files/collections when the API changes

## 🧪 Final Tests

### Verify Your Learning

1. Use curl to fetch a specific product and save the response to a file:

   ```bash
   curl -X GET https://localhost:7188/api/basic/products/3fa85f64-5717-4562-b3fc-2c963f66afa6 -k > product.json
   ```

2. Create a comprehensive REST Client file with at least three different requests.

3. In Postman:
   - Create a collection with at least three saved requests
   - Set up an environment with at least two variables
   - Add tests to at least one request
   - Run the collection and verify all requests succeed

✅ **Expected Results**

- You can successfully make API requests using all three tools
- You understand the advantages and limitations of each tool
- You can organize requests in REST Client files and Postman collections
- You know how to use environment variables for flexible testing
- You can write basic tests in Postman to validate responses
- You've saved examples of each approach for future reference

## 🔧 Troubleshooting

If you encounter issues:

- **Certificate errors**:
  - For curl, use the `-k` flag to skip certificate validation
  - For REST Client, add `@ignoreCerts = true` at the top of your file
  - For Postman, disable SSL certificate verification in Settings

- **Connection refused errors**:
  - Make sure your API is running and the port number is correct
  - Check for any firewall issues blocking local connections

- **Authentication errors** (if your API uses authentication):
  - Ensure you're including the required headers/tokens
  - Check that credentials are correct for your environment

- **JSON parsing errors**:
  - Verify the response is valid JSON using a tool like [JSONLint](https://jsonlint.com/)

## 🚀 Optional Challenge

Want to take your learning further? Try:

1. **Advanced curl**: Use curl to make a request that includes custom headers and saves both headers and response body to separate files

2. **REST Client variables**: Create multiple environment files (development, testing) and learn to switch between them

3. **Postman automation**: Create a Postman Collection Runner that runs all your API tests in sequence with a single click

4. **Comparing responses**: Make the same request with all three tools and write a brief analysis of the differences in responses, usability, and workflow

## 📚 Further Reading

- [curl Documentation](https://curl.se/docs/) - Official documentation for all curl options
- [REST Client Extension Documentation](https://github.com/Huachao/vscode-restclient/blob/master/README.md) - Comprehensive guide to all features
- [Postman Learning Center](https://learning.postman.com/) - Official tutorials and guides for Postman
- [HTTP Status Codes](https://httpstatuses.com/) - Reference for understanding HTTP response codes
- [API Testing Best Practices](https://blog.postman.com/api-testing-best-practices/) - Postman's guide to effective API testing

## Done! 🎉

Congratulations! You've learned how to test your API using three different tools: curl, REST Client, and Postman. Each tool has its own advantages and use cases, and having all three in your toolkit makes you a more versatile API developer.

As you continue developing your MerchStore application, you can use these tools to test new endpoints, debug issues, and ensure your API works as expected across different scenarios. The ability to quickly and effectively test API endpoints is an essential skill for modern web development. 🚀

## What's Next?

In future exercises, you'll learn:

1. **How to secure your API with authentication**
2. **How to implement Minimal APIs for a more concise approach**
3. **How to create a versioned API using CQRS pattern**

Each of these topics will build on the API testing skills you've developed in this exercise.
