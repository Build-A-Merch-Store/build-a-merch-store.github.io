---
title: "Implement the Infrastructure Layer for Products"
date: "2025-04-14"
lastmod: "2025-04-14"
draft: false
weight: 203
toc: true
---

## 🎯 Goal

Implement the infrastructure layer components needed to persist and retrieve Product entities using Entity Framework Core, following Clean Architecture principles.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 1 (Setting Up a Clean Architecture Solution)
- Have completed Exercise 2 (Creating a Simple Product Entity)
- Understand basic Entity Framework Core concepts
- Be familiar with repository pattern implementation

## 📚 Learning Objectives

By the end of this exercise, you will:

- Create an **Entity Framework Core DbContext** for the application
- Configure **Entity Type Configuration** for the Product entity
- Implement the **Repository Pattern** using Entity Framework Core
- Set up a **Unit of Work** pattern for transaction management
- Create a **Database Seeder** for sample data
- Configure **Dependency Injection** for infrastructure components

## 🔍 Why This Matters

In real-world applications, the infrastructure layer implementation is crucial because:

- It provides the actual persistence mechanism for your domain entities
- It isolates data access code from your domain and application layers
- It enables testability by allowing you to substitute real repositories with test doubles
- It provides a clean implementation of interfaces defined in the domain layer
- It handles technical concerns like connection strings, migrations, and database setup

## 📝 Step-by-Step Instructions

### Step 1: Add Entity Framework Core Packages

**Introduction**: First, we need to add the required NuGet packages to the Infrastructure project. These packages provide Entity Framework Core functionality, including the in-memory database provider we'll use for development.

1. Navigate to your project directory and add Entity Framework Core packages to the Infrastructure project:

    ```bash
    # Add Entity Framework Core packages
    dotnet add src/MerchStore.Infrastructure/MerchStore.Infrastructure.csproj package Microsoft.EntityFrameworkCore
    
    dotnet add src/MerchStore.Infrastructure/MerchStore.Infrastructure.csproj package Microsoft.EntityFrameworkCore.InMemory
    
    dotnet add src/MerchStore.Infrastructure/MerchStore.Infrastructure.csproj package Microsoft.EntityFrameworkCore.Relational
    ```

2. Update the Infrastructure project file to ensure it has the correct references:

    > `src/MerchStore.Infrastructure/MerchStore.Infrastructure.csproj`

    ```xml
    <Project Sdk="Microsoft.NET.Sdk">

      <ItemGroup>
        <ProjectReference Include="..\MerchStore.Application\MerchStore.Application.csproj" />
      </ItemGroup>

      <ItemGroup>
        <PackageReference Include="Microsoft.EntityFrameworkCore" Version="9.0.0-preview.2.24128.4" />
        <PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="9.0.0-preview.2.24128.4" />
        <PackageReference Include="Microsoft.EntityFrameworkCore.Relational" Version="9.0.0-preview.2.24128.4" />
      </ItemGroup>

      <PropertyGroup>
        <TargetFramework>net9.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <Nullable>enable</Nullable>
      </PropertyGroup>

    </Project>
    ```

> 💡 **Information**
>
> - **EntityFrameworkCore**: The core ORM (Object-Relational Mapping) library
> - **InMemory Provider**: Useful for development and testing without needing a real database
> - **Relational**: Provides base functionality for relational database providers
>
> ⚠️ **Common Mistakes**
>
> - Using incompatible package versions can lead to runtime errors
> - Forgetting to reference the Application project means you won't have access to application interfaces

### Step 2: Create the Database Context

**Introduction**: The DbContext is the primary class that coordinates Entity Framework functionality for a data model. It represents a session with the database and provides APIs for querying and saving entities.

1. Create a folder for persistence-related classes:

    ```bash
    mkdir -p src/MerchStore.Infrastructure/Persistence
    ```

2. Create the `AppDbContext` class:

    > `src/MerchStore.Infrastructure/Persistence/AppDbContext.cs`

    ```csharp
    using Microsoft.EntityFrameworkCore;
    using MerchStore.Domain.Entities;

    namespace MerchStore.Infrastructure.Persistence;

    /// <summary>
    /// The database context that provides access to the database through Entity Framework Core.
    /// This is the central class in EF Core and serves as the primary point of interaction with the database.
    /// </summary>
    public class AppDbContext : DbContext
    {
        /// <summary>
        /// DbSet represents a collection of entities of a specific type in the database.
        /// Each DbSet typically corresponds to a database table.
        /// </summary>
        public DbSet<Product> Products { get; set; }

        /// <summary>
        /// Constructor that accepts DbContextOptions, which allows for configuration to be passed in.
        /// This enables different database providers (SQL Server, In-Memory, etc.) to be used with the same context.
        /// </summary>
        /// <param name="options">The options to be used by the DbContext</param>
        public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
        {
        }

        /// <summary>
        /// This method is called when the model for a derived context is being created.
        /// It allows for configuration of entities, relationships, and other model-building activities.
        /// </summary>
        /// <param name="modelBuilder">Provides a simple API for configuring the model</param>
        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);

            // Apply entity configurations from the current assembly
            // This scans for all IEntityTypeConfiguration implementations and applies them
            modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
        }
    }
    ```

> 💡 **Information**
>
> - **DbContext**: Acts as a session with the database, tracking changes to entities
> - **`DbSet<T>`**: Represents a collection of entities that can be queried and saved
> - **ApplyConfigurationsFromAssembly**: Automatically discovers and applies entity configurations
> - **XML Comments**: Provide helpful documentation for other developers

### Step 3: Create Entity Type Configuration for Product

**Introduction**: Entity Type Configurations allow you to configure how your domain entities map to database tables. This approach keeps entity mapping separate from your domain classes, maintaining clean separation of concerns.

1. Create a folder for entity configurations:

    ```bash
    mkdir -p src/MerchStore.Infrastructure/Persistence/Configurations
    ```

2. Create the Product configuration class:

    > `src/MerchStore.Infrastructure/Persistence/Configurations/ProductConfiguration.cs`

    ```csharp
    using Microsoft.EntityFrameworkCore;
    using Microsoft.EntityFrameworkCore.Metadata.Builders;
    using MerchStore.Domain.Entities;

    namespace MerchStore.Infrastructure.Persistence.Configurations;

    /// <summary>
    /// Configuration class for the Product entity.
    /// This defines how a Product is mapped to the database.
    /// </summary>
    public class ProductConfiguration : IEntityTypeConfiguration<Product>
    {
        /// <summary>
        /// Configures the entity mapping using EF Core's Fluent API.
        /// </summary>
        /// <param name="builder">Entity type builder used to configure the entity</param>
        public void Configure(EntityTypeBuilder<Product> builder)
        {
            // Define the table name explicitly
            builder.ToTable("Products");

            // Configure the primary key
            builder.HasKey(p => p.Id);

            // Configure Name property
            builder.Property(p => p.Name)
                .IsRequired() // NOT NULL constraint
                .HasMaxLength(100); // VARCHAR(100)

            // Configure Description property
            builder.Property(p => p.Description)
                .IsRequired() // NOT NULL constraint
                .HasMaxLength(500); // VARCHAR(500)

            // Configure StockQuantity property
            builder.Property(p => p.StockQuantity)
                .IsRequired(); // NOT NULL constraint

            // Configure ImageUrl property - it's nullable
            builder.Property(p => p.ImageUrl)
                .IsRequired(false); // NULL allowed

            // Configure the owned entity Money as a complex type
            // This maps the Money value object to columns in the Products table
            builder.OwnsOne(p => p.Price, priceBuilder =>
            {
                // Map Amount property to a column named Price
                priceBuilder.Property(m => m.Amount)
                    .HasColumnName("Price")
                    .IsRequired();

                // Map Currency property to a column named Currency
                priceBuilder.Property(m => m.Currency)
                    .HasColumnName("Currency")
                    .HasMaxLength(3)
                    .IsRequired();
            });

            // Add an index on the Name for faster lookups
            builder.HasIndex(p => p.Name);
        }
    }
    ```

> 💡 **Information**
>
> - **IEntityTypeConfiguration**: Interface for configuring entity types in a separate class
> - **Fluent API**: EF Core's fluent API for configuring entities, more powerful than data annotations
> - **Owned Entities**: Value objects like Money are configured as owned entities in EF Core
> - **Indexing**: Adding indexes improves query performance for frequently used columns

### Step 4: Implement Generic Repository Pattern

**Introduction**: A generic repository provides a standard set of data access methods for any entity type. It implements the repository interfaces defined in the domain layer, providing the actual persistence logic.

1. Create a folder for repositories:

    ```bash
    mkdir -p src/MerchStore.Infrastructure/Persistence/Repositories
    ```

2. Create the generic Repository base class:

    > `src/MerchStore.Infrastructure/Persistence/Repositories/Repository.cs`

    ```csharp
    using Microsoft.EntityFrameworkCore;
    using MerchStore.Domain.Common;
    using MerchStore.Domain.Interfaces;
    
    namespace MerchStore.Infrastructure.Persistence.Repositories;
    
    /// <summary>
    /// A generic repository implementation that works with any entity type.
    /// This provides standard CRUD operations for all entities.
    /// </summary>
    /// <typeparam name="TEntity">The entity type this repository works with</typeparam>
    /// <typeparam name="TId">The ID type of the entity</typeparam>
    public class Repository<TEntity, TId> : IRepository<TEntity, TId>
        where TEntity : Entity<TId>
        where TId : notnull
    {
        // The DbContext instance - protected so derived classes can access it
        protected readonly AppDbContext _context;
        
        // DbSet for the specific entity type
        protected readonly DbSet<TEntity> _dbSet;
    
        /// <summary>
        /// Constructor that accepts a DbContext
        /// </summary>
        /// <param name="context">The database context to use</param>
        public Repository(AppDbContext context)
        {
            _context = context;
            _dbSet = context.Set<TEntity>(); // Get the DbSet for this entity type
        }
    
        /// <summary>
        /// Retrieves an entity by its ID
        /// </summary>
        /// <param name="id">The entity's ID</param>
        /// <returns>The entity if found, null otherwise</returns>
        public virtual async Task<TEntity?> GetByIdAsync(TId id)
        {
            return await _dbSet.FindAsync(id);
        }
    
        /// <summary>
        /// Retrieves all entities of this type
        /// </summary>
        /// <returns>A collection of all entities</returns>
        public virtual async Task<IEnumerable<TEntity>> GetAllAsync()
        {
            return await _dbSet.ToListAsync();
        }
    
        /// <summary>
        /// Adds a new entity to the database
        /// </summary>
        /// <param name="entity">The entity to add</param>
        public virtual async Task AddAsync(TEntity entity)
        {
            await _dbSet.AddAsync(entity);
        }
    
        /// <summary>
        /// Updates an existing entity in the database
        /// </summary>
        /// <param name="entity">The entity to update</param>
        public virtual Task UpdateAsync(TEntity entity)
        {
            // Mark the entity as modified
            _context.Entry(entity).State = EntityState.Modified;
            return Task.CompletedTask;
        }
    
        /// <summary>
        /// Removes an entity from the database
        /// </summary>
        /// <param name="entity">The entity to remove</param>
        public virtual Task RemoveAsync(TEntity entity)
        {
            _dbSet.Remove(entity);
            return Task.CompletedTask;
        }
    }
    ```

3. Create the Product Repository implementation:

    > `src/MerchStore.Infrastructure/Persistence/Repositories/ProductRepository.cs`

    ```csharp
    using MerchStore.Domain.Entities;
    using MerchStore.Domain.Interfaces;
    
    namespace MerchStore.Infrastructure.Persistence.Repositories;
    
    /// <summary>
    /// Repository implementation for managing Product entities.
    /// This class inherits from the generic Repository class and adds product-specific functionality.
    /// </summary>
    public class ProductRepository : Repository<Product, Guid>, IProductRepository
    {
        /// <summary>
        /// Constructor that passes the context to the base Repository class
        /// </summary>
        /// <param name="context">The database context</param>
        public ProductRepository(AppDbContext context) : base(context)
        {
        }
    
        // You can add product-specific methods here if needed
    }
    ```

> 💡 **Information**
>
> - **Generic Repository**: Provides a reusable implementation for all entity types
> - **Base Repository**: Implements common CRUD operations defined in the domain interfaces
> - **Specific Repositories**: Inherit from the base and add entity-specific operations
> - **Task-based Async Pattern**: All methods return Tasks for asynchronous operation
>
> ⚠️ **Common Mistakes**
>
> - Not awaiting asynchronous operations can lead to incomplete data persistence
> - Mixing synchronous and asynchronous code can cause deadlocks

### Step 5: Implement the Unit of Work Pattern

**Introduction**: The Unit of Work pattern coordinates the work of multiple repositories, ensuring that all operations are committed together as a transaction. It helps maintain data consistency across multiple entity changes.

1. First, create the application layer interface. We'll assume you don't have this yet:

    ```bash
    mkdir -p src/MerchStore.Application/Common/Interfaces
    ```

2. Create the `IUnitOfWork` interface:

    > `src/MerchStore.Application/Common/Interfaces/IUnitOfWork.cs`

    ```csharp
    namespace MerchStore.Application.Common.Interfaces;
    
    /// <summary>
    /// Interface for the Unit of Work pattern.
    /// This defines operations for transactional work that spans multiple repositories.
    /// </summary>
    /// <remarks>
    /// The Unit of Work pattern offers several significant benefits in a Clean Architecture and CQRS setup:
    /// 
    /// 1. Atomicity: Ensures that a series of database operations either all succeed or all fail,
    ///    maintaining data integrity across multiple repository operations.
    /// 
    /// 2. Consistency: Maintains the consistent state of the database by grouping changes
    ///    and applying them as a single unit.
    /// 
    /// 3. Performance Optimization: Reduces database roundtrips by batching multiple
    ///    changes into a single transaction.
    /// 
    /// 4. Domain Invariants Protection: Ensures that domain rules spanning multiple entities
    ///    are enforced as a whole, preventing partial updates that could violate business rules.
    /// 
    /// 5. Separation from Repository Pattern: While repositories focus on data access operations
    ///    for specific entity types, UnitOfWork coordinates across all repositories.
    /// 
    /// 6. CQRS Support: In a CQRS architecture, commands often need to update multiple aggregates
    ///    atomically, which UnitOfWork facilitates.
    /// 
    /// 7. Transaction Management: Provides explicit control over transaction boundaries,
    ///    offering more granular control than implicit transactions.
    /// 
    /// 8. Testability: Makes it easier to test business logic by allowing transaction
    ///    rollback after tests, leaving the database in its original state.
    /// </remarks>
    public interface IUnitOfWork
    {
        /// <summary>
        /// Saves all changes made in the context to the database
        /// </summary>
        /// <param name="cancellationToken">A token to cancel the operation if needed</param>
        /// <returns>The number of affected entities</returns>
        /// <remarks>
        /// This method persists all tracked changes to the database but operates outside
        /// of an explicit transaction. For simple operations affecting a single entity or
        /// when transaction coordination isn't necessary, this offers better performance.
        /// </remarks>
        Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
        
        /// <summary>
        /// Begins a new transaction
        /// </summary>
        /// <remarks>
        /// This starts a new database transaction that will encompass all subsequent
        /// data operations until explicitly committed or rolled back. This creates
        /// a transactional boundary to ensure multiple operations succeed or fail as a unit.
        /// 
        /// In CQRS, this is typically called at the beginning of a command handler that
        /// will modify multiple aggregates or entities.
        /// </remarks>
        Task BeginTransactionAsync();
        
        /// <summary>
        /// Commits all changes made in the current transaction
        /// </summary>
        /// <remarks>
        /// This finalizes all changes made within the current transaction and persists them 
        /// to the database. Once committed, the changes cannot be rolled back, and the
        /// transaction is complete.
        /// 
        /// In a Command pattern, this is typically called at the end of a command handler
        /// after all business operations have been successfully performed.
        /// </remarks>
        Task CommitTransactionAsync();
        
        /// <summary>
        /// Rolls back all changes made in the current transaction
        /// </summary>
        /// <remarks>
        /// This discards all changes made within the current transaction, reverting the 
        /// database to its state before the transaction began. This is typically called
        /// when an error occurs during processing or when business rules are violated.
        /// 
        /// In error handling scenarios, this ensures no partial updates are applied,
        /// maintaining data consistency even when operations fail.
        /// </remarks>
        Task RollbackTransactionAsync();
    }
    ```

3. Create the repository manager interface:

    > `src/MerchStore.Application/Common/Interfaces/IRepositoryManager.cs`

    ```csharp
    using MerchStore.Domain.Interfaces;
    
    namespace MerchStore.Application.Common.Interfaces;
    
    /// <summary>
    /// Interface for providing access to all repositories.
    /// This acts as an abstraction over individual repositories.
    /// </summary>
    /// <remarks>
    /// The Repository Manager pattern offers several benefits:
    /// 
    /// 1. Single Entry Point: Provides a unified interface to access all repositories,
    ///    reducing the need to inject multiple repositories into services/handlers.
    /// 
    /// 2. Dependency Inversion: The application layer depends on abstractions (this interface)
    ///    rather than concrete implementations, following the Dependency Inversion Principle.
    /// 
    /// 3. Transaction Management: Centralizes transaction handling through the UnitOfWork property,
    ///    ensuring atomic operations across multiple repositories.
    /// 
    /// 4. Testability: Makes it easy to mock repository access in unit tests by substituting
    ///    a test implementation of this interface.
    /// 
    /// 5. Separation of Concerns: Keeps repository access logic separate from business logic
    ///    in the application layer, adhering to Clean Architecture principles.
    /// 
    /// 6. Reduced Coupling: Application layer components (command/query handlers) aren't directly
    ///    coupled to infrastructure concerns like data access implementations.
    /// 
    /// 7. Consistency: Ensures a consistent approach to data access across the application.
    /// </remarks>
    public interface IRepositoryManager
    {
        /// <summary>
        /// Gets the product repository.
        /// </summary>
        /// <remarks>
        /// This property provides access to product-specific data operations without exposing
        /// the concrete implementation details to the application layer.
        /// </remarks>
        IProductRepository ProductRepository { get; }
    
        /// <summary>
        /// Gets the unit of work to commit transactions.
        /// </summary>
        /// <remarks>
        /// The Unit of Work pattern coordinates the work of multiple repositories by
        /// tracking changes made during a business transaction and persisting them in one operation.
        /// This is particularly important in CQRS architecture where commands may affect
        /// multiple aggregates that need to be saved atomically.
        /// </remarks>
        IUnitOfWork UnitOfWork { get; }
    }
    ```

4. Now, implement the `UnitOfWork` class:

    > `src/MerchStore.Infrastructure/Persistence/UnitOfWork.cs`

    ```csharp
    using MerchStore.Application.Common.Interfaces;
    using MerchStore.Infrastructure.Persistence;

    namespace MerchStore.Infrastructure.Persistence;

    /// <summary>
    /// The Unit of Work pattern provides a way to group multiple database operations
    /// into a single transaction that either all succeed or all fail together.
    /// </summary>
    public class UnitOfWork : IUnitOfWork
    {
        private readonly AppDbContext _context;

        /// <summary>
        /// Constructor that accepts a DbContext
        /// </summary>
        /// <param name="context">The database context to use</param>
        public UnitOfWork(AppDbContext context)
        {
            _context = context;
        }

        /// <summary>
        /// Saves all changes made in the context to the database
        /// </summary>
        /// <returns>The number of affected entities</returns>
        public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
        {
            return await _context.SaveChangesAsync(cancellationToken);
        }

        /// <summary>
        /// Begins a new transaction
        /// </summary>
        public async Task BeginTransactionAsync()
        {
            await _context.Database.BeginTransactionAsync();
        }

        /// <summary>
        /// Commits all changes made in the current transaction
        /// </summary>
        public async Task CommitTransactionAsync()
        {
            await _context.Database.CommitTransactionAsync();
        }

        /// <summary>
        /// Rolls back all changes made in the current transaction
        /// </summary>
        public async Task RollbackTransactionAsync()
        {
            await _context.Database.RollbackTransactionAsync();
        }
    }
    ```

5. Implement the Repository Manager:

    > `src/MerchStore.Infrastructure/Persistence/Repositories/RepositoryManager.cs`

    ```csharp
    using MerchStore.Application.Common.Interfaces;
    using MerchStore.Domain.Interfaces;
    
    namespace MerchStore.Infrastructure.Persistence.Repositories;
    
    /// <summary>
    /// Implementation of the Repository Manager pattern.
    /// This class provides a single point of access to all repositories and the unit of work.
    /// </summary>
    public class RepositoryManager : IRepositoryManager
    {
        private readonly IProductRepository _productRepository;
        private readonly IUnitOfWork _unitOfWork;
    
        /// <summary>
        /// Constructor that accepts all required repositories and the unit of work
        /// </summary>
        /// <param name="productRepository">The product repository</param>
        /// <param name="orderRepository">The order repository</param>
        /// <param name="unitOfWork">The unit of work</param>
        public RepositoryManager(IProductRepository productRepository, IUnitOfWork unitOfWork)
        {
            _productRepository = productRepository;
            _unitOfWork = unitOfWork;
        }
    
        /// <inheritdoc/>
        public IProductRepository ProductRepository => _productRepository;
    
    
        /// <inheritdoc/>
        public IUnitOfWork UnitOfWork => _unitOfWork;
    }
    ```

> 💡 **Information**
>
> - **Unit of Work**: Coordinates multiple operations to ensure atomic transactions
> - **Repository Manager**: Provides a facade to access all repositories through one interface
> - **Transaction Management**: Explicit transaction control with begin/commit/rollback
> - **SaveChangesAsync**: Persists tracked entity changes to the database
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to call SaveChangesAsync means your changes won't be persisted
> - Not handling transaction properly could leave database in inconsistent state

### Step 6: Create a Database Seeder

**Introduction**: A database seeder provides initial data for development and testing. It ensures your application has useful data from the start, rather than beginning with an empty database.

1. Create the Database Seeder class:

    > `src/MerchStore.Infrastructure/Persistence/AppDbContextSeeder.cs`

    ```csharp
    using Microsoft.EntityFrameworkCore;
    using Microsoft.Extensions.Logging;
    using MerchStore.Domain.Entities;
    using MerchStore.Domain.ValueObjects;
    
    namespace MerchStore.Infrastructure.Persistence;
    
    /// <summary>
    /// Class for seeding the database with initial data.
    /// This is useful for development, testing, and demos.
    /// </summary>
    public class AppDbContextSeeder
    {
        private readonly ILogger<AppDbContextSeeder> _logger;
        private readonly AppDbContext _context;
    
        /// <summary>
        /// Constructor that accepts the context and a logger
        /// </summary>
        /// <param name="context">The database context to seed</param>
        /// <param name="logger">The logger for logging seed operations</param>
        public AppDbContextSeeder(AppDbContext context, ILogger<AppDbContextSeeder> logger)
        {
            _context = context;
            _logger = logger;
        }
    
        /// <summary>
        /// Seeds the database with initial data
        /// </summary>
        public virtual async Task SeedAsync()
        {
            try
            {
                // Ensure the database is created (only needed for in-memory database)
                // For SQL Server, you would use migrations instead
                await _context.Database.EnsureCreatedAsync();
    
                // Seed products if none exist
                await SeedProductsAsync();
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "An error occurred while seeding the database.");
                throw;
            }
        }
        
        /// <summary>
        /// Seeds the database with sample products
        /// </summary>
        private async Task SeedProductsAsync()
        {
            // Check if we already have products (to avoid duplicate seeding)
            if (!await _context.Products.AnyAsync())
            {
                _logger.LogInformation("Seeding products...");
                
                // Add sample products
                var products = new List<Product>
                {
                    new Product(
                        "Conference T-Shirt", 
                        "A comfortable cotton t-shirt with the conference logo.", 
                        // new Uri("https://example.com/images/tshirt.jpg"),
                        new Uri("https://merchstore202503311226.blob.core.windows.net/images/tshirt.png"),
                        Money.FromSEK(249.99m),
                        50),
                    
                    new Product(
                        "Developer Mug", 
                        "A ceramic mug with a funny programming joke.", 
                        // new Uri("https://example.com/images/mug.jpg"),
                        new Uri("https://merchstore202503311226.blob.core.windows.net/images/mug.png"),
                        Money.FromSEK(149.50m),
                        100),
                    
                    new Product(
                        "Laptop Sticker Pack", 
                        "A set of 5 programming language stickers for your laptop.", 
                        // new Uri("https://example.com/images/stickers.jpg"),
                        new Uri("https://merchstore202503311226.blob.core.windows.net/images/stickers.png"),
                        Money.FromSEK(79.99m),
                        200),
                    
                    new Product(
                        "Branded Hoodie", 
                        "A warm hoodie with the company logo, perfect for cold offices.", 
                        // new Uri("https://example.com/images/hoodie.jpg"),
                        new Uri("https://merchstore202503311226.blob.core.windows.net/images/hoodie.png"),
                        Money.FromSEK(499.99m),
                        25)
                };
    
                await _context.Products.AddRangeAsync(products);
                await _context.SaveChangesAsync();
                
                _logger.LogInformation("Product seeding completed successfully.");
            }
            else
            {
                _logger.LogInformation("Database already contains products. Skipping product seed.");
            }
        }
    }
    ```

> 💡 **Information**
>
> - **Seed Data**: Provides realistic data for development and testing
> - **Conditional Seeding**: Only adds data if the database is empty
> - **Sample Products**: Represents realistic merchandise items for the store
> - **Logging**: Tracks the seeding process for troubleshooting

### Step 7: Create an Infrastructure Dependency Injection Extension

**Introduction**: Dependency Injection Extensions provide a clean way to register all infrastructure services with the application's service container. This centralizes registration logic and makes it reusable across different application entry points.

1. Create a DependencyInjection class in the Infrastructure project root:

    > `src/MerchStore.Infrastructure/DependencyInjection.cs`

    ```csharp
    using Microsoft.EntityFrameworkCore;
    using Microsoft.Extensions.Configuration;
    using Microsoft.Extensions.DependencyInjection;
    using MerchStore.Application.Common.Interfaces;
    using MerchStore.Domain.Interfaces;
    using MerchStore.Infrastructure.Persistence;
    using MerchStore.Infrastructure.Persistence.Repositories;
    
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
> - **Extension Methods**: Clean way to extend IServiceCollection with infrastructure registrations
> - **Service Lifetimes**: Scoped services are created once per request, appropriate for repositories
> - **Configuration Injection**: Allows passing connection strings and other settings
> - **Service Provider Extension**: Helper method for seeding the database from the application startup
> - **Centralized Registration**: All infrastructure services registered in one place

## 🧪 Final Tests

### Ensure Your Implementation Works

1. Build the project to make sure everything compiles:

   ```bash
   dotnet build
   ```

2. Update the `Program.cs` file in the Web project to use your Infrastructure layer:

   ```csharp
    using MerchStore.Infrastructure;
    
    var builder = WebApplication.CreateBuilder(args);
    
    // Add services to the container.
    builder.Services.AddControllersWithViews();
    
    // Add Infrastructure services - this includes DbContext, Repositories, etc.
    builder.Services.AddInfrastructure(builder.Configuration);
    
    var app = builder.Build();
    
    // Configure the HTTP request pipeline.
    if (!app.Environment.IsDevelopment())
    {
        app.UseExceptionHandler("/Home/Error");
        // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
        app.UseHsts();
    }
    else
    {
        // In development, seed the database with test data using the extension method
        app.Services.SeedDatabaseAsync().Wait();
    }
    
    app.UseHttpsRedirection();
    app.UseRouting();
    
    app.UseAuthorization();
    
    app.MapStaticAssets();
    
    app.MapControllerRoute(
        name: "default",
        pattern: "{controller=Home}/{action=Index}/{id?}")
        .WithStaticAssets();
    
    
    app.Run();
   ```

3. Run the web project to verify it starts with the infrastructure layer:

   ```bash
   dotnet run --project src/MerchStore.WebUI
   ```

✅ **Expected Results**

- The application builds successfully
- The web application starts without errors
- Database seeding is successful (check logs)
- The application has access to the product data through repositories

## 🔧 Troubleshooting

If you encounter issues:

- Check that all namespaces match your project structure
- Ensure the application layer interfaces match the infrastructure implementations
- Verify that the dependency injection registration includes all required services
- Check for missing package references or version conflicts

## 🚀 Optional Challenge

Want to take your learning further? Try:

- Adding SQL Server support with migrations instead of the in-memory database
- Implementing a more sophisticated repository with filtering, sorting, and paging
- Adding audit logs to track entity changes in the infrastructure layer
- Implementing soft delete functionality for products

## 📚 Further Reading

- [Entity Framework Core Documentation](https://docs.microsoft.com/en-us/ef/core/) - Microsoft's documentation for EF Core
- [Repository Pattern in .NET Core](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-implementation-entity-framework-core) - Microsoft's guidance on repository pattern
- [Unit of Work Pattern](https://martinfowler.com/eaaCatalog/unitOfWork.html) - Martin Fowler's description of the pattern
- [EF Core Best Practices](https://www.thereformedprogrammer.net/entity-framework-core-performance-tuning-a-worked-example/) - Performance tuning for EF Core

## Done! 🎉

Congratulations! You've successfully implemented the infrastructure layer for the Product entity, following Clean Architecture principles. You now have a complete data access implementation that includes a DbContext, entity configuration, repository implementation, and unit of work. This foundation will allow you to persist and retrieve Product entities, while maintaining a clean separation between your domain, application, and infrastructure layers. 🚀

## Appendix: Understanding Entity Framework Core's Change Tracking

Entity Framework Core relies on a sophisticated change tracking system to know which entities need to be inserted, updated, or deleted when you call `SaveChanges` or `SaveChangesAsync`. Understanding how this works can help you write more efficient repository implementations.

### How Change Tracking Works

1. **Entry Point**: When you retrieve an entity from the database or add a new entity to a DbSet, EF Core starts tracking that entity.

2. **State Management**: Each tracked entity has an associated state:
   - `Added`: The entity is new and will be inserted into the database.
   - `Modified`: The entity exists in the database and has been changed.
   - `Unchanged`: The entity exists in the database and hasn't changed.
   - `Deleted`: The entity exists in the database and will be deleted.
   - `Detached`: The entity is not being tracked.

3. **Property Snapshots**: For tracked entities, EF Core takes a snapshot of the original property values when the entity is loaded from the database.

4. **Change Detection**: When you call `SaveChanges`, EF Core compares the current values with the original snapshot to detect changes.

### How This Affects Repository Implementation

1. **Adding Entities**: When you call `_dbSet.Add(entity)` or `_dbSet.AddAsync(entity)`, the entity is marked as `Added`.

2. **Updating Entities**: For updates, there are several approaches:
   - If you retrieved the entity from the database, modified its properties, and call `SaveChanges`, EF Core automatically detects the changes.
   - If you detached an entity and want to update it later, you need to explicitly mark it as `Modified` using `_context.Entry(entity).State = EntityState.Modified`.

3. **Deleting Entities**: When you call `_dbSet.Remove(entity)`, the entity is marked as `Deleted`.

4. **Explicit vs. Implicit Change Tracking**:
   - **Implicit**: Let EF Core detect changes automatically (more convenient but can be less performant for many changes).
   - **Explicit**: Manually specify which entities and properties have changed (more work but can be more efficient).

### Best Practices

1. **Use AsNoTracking for Read-Only Queries**: If you're just reading data and won't modify it, use `AsNoTracking()` to avoid the overhead of change tracking:

   ```csharp
   var products = await _dbSet.AsNoTracking().ToListAsync();
   ```

2. **Be Careful with Detached Entities**: If an entity becomes detached (e.g., sent to the client and back), you need to handle reattachment carefully:

   ```csharp
   _context.Attach(entity);  // Attach without marking as modified
   _context.Entry(entity).State = EntityState.Modified;  // Mark as modified
   ```

3. **Consider Selective Updates**: For large entities, you can update only specific properties:

   ```csharp
   var entry = _context.Entry(product);
   entry.Property(p => p.Name).IsModified = true;
   entry.Property(p => p.Price).IsModified = true;
   // Other properties won't be updated
   ```

4. **Use ChangeTracker.DetectChanges Carefully**: EF Core calls this internally before SaveChanges, but you can call it explicitly if needed:

   ```csharp
   _context.ChangeTracker.DetectChanges();
   ```

5. **Use ValueObjects Correctly**: As shown in the ProductConfiguration, owned entities like Money need special configuration:

   ```csharp
   builder.OwnsOne(p => p.Price, priceBuilder => {
       priceBuilder.Property(m => m.Amount).HasColumnName("Price");
       priceBuilder.Property(m => m.Currency).HasColumnName("Currency");
   });
   ```

By understanding how EF Core tracks changes, you can optimize your repository implementations for better performance and more reliable data operations.
