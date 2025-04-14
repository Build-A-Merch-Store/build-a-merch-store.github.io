---
title: "Test Infrastructure with Polyglot Notebooks"
date: "2025-04-14"
lastmod: "2025-04-14"
draft: false
weight: 204
toc: true
---

## 🎯 Goal

Set up and use Polyglot Notebooks (.dib files) to interactively test your infrastructure layer components, particularly focusing on the Product repository implementation.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 1 (Setting Up a Clean Architecture Solution)
- Have completed Exercise 2 (Creating a Simple Product Entity)
- Have completed Exercise 4 (Implementing the Infrastructure Layer for Products)
- Have Visual Studio Code installed
- Be familiar with basic C# syntax and Entity Framework Core concepts

## 📚 Learning Objectives

By the end of this exercise, you will:

- Install and configure the **Polyglot Notebooks extension** for Visual Studio Code
- Create an **interactive notebook** to test your infrastructure layer
- Use **C# scripting** in notebooks to work with your repositories
- Test **CRUD operations** against an in-memory database
- Understand how to **manually test** components outside the context of a running application
- Get immediate **visual feedback** on code execution and data exploration

## 🔍 Why This Matters

In real-world applications, interactive testing of infrastructure components is valuable because:

- It allows you to test repositories and data access code outside of a full application
- It provides quick feedback loops for exploring how components work
- It serves as living documentation of how to use your infrastructure layer
- It helps diagnose issues with data access code through immediate visual results
- It bridges the gap between unit tests and integration tests with a more exploratory approach

## 📝 Step-by-Step Instructions

### Step 1: Install the Polyglot Notebooks Extension

**Introduction**: Polyglot Notebooks (formerly .NET Interactive Notebooks) allow you to mix executable code, text, and visualizations in one document. This makes them perfect for exploratory testing, documentation, and learning. In this step, we'll install the necessary extension in Visual Studio Code.

1. Open Visual Studio Code.
2. Go to the Extensions view by clicking the Extensions icon in the Activity Bar on the side of the window or press `Ctrl+Shift+X` (Windows/Linux) or `Cmd+Shift+X` (macOS).
3. Search for "Polyglot Notebooks" in the Extensions marketplace.
4. Click "Install" to install the extension.

> 💡 **Information**
>
> - **Polyglot Notebooks**: A VS Code extension that enables interactive code notebooks
> - **Multiple Languages**: Supports multiple languages, including C#, F#, SQL, JavaScript, and more
> - **Interactive Execution**: Allows you to execute code cells one at a time and see results immediately
> - **Rich Output**: Displays tables, charts, and other rich output directly in the notebook
>
> ⚠️ **Common Mistakes**
>
> - Confusing .dib files with .ipynb files (both are notebook formats but work with different extensions)
> - Not installing the .NET SDK, which is required for the C# kernel to work
> - Not restarting VS Code after installing the extension if it doesn't appear to be working

### Step 2: Create a New Notebook for Infrastructure Testing

**Introduction**: Now that we have the necessary extension installed, we'll create a new notebook file specifically for testing our infrastructure layer components.

1. Create a new file in your solution with a `.dib` extension:

    ```bash
    # Navigate to your Infrastructure project
    cd src/MerchStore.Infrastructure

    # Create a new notebook file
    touch "TestInfrastructure.dib"
    ```

2. Open the file in Visual Studio Code. If prompted, select "Polyglot Notebook" as the file type.

3. Add a title and introduction to your notebook:

    ```markdown
    # MerchStore Infrastructure Testing Notebook

    This notebook demonstrates how to manually test the MerchStore Infrastructure layer components, 
    particularly the Product repository implementation with Entity Framework Core.

    We'll test basic CRUD operations against an in-memory database to verify the repository works as expected.
    ```

> 💡 **Information**
>
> - **DIB Extension**: Stands for "dotnet interactive book", the file format for Polyglot Notebooks
> - **Cell Types**: Notebooks support code cells (C#, SQL, etc.) and Markdown cells for documentation
> - **Adding Cells**: Use the + button in the toolbar or keyboard shortcuts to add new cells
> - **Cell Navigation**: Use arrow keys or mouse clicks to navigate between cells
>
> ⚠️ **Common Mistakes**
>
> - Not saving the file with the `.dib` extension
> - Not including proper markdown formatting in markdown cells

### Step 3: Set Up the Notebook Environment

**Introduction**: Before we can test our infrastructure components, we need to reference our project assemblies and any required NuGet packages. This is similar to adding references to a project, but done within the notebook script environment.

1. Add a new C# cell for references to your project assemblies:

    ```csharp
    #r "../MerchStore.Infrastructure/bin/Debug/net9.0/MerchStore.Infrastructure.dll"
    #r "../MerchStore.Infrastructure/bin/Debug/net9.0/MerchStore.Application.dll"
    #r "../MerchStore.Infrastructure/bin/Debug/net9.0/MerchStore.Domain.dll"
    
    #r "nuget: Microsoft.EntityFrameworkCore.InMemory, 9.0.0-preview.2.24128.4"
    #r "nuget: Microsoft.EntityFrameworkCore, 9.0.0-preview.2.24128.4"
    #r "nuget: Microsoft.EntityFrameworkCore.Relational, 9.0.0-preview.2.24128.4"
    
    #r "nuget: Microsoft.Extensions.Logging.Console"
    ```

2. Add a markdown cell explaining the purpose of these references:

    ```markdown
    ## Setting Up the Environment

    Above, we're referencing our project assemblies and necessary NuGet packages. This allows us to:
    - Use our domain entities (Product, Money)
    - Access our infrastructure implementations (repositories, DbContext)
    - Use Entity Framework Core with an in-memory database
    - Configure logging for better visibility

    The `#r` directive is used to reference assemblies, while the `#r "nuget:"` syntax is used to 
    reference packages directly from NuGet.
    ```

> 💡 **Information**
>
> - **#r Directive**: Used to reference assemblies in C# scripting
> - **#r "nuget:"**: A special directive to reference NuGet packages directly
> - **Assembly References**: Must point to the compiled DLLs of your projects
> - **Path Resolution**: Paths are relative to the location of the .dib file
>
> ⚠️ **Common Mistakes**
>
> - Referencing assemblies that don't exist or have incorrect paths
> - Using incompatible package versions
> - Not building your projects before trying to reference them
> - Using absolute paths that won't work across different machines

### Step 4: Create and Configure the DbContext and Repository

**Introduction**: Now that we have our references set up, we'll create instances of our AppDbContext and ProductRepository. This is normally handled by the DI container in a running application, but for our interactive testing, we'll create them manually.

1. Add a markdown cell introducing this section:

    ```markdown
    ## Creating the DbContext and Repository

    In a normal application, the DbContext and repositories are created and managed by the 
    dependency injection container. For our interactive testing, we need to create these 
    manually with the appropriate configuration.

    We'll use EF Core's in-memory database provider for testing, which doesn't require a 
    real database server. This is perfect for testing repository functionality.
    ```

2. Add a C# cell to create the DbContext and repository:

    ```csharp
    using Microsoft.EntityFrameworkCore;
    using MerchStore.Domain.Entities;
    using MerchStore.Domain.ValueObjects;
    using MerchStore.Infrastructure.Persistence;
    using MerchStore.Infrastructure.Persistence.Repositories;
    
    // Create options for an in-memory database
    var options = new DbContextOptionsBuilder<AppDbContext>()
        .UseInMemoryDatabase("InMemTestDb")
        .Options;
    
    // Create the context and repository
    var context = new AppDbContext(options);
    var repository = new ProductRepository(context);
    
    // Display confirmation
    Console.WriteLine("DbContext and ProductRepository created successfully");
    ```

3. Add a C# cell to configure logging and create a seeder:

    ```csharp
    using Microsoft.Extensions.Logging;
    using Microsoft.Extensions.Logging.Console;

    // Create a logger factory and logger
    var loggerFactory = LoggerFactory.Create(builder => builder.AddConsole());
    var logger = loggerFactory.CreateLogger<AppDbContextSeeder>();

    // Create the seeder
    var seeder = new AppDbContextSeeder(context, logger);

    // Seed the database
    await seeder.SeedAsync();

    Console.WriteLine("Database seeded successfully");
    ```

> 💡 **Information**
>
> - **In-Memory Database**: Perfect for testing as it doesn't require a real database server
> - **DbContextOptionsBuilder**: Used to configure the DbContext with the in-memory provider
> - **Logger Configuration**: Helps see log messages during testing
> - **Database Seeder**: Populates the database with sample data for testing
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to create a new in-memory database instance
> - Not using async/await with asynchronous methods like SeedAsync
> - Using different database names across cells, which would create separate in-memory databases

### Step 5: Test Reading Products

**Introduction**: Now that we've seeded product, let's test retrieving products from the database using our repository.

1. Add a markdown cell explaining the read operations:

    ```markdown
    ## Reading Products

    Let's test querying products from the database using our repository. We'll:
    1. Use GetAllAsync to retrieve all products
    2. Count and display the total number of products
    3. Show details of each product to verify they were retrieved correctly
    ```

2. Add a C# cell to retrieve and display products:

    ```csharp
    // Get all products
    var products = await repository.GetAllAsync();
    
    // Display the count
    Console.WriteLine($"Total products: {products.Count()}");

    // Display product details in a table
    var productTable = products.Select(p => new {
        Id = p.Id,
        Name = p.Name,
        Price = p.Price.ToString(),
        Stock = p.StockQuantity,
        ImageUrl = p.ImageUrl?.ToString() ?? "No image"
    }).ToList();

    // Display as a formatted table
    display(productTable);
    ```

3. Add a C# cell to test retrieving a specific product by ID:

    ```csharp
    // Get the first product's ID
    var firstProductId = (await repository.GetAllAsync()).First().Id;

    // Get a specific product by ID
    var specificProduct = await repository.GetByIdAsync(firstProductId);

    // Display the result
    if (specificProduct != null)
    {
        Console.WriteLine($"Found product: {specificProduct.Name}");
        Console.WriteLine($"Price: {specificProduct.Price}");
        Console.WriteLine($"Description: {specificProduct.Description}");
    }
    else
    {
        Console.WriteLine($"Product with ID {firstProductId} not found!");
    }
    ```

> 💡 **Information**
>
> - **GetAllAsync**: Retrieves all products from the database
> - **GetByIdAsync**: Retrieves a specific product by its ID
> - **display()**: A special notebook function that formats objects as tables
> - **Anonymous Types**: Used to select specific properties for display
>
> ⚠️ **Common Mistakes**
>
> - Trying to access results without awaiting the async method call
> - Not handling the case where GetByIdAsync might return null
> - Using ToList() on large datasets, which could cause performance issues

### Step 6: Test Creating a New Product

**Introduction**: Let's test our first CRUD operation: creating a new product. This will verify that our repository's AddAsync method works correctly and that we can persist entities to the database.

1. Add a markdown cell explaining the create operation:

    ```markdown
    ## Creating a New Product

    Let's test adding a new product using our repository. This will:
    1. Create a new Product entity with valid data
    2. Call the repository's AddAsync method to add it to the context
    3. Call SaveChangesAsync to persist the changes to the database
    4. Display the new product's ID as confirmation
    ```

2. Add a C# cell to create a new product:

    ```csharp
    // Create a new product
    var product = new Product(
        "Testing Notebook",
        "A product created from the Polyglot Notebook",
        new Uri("https://example.com/notebook.jpg"),
        Money.FromSEK(199.99m),
        25);
        
    // Add it to the repository
    await repository.AddAsync(product);
    
    // Save changes to the database
    await context.SaveChangesAsync();

    // Display the result
    Console.WriteLine($"Added product with ID: {product.Id}");
    Console.WriteLine($"Name: {product.Name}");
    Console.WriteLine($"Price: {product.Price}");
    Console.WriteLine($"Stock: {product.StockQuantity}");
    ```

> 💡 **Information**
>
> - **Entity Creation**: Creates a new Product with all required properties
> - **AddAsync**: Adds the entity to the context but doesn't save it yet
> - **SaveChangesAsync**: Persists changes to the database
> - **Immediate Feedback**: The output shows the generated ID and product details
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to call SaveChangesAsync after AddAsync
> - Not using await with async methods
> - Creating entities with invalid data that would fail validation

### Step 7: Test Updating a Product

**Introduction**: Let's test updating a product. This will verify that our repository's UpdateAsync method works correctly and that changes are properly persisted to the database.

1. Add a markdown cell explaining the update operation:

    ```markdown
    ## Updating a Product

    Now let's test updating an existing product using our repository. We'll:
    1. Retrieve a product from the database
    2. Update some of its properties
    3. Call the repository's UpdateAsync method
    4. Save the changes
    5. Verify the update by retrieving the product again
    ```

2. Add a C# cell to update a product:

    ```csharp
    // Get the first product
    var productToUpdate = (await repository.GetAllAsync()).First();
    
    // Display before update
    Console.WriteLine("BEFORE UPDATE:");
    Console.WriteLine($"Name: {productToUpdate.Name}");
    Console.WriteLine($"Price: {productToUpdate.Price}");
    Console.WriteLine($"Stock: {productToUpdate.StockQuantity}");
    
    // Update the product
    productToUpdate.UpdatePrice(Money.FromSEK(productToUpdate.Price.Amount + 50m));
    productToUpdate.UpdateStock(productToUpdate.StockQuantity + 10);
    
    // Call update method
    await repository.UpdateAsync(productToUpdate);
    
    // Save changes
    await context.SaveChangesAsync();
    
    // Retrieve the updated product
    var updatedProduct = await repository.GetByIdAsync(productToUpdate.Id);
    
    // Display after update
    Console.WriteLine("\nAFTER UPDATE:");
    Console.WriteLine($"Name: {updatedProduct!.Name}");
    Console.WriteLine($"Price: {updatedProduct.Price}");
    Console.WriteLine($"Stock: {updatedProduct.StockQuantity}");
    ```

> 💡 **Information**
>
> - **Domain Method Usage**: Uses the domain entity's methods for updates, not direct property setting
> - **UpdateAsync**: Marks the entity as modified in the context
> - **State Verification**: Retrieves the entity again to verify the update was saved
> - **Entity Tracking**: EF Core automatically tracks changes to entities retrieved from the context
>
> ⚠️ **Common Mistakes**
>
> - Trying to modify properties directly when they have private setters
> - Forgetting to call SaveChangesAsync after UpdateAsync
> - Not using the domain entity's methods for updates, bypassing business rules

### Step 8: Test Deleting Products

**Introduction**: Finally, let's test deleting products. This will verify that our repository's RemoveAsync method works correctly and that entities can be properly removed from the database.

1. Add a markdown cell explaining the delete operation:

    ```markdown
    ## Deleting Products

    Finally, let's test deleting products using our repository. We'll:
    1. Get all products
    2. Delete them one by one
    3. Verify the deletion by counting the remaining products
    ```

2. Add a C# cell to delete products:

    ```csharp
    // Get all products
    var productsToDelete = await repository.GetAllAsync();
    var initialCount = productsToDelete.Count();
    
    Console.WriteLine($"Initial product count: {initialCount}");

    // Delete products one by one
    foreach (var p in productsToDelete)
    {
        await repository.RemoveAsync(p);
        await context.SaveChangesAsync();
        Console.WriteLine($"Deleted product: {p.Name} (ID: {p.Id})");
    }

    // Verify all products are deleted
    var remainingProducts = await repository.GetAllAsync();
    Console.WriteLine($"Remaining product count: {remainingProducts.Count()}");

    if (!remainingProducts.Any())
    {
        Console.WriteLine("All products deleted successfully!");
    }
    ```

> 💡 **Information**
>
> - **RemoveAsync**: Marks the entity for deletion in the context
> - **SaveChangesAsync**: Executes the DELETE operation in the database
> - **Verification**: Counts remaining products to ensure deletion was successful
> - **Per-Entity SaveChanges**: Saves after each deletion for clearer feedback
>
> ⚠️ **Common Mistakes**
>
> - Trying to iterate and delete from the same collection
> - Forgetting to call SaveChangesAsync after RemoveAsync
> - Not considering how deletion affects related entities (if any)

### Step 9: Add Visualization and Summary

**Introduction**: Let's enhance our notebook with some visualization and a summary section to make the testing more informative and visually appealing.

1. Add a markdown cell for a Mermaid diagram showing the repository pattern:

    ```markdown
    ## Visual Repository Pattern Overview

    The following diagram shows how the components we've tested fit into the Clean Architecture:

    ```

2. Add a Mermaid cell with a diagram:

    ```sh
    #!mermaid

    flowchart TB
        subgraph Domain
            E[Product Entity]
            I[IProductRepository]
        end
        
        subgraph Infrastructure
            R[ProductRepository]
            DB[(Database)]
        end
        
        subgraph Application
            S[Service/Handler]
        end
        
        S --> I
        I -.-> E
        R --> E
        R --> DB
        R -.implements.-> I
        
        style Domain fill:#f9f,stroke:#333,stroke-width:2px
        style Infrastructure fill:#bbf,stroke:#333,stroke-width:2px
        style Application fill:#bfb,stroke:#333,stroke-width:2px
    ```

3. Add a markdown summary cell:

    ```markdown
    ## Summary

    In this notebook, we've successfully tested:

    1. **Setup**: Creating an in-memory database and repository
    2. **Create**: Adding new products to the database
    3. **Read**: Retrieving all products and specific products by ID
    4. **Update**: Modifying product properties and saving changes
    5. **Delete**: Removing products from the database

    This confirms that our infrastructure layer is working correctly and can perform all the necessary 
    CRUD operations on the Product entity. The repository pattern provides a clean abstraction over 
    the database operations, letting us work with domain entities directly.

    The interactive nature of this notebook makes it easy to explore the behavior of our infrastructure 
    components without having to run the full application. This approach is valuable for:

    - Development-time testing and exploration
    - Demonstrating how components work to other team members
    - Debugging data access issues
    - Creating documentation with live code examples
    ```

> 💡 **Information**
>
> - **Mermaid Diagrams**: A powerful way to create diagrams directly in the notebook
> - **Visualization**: Helps understand how components relate to each other
> - **Summary**: Reinforces what was learned and provides context
> - **Documentation**: Turns the notebook into a learning resource for the team

## 🧪 Final Tests

### Verify Everything Works

1. Make sure your projects are built and the DLLs are available:

   ```bash
   dotnet build
   ```

2. Open the `.dib` file in VS Code with the Polyglot Notebooks extension installed.

3. Run each cell in order (use the "Run Cell" button or keyboard shortcut).

✅ **Expected Results**

- No errors when loading references
- Database context and repository are created successfully
- Products can be created, retrieved, updated, and deleted
- All operations are visible with immediate feedback
- The diagram renders correctly

## 🔧 Troubleshooting

If you encounter issues:

- Make sure your project paths in the `#r` directives match your actual project structure
- Verify that you've built your projects so the DLLs are available
- Check that you're using the correct namespace imports
- Ensure entity properties are being modified using domain methods, not direct property access
- Verify that you're awaiting async methods correctly

## 🚀 Optional Challenge

Want to take your learning further? Try:

- Adding filtering and sorting capabilities to test more advanced repository methods
- Creating a more complex visualization of the data using charts or graphs
- Testing error scenarios and exception handling
- Extending the notebook to test other repositories and entities
- Using SQL cells to inspect the underlying database state directly

## 📚 Further Reading

- [Polyglot Notebooks Documentation](https://github.com/dotnet/interactive) - GitHub repository with documentation
- [.NET Interactive Documentation](https://github.com/dotnet/interactive/blob/main/docs/README.md) - More details on .NET Interactive features
- [Mermaid Diagram Syntax](https://mermaid-js.github.io/mermaid/#/) - Learn more about creating diagrams
- [EF Core Testing Best Practices](https://docs.microsoft.com/en-us/ef/core/testing/) - Microsoft's guidance on testing with EF Core

## Done! 🎉

Congratulations! You've created an interactive notebook for testing your infrastructure layer. This approach provides a more exploratory and visual way to test your repositories compared to traditional unit testing. It's a valuable skill for debugging, learning, and documenting how your infrastructure components work. 🚀
