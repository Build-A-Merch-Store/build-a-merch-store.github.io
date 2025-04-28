---
title: "Implementing the Domain Layer for Reviews"
date: "2025-04-28"
lastmod: "2025-04-28"
draft: false
weight: 501
toc: true
---

## 🎯 Goal

Enhance your application by implementing the domain layer components needed to support product reviews, preparing for integration with an external review API.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed previous exercises setting up the domain layer
- Understand basic Domain-Driven Design concepts
- Be familiar with value objects and entities in Clean Architecture
- Have a basic understanding of domain modeling

## 📚 Learning Objectives

By the end of this exercise, you will:

- Create a **Review entity** with appropriate validation and behavior
- Implement a **ReviewStats value object** to encapsulate review statistics
- Define a **ReviewStatus enum** to represent the review approval workflow
- Create the **IReviewRepository interface** to define repository operations
- Understand the **separation of concerns** in the domain layer
- Prepare for **external service integration** while maintaining clean architecture

## 🔍 Why This Matters

In real-world applications, properly designed domain models are crucial because:

- They form the foundation of your application's core business logic
- They ensure data integrity through proper encapsulation and validation
- They enable integration with external services while maintaining clean architecture
- They separate what data looks like (domain model) from how it's stored or retrieved
- They make the application more maintainable by isolating business rules from infrastructure concerns
- They allow for better testing by decoupling domain logic from external dependencies

## 📝 Step-by-Step Instructions

### Step 1: Create the ReviewStatus Enum

**Introduction**: First, we'll define an enum to represent the different states a review can be in. This encapsulates the concept of a review approval workflow and makes the code more expressive and type-safe.

1. Create a directory for enums if it doesn't exist:

    ```bash
    mkdir -p src/MerchStore.Domain/Enums
    ```

2. Create the ReviewStatus enum:

    > `src/MerchStore.Domain/Enums/ReviewStatus.cs`

    ```csharp
    namespace MerchStore.Domain.Enums;

    /// <summary>
    /// Represents the status of a product review
    /// </summary>
    public enum ReviewStatus
    {
        Pending,
        Approved,
        Rejected
    }
    ```

> 💡 **Information**
>
> - **Enums vs. String Constants**: Using an enum instead of string constants provides type safety and makes the code more readable
> - **Domain Language**: The enum values reflect the business process of review moderation
> - **Workflow States**: These states represent a simple approval workflow that customer reviews typically go through
>
> ⚠️ **Common Mistakes**
>
> - Using strings to represent status values, which can lead to typos and inconsistencies
> - Not documenting the meaning of each status value
> - Adding too many statuses that overcomplicate the domain model

### Step 2: Create the ReviewStats Value Object

**Introduction**: Next, we'll create a value object to represent review statistics for a product. Value objects are immutable and are defined by their attributes rather than identity.

1. Create a directory for value objects if it doesn't exist:

    ```bash
    mkdir -p src/MerchStore.Domain/ValueObjects
    ```

2. Create the ReviewStats value object:

    > `src/MerchStore.Domain/ValueObjects/ReviewStats.cs`

    ```csharp
    namespace MerchStore.Domain.ValueObjects;

    public record ReviewStats
    {
        public Guid ProductId { get; }
        public double AverageRating { get; }
        public int ReviewCount { get; }

        public ReviewStats(Guid productId, double averageRating, int reviewCount)
        {
            if (productId == Guid.Empty)
                throw new ArgumentException("Product ID cannot be empty", nameof(productId));

            if (averageRating < 0 || averageRating > 5)
                throw new ArgumentOutOfRangeException(nameof(averageRating), "Average rating must be between 0 and 5");

            if (reviewCount < 0)
                throw new ArgumentOutOfRangeException(nameof(reviewCount), "Review count cannot be negative");

            ProductId = productId;
            AverageRating = averageRating;
            ReviewCount = reviewCount;
        }
    }
    ```

> 💡 **Information**
>
> - **Record Type**: Using C#'s `record` type for value objects provides built-in immutability and value equality
> - **Validation**: Each property is validated in the constructor to ensure the value object is always in a valid state
> - **Immutability**: Properties have getters but no setters, making the object immutable after creation
> - **Domain Logic**: The validation rules (rating between 0-5, counts non-negative) express domain constraints
>
> ⚠️ **Common Mistakes**
>
> - Not validating inputs in the constructor, which can lead to invalid states
> - Using mutable properties, which can make tracking changes difficult
> - Exposing constructors without validation, allowing invalid objects to be created

### Step 3: Create the Review Entity

**Introduction**: Now we'll create the Review entity, which represents a customer's review of a product. Entities have identity and lifecycle, and often contain business rules and behavior.

1. Create a directory for entities if it doesn't exist:

    ```bash
    mkdir -p src/MerchStore.Domain/Entities
    ```

2. Create the Review entity:

    > `src/MerchStore.Domain/Entities/Review.cs`

    ```csharp
    using MerchStore.Domain.Common;
    using MerchStore.Domain.Enums;

    namespace MerchStore.Domain.Entities;

    public class Review : Entity<Guid>
    {
        // Properties with private setters for encapsulation
        public Guid ProductId { get; private set; }
        public string CustomerName { get; private set; } = string.Empty;
        public string Title { get; private set; } = string.Empty;
        public string Content { get; private set; } = string.Empty;
        public int Rating { get; private set; }
        public DateTime CreatedAt { get; private set; }
        public ReviewStatus Status { get; private set; }

        // Private parameterless constructor for EF Core
        private Review() { }

        // Public constructor with required parameters
        public Review(
            Guid id,
            Guid productId,
            string customerName,
            string title,
            string content,
            int rating,
            DateTime createdAt,
            ReviewStatus status) : base(id)
        {
            // Validate parameters
            if (productId == Guid.Empty)
                throw new ArgumentException("Product ID cannot be empty", nameof(productId));

            if (string.IsNullOrWhiteSpace(customerName))
                throw new ArgumentException("Customer name cannot be empty", nameof(customerName));

            if (string.IsNullOrWhiteSpace(title))
                throw new ArgumentException("Review title cannot be empty", nameof(title));

            if (string.IsNullOrWhiteSpace(content))
                throw new ArgumentException("Review content cannot be empty", nameof(content));

            if (rating < 1 || rating > 5)
                throw new ArgumentOutOfRangeException(nameof(rating), "Rating must be between 1 and 5");

            // Set properties
            ProductId = productId;
            CustomerName = customerName;
            Title = title;
            Content = content;
            Rating = rating;
            CreatedAt = createdAt;
            Status = status;
        }
    }
    ```

> 💡 **Information**
>
> - **Inheritance**: The entity inherits from a base `Entity<TId>` class that provides identity and equality behavior
> - **Encapsulation**: Properties have private setters to protect the entity's invariants
> - **Validation**: The constructor validates all parameters to ensure the entity is always in a valid state
> - **ORM Support**: The private parameterless constructor allows Entity Framework Core to create instances
> - **Domain Constraints**: Business rules like "rating must be between 1 and 5" are enforced at the domain level
>
> ⚠️ **Common Mistakes**
>
> - Using public setters that allow bypassing validation
> - Not validating parameters in the constructor
> - Forgetting the parameterless constructor needed for ORM tools
> - Missing proper encapsulation of domain behavior and rules

### Step 4: Create the IReviewRepository Interface

**Introduction**: Finally, we'll define the repository interface for reviews. This interface belongs in the domain layer and defines the operations that can be performed on reviews, but without specifying how they are implemented.

1. Create a directory for interfaces if it doesn't exist:

    ```bash
    mkdir -p src/MerchStore.Domain/Interfaces
    ```

2. Create the IReviewRepository interface:

    > `src/MerchStore.Domain/Interfaces/IReviewRepository.cs`

    ```csharp
    using MerchStore.Domain.Entities;
    using MerchStore.Domain.ValueObjects;

    namespace MerchStore.Domain.Interfaces;

    /// <summary>
    /// Repository interface for accessing product reviews
    /// </summary>
    public interface IReviewRepository
    {
        /// <summary>
        /// Gets both reviews and statistics for a product in a single operation
        /// </summary>
        /// <param name="productId">The product ID</param>
        /// <returns>A tuple containing reviews and review statistics</returns>
        Task<(IEnumerable<Review> Reviews, ReviewStats Stats)> GetProductReviewsAsync(Guid productId);
    }
    ```

> 💡 **Information**
>
> - **Repository Pattern**: The repository provides a clean abstraction for data access operations
> - **Domain Focus**: The interface focuses on domain concepts rather than data access details
> - **Tuple Return**: Using a tuple return value allows fetching both reviews and statistics in a single operation
> - **Interface Segregation**: The interface defines only the operations needed by the domain
> - **Persistence Ignorance**: The domain layer doesn't know or care how the data is actually stored or retrieved
>
> ⚠️ **Common Mistakes**
>
> - Including infrastructure concerns (like database connections) in the domain interface
> - Defining too many specific methods rather than focusing on domain operations
> - Not using asynchronous methods for potentially long-running operations
> - Leaking implementation details through the interface

## 🧪 Final Tests

### Verify Your Domain Components

Since this exercise focuses just on the domain layer without implementation, you'll verify your work by checking for:

1. Correct namespace organization
2. Proper encapsulation and validation in your entity and value object
3. Clear and focused repository interface
4. XML documentation for all public members

✅ **Expected Results**

- A properly defined `ReviewStatus` enum that represents the different states a review can be in
- A `ReviewStats` value object that encapsulates review statistics with proper validation
- A `Review` entity that represents a customer review with encapsulated properties and validation
- An `IReviewRepository` interface that defines how to interact with reviews at the domain level

## 🔧 Troubleshooting

If you encounter issues:

- Check that your namespaces match your project structure
- Ensure all required validation is implemented in constructors
- Verify that properties have appropriate accessibility (private setters for encapsulation)
- Make sure your entity inherits from the base Entity class correctly
- Confirm your repository interface follows the repository pattern correctly

## 🚀 Optional Challenge

Want to take your learning further? Try:

1. Add methods to the `Review` entity for updating the review status with appropriate validation
2. Implement a `ReviewPolicyService` domain service to encapsulate approval rules
3. Extend the repository interface with methods for adding or updating reviews
4. Create a `ReviewSearchCriteria` value object for filtering reviews

## 📚 Further Reading

- [Value Objects in Domain-Driven Design](https://domaincentric.net/blog/value-objects-in-domain-driven-design) - More about value objects and why they matter
- [Domain-Driven Design: Tackling Complexity in the Heart of Software](https://www.domainlanguage.com/ddd/) - Eric Evans' seminal book on DDD
- [Repository Pattern in DDD](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design) - Microsoft's guide to repositories
- [Effective Aggregate Design](https://dddcommunity.org/library/vernon_2011/) - How to design effective domain models

## Done! 🎉

Great job! You've successfully created the domain layer components needed to work with product reviews. This provides a solid foundation for integrating with an external review API while maintaining clean architecture principles.

The domain layer you've created focuses on the business concepts (reviews, ratings, approval workflow) rather than the technical details of how reviews are stored or retrieved. This separation of concerns will make it easier to implement the actual integration with the external review API in future exercises.

In the next exercise, we'll build on this foundation by implementing the application and infrastructure layers to connect to the external review API while maintaining the clean architecture principles. 🚀
