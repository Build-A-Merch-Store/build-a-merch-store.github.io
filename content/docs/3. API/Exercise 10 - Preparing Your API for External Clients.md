---
title: "Preparing Your API for External Clients"
date: "2025-04-20"
lastmod: "2025-04-20"
draft: false
weight: 304
toc: true
---

## 🎯 Goal

Prepare your Basic Products API for consumption by external client applications by implementing CORS support, configuring JSON formatting with snake case, and creating a simple JavaScript client that demonstrates how to interact with your API.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 7 (Implementing a Basic Product Catalog API)
- Have completed Exercise 9 (Adding API Key Authentication to Your Basic API)
- Have your API running and accessible
- Understand basic JavaScript and HTML concepts
- Be familiar with HTTP requests from client-side JavaScript

## 📚 Learning Objectives

By the end of this exercise, you will:

- Configure **Cross-Origin Resource Sharing (CORS)** to allow access from external domains
- Implement **snake_case JSON formatting** for API responses
- Create a **simple JavaScript client** that consumes your API
- Understand **security implications** of different CORS policies
- Learn to build **resilient client applications** that handle API errors gracefully
- Experience the **full API lifecycle** from implementation to consumption

## 🔍 Why This Matters

In real-world applications, preparing APIs for external clients is crucial because:

- Modern web applications often use a separate frontend and backend
- APIs need to be accessible from different domains or origins
- Consistent naming conventions improve developer experience
- Error handling on both server and client sides creates resilient applications
- Understanding both sides of the API interaction gives you a complete picture
- These skills are directly applicable to professional web development

## 📝 Step-by-Step Instructions

### Step 1: Configure CORS for Your API

**Introduction**: Cross-Origin Resource Sharing (CORS) is a security feature implemented by browsers that blocks web applications from making requests to a different domain than the one that served the web application. To allow your API to be accessed from applications hosted on different domains, you need to configure CORS.

1. Update `Program.cs` to add CORS services and middleware:

    > `src/MerchStore.WebUI/Program.cs`

    ```csharp
    // Add this after other service registrations
    builder.Services.AddCors(options =>
    {
        options.AddPolicy("AllowAllOrigins",
            builder =>
            {
                builder.AllowAnyOrigin()  // Allow requests from any origin
                       .AllowAnyHeader()  // Allow any headers
                       .AllowAnyMethod(); // Allow any HTTP method
            });
    });

    // ... existing code ...

    // Add this before authentication middleware (app.UseAuthentication())
    app.UseCors("AllowAllOrigins");
    ```

> 💡 **Information**
>
> - **Same-Origin Policy**: Browsers prevent web pages from making requests to a different domain by default
> - **CORS Headers**: The server sends special headers that tell the browser which origins are allowed to access the resources
> - **AllowAnyOrigin**: In production, you'd typically restrict to specific origins rather than allowing any origin
> - **Middleware Order**: CORS middleware should be registered before authentication middleware
>
> ⚠️ **Common Mistakes**
>
> - Not placing the `UseCors` call in the correct position in the middleware pipeline
> - Using `AllowAnyOrigin` in production environments (security risk)
> - Forgetting to allow the necessary HTTP methods or headers
> - Not understanding that CORS is enforced by browsers, not by the server itself

### Step 2: Configure Snake Case JSON Formatting

**Introduction**: Many API developers prefer using snake_case for JSON property names because it's more readable and is a common convention in many popular APIs. Let's configure ASP.NET Core to output JSON with snake_case property names.

1. Since .NET doesn't have a built-in snake case naming policy, we need to create one:

    > `src/MerchStore.WebUI/Infrastructure/JsonSnakeCaseNamingPolicy.cs`

    ```csharp
    using System.Text;
    using System.Text.Json;

    namespace MerchStore.WebUI.Infrastructure;

    /// <summary>
    /// A JSON naming policy that converts property names to snake_case.
    /// </summary>
    public class JsonSnakeCaseNamingPolicy : JsonNamingPolicy
    {
        /// <summary>
        /// Converts a property name to snake_case.
        /// </summary>
        /// <param name="name">The property name to convert.</param>
        /// <returns>The snake_case version of the property name.</returns>
        public override string ConvertName(string name)
        {
            if (string.IsNullOrEmpty(name))
            {
                return name;
            }

            var builder = new StringBuilder();
            
            for (int i = 0; i < name.Length; i++)
            {
                if (i > 0 && char.IsUpper(name[i]))
                {
                    builder.Append('_');
                }
                
                builder.Append(char.ToLowerInvariant(name[i]));
            }
            
            return builder.ToString();
        }
    }
    ```

2. Update `Program.cs` to use our custom snake case naming policy:

    > `src/MerchStore.WebUI/Program.cs`

    ```csharp
    using MerchStore.WebUI.Infrastructure;

    // Update the JSON options configuration to use our custom policy
    builder.Services.AddControllersWithViews()
        .AddJsonOptions(options =>
        {
            // Use snake_case for JSON serialization
            options.JsonSerializerOptions.PropertyNamingPolicy = new JsonSnakeCaseNamingPolicy();
            options.JsonSerializerOptions.DictionaryKeyPolicy = new JsonSnakeCaseNamingPolicy();
            options.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());
        });
    ```

> 💡 **Information**
>
> - **Naming Conventions**: snake_case is widely used in APIs from companies like Stripe, GitHub, and Slack
> - **System.Text.Json**: The modern built-in JSON serializer in .NET Core
> - **PropertyNamingPolicy**: Controls how property names are formatted in JSON output
> - **JsonStringEnumConverter**: Converts enum values to strings instead of numbers, which is more readable
>
> ⚠️ **Common Mistakes**
>
> - Not consistently applying the same naming convention across all API endpoints
> - Forgetting to consider the impact on client code when changing naming conventions
> - Not documenting your API's naming convention for API consumers

### Step 3: Create a Simple JavaScript Client

**Introduction**: Let's create a basic web client that demonstrates how to consume your API. This will include HTML for structure, CSS for styling, and JavaScript for making API requests and handling responses.

1. Create a new folder to house the client application:

    ```bash
    mkdir -p src/MerchStore.WebUI/wwwroot/client
    ```

2. Create an HTML file for the client app:

    > `src/MerchStore.WebUI/wwwroot/client/index.html`

    ```html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>MerchStore Client</title>
        <link rel="stylesheet" href="styles.css">
    </head>
    <body>
        <header>
            <h1>MerchStore Product Catalog</h1>
            <p class="subtitle">External Client Demo</p>
        </header>
        
        <main>
            <!-- API Connection Status -->
            <section class="status-container">
                <div id="api-status" class="status-indicator pending">
                    Connecting to API...
                </div>
            </section>
            
            <!-- Product Grid -->
            <section class="products-container">
                <h2>Products</h2>
                <div id="products-grid" class="products-grid">
                    <div class="loader"></div>
                </div>
            </section>
            
            <!-- Product Details Modal -->
            <div id="product-modal" class="modal">
                <div class="modal-content">
                    <span class="close-button">&times;</span>
                    <div id="product-details" class="product-details">
                        <!-- Product details will be inserted here -->
                    </div>
                </div>
            </div>
        </main>
        
        <!-- Product Card Template -->
        <template id="product-card-template">
            <div class="product-card">
                <div class="product-image">
                    <img src="" alt="Product Image">
                </div>
                <div class="product-info">
                    <h3 class="product-name"></h3>
                    <p class="product-price"></p>
                    <p class="product-stock"></p>
                    <button class="view-details-button">View Details</button>
                </div>
            </div>
        </template>
        
        <footer>
            <p>&copy; 2025 MerchStore Demo Client</p>
        </footer>
        
        <!-- Include JavaScript file -->
        <script src="app.js"></script>
    </body>
    </html>
    ```

3. Create a CSS file for styling:

    > `src/MerchStore.WebUI/wwwroot/client/styles.css`

    ```css
    /* Global Styles */
    :root {
        --primary-color: #3498db;
        --secondary-color: #2ecc71;
        --text-color: #333;
        --light-gray: #f9f9f9;
        --border-color: #ddd;
        --error-color: #e74c3c;
        --success-color: #2ecc71;
        --warning-color: #f39c12;
    }

    * {
        box-sizing: border-box;
        margin: 0;
        padding: 0;
    }

    body {
        font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        line-height: 1.6;
        color: var(--text-color);
        background-color: var(--light-gray);
    }

    /* Header */
    header {
        background-color: var(--primary-color);
        color: white;
        padding: 2rem;
        text-align: center;
    }

    header h1 {
        margin-bottom: 0.5rem;
    }

    .subtitle {
        font-size: 1.1rem;
        opacity: 0.8;
    }

    /* Main Content */
    main {
        max-width: 1200px;
        margin: 0 auto;
        padding: 2rem;
    }

    /* API Status Indicator */
    .status-container {
        margin-bottom: 2rem;
    }

    .status-indicator {
        display: inline-block;
        padding: 0.5rem 1rem;
        border-radius: 4px;
        font-weight: bold;
    }

    .pending {
        background-color: var(--warning-color);
        color: white;
    }

    .connected {
        background-color: var(--success-color);
        color: white;
    }

    .error {
        background-color: var(--error-color);
        color: white;
    }

    /* Products Grid */
    .products-container h2 {
        margin-bottom: 1rem;
    }

    .products-grid {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
        gap: 2rem;
        min-height: 300px;
        position: relative;
    }

    /* Product Card */
    .product-card {
        background-color: white;
        border-radius: 8px;
        overflow: hidden;
        box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
        transition: transform 0.3s ease;
    }

    .product-card:hover {
        transform: translateY(-5px);
    }

    .product-image {
        height: 200px;
        overflow: hidden;
    }

    .product-image img {
        width: 100%;
        height: 100%;
        object-fit: cover;
    }

    .product-info {
        padding: 1rem;
    }

    .product-name {
        margin-bottom: 0.5rem;
        font-size: 1.2rem;
    }

    .product-price {
        font-weight: bold;
        color: var(--primary-color);
        margin-bottom: 0.5rem;
    }

    .product-stock {
        margin-bottom: 1rem;
        font-size: 0.9rem;
    }

    .in-stock {
        color: var(--success-color);
    }

    .out-of-stock {
        color: var(--error-color);
    }

    .view-details-button {
        background-color: var(--primary-color);
        color: white;
        border: none;
        padding: 0.5rem 1rem;
        border-radius: 4px;
        cursor: pointer;
        width: 100%;
        font-size: 1rem;
        transition: background-color 0.3s ease;
    }

    .view-details-button:hover {
        background-color: #2980b9;
    }

    /* Modal */
    .modal {
        display: none;
        position: fixed;
        z-index: 1000;
        left: 0;
        top: 0;
        width: 100%;
        height: 100%;
        background-color: rgba(0, 0, 0, 0.5);
    }

    .modal-content {
        background-color: white;
        margin: 10% auto;
        padding: 2rem;
        border-radius: 8px;
        max-width: 600px;
        position: relative;
    }

    .close-button {
        position: absolute;
        top: 1rem;
        right: 1rem;
        font-size: 1.5rem;
        cursor: pointer;
    }

    .product-details {
        margin-top: 1rem;
    }

    .product-details h2 {
        margin-bottom: 1rem;
        color: var(--primary-color);
    }

    .product-details-image {
        width: 100%;
        max-height: 300px;
        object-fit: cover;
        border-radius: 4px;
        margin-bottom: 1rem;
    }

    .product-description {
        margin-bottom: 1rem;
        line-height: 1.8;
    }

    .product-meta {
        display: flex;
        justify-content: space-between;
        margin-top: 2rem;
        padding-top: 1rem;
        border-top: 1px solid var(--border-color);
    }

    /* Loader */
    .loader {
        border: 5px solid #f3f3f3;
        border-top: 5px solid var(--primary-color);
        border-radius: 50%;
        width: 50px;
        height: 50px;
        animation: spin 1s linear infinite;
        position: absolute;
        top: 50%;
        left: 50%;
        margin-top: -25px;
        margin-left: -25px;
    }

    @keyframes spin {
        0% { transform: rotate(0deg); }
        100% { transform: rotate(360deg); }
    }

    /* Footer */
    footer {
        text-align: center;
        padding: 2rem;
        background-color: var(--text-color);
        color: white;
    }

    /* Responsive Design */
    @media (max-width: 768px) {
        .products-grid {
            grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
        }
        
        .modal-content {
            margin: 20% auto;
            width: 90%;
        }
    }
    ```

4. Create a JavaScript file for the client logic:

    > `src/MerchStore.WebUI/wwwroot/client/app.js`

    ```javascript
    // Configuration
    const API_CONFIG = {
        baseUrl: 'https://localhost:7188',  // Update with your API's URL
        endpoints: {
            products: '/api/basic/products'
        },
        headers: {
            'X-API-Key': 'API_KEY',  // Update with your API key
            'Content-Type': 'application/json',
            'Accept': 'application/json'
        }
    };

    // DOM Elements
    const apiStatus = document.getElementById('api-status');
    const productsGrid = document.getElementById('products-grid');
    const productModal = document.getElementById('product-modal');
    const productDetails = document.getElementById('product-details');
    const closeButton = document.querySelector('.close-button');
    const productCardTemplate = document.getElementById('product-card-template');

    // Event Listeners
    document.addEventListener('DOMContentLoaded', initialize);
    closeButton.addEventListener('click', closeModal);
    window.addEventListener('click', (event) => {
        if (event.target === productModal) {
            closeModal();
        }
    });

    /**
     * Initialize the application
     */
    function initialize() {
        // Always render the UI first, regardless of API connectivity
        renderWelcomeMessage();
        
        // Then attempt to connect to the API
        fetchProducts()
            .then(products => {
                updateApiStatus('connected', 'Connected to API');
                renderProductGrid(products);
            })
            .catch(error => {
                console.error('Error connecting to API:', error);
                updateApiStatus('error', `API Connection Error: ${error.message}`);
                renderErrorMessage('Could not load products from the API. Please try again later.');
            });
    }

    /**
     * Render a welcome message in the products grid
     */
    function renderWelcomeMessage() {
        productsGrid.innerHTML = `
            <div class="welcome-message">
                <h3>Welcome to the MerchStore Client Demo!</h3>
                <p>This application demonstrates how to consume the MerchStore API from a JavaScript client.</p>
                <p>Loading products from the API...</p>
            </div>
        `;
    }

    /**
     * Render an error message in the products grid
     */
    function renderErrorMessage(message) {
        productsGrid.innerHTML = `
            <div class="error-message">
                <h3>Oops! Something went wrong</h3>
                <p>${message}</p>
                <button onclick="initialize()" class="retry-button">Retry</button>
            </div>
        `;
    }

    /**
     * Update the API status indicator
     */
    function updateApiStatus(status, message) {
        apiStatus.className = `status-indicator ${status}`;
        apiStatus.textContent = message;
    }

    /**
     * Fetch all products from the API
     */
    async function fetchProducts() {
        const response = await fetch(`${API_CONFIG.baseUrl}${API_CONFIG.endpoints.products}`, {
            method: 'GET',
            headers: API_CONFIG.headers
        });

        if (!response.ok) {
            throw new Error(`API returned status: ${response.status}`);
        }

        return await response.json();
    }

    /**
     * Fetch a specific product by ID
     */
    async function fetchProductById(productId) {
        const response = await fetch(`${API_CONFIG.baseUrl}${API_CONFIG.endpoints.products}/${productId}`, {
            method: 'GET',
            headers: API_CONFIG.headers
        });

        if (!response.ok) {
            throw new Error(`API returned status: ${response.status}`);
        }

        return await response.json();
    }

    /**
     * Render the products grid
     */
    function renderProductGrid(products) {
        // Clear the loader
        productsGrid.innerHTML = '';
        
        // Handle empty products array
        if (!products || products.length === 0) {
            productsGrid.innerHTML = '<p>No products available.</p>';
            return;
        }
        
        // Create a document fragment to improve performance
        const fragment = document.createDocumentFragment();
        
        // Create product cards
        products.forEach(product => {
            const productCard = createProductCard(product);
            fragment.appendChild(productCard);
        });
        
        // Append all products to the grid
        productsGrid.appendChild(fragment);
    }

    /**
     * Create a product card from the template
     */
    function createProductCard(product) {
        // Clone the template
        const productCard = productCardTemplate.content.cloneNode(true);
        
        // Set product data
        const image = productCard.querySelector('.product-image img');
        if (product.image_url) {
            image.src = product.image_url;
            image.alt = product.name;
        } else {
            image.src = 'https://via.placeholder.com/300x200?text=No+Image';
            image.alt = 'No image available';
        }
        
        productCard.querySelector('.product-name').textContent = product.name;
        productCard.querySelector('.product-price').textContent = `${product.price} ${product.currency}`;
        
        const stockElement = productCard.querySelector('.product-stock');
        if (product.in_stock) {
            stockElement.textContent = `In Stock (${product.stock_quantity})`;
            stockElement.classList.add('in-stock');
        } else {
            stockElement.textContent = 'Out of Stock';
            stockElement.classList.add('out-of-stock');
        }
        
        // Add event listener to view details button
        const viewDetailsButton = productCard.querySelector('.view-details-button');
        viewDetailsButton.addEventListener('click', () => openProductDetails(product.id));
        
        return productCard;
    }

    /**
     * Open the product details modal
     */
    async function openProductDetails(productId) {
        try {
            // Show loading state
            productDetails.innerHTML = '<div class="loader"></div>';
            productModal.style.display = 'block';
            
            // Fetch the product details
            const product = await fetchProductById(productId);
            
            // Render the product details
            productDetails.innerHTML = `
                <h2>${product.name}</h2>
                ${product.image_url ? 
                    `<img src="${product.image_url}" alt="${product.name}" class="product-details-image">` : 
                    '<div class="no-image">No image available</div>'}
                <p class="product-description">${product.description}</p>
                <div class="product-meta">
                    <div class="product-price-details">
                        <strong>Price:</strong> ${product.price} ${product.currency}
                    </div>
                    <div class="product-stock-details">
                        <strong>Stock:</strong> 
                        <span class="${product.in_stock ? 'in-stock' : 'out-of-stock'}">
                            ${product.in_stock ? `In Stock (${product.stock_quantity})` : 'Out of Stock'}
                        </span>
                    </div>
                </div>
            `;
        } catch (error) {
            console.error('Error fetching product details:', error);
            productDetails.innerHTML = `
                <div class="error-message">
                    <h3>Error Loading Product</h3>
                    <p>Could not load product details. Please try again later.</p>
                </div>
            `;
        }
    }

    /**
     * Close the product details modal
     */
    function closeModal() {
        productModal.style.display = 'none';
    }
    ```

> 💡 **Information**
>
> - **Single Page Application**: The client is a simple SPA that makes API requests using the Fetch API
> - **Progressive Enhancement**: The client always renders the basic UI first, then attempts to load data
> - **Error Handling**: Both API connection errors and individual request errors are handled gracefully
> - **Modal Pattern**: Product details are displayed in a modal dialog for a smoother user experience
> - **Template Element**: HTML templates are used to create product cards dynamically
>
> ⚠️ **Common Mistakes**
>
> - Hardcoding API URLs and credentials in client-side code (in production, use environment variables)
> - Not handling API errors or network failures in the client
> - Blocking UI rendering until API data is loaded (making the app feel unresponsive)
> - Not considering cross-browser compatibility in your JavaScript code

### Step 4: Test the Complete Solution

**Introduction**: Now that we've configured CORS, implemented snake_case for JSON responses, and created a JavaScript client, let's test the complete solution to ensure everything works together.

1. Run your application:

    ```bash
    dotnet run --project src/MerchStore.WebUI
    ```

2. Open a browser and navigate to the client application:

    ```sh
    https://localhost:7188/client/index.html
    ```

3. Verify that:
   - The client application loads correctly
   - The API connection status shows as "Connected to API"
   - Products are displayed in the grid with their images, names, prices, and stock status
   - Clicking "View Details" opens a modal with more product information
   - The API responses use snake_case property names

4. Use the browser's developer tools to:
   - Inspect the network requests and confirm the correct headers are sent/received
   - Check the console for any errors
   - Verify that CORS is allowing the requests

> 💡 **Information**
>
> - **Developer Tools**: Press F12 in most browsers to open the developer tools
> - **Network Tab**: Shows all HTTP requests, including headers, responses, and timing
> - **Console Tab**: Displays JavaScript errors and log messages
> - **Application Tab**: Can be used to inspect cookies, local storage, and session storage
>
> ⚠️ **Common Mistakes**
>
> - Not checking browser console for JavaScript errors
> - Missing API key or incorrect API URL in the client configuration
> - Forgetting to update JSON property names in the client when switching to snake_case

## 🧪 Final Tests

### Verify Your Cross-Origin Setup

To test that CORS is properly configured, you can:

1. Serve your client app from a different port/domain (using a simple HTTP server)
2. Modify the API URL in the client to point to your ASP.NET Core API
3. Verify that the requests succeed despite coming from a different origin

For example, using Node.js with `http-server`:

```bash
# Install http-server globally if you don't have it
npm install -g http-server

# Serve the existing client files directly
http-server ./src/MerchStore.WebUI/wwwroot/client -p 8080
```

Now visit `http://localhost:8080` and verify that the API requests still work.

### Verify Your JSON Formatting

Check that your API responses are properly formatted with snake_case:

1. Use the client app to trigger API requests
2. Inspect the network requests in the browser's developer tools
3. Check that property names are in snake_case format (e.g., `product_name` instead of `productName`)

✅ **Expected Results**

- The client application loads and displays products from the API
- API requests from different origins are allowed by CORS
- JSON responses use snake_case property names
- The application handles errors gracefully when the API is unavailable
- Product details can be viewed in a modal dialog

## 🔧 Troubleshooting

If you encounter issues:

- **CORS Errors**:
  - Look for errors like "Access to fetch API at ... from origin ... has been blocked by CORS policy"
  - Check that the CORS middleware is registered correctly in Program.cs
  - Ensure the CORS middleware is registered in the correct order (before authentication)

- **JSON Formatting Issues**:
  - Check that your custom naming policy is being applied correctly
  - Verify that property names in API responses are in snake_case format
  - Update client-side code to use the new property names

- **Client Application Problems**:
  - Look for JavaScript errors in the browser console
  - Verify that the API URL and key in the client configuration are correct
  - Check that static files are being served correctly from the wwwroot directory

## 🚀 Optional Challenge

Want to take your learning further? Try:

1. **API Versioning**: Add API versioning to your endpoints and update the client to specify the API version
2. **Pagination**: Implement pagination for the products endpoint and update the client to handle paged results
3. **Search and Filter**: Add search and filter capabilities to the API and client
4. **Error Handling**: Enhance the error handling in both the API and client for a better user experience
5. **Client-side Caching**: Implement caching for API responses using localStorage to improve performance

## 📚 Further Reading

- [CORS in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/cors) - Microsoft's official documentation
- [Naming Conventions in API Design](https://restfulapi.net/resource-naming/) - REST API naming best practices
- [System.Text.Json Documentation](https://docs.microsoft.com/en-us/dotnet/standard/serialization/system-text-json-overview) - Microsoft's JSON serialization library
- [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) - MDN documentation on the Fetch API
- [Progressive Enhancement](https://developer.mozilla.org/en-US/docs/Glossary/Progressive_Enhancement) - Building resilient web applications

## Done! 🎉

Congratulations! You've successfully prepared your API for external consumption by implementing CORS, configuring snake_case JSON formatting, and creating a JavaScript client application. This gives you a complete end-to-end solution that demonstrates the full lifecycle of API development and consumption.

These skills are directly applicable to real-world web development, where separation of backend and frontend concerns is common. The ability to build both robust APIs and resilient clients that can communicate effectively is a valuable skillset in modern web development. 🚀

## What's Next?

In future exercises, you'll explore:

1. **Implementing Minimal APIs**: Creating lightweight API endpoints using .NET's Minimal API approach
2. **Versioned APIs with CQRS**: Building more sophisticated APIs using CQRS pattern with versioning
3. **Real-time Communication**: Adding SignalR for real-time updates between server and clients
4. **Advanced Authentication**: Implementing more sophisticated authentication schemes like JWT
5. **API Documentation**: Creating comprehensive API documentation with Swagger/OpenAPI
