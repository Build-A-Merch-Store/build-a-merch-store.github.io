---
title: "Creating a Product Entity"
date: "2025-04-12"
lastmod: "2025-04-12"
draft: false
weight: 201
toc: true
---

## 🎯 Goal

Implement a foundational Product entity in the Domain layer of our Clean Architecture solution to represent merchandise items in our store.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 1 (Setting Up a Clean Architecture Solution)
- Understand basic C# class implementation
- Be familiar with object-oriented principles like encapsulation

## 📚 Learning Objectives

By the end of this exercise, you will:

- Create a **domain entity** that implements proper encapsulation
- Use **value objects** to represent complex properties
- Implement **domain validation** to enforce business rules
- Define **repository interfaces** in the Domain layer
- Understand how **entities** fit into Clean Architecture

## 🔍 Why This Matters

In real-world applications, well-designed domain entities are crucial because:

- They represent the core business concepts and rules of your application
- They encapsulate business logic and validation where it belongs
- They form the foundation for all operations in your application
- They help maintain consistency and integrity of your business data

## 📝 Step-by-Step Instructions

### Step 1: Create a Generic Base Entity Class

**Introduction**: We'll start by creating a generic base class for all domain entities. This approach is more flexible than a non-generic base class since it allows different ID types (Guid, int, string) for different entities while still providing common functionality for identity and equality.

1. Navigate to the `src/MerchStore.Domain/Common` folder.
2. Create a new file named `Entity.cs` with the following code:

    > `src/MerchStore.Domain/Common/Entity.cs`

    ```csharp
    namespace MerchStore.Domain.Common;

    public abstract class Entity<TId> : IEquatable<Entity<TId>> where TId : notnull
    {
        public TId Id { get; protected set; }

        protected Entity(TId id)
        {
            // Basic validation, can be expanded
            if (EqualityComparer<TId>.Default.Equals(id, default))
            {
                throw new ArgumentException("The entity ID cannot be the default value.", nameof(id));
            }
            Id = id;
        }
    
        // Required for EF Core, even if using private setters elsewhere
        #pragma warning disable CS8618 // Non-nullable field must contain a non-null value when exiting constructor. Consider adding the 'required' modifier or declaring as nullable.
        protected Entity() { }
        #pragma warning restore CS8618 // Non-nullable field must contain a non-null value when exiting constructor. Consider adding the 'required' modifier or declaring as nullable.
    
        public override bool Equals(object? obj)
        {
            return obj is Entity<TId> entity && Id.Equals(entity.Id);
        }
    
        public bool Equals(Entity<TId>? other)
        {
            return Equals((object?)other);
        }
    
        public static bool operator ==(Entity<TId> left, Entity<TId> right)
        {
            return Equals(left, right);
        }
    
        public static bool operator !=(Entity<TId> left, Entity<TId> right)
        {
            return !Equals(left, right);
        }
    
        public override int GetHashCode()
        {
            return Id.GetHashCode();
        }
    }
    ```

> 💡 **Information**
>
> - **Generic Base Class**: This approach is more flexible than a non-generic base class because it allows entities to use different ID types (Guid, int, string, etc.) based on their specific requirements
> - **IEquatable Interface**: Implementing this interface improves performance for equality operations and makes the class more compatible with .NET collections and LINQ operations
> - **Entity Equality**: Entities are compared by ID, not by their property values, which aligns with domain-driven design principles
> - **Protected Setters**: Allow property modification in derived classes and EF Core, but not from outside
> - **Constructor Validation**: Enforces that entities cannot be created with default ID values, which helps maintain data integrity [See Appendix A]
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to call the base constructor when implementing derived entity classes
> - Using public setters for the Id property, which would violate entity identity principles
> - Creating entities with default ID values (e.g., Guid.Empty), which would cause validation errors

### Step 2: Create a Money Value Object

**Introduction**: Money is a classic example of a value object - something defined by its attributes rather than an identity. Creating a dedicated Money type helps prevent common errors like mixing currencies or performing invalid operations.

1. Navigate to the `src/MerchStore.Domain/ValueObjects` folder.
2. Create a new file named `Money.cs`.
3. Add the following code:

    > `src/MerchStore.Domain/ValueObjects/Money.cs`

    ```csharp
    namespace MerchStore.Domain.ValueObjects;

    public record Money
    {
        public decimal Amount { get; }
        public string Currency { get; }

        public Money(decimal amount, string currency)
        {
            // Validate the amount
            if (amount < 0)
                throw new ArgumentException("Money Amount cannot be negative", nameof(amount));

            // Validate the currency
            if (string.IsNullOrWhiteSpace(currency))
                throw new ArgumentException("Money Currency cannot be empty", nameof(currency));

            if (currency.Length != 3)
                throw new ArgumentException("Money Currency code must be 3 characters (ISO 4217 format)", nameof(currency));

            Amount = amount;
            Currency = currency.ToUpper(); // Standardize to uppercase
        }

        // Create SEK currency shorthand using the Static Factory Method design pattern
        public static Money FromSEK(decimal amount) => new Money(amount, "SEK");

        // Add two money values (only if same currency)
        public static Money operator +(Money left, Money right)
        {
            if (left.Currency != right.Currency)
                throw new InvalidOperationException("Cannot add money values with different currencies");

            return new Money(left.Amount + right.Amount, left.Currency);
        }
        
        // Multiply money by a scalar value (e.g., quantity)
        public static Money operator *(Money money, int multiplier)
        {
            return new Money(money.Amount * multiplier, money.Currency);
        }
        
        // Multiply money by a decimal value (for percentages, etc.)
        public static Money operator *(Money money, decimal multiplier)
        {
            if (multiplier < 0)
                throw new ArgumentException("Cannot multiply money by a negative value", nameof(multiplier));
                
            return new Money(money.Amount * multiplier, money.Currency);
        }
        
        // Support for commutative property (int * Money) (var total = 3 * price; // Same as price * 3, resulting in 150 SEK)
        public static Money operator *(int multiplier, Money money)
        {
            return money * multiplier;
        }
        
        // Support for commutative property (decimal * Money)
        public static Money operator *(decimal multiplier, Money money)
        {
            return money * multiplier;
        }

        // Format as string with currency
        // public override string ToString() => $"{Amount:F2} {Currency}";
        // Format as string with invariant culture to avoid localization issues (e.g., decimal separator)
        public override string ToString() => $"{Amount.ToString("F2", System.Globalization.CultureInfo.InvariantCulture)} {Currency}";
    }
    ```

> 💡 **Information**
>
> - **Records**: C# records provide built-in value equality and immutability, making them perfect for value objects
> - **Validation**: Placing validation in the constructor ensures the object is always in a valid state
> - **Money as Value Object**: Financial values should not be represented as primitive types
> - **Operations**: Adding operator overloading allows natural arithmetic operations with proper domain rules
>
> ⚠️ **Common Mistakes**
>
> - Using primitive types (like decimal) for monetary values can lead to currency confusion
> - Without validation, you might create invalid monetary values (negative prices, invalid currencies)
> - Forgetting immutability can result in objects that change unexpectedly

### Step 3: Create a Product Entity

**Introduction**: Now we'll create the actual Product entity - the core business object that represents items in our merchandise store. This entity will enforce business rules and encapsulate product-related behavior.

1. Navigate to the `src/MerchStore.Domain/Entities` folder.
2. Create a new file named `Product.cs`.
3. Add the following code:

    > `src/MerchStore.Domain/Entities/Product.cs`

    ```csharp
    using MerchStore.Domain.Common;
    using MerchStore.Domain.ValueObjects;

    namespace MerchStore.Domain.Entities;

    public class Product : Entity<Guid>
    {
        // Properties with private setters for encapsulation
        public string Name { get; private set; } = string.Empty;
        public string Description { get; private set; } = string.Empty;
        public Money Price { get; private set; } = Money.FromSEK(0);
        public int StockQuantity { get; private set; } = 0;
        public Uri? ImageUrl { get; private set; } = null;

        // Private parameterless constructor for EF Core
        private Product() 
        { 
            // Required for EF Core, but we don't want it to be used directly
        }

        // Public constructor with required parameters
        public Product(string name, string description, Uri? imageUrl, Money price, int stockQuantity) : base(Guid.NewGuid())
        {
            // Validate parameters
            if (string.IsNullOrWhiteSpace(name))
                throw new ArgumentException("Product name cannot be empty", nameof(name));

            if (name.Length > 100)
                throw new ArgumentException("Product name cannot exceed 100 characters", nameof(name));

            if (string.IsNullOrWhiteSpace(description))
                throw new ArgumentException("Product description cannot be empty", nameof(description));

            if (description.Length > 500)
                throw new ArgumentException("Product description cannot exceed 500 characters", nameof(description));

            // Image URI validation
            if (imageUrl != null)
            {
                // Validate URI scheme (only allow http and https)
                if (imageUrl.Scheme != "http" && imageUrl.Scheme != "https")
                    throw new ArgumentException("Image URL must use HTTP or HTTPS protocol", nameof(imageUrl));
                    
                // Validate URI length - using AbsoluteUri to get the full string representation
                if (imageUrl.AbsoluteUri.Length > 2000)
                    throw new ArgumentException("Image URL exceeds maximum length of 2000 characters", nameof(imageUrl));
                    
                // Optional: Validate file extension for images
                string extension = Path.GetExtension(imageUrl.AbsoluteUri).ToLowerInvariant();
                string[] validExtensions = { ".jpg", ".jpeg", ".png", ".gif", ".webp" };
                
                if (!validExtensions.Contains(extension))
                    throw new ArgumentException("Image URL must point to a valid image file (jpg, jpeg, png, gif, webp)", nameof(imageUrl));
            }

            if (price is null)
                throw new ArgumentNullException(nameof(price));

            if (stockQuantity < 0)
                throw new ArgumentException("Stock quantity cannot be negative", nameof(stockQuantity));

            // Set properties
            Name = name;
            Description = description;
            ImageUrl = imageUrl;
            Price = price;
            StockQuantity = stockQuantity;
        }

        // Domain methods that encapsulate business logic
        public void UpdateDetails(string name, string description, Uri? imageUrl)
        {
            // Validate name with clear domain rules
            if (string.IsNullOrWhiteSpace(name))
                throw new ArgumentException("Name cannot be empty", nameof(name));

            if (name.Length > 100)
                throw new ArgumentException("Name cannot exceed 100 characters", nameof(name));

            // Validate description with clear domain rules
            if (string.IsNullOrWhiteSpace(description))
                throw new ArgumentException("Description cannot be empty", nameof(description));
                
            if (description.Length > 500)
                throw new ArgumentException("Description cannot exceed 500 characters", nameof(description));
            
            // Image URI validation
            if (imageUrl != null)
            {
                // Validate URI scheme (only allow http and https)
                if (imageUrl.Scheme != "http" && imageUrl.Scheme != "https")
                    throw new ArgumentException("Image URL must use HTTP or HTTPS protocol", nameof(imageUrl));
                    
                // Validate URI length - using AbsoluteUri to get the full string representation
                if (imageUrl.AbsoluteUri.Length > 2000)
                    throw new ArgumentException("Image URL exceeds maximum length of 2000 characters", nameof(imageUrl));
                    
                // Optional: Validate file extension for images
                string extension = Path.GetExtension(imageUrl.AbsoluteUri).ToLowerInvariant();
                string[] validExtensions = { ".jpg", ".jpeg", ".png", ".gif", ".webp" };
                
                if (!validExtensions.Contains(extension))
                    throw new ArgumentException("Image URL must point to a valid image file (jpg, jpeg, png, gif, webp)", nameof(imageUrl));
            }
            
            // Update properties after all validation passes
            Name = name;
            Description = description;
            ImageUrl = imageUrl;  // Assuming the property name has been updated to imageUrl
        }

        public void UpdatePrice(Money newPrice)
        {
            ArgumentNullException.ThrowIfNull(newPrice);

            Price = newPrice;
        }

        public void UpdateStock(int quantity)
        {
            if (quantity < 0)
                throw new ArgumentException("Stock quantity cannot be negative", nameof(quantity));

            StockQuantity = quantity;
        }


        public bool DecrementStock(int quantity = 1)
        {
            if (quantity <= 0)
                throw new ArgumentException("Quantity must be positive", nameof(quantity));

            if (StockQuantity < quantity)
                return false; // Not enough stock

            StockQuantity -= quantity;
            return true;
        }

        public void IncrementStock(int quantity)
        {
            if (quantity <= 0)
                throw new ArgumentException("Quantity must be positive", nameof(quantity));

            StockQuantity += quantity;
        }
    }
    ```

> 💡 **Information**
>
> - **Domain Entity**: Represents a core business concept with identity
> - **Encapsulation**: Private setters protect properties from invalid direct modification
> - **Validation**: Business rules are enforced within the entity methods
> - **EF Core Support**: Private constructor and protected setters allow Entity Framework to create and track the entity [See Appendix C]
>
> ⚠️ **Common Mistakes**
>
> - Using public setters would allow properties to be changed without validation
> - Not having a parameterless constructor would cause issues with Entity Framework Core
> - Putting too much logic in the entity could violate Single Responsibility Principle
> - Forgetting to validate parameters could lead to inconsistent state

### Step 4: Create a Generic Repository Interface and Product Repository

**Introduction**: To promote code reuse and consistency, we'll first create a generic repository interface for standard CRUD operations, then create a specific repository interface for products that extends this base interface with product-specific operations.

1. Navigate to the `src/MerchStore.Domain/Interfaces` folder.
2. Create a new file named `IRepository.cs`.
3. Add the following code:

    > `src/MerchStore.Domain/Interfaces/IRepository.cs`

    ```csharp
    using MerchStore.Domain.Common;
    
    namespace MerchStore.Domain.Interfaces;
    
    // Generic repository interface for standard CRUD operations
    public interface IRepository<TEntity, TId> 
        where TEntity : Entity<TId> 
        where TId : notnull
    {
        Task<TEntity?> GetByIdAsync(TId id);
        Task<IEnumerable<TEntity>> GetAllAsync();
        Task AddAsync(TEntity entity);
        Task UpdateAsync(TEntity entity);
        Task RemoveAsync(TEntity entity);
    }
    ```

4. Now, create a new file named `IProductRepository.cs`.
5. Add the following code:

    > `src/MerchStore.Domain/Interfaces/IProductRepository.cs`

    ```csharp
    using MerchStore.Domain.Entities;
    
    namespace MerchStore.Domain.Interfaces;
    
    public interface IProductRepository : IRepository<Product, Guid>
    {
        // You can add product-specific methods here if needed
    }
    ```

> 💡 **Information**
>
> - **Generic Repository Pattern**: The base interface provides a template for common operations that all repositories can inherit [See Appendix B]
> - **Type Constraints**: The `where` clauses ensure the interface works only with proper entity types
> - **Separation of Concerns**: The specific repository only needs to define operations unique to products
> - **Interface Inheritance**: IProductRepository inherits all the standard CRUD methods from IRepository
>
> This approach reduces code duplication and enforces consistency across all repository implementations. As your application grows, you can create additional repository interfaces that all follow the same pattern.
>
> ⚠️ **Common Mistakes**
>
> - Creating separate repository interfaces without sharing common operations
> - Defining data access details (like database connections) in the repository interface
> - Adding UI or application-specific concerns to the repository interface
> - Not using async/await for potentially long-running data operations

## 🧪 Final Tests

### Run the Application and Validate Your Work

1. Build the entire solution to ensure all code compiles:

   ```bash
   dotnet build
   ```

2. Run the WebUI project:

   ```bash
   dotnet run --project src/MerchStore.WebUI
   ```

✅ **Expected Results**

- The application should build without errors
- The default ASP.NET Core page should load correctly in the browser

## 🔧 Troubleshooting

If you encounter issues:

- Check that your namespace declarations match the project structure
- Ensure all required `using` statements are included
- Verify that your class names and method signatures match exactly
- Make sure you've created all the necessary folders and files

## 🚀 Optional Challenge

Want to take your learning further? Try:

- Adding inventory tracking functionality to the Product entity
- Creating a method to apply discounts to products
- Adding a property to track product creation and update dates
- Implementing additional value objects for product characteristics (like dimensions or weight)

## 📚 Further Reading

- [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/) - The seminal work on DDD
- [Value Objects in Domain-Driven Design](https://enterprisecraftsmanship.com/posts/value-objects-explained/) - Detailed explanation
- [Entity vs Value Object](https://enterprisecraftsmanship.com/posts/entity-vs-value-object-the-ultimate-list-of-differences/) - Understanding the differences

## Done! 🎉

Great job! You've successfully created a foundational Product entity with proper encapsulation, validation, and domain logic. This entity will serve as a cornerstone for our merchandise store application, representing the products we'll sell to customers. You've also learned how to use value objects to represent complex properties like Money, and how to ensure your domain enforces important business rules. 🚀

## Appendix

### Appendix A: Understanding Constructor Validation in Entity Classes

In the `Entity<TId>` base class, the parameterized constructor includes validation to prevent entities from being created with default ID values:

```csharp
protected Entity(TId id)
{
    // Basic validation, can be expanded
    if (EqualityComparer<TId>.Default.Equals(id, default))
    {
        throw new ArgumentException("The entity ID cannot be the default value.", nameof(id));
    }
    Id = id;
}
```

This validation is important for several reasons:

1. **What are default values?**
   - For `Guid`, the default value is `Guid.Empty` (all zeros)
   - For `int`, the default value is `0`
   - For `string`, the default value is `null`
   - For reference types, the default value is generally `null`

2. **Why validate against default values?**
   - Default values often indicate uninitialized or "empty" data
   - In databases, using default values like `0` or empty GUIDs as IDs could lead to incorrect data relationships or overwritten records
   - Entities need proper identity to maintain domain integrity
   - It prevents the accidental creation of entities without explicitly assigning an ID

3. **How does the validation work?**
   - `EqualityComparer<TId>.Default` provides a type-specific way to compare values of type `TId`
   - `.Equals(id, default)` checks if the provided ID equals the default value for that type
   - If they match (meaning the ID is a default value), an exception is thrown

4. **Benefits for data integrity:**
   - Ensures all entities have valid, non-default identifiers
   - Prevents data corruption scenarios where entities might accidentally share the same default ID
   - Makes domain code more robust by failing early if an entity is created improperly
   - Enforces the principle that entity identity is meaningful and required
   - Creates a consistent rule across all entity types in the application

For example, when you create a new Product entity with:

```csharp
public Product(string name, Money price, string sku) : base(Guid.NewGuid())
{
    // Rest of the constructor...
}
```

The base constructor checks that `Guid.NewGuid()` isn't equal to `Guid.Empty`. Since `Guid.NewGuid()` generates a random GUID, this validation will pass. But if someone tried to create an entity with `Guid.Empty` or `default(Guid)`, the validation would catch this and throw an exception.

### Appendix B: The Repository Pattern and Generic Interfaces

The Repository Pattern is a design pattern that mediates between the domain model and the data mapping layers. It's a crucial part of implementing Clean Architecture and Domain-Driven Design effectively.

#### Core Concepts of the Repository Pattern

1. **Separation of Concerns**
   - Repositories separate business logic from data access logic
   - Domain entities remain focused on business rules without being concerned with how they're persisted
   - Application services can manipulate domain objects without worrying about database operations

2. **Abstraction of Data Access**
   - The repository provides a collection-like interface for accessing domain objects
   - It hides the details of data access technologies (SQL, ORM, etc.) from the rest of the application
   - This abstraction makes it easier to change data access technologies without affecting business logic

3. **Domain-Centric Approach**
   - Repositories are defined in terms of the domain model
   - They work with fully-constructed domain objects, not primitive data types
   - All database queries and commands are translated to and from domain objects

#### Benefits of Using Generic Repositories

The generic repository interface we've created (`IRepository<TEntity, TId>`) provides several advantages:

1. **Code Reuse**
   - Common CRUD operations are defined once and reused for all entity types
   - Reduces repetitive code and ensures consistency
   - Makes adding new entity repositories much faster

2. **Type Safety**
   - Generic type parameters with constraints ensure compile-time type checking
   - Prevents using the wrong ID type with an entity
   - Ensures only valid entity types can be used with repositories

3. **Interface Segregation**
   - Base repository interface contains only operations common to all entities
   - Specific repository interfaces (like `IProductRepository`) add operations relevant to that entity type
   - Clients only depend on the operations they actually need

4. **Testability**
   - Interfaces make it easy to create mock repositories for unit testing
   - Business logic can be tested without hitting the actual database
   - Test doubles can simulate different data scenarios

#### Implementation Strategy in Clean Architecture

In our Clean Architecture solution:

1. **Domain Layer**
   - Contains repository interfaces (`IRepository<TEntity, TId>` and `IProductRepository`)
   - Defines what operations are needed from a business perspective
   - Has no knowledge of how these operations are implemented

2. **Infrastructure Layer**
   - Contains concrete repository implementations
   - Typically uses Entity Framework Core or other data access technologies
   - Implements the interfaces defined in the domain layer

3. **Application Layer**
   - Uses repositories through their interfaces
   - Orchestrates domain objects and repository operations
   - Never directly depends on repository implementations

#### The Importance of Asynchronous Repository Methods

Most of our repository methods use the async/await pattern, which is crucial for modern web applications:

1. **Scalability Benefits**
   - Asynchronous methods free up threads while waiting for I/O operations
   - A server can handle more concurrent requests with the same number of threads
   - Particularly important for database and network operations that have latency

2. **Responsiveness**
   - The application remains responsive even during long-running data operations
   - The UI thread doesn't get blocked in client applications
   - Server resources are used more efficiently

3. **Modern Best Practice**
   - All modern .NET data access APIs (Entity Framework Core, Dapper, etc.) support async operations
   - Using async repositories ensures compatibility with these APIs
   - Aligns with Microsoft's recommendations for ASP.NET Core applications

4. **Practical Implementation**
   - Methods that return data use `Task<T>` return type
   - Methods that don't return data use `Task` return type
   - Implementations use `await` when calling database operations
   - Consumers use `await` when calling repository methods

```csharp
// Example repository implementation using Entity Framework Core
public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _context;
    
    public ProductRepository(AppDbContext context)
    {
        _context = context;
    }
    
    public async Task<Product?> GetByIdAsync(Guid id)
    {
        // The await keyword releases the thread while the database query executes
        return await _context.Products.FindAsync(id);
    }
    
    public async Task<IEnumerable<Product>> GetAllAsync()
    {
        return await _context.Products.ToListAsync();
    }
    
    public async Task AddAsync(Product product)
    {
        await _context.Products.AddAsync(product);
        await _context.SaveChangesAsync();
    }
    
    public async Task UpdateAsync(Product product)
    {
        _context.Products.Update(product);
        await _context.SaveChangesAsync();
    }
    
    public async Task RemoveAsync(Product product)
    {
        _context.Products.Remove(product);
        await _context.SaveChangesAsync();
    }
    
    // Additional methods...
}

// Example usage in an application service
public class ProductService
{
    private readonly IProductRepository _productRepository;
    
    public ProductService(IProductRepository productRepository)
    {
        _productRepository = productRepository;
    }
    
    public async Task<ProductDto> GetProductAsync(Guid id)
    {
        var product = await _productRepository.GetByIdAsync(id);
        
        if (product == null)
            throw new NotFoundException($"Product with ID {id} not found");
            
        return new ProductDto(product.Id, product.Name, product.Price.Amount, product.SKU);
    }
}
```

By defining repositories as interfaces with async methods in the domain layer, we establish a clear contract for data access that supports scalable, responsive applications while maintaining the separation of concerns that Clean Architecture requires.

### Appendix C: POCO Classes in DDD and Clean Architecture

POCO stands for "Plain Old CLR Objects" in .NET (inspired by Java's "POJOs"). These simple classes form the foundation of domain modeling in both Domain-Driven Design and Clean Architecture.

#### What Makes a Class a POCO?

A POCO class:

- Doesn't inherit from framework-specific base classes
- Doesn't implement framework-specific interfaces
- Doesn't have framework-specific attributes or annotations
- Doesn't have special requirements like parameterless constructors

However, a POCO *can* inherit from domain-focused base classes (like our `Entity<TId>`) while still maintaining its "POCO-ness" as long as those base classes themselves don't introduce framework dependencies.

#### POCOs in Domain-Driven Design

In DDD, POCOs serve as:

- **Entities**: Objects with identity and lifecycle (like `Product`)
- **Value Objects**: Immutable objects defined by their attributes (like `Money`)
- **Aggregates**: Clusters of entities and value objects with a root entity

The key advantage is that these domain objects can express business concepts and rules clearly without being cluttered by technical concerns.

```csharp
// A proper POCO entity in our domain model
public class Order : Entity<Guid>
{
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    public Customer Customer { get; private set; }
    public OrderStatus Status { get; private set; }
    public DateTime OrderDate { get; private set; }
    
    // Domain constructor
    public Order(Guid id, Customer customer) : base(id)
    {
        Customer = customer ?? throw new ArgumentNullException(nameof(customer));
        OrderDate = DateTime.UtcNow;
        Status = OrderStatus.New;
    }
    
    // Domain behavior/methods
    public void AddItem(Product product, int quantity)
    {
        if (product == null)
            throw new ArgumentNullException(nameof(product));
            
        if (quantity <= 0)
            throw new ArgumentException("Quantity must be positive", nameof(quantity));
            
        _items.Add(new OrderItem(Guid.NewGuid(), this, product, quantity));
    }
    
    public void Submit()
    {
        if (!_items.Any())
            throw new DomainException("Cannot submit an empty order");
            
        Status = OrderStatus.Submitted;
    }
}
```

#### POCO in Clean Architecture

Within our Clean Architecture approach, POCOs help enforce the Dependency Rule:

- Domain POCOs have no outward dependencies
- Application services use these POCOs but don't change their nature
- Infrastructure concerns (like persistence) adapt to POCOs, not vice versa

This ensures our business rules remain uncontaminated by technical details.

#### Working with Entity Framework

A common challenge is using POCOs with frameworks like Entity Framework Core. We handle this by:

1. Using private parameterless constructors (for EF Core)
2. Employing private/protected setters (for encapsulation)
3. Configuring mapping through Fluent API instead of attributes

```csharp
// EF Core configuration (in Infrastructure layer)
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        
        builder.Property(o => o.OrderDate)
               .IsRequired();
               
        builder.Property(o => o.Status)
               .IsRequired()
               .HasConversion<string>();
               
        builder.HasOne(o => o.Customer)
               .WithMany()
               .IsRequired();
               
        builder.HasMany(o => o.Items)
               .WithOne()
               .HasForeignKey("OrderId");
    }
}
```

#### Benefits for Domain Modeling

Using POCOs in our application delivers several advantages:

- **Domain Focus**: Models express business concepts without technical distractions
- **Testability**: Domain objects can be tested without infrastructure dependencies
- **Maintainability**: Changes to infrastructure don't affect domain logic
- **Flexibility**: Can swap out persistence or UI frameworks without changing domain
- **Clarity**: New developers can understand the business rules by reading the domain model

By maintaining true POCOs in our domain layer, we create a clean separation between business logic and technical concerns, which is the essence of both DDD and Clean Architecture.

### Appendix D: Entity Framework Core Support in Domain Entities

Entity Framework Core (EF Core) is an Object-Relational Mapper (ORM) that needs special accommodations in our domain entities to work properly without compromising domain-driven design principles.

#### Why Entity Framework Core Needs Special Accommodations

EF Core needs to:

1. **Create entities** from database records without calling public constructors
2. **Set property values** directly from database values
3. **Track changes** to entity properties
4. **Lazy load** related entities when needed

However, these requirements can conflict with good domain-driven design principles like encapsulation and validation.

#### Private Parameterless Constructor

```csharp
// Private parameterless constructor for EF Core
private Product() 
{ 
    Name = string.Empty;
    Description = string.Empty;
    Price = Money.FromUsd(0);
}
```

Here's why this is necessary:

1. **Entity Creation Process**
   - When EF Core retrieves data from the database, it needs to create entity instances without calling the public constructor. It does this by:
   - Creating an uninitialized instance using the parameterless constructor
   - Setting property values directly from database values

2. **Accessibility Level**
   - Making the constructor `private` prevents application code from creating invalid entities
   - EF Core can still use it through reflection (special .NET functionality that allows accessing private members)

3. **Default Values**
   - The empty string and default value assignments ensure non-nullable properties have valid initial values
   - This prevents null reference exceptions if code tries to access these properties before EF Core populates them

#### Protected Setters

```csharp
public string Name { get; private set; }
public Money Price { get; private set; }
```

Here's why protected/private setters are important:

1. **Encapsulation Benefits**:
   - `private set` restricts property modifications to methods within the class
   - This ensures all changes go through validation logic
   - It prevents accidental or malicious bypassing of business rules

2. **EF Core Requirements**:
   - EF Core needs to set property values when loading data from the database
   - It can set properties with private setters through reflection
   - When tracking changes, EF Core needs to update property values

3. **Change Tracking**:
   - EF Core detects changes to properties by hooking into property setters
   - Even with private setters, EF Core can detect these changes
   - This allows proper change tracking for SaveChanges operations

#### Example of How It Works

When loading a Product from the database:

1. EF Core executes a SQL query
2. For each row returned, EF Core:
   - Calls the private parameterless constructor using reflection
   - Sets each property value directly using reflection, bypassing the private setters
   - Marks the entity as "unchanged" in its change tracker

When you modify an entity property through a domain method:

```csharp
product.UpdatePrice(new Money(29.99m, "USD"));
```

1. The `UpdatePrice` method validates the new price and sets the property
2. EF Core detects this change through its change tracking
3. When `SaveChangesAsync` is called, EF Core generates and executes the appropriate UPDATE SQL

#### Benefits of This Approach

1. **Domain Integrity**: Your domain entities maintain proper encapsulation and validation
2. **Framework Compatibility**: EF Core can still work with your entities
3. **Clean Code**: Application code must use the proper domain methods to modify entities
4. **Best of Both Worlds**: You get the benefits of both ORM functionality and good domain design

This approach allows you to design your domain entities according to DDD principles without compromising on the practicalities of using Entity Framework Core for data persistence.
