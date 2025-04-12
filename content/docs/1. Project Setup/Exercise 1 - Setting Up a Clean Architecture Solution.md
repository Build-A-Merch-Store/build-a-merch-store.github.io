---
title: "Setting Up A Clean Architecture Solution"
date: "2023-09-07"
lastmod: "2025-04-12"
draft: false
weight: 101
toc: true
---

## 🎯 Goal

Create the foundation for a merchandise store application by setting up a Clean Architecture solution structure with the correct project dependencies, organizing all source code in a `src` directory.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have .NET SDK 9.0 or newer installed
- Have VS Code with C# Dev Kit extension installed
- Have basic understanding of ASP.NET Core MVC

## 📚 Learning Objectives

By the end of this exercise, you will:

- Set up a **Clean Architecture** solution structure from scratch
- Create projects for **Domain**, **Application**, **Infrastructure**, and **Presentation** layers
- Configure **project references** to enforce proper dependency flow
- Organize projects in a **src directory** for better code organization
- Verify the solution builds successfully

## 🔍 Why This Matters

In real-world applications, Clean Architecture is crucial because:

- It enables separation of concerns, making the codebase more maintainable
- It's an industry standard approach for building scalable enterprise applications
- It will be foundational for all future functionality we'll build throughout this course
- It makes the application more testable and less coupled to external frameworks

## 📝 Step-by-Step Instructions

### Step 1: Create the Solution Structure

1. Open a terminal and navigate to where you want to create your project.
2. Create a new solution and set up the `src` directory:

    ```bash
    # Create solution
    dotnet new sln -n MerchStore

    # Create src directory
    mkdir -p src

    # Create a gitignore file for .Net
    dotnet new gitignore
    ```

3. Create projects for each Clean Architecture layer in the `src` directory:

    ```bash
    # Domain Layer
    dotnet new classlib -o src/MerchStore.Domain

    # Application Layer
    dotnet new classlib -o src/MerchStore.Application

    # Infrastructure Layer
    dotnet new classlib -o src/MerchStore.Infrastructure

    # Presentation Layer (WebUI)
    dotnet new mvc -o src/MerchStore.WebUI
    ```

4. Add all projects to the solution:

    ```bash
    # Add projects to solution
    dotnet sln add src/MerchStore.Domain/MerchStore.Domain.csproj
    dotnet sln add src/MerchStore.Application/MerchStore.Application.csproj
    dotnet sln add src/MerchStore.Infrastructure/MerchStore.Infrastructure.csproj
    dotnet sln add src/MerchStore.WebUI/MerchStore.WebUI.csproj
    ```

> 💡 **Information**
>
> - **src Directory**: Separates source code from other solution artifacts like tests, docs, etc.
> - **Domain Layer**: Contains enterprise/business logic and entities - the heart of your application
> - **Application Layer**: Contains business rules specific to the application and orchestrates domain objects
> - **Infrastructure Layer**: Implements interfaces defined in inner layers, contains database, external APIs
> - **WebUI**: Acts as the presentation layer in Clean Architecture, handling user interactions
>
> ⚠️ **Common Mistakes**
>
> - Forgetting to organize projects in the src directory will make it harder to add test projects later
> - Creating direct dependencies from inner to outer layers breaks Clean Architecture principles
> - Having too many project references can lead to circular dependencies

### Step 2: Configure Project Dependencies

1. Set up project references according to Clean Architecture principles (dependencies always point inward):

    ```bash
    # Application depends on Domain
    dotnet add src/MerchStore.Application/MerchStore.Application.csproj reference src/MerchStore.Domain/MerchStore.Domain.csproj

    # Infrastructure depends on Application (which already references Domain)
    dotnet add src/MerchStore.Infrastructure/MerchStore.Infrastructure.csproj reference src/MerchStore.Application/MerchStore.Application.csproj

    # WebUI depends on Infrastructure and Application
    dotnet add src/MerchStore.WebUI/MerchStore.WebUI.csproj reference src/MerchStore.Infrastructure/MerchStore.Infrastructure.csproj
    dotnet add src/MerchStore.WebUI/MerchStore.WebUI.csproj reference src/MerchStore.Application/MerchStore.Application.csproj
    ```

> 💡 **Information**
>
> - **Dependency Rule**: This reference structure enforces the Dependency Rule of Clean Architecture
> - **Inner Layer Independence**: Inner layers have no knowledge of outer layers
> - **Domain Purity**: Domain has no dependencies on other project layers
> - This approach ensures that business logic doesn't depend on UI or database implementations

### Step 3: Set Up Basic Folder Structure

1. Create the base folder structure in each project:

    ```bash
    # Domain Layer folders
    mkdir -p src/MerchStore.Domain/Entities
    mkdir -p src/MerchStore.Domain/ValueObjects
    mkdir -p src/MerchStore.Domain/Interfaces
    mkdir -p src/MerchStore.Domain/Common

    # Application Layer folders - service-based approach
    mkdir -p src/MerchStore.Application/Common/Interfaces
    mkdir -p src/MerchStore.Application/Products/Services
    mkdir -p src/MerchStore.Application/DTOs

    # Infrastructure Layer folders
    mkdir -p src/MerchStore.Infrastructure/Persistence
    mkdir -p src/MerchStore.Infrastructure/Persistence/Repositories
    mkdir -p src/MerchStore.Infrastructure/Services
    ```

> 💡 **Information**
>
> - **Entities**: Domain objects with identity and lifecycle
> - **Value Objects**: Immutable objects defined by their attributes
> - **Repositories**: Data access patterns that abstract the underlying data store
> - **Services**: Contain business logic that doesn't naturally fit in domain objects
> - This folder structure promotes a good separation of concerns in your application

### Step 4: Remove the Example Class1.cs Files

1. When you create a class library, an example class `Class1.cs` is created. You can safely remove these files:

    ```bash
    find . -name "Class1.cs" -type f -delete
    ```

> ⚠️ **Common Mistakes**
>
> - Leaving unused template files can cause confusion for new developers
> - Be careful not to delete other important files when running this command

## 🧪 Final Tests

### Run the Application and Validate Your Work

1. Build the entire solution to ensure all projects are correctly set up:

   ```bash
   dotnet build
   ```

2. Run the WebUI project to confirm the application starts correctly:

   ```bash
   dotnet run --project src/MerchStore.WebUI
   ```

3. Open a browser and navigate to:

    ```sh
    https://localhost:<port>
    ```

4. Verify the default ASP.NET Core template loads correctly.

✅ **Expected Results**

- Build succeeds without errors
- ASP.NET Core default page loads correctly in the browser
- The solution has the following structure inside the `src` directory:
  - MerchStore.Domain (Class Library)
  - MerchStore.Application (Class Library - references Domain)
  - MerchStore.Infrastructure (Class Library - references Application)
  - MerchStore.WebUI (ASP.NET Core MVC - references Infrastructure and Application)

## 🔧 Troubleshooting

If you encounter issues:

- Check that your .NET SDK version is 9.0 or newer with `dotnet --version`
- Ensure all project references are correctly set up
- Verify that the folder paths are created correctly
- Make sure the WebUI project runs with the default template

## 📚 Further Reading

- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) - The definitive guide
- [ASP.NET Core Clean Architecture](https://docs.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures) - Microsoft documentation
- [Clean Architecture with .NET Core](https://jasontaylor.dev/clean-architecture-getting-started/) - Jason Taylor's practical approach

## Done! 🎉

Great job! You've successfully set up the foundation for a Clean Architecture solution with proper organization using a `src` directory. This structure separates your source code from other solution artifacts, making it easier to add test projects, documentation, and other resources later. 🚀
