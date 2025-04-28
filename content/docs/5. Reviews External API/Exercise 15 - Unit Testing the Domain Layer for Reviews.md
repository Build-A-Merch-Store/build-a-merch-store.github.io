---
title: "Unit Testing the Domain Layer for Reviews"
date: "2025-04-28"
lastmod: "2025-04-28"
draft: false
weight: 502
toc: true
---

## 🎯 Goal

Create comprehensive unit tests for the review-related domain components implemented in Exercise 14, ensuring that your business rules are properly enforced and domain behavior functions as expected.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have completed Exercise 14 (Implementing the Domain Layer for Reviews)
- Understand basic unit testing concepts and patterns
- Be familiar with the xUnit testing framework
- Have basic knowledge of mocking using Moq

## 📚 Learning Objectives

By the end of this exercise, you will:

- Write **unit tests** for domain entities and value objects
- Organize tests using the **Arrange-Act-Assert** pattern
- Create **test fixtures** to share setup code across multiple tests
- Implement tests for **validation logic** and business rules
- Use **theory-based tests** for parameterized testing
- Understand how to test **domain invariants** and constraints
- Mock interfaces to isolate domain logic for testing

## 🔍 Why This Matters

In real-world applications, thorough unit testing is crucial because:

- It verifies that your domain model correctly implements business rules
- It provides confidence when making changes or refactoring code
- It serves as executable documentation for how your domain behaves
- It catches bugs and edge cases early in the development process
- It ensures that validation rules are properly enforced
- It helps new team members understand how the domain model works
- It provides a safety net when evolving your code in future iterations

## 📝 Step-by-Step Instructions

### Step 1: Set Up the Test Project and Framework

**Introduction**: First, we'll create a dedicated test project for our domain layer and install the necessary testing packages.

1. Create a directory for your test project:

    ```bash
    mkdir -p tests/MerchStore.Domain.Tests
    ```

2. Create a new test project:

    ```bash
    cd tests/MerchStore.Domain.Tests
    dotnet new xunit
    ```

3. Add a reference to your domain project and install required packages:

    ```bash
    dotnet add reference ../../src/MerchStore.Domain/MerchStore.Domain.csproj
    
    dotnet add tests/MerchStore.Domain.UnitTests/MerchStore.Domain.UnitTests.csproj package FluentAssertions
    dotnet add tests/MerchStore.Domain.UnitTests/MerchStore.Domain.UnitTests.csproj package Moq
    ```

4. Create a directory structure for your tests:

    ```bash
    mkdir -p Entities
    mkdir -p ValueObjects
    mkdir -p Interfaces
    ```

> 💡 **Information**
>
> - **Separate Test Project**: Keeps tests isolated from production code
> - **xUnit**: A popular testing framework for .NET that's lightweight and extensible
> - **Moq**: A mocking library that helps isolate the code being tested
> - **FluentAssertions**: Makes assertions more readable and provides better error messages
> - **Project Structure**: Organizing tests to mirror the structure of the production code makes it easier to locate tests
>
> ⚠️ **Common Mistakes**
>
> - Placing tests in the same project as production code
> - Not using a consistent naming convention for test classes and methods
> - Creating an overly complex directory structure that doesn't reflect the code being tested

### Step 2: Create Tests for the ReviewStatus Enum

**Introduction**: Let's start with some simple tests for our `ReviewStatus` enum to verify its values and behavior.

1. Create a test file for the ReviewStatus enum:

    > `tests/MerchStore.Domain.Tests/Enums/ReviewStatusTests.cs`

    ```csharp
    using System;
    using FluentAssertions;
    using MerchStore.Domain.Enums;
    using Xunit;

    namespace MerchStore.Domain.Tests.Enums;

    public class ReviewStatusTests
    {
        [Fact]
        public void ReviewStatus_ShouldHaveExpectedValues()
        {
            // Arrange & Act - for enums, these steps are typically combined
            var values = Enum.GetValues<ReviewStatus>();
            
            // Assert
            values.Should().Contain(ReviewStatus.Pending);
            values.Should().Contain(ReviewStatus.Approved);
            values.Should().Contain(ReviewStatus.Rejected);
            values.Should().HaveCount(3); // Ensures no unexpected values are added
        }

        [Theory]
        [InlineData(ReviewStatus.Pending, "Pending")]
        [InlineData(ReviewStatus.Approved, "Approved")]
        [InlineData(ReviewStatus.Rejected, "Rejected")]
        public void ReviewStatus_ToString_ShouldReturnExpectedString(ReviewStatus status, string expected)
        {
            // Act
            string result = status.ToString();
            
            // Assert
            result.Should().Be(expected);
        }

        [Fact]
        public void ReviewStatus_DefaultValue_ShouldBePending()
        {
            // Arrange
            ReviewStatus defaultValue = default;
            
            // Act & Assert
            defaultValue.Should().Be(ReviewStatus.Pending);
        }
    }
    ```

> 💡 **Information**
>
> - **Fact vs. Theory**: `[Fact]` represents a single test case, while `[Theory]` allows parameterized tests with multiple data points
> - **InlineData**: Provides test data for theory-based tests
> - **FluentAssertions**: Uses a more readable syntax for assertions with better error messages
> - **Complete Testing**: Checks not just values but behavior like default values and string conversion
>
> ⚠️ **Common Mistakes**
>
> - Not testing for the exact count of enum values, which can lead to undetected additions
> - Forgetting to check the default value of the enum
> - Using string literals instead of the actual enum values in tests, making tests brittle

Try to run the tests:

```bash
dotnet test tests/MerchStore.Domain.UnitTests/MerchStore.Domain.UnitTests.csproj --filter "FullyQualifiedName~ReviewStatusTests"
```

### Step 3: Create Tests for the ReviewStats Value Object

**Introduction**: Now let's create tests for the `ReviewStats` value object, focusing on validation, construction, and equality behavior.

1. Create a test file for the ReviewStats value object:

    > `tests/MerchStore.Domain.Tests/ValueObjects/ReviewStatsTests.cs`

    ```csharp
    using System;
    using FluentAssertions;
    using MerchStore.Domain.ValueObjects;
    using Xunit;

    namespace MerchStore.Domain.Tests.ValueObjects;

    public class ReviewStatsTests
    {
        private readonly Guid _validProductId = Guid.NewGuid();
        private const double _validAverageRating = 4.5;
        private const int _validReviewCount = 10;

        [Fact]
        public void Constructor_WithValidParameters_ShouldCreateInstance()
        {
            // Arrange & Act
            var stats = new ReviewStats(_validProductId, _validAverageRating, _validReviewCount);
            
            // Assert
            stats.ProductId.Should().Be(_validProductId);
            stats.AverageRating.Should().Be(_validAverageRating);
            stats.ReviewCount.Should().Be(_validReviewCount);
        }

        [Fact]
        public void Constructor_WithEmptyProductId_ShouldThrowArgumentException()
        {
            // Arrange
            Guid emptyProductId = Guid.Empty;
            
            // Act
            Action act = () => new ReviewStats(emptyProductId, _validAverageRating, _validReviewCount);
            
            // Assert
            act.Should().Throw<ArgumentException>()
                .WithMessage("*Product ID cannot be empty*")
                .And.ParamName.Should().Be("productId");
        }

        [Theory]
        [InlineData(-0.1)]  // Slightly below minimum
        [InlineData(5.1)]   // Slightly above maximum
        [InlineData(double.NegativeInfinity)]
        [InlineData(double.PositiveInfinity)]
        public void Constructor_WithInvalidAverageRating_ShouldThrowArgumentOutOfRangeException(double invalidRating)
        {
            // Arrange & Act
            Action act = () => new ReviewStats(_validProductId, invalidRating, _validReviewCount);
            
            // Assert
            act.Should().Throw<ArgumentOutOfRangeException>()
                .WithMessage("*Average rating must be between 0 and 5*")
                .And.ParamName.Should().Be("averageRating");
        }

        [Theory]
        [InlineData(0)]     // Edge case - valid
        [InlineData(5)]     // Edge case - valid
        [InlineData(2.5)]   // Middle value - valid
        public void Constructor_WithValidAverageRating_ShouldNotThrow(double validRating)
        {
            // Arrange & Act
            Action act = () => new ReviewStats(_validProductId, validRating, _validReviewCount);
            
            // Assert
            act.Should().NotThrow();
        }

        [Theory]
        [InlineData(-1)]
        [InlineData(int.MinValue)]
        public void Constructor_WithNegativeReviewCount_ShouldThrowArgumentOutOfRangeException(int invalidCount)
        {
            // Arrange & Act
            Action act = () => new ReviewStats(_validProductId, _validAverageRating, invalidCount);
            
            // Assert
            act.Should().Throw<ArgumentOutOfRangeException>()
                .WithMessage("*Review count cannot be negative*")
                .And.ParamName.Should().Be("reviewCount");
        }

        [Theory]
        [InlineData(0)]     // Edge case - valid
        [InlineData(1)]     // Minimum positive value - valid
        [InlineData(1000)]  // Large value - valid
        public void Constructor_WithValidReviewCount_ShouldNotThrow(int validCount)
        {
            // Arrange & Act
            Action act = () => new ReviewStats(_validProductId, _validAverageRating, validCount);
            
            // Assert
            act.Should().NotThrow();
        }

        [Fact]
        public void EqualityOperator_WithSameValues_ShouldBeEqual()
        {
            // Arrange
            var stats1 = new ReviewStats(_validProductId, _validAverageRating, _validReviewCount);
            var stats2 = new ReviewStats(_validProductId, _validAverageRating, _validReviewCount);
            
            // Act & Assert
            stats1.Should().Be(stats2);
            (stats1 == stats2).Should().BeTrue();
            (stats1 != stats2).Should().BeFalse();
            stats1.GetHashCode().Should().Be(stats2.GetHashCode());
        }

        [Fact]
        public void EqualityOperator_WithDifferentValues_ShouldNotBeEqual()
        {
            // Arrange
            var stats1 = new ReviewStats(_validProductId, _validAverageRating, _validReviewCount);
            var stats2 = new ReviewStats(_validProductId, _validAverageRating, _validReviewCount + 1);
            
            // Act & Assert
            stats1.Should().NotBe(stats2);
            (stats1 == stats2).Should().BeFalse();
            (stats1 != stats2).Should().BeTrue();
        }
    }
    ```

> 💡 **Information**
>
> - **Field Initialization**: Using class fields for common test values improves readability and maintainability
> - **Edge Case Testing**: Tests both valid and invalid boundary values (e.g., 0, 5, -0.1, 5.1)
> - **Error Message Verification**: Ensures exception messages are descriptive and parameter names are correct
> - **Value Object Testing**: Focuses on constructor validation and equality behavior, which are key for value objects
> - **Theory Tests**: Uses parameterized tests to cover multiple scenarios with less code
>
> ⚠️ **Common Mistakes**
>
> - Testing only the happy path without checking validation and error conditions
> - Forgetting to test equality behavior for value objects
> - Not verifying both the exception type and the error message
> - Missing edge cases in validation testing

### Step 4: Create Tests for the Review Entity

**Introduction**: Next, let's create comprehensive tests for the `Review` entity, focusing on validation, construction, and domain behavior.

1. Create a test file for the Review entity:

    > `tests/MerchStore.Domain.Tests/Entities/ReviewTests.cs`

    ```csharp
    using System;
    using FluentAssertions;
    using MerchStore.Domain.Entities;
    using MerchStore.Domain.Enums;
    using Xunit;
    
    namespace MerchStore.Domain.Tests.Entities;
    
    public class ReviewTests
    {
        private readonly Guid _validId = Guid.NewGuid();
        private readonly Guid _validProductId = Guid.NewGuid();
        private const string _validCustomerName = "John Doe";
        private const string _validTitle = "Great Product";
        private const string _validContent = "This product exceeded my expectations!";
        private const int _validRating = 5;
        private readonly DateTime _validCreatedAt = DateTime.UtcNow;
        private const ReviewStatus _validStatus = ReviewStatus.Approved;
    
        [Fact]
        public void Constructor_WithValidParameters_ShouldCreateInstance()
        {
            // Arrange & Act
            var review = new Review(
                _validId,
                _validProductId,
                _validCustomerName,
                _validTitle,
                _validContent,
                _validRating,
                _validCreatedAt,
                _validStatus);
    
            // Assert
            review.Id.Should().Be(_validId);
            review.ProductId.Should().Be(_validProductId);
            review.CustomerName.Should().Be(_validCustomerName);
            review.Title.Should().Be(_validTitle);
            review.Content.Should().Be(_validContent);
            review.Rating.Should().Be(_validRating);
            review.CreatedAt.Should().Be(_validCreatedAt);
            review.Status.Should().Be(_validStatus);
        }
    
        [Fact]
        public void Constructor_WithEmptyId_ShouldThrowArgumentException()
        {
            // Arrange & Act
            Action act = () => new Review(
                Guid.Empty,
                _validProductId,
                _validCustomerName,
                _validTitle,
                _validContent,
                _validRating,
                _validCreatedAt,
                _validStatus);
    
            // Assert
            act.Should().Throw<ArgumentException>()
                .WithMessage("*entity ID cannot be*");
        }
    
        [Fact]
        public void Constructor_WithEmptyProductId_ShouldThrowArgumentException()
        {
            // Arrange & Act
            Action act = () => new Review(
                _validId,
                Guid.Empty,
                _validCustomerName,
                _validTitle,
                _validContent,
                _validRating,
                _validCreatedAt,
                _validStatus);
    
            // Assert
            act.Should().Throw<ArgumentException>()
                .WithMessage("*Product ID cannot be empty*")
                .And.ParamName.Should().Be("productId");
        }
    
        [Theory]
        [InlineData(null)]
        [InlineData("")]
        [InlineData("   ")]
        public void Constructor_WithInvalidCustomerName_ShouldThrowArgumentException(string? invalidName)
        {
            // Arrange & Act
            Action act = () => new Review(
                _validId,
                _validProductId,
                invalidName!,
                _validTitle,
                _validContent,
                _validRating,
                _validCreatedAt,
                _validStatus);
    
            // Assert
            act.Should().Throw<ArgumentException>()
                .WithMessage("*Customer name cannot be empty*")
                .And.ParamName.Should().Be("customerName");
        }
    
        [Theory]
        [InlineData(null)]
        [InlineData("")]
        [InlineData("   ")]
        public void Constructor_WithInvalidTitle_ShouldThrowArgumentException(string? invalidTitle)
        {
            // Arrange & Act
            Action act = () => new Review(
                _validId,
                _validProductId,
                _validCustomerName,
                invalidTitle!,
                _validContent,
                _validRating,
                _validCreatedAt,
                _validStatus);
    
            // Assert
            act.Should().Throw<ArgumentException>()
                .WithMessage("*Review title cannot be empty*")
                .And.ParamName.Should().Be("title");
        }
    
        [Theory]
        [InlineData(null)]
        [InlineData("")]
        [InlineData("   ")]
        public void Constructor_WithInvalidContent_ShouldThrowArgumentException(string? invalidContent)
        {
            // Arrange & Act
            Action act = () => new Review(
                _validId,
                _validProductId,
                _validCustomerName,
                _validTitle,
                invalidContent!,
                _validRating,
                _validCreatedAt,
                _validStatus);
    
            // Assert
            act.Should().Throw<ArgumentException>()
                .WithMessage("*Review content cannot be empty*")
                .And.ParamName.Should().Be("content");
        }
    
        [Theory]
        [InlineData(0)]  // Too low
        [InlineData(-1)] // Negative
        [InlineData(6)]  // Too high
        [InlineData(int.MinValue)]
        [InlineData(int.MaxValue)]
        public void Constructor_WithInvalidRating_ShouldThrowArgumentOutOfRangeException(int invalidRating)
        {
            // Arrange & Act
            Action act = () => new Review(
                _validId,
                _validProductId,
                _validCustomerName,
                _validTitle,
                _validContent,
                invalidRating,
                _validCreatedAt,
                _validStatus);
    
            // Assert
            act.Should().Throw<ArgumentOutOfRangeException>()
                .WithMessage("*Rating must be between 1 and 5*")
                .And.ParamName.Should().Be("rating");
        }
    
        [Theory]
        [InlineData(1)] // Minimum valid
        [InlineData(3)] // Middle value
        [InlineData(5)] // Maximum valid
        public void Constructor_WithValidRating_ShouldNotThrow(int validRating)
        {
            // Arrange & Act
            Action act = () => new Review(
                _validId,
                _validProductId,
                _validCustomerName,
                _validTitle,
                _validContent,
                validRating,
                _validCreatedAt,
                _validStatus);
    
            // Assert
            act.Should().NotThrow();
        }
    
        [Fact]
        public void Equals_WithSameId_ShouldBeTrue()
        {
            // Arrange
            var review1 = new Review(
                _validId,
                _validProductId,
                _validCustomerName,
                _validTitle,
                _validContent,
                _validRating,
                _validCreatedAt,
                _validStatus);
    
            var review2 = new Review(
                _validId, // Same ID
                Guid.NewGuid(), // Different product
                "Different Name",
                "Different Title",
                "Different Content",
                4, // Different rating
                DateTime.UtcNow.AddDays(-1), // Different date
                ReviewStatus.Pending); // Different status
    
            // Act & Assert
            review1.Equals(review2).Should().BeTrue();
            (review1 == review2).Should().BeTrue();
            (review1 != review2).Should().BeFalse();
            review1.GetHashCode().Should().Be(review2.GetHashCode());
        }
    
        [Fact]
        public void Equals_WithDifferentId_ShouldBeFalse()
        {
            // Arrange
            var review1 = new Review(
                _validId,
                _validProductId,
                _validCustomerName,
                _validTitle,
                _validContent,
                _validRating,
                _validCreatedAt,
                _validStatus);
    
            var review2 = new Review(
                Guid.NewGuid(), // Different ID
                _validProductId, // Same product
                _validCustomerName, // Same name
                _validTitle, // Same title
                _validContent, // Same content
                _validRating, // Same rating
                _validCreatedAt, // Same date
                _validStatus); // Same status
    
            // Act & Assert
            review1.Equals(review2).Should().BeFalse();
            (review1 == review2).Should().BeFalse();
            (review1 != review2).Should().BeTrue();
        }
    }
    ```

> 💡 **Information**
>
> - **Test Organization**: Tests are organized by feature/behavior to make them easier to understand
> - **Comprehensive Validation Testing**: Checks all validation rules in the constructor
> - **Entity Equality Testing**: Focuses on ID-based equality, which is the correct behavior for entities
> - **Descriptive Test Names**: Method names clearly describe what is being tested
> - **Theory for String Validation**: Uses multiple types of empty strings (null, empty, whitespace)
>
> ⚠️ **Common Mistakes**
>
> - Not testing all validation rules in the constructor
> - Testing equality based on property values rather than ID for entities
> - Missing important edge cases for validation
> - Creating test data directly in each test, leading to duplication and maintenance issues

### Step 5: Create Tests for the IReviewRepository Interface

**Introduction**: Finally, let's create tests for code that uses the `IReviewRepository` interface. Since we're just testing the domain layer, we'll use mocks to isolate our tests from actual implementations.

1. Create a test file for the IReviewRepository interface:

    > `tests/MerchStore.Domain.Tests/Interfaces/ReviewRepositoryTests.cs`

    ```csharp
    using FluentAssertions;
    using MerchStore.Domain.Entities;
    using MerchStore.Domain.Enums;
    using MerchStore.Domain.Interfaces;
    using MerchStore.Domain.ValueObjects;
    using Moq;
    
    namespace MerchStore.Domain.Tests.Interfaces;
    
    public class ReviewRepositoryTests
    {
        private readonly Guid _productId = Guid.NewGuid();
        private readonly Mock<IReviewRepository> _repositoryMock;
    
        public ReviewRepositoryTests()
        {
            _repositoryMock = new Mock<IReviewRepository>();
        }
    
        [Fact]
        public async Task GetProductReviewsAsync_ShouldReturnReviewsAndStats()
        {
            // Arrange
            var reviews = new List<Review>
            {
                new Review(
                    Guid.NewGuid(),
                    _productId,
                    "Customer 1",
                    "Great Product",
                    "I love it!",
                    5,
                    DateTime.UtcNow.AddDays(-2),
                    ReviewStatus.Approved),
                new Review(
                    Guid.NewGuid(),
                    _productId,
                    "Customer 2",
                    "Good Product",
                    "Pretty good.",
                    4,
                    DateTime.UtcNow.AddDays(-1),
                    ReviewStatus.Approved)
            };
    
            var stats = new ReviewStats(_productId, 4.5, 2);
    
            _repositoryMock.Setup(r => r.GetProductReviewsAsync(_productId))
                .ReturnsAsync((reviews, stats));
    
            // Act
            var result = await _repositoryMock.Object.GetProductReviewsAsync(_productId);
    
            // Assert
            result.Reviews.Should().BeEquivalentTo(reviews);
            result.Stats.Should().Be(stats);
        }
    
        [Fact]
        public async Task GetProductReviewsAsync_WithNoReviews_ShouldReturnEmptyListAndZeroStats()
        {
            // Arrange
            var emptyReviews = Enumerable.Empty<Review>();
            var zeroStats = new ReviewStats(_productId, 0, 0);
    
            _repositoryMock.Setup(r => r.GetProductReviewsAsync(_productId))
                .ReturnsAsync((emptyReviews, zeroStats));
    
            // Act
            var result = await _repositoryMock.Object.GetProductReviewsAsync(_productId);
    
            // Assert
            result.Reviews.Should().BeEmpty();
            result.Stats.AverageRating.Should().Be(0);
            result.Stats.ReviewCount.Should().Be(0);
        }
    }
    ```

> 💡 **Information**
>
> - **Mocking**: Using Moq to create a fake implementation of IReviewRepository
> - **Setup**: Configuring mock behavior to return specific values for testing
> - **Constructor for Test Class**: Using constructor to initialize common test objects
> - **Async Testing**: Using async/await to test asynchronous repository methods
> - **Edge Cases**: Testing both with reviews and with empty reviews
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to use async/await when testing async methods
> - Not testing edge cases like empty collections
> - Setting up mocks too broadly, which can hide issues in tests
> - Not verifying that the mock was called with the expected parameters

## 🧪 Final Tests

### Run the Tests and Verify Coverage

1. Run all tests from the command line:

    ```bash
    cd tests/MerchStore.Domain.Tests
    dotnet test
    ```

2. If you have a coverage tool, you can also run the tests with coverage:

    ```bash
    dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover
    ```

✅ **Expected Results**

- All tests should pass, indicating that your domain components are functioning correctly
- You should have good test coverage for all domain components:
  - ReviewStatus enum tests verify its values and behavior
  - ReviewStats value object tests verify validation, construction, and equality
  - Review entity tests verify validation, construction, and identity-based equality
  - IReviewRepository tests verify that the interface works as expected with mocked implementations

## 🔧 Troubleshooting

If you encounter issues:

- Make sure all test classes and the domain project have the same framework version
- Check for typos in test method names (they should start with uppercase letters)
- Verify that fact and theory attributes are properly applied
- Ensure you're using the correct assertion methods from FluentAssertions
- Check that async tests are awaiting the results properly

## 🚀 Optional Challenge

Want to take your learning further? Try:

1. Add **test cases** for additional boundary conditions or edge cases
2. Calculate and improve your **test coverage metrics**
3. Create a **test fixture for common test data** to reduce duplication across test classes
4. Add **integration tests** that use a real repository implementation with an in-memory database

## 📚 Further Reading

- [xUnit Documentation](https://xunit.net/#documentation) - Comprehensive guide to the xUnit testing framework
- [FluentAssertions Documentation](https://fluentassertions.com/introduction) - Learn more about the expressive assertion library
- [Moq Quickstart](https://github.com/moq/moq4) - Guide to the Moq mocking library
- [Clean Code: Unit Tests](https://blog.cleancoder.com/uncle-bob/2013/05/27/TheTransformationPriorityPremise.html) - Robert C. Martin on writing good unit tests
- [The Art of Unit Testing](https://www.artofunittesting.com/) - Roy Osherove's guide to effective unit testing

## Done! 🎉

Great job! You've successfully created comprehensive unit tests for your review domain components. These tests verify that your domain model correctly implements business rules and behaves as expected in various scenarios.

Having a solid suite of unit tests gives you confidence that your domain model is working correctly and will continue to work as expected even as you make changes to the codebase. This is especially important in a Clean Architecture approach, where the domain layer forms the foundation of your application.

In the next exercise, we'll build on this foundation by implementing the application layer for reviews, connecting the domain model to the rest of your application. 🚀
