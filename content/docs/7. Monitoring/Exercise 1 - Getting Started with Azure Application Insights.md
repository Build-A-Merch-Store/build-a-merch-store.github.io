---
title: "Getting Started with Azure Application Insights"
date: "2025-05-12"
lastmod: "2025-05-12"
draft: false
weight: 701
toc: true
---

## 🎯 Goal

Set up Azure Application Insights for your MerchStore application to establish a foundation for application performance monitoring (APM), ensuring proper configuration for both development and production environments.

## 📋 Prerequisites

Before beginning this exercise, you should:

- Have a functioning MerchStore ASP.NET Core application
- Have an Azure subscription
- Be familiar with basic ASP.NET Core concepts
- Have .NET SDK installed (.NET 6 or later)

## 📚 Learning Objectives

By the end of this exercise, you will:

- Create an **Application Insights resource** in Azure
- Configure **Application Insights SDK** with different settings for development and production
- Understand the concept of **telemetry channels** and their impact on data flow
- Verify that **telemetry data** is successfully reaching Azure
- Learn the difference between **development** and **production** monitoring configurations

## 🔍 Why This Matters

In real-world applications, proper monitoring setup is crucial because:

- It provides visibility into application behavior without affecting performance
- Different environments require different monitoring configurations
- Early detection of issues saves time and prevents customer impact
- Good monitoring practices are essential for production-grade applications

## 📝 Step-by-Step Instructions

### Step 1: Create an Application Insights Resource in Azure

1. Navigate to the [Azure Portal](https://portal.azure.com) and sign in with your account.

2. Click on **+ Create a resource** in the left navigation menu.

3. Search for "Application Insights" and select it from the results.

4. Click **Create**.

5. Fill in the required information:
   - **Subscription**: Choose your Azure subscription
   - **Resource Group**: Create a new one (e.g., `MerchStore-Monitoring-RG`)
   - **Name**: Give your resource a unique name (e.g., `merchstore-appinsights-[yourname]`)
   - **Region**: Choose a region close to your location
   - **Resource Mode**: Select "Workspace-based"
   - **Log Analytics Workspace**: Create a new one or select an existing one

6. Click **Review + create**, then **Create**.

7. Once the deployment is complete, navigate to your new Application Insights resource and locate the **Connection String** on the Overview page. Copy this string for use in the next steps.

> 💡 **Information**
>
> - **Application Insights** is part of Azure Monitor and provides APM capabilities
> - **Workspace-based** mode is the modern approach that provides better querying capabilities
> - **Connection String** contains all information needed for your app to send telemetry to Azure
> - The resource takes a few minutes to fully deploy and become available
>
> ⚠️ **Common Mistakes**
>
> - Using the old Instrumentation Key instead of the Connection String (deprecated)
> - Choosing classic mode instead of workspace-based mode
> - Creating the resource in a region far from your application (increases latency)

### Step 2: Add Application Insights SDK to Your MerchStore Project

1. Open your MerchStore solution in your preferred IDE.

2. Add the Application Insights SDK NuGet package to your project using the command line:

   ```bash
   dotnet add package Microsoft.ApplicationInsights.AspNetCore
   ```

3. Verify that the package has been added to your project file:

   > `MerchStore.csproj`

   ```xml
   <ItemGroup>
     <!-- Other package references -->
     <PackageReference Include="Microsoft.ApplicationInsights.AspNetCore" Version="2.22.0" />
   </ItemGroup>
   ```

> 💡 **Information**
>
> - The SDK includes automatic collectors for common telemetry types (requests, dependencies, exceptions)
> - The package automatically integrates with ASP.NET Core's request pipeline
> - Version numbers may vary - always use the latest stable version
>
> ⚠️ **Common Mistakes**
>
> - Adding only the core Application Insights package without the ASP.NET Core integration
> - Using preview versions in production environments

### Step 3: Configure Connection String in appsettings.json

1. Open `appsettings.json` and add your Application Insights connection string:

   > `appsettings.json`

   ```json
   {
      "ConnectionStrings": {
         "DefaultConnection": "Data Source=merchstore.db"
      },
      "ApplicationInsights": {
         "ConnectionString": "YOUR_CONNECTION_STRING_HERE"
      },
      "Logging": {
         "LogLevel": {
            "Default": "Information",
            "Microsoft.AspNetCore": "Warning"
         },
         "ApplicationInsights": {
            "LogLevel": {
               "Default": "Information"
            }
         }
      },
     "AllowedHosts": "*"
   }
   ```

   Replace `YOUR_CONNECTION_STRING_HERE` with the connection string you copied from the Azure Portal.

> 💡 **Information**
>
> - Configuration files use a hierarchy - Development settings override base settings
> - Different log levels help control the volume of telemetry in different environments
> - The connection string works for all environments - it's the SDK configuration that differs
>
> ⚠️ **Common Mistakes**
>
> - Putting sensitive information like connection strings in source control
> - Not using environment-specific configuration files

### Step 4: Configure Application Insights with Environment-Specific Settings

Now, let's configure Application Insights differently for development and production environments.

1. Open `Program.cs` and add the necessary using statements:

   > `Program.cs`

   ```csharp
   using Microsoft.ApplicationInsights.Channel;
   using Microsoft.ApplicationInsights.Extensibility;
   ```

2. Configure Application Insights with environment-specific settings:

   > `Program.cs`

   ```csharp
   var builder = WebApplication.CreateBuilder(args);

   ...

   // Configure Application Insights based on environment
   if (!builder.Environment.IsDevelopment())
   {
      // Production configuration with performance optimization
      builder.Services.AddApplicationInsightsTelemetry();
   }
   else
   {
      // Development-specific configuration for immediate telemetry
      builder.Services.AddApplicationInsightsTelemetry(options =>
      {
         // Use InMemoryChannel for immediate telemetry transmission in development
         options.EnableAdaptiveSampling = false;
         options.ApplicationVersion = "dev-" + DateTime.Now.ToString("yyyyMMdd-HHmm");
         options.EnableDebugLogger = true;
      });
      
      // Configure telemetry channel separately
      builder.Services.Configure<TelemetryConfiguration>(config =>
      {
         config.TelemetryChannel = new InMemoryChannel
         {
               DeveloperMode = true // Sends telemetry immediately without batching
         };
      });
   }

   ...

   var app = builder.Build();
   ```

> 💡 **Information**
>
> - **InMemoryChannel**: Sends telemetry immediately, perfect for development debugging
> - **DeveloperMode**: Bypasses normal batching and sends data in real-time
> - **Adaptive Sampling**: Automatically reduces telemetry volume in high-traffic scenarios
> - **ApplicationVersion**: Helps filter and identify telemetry from different deployments
>
> ⚠️ **Common Mistakes**
>
> - Using developer mode in production (causes performance issues)
> - Disabling sampling in production (can lead to high costs)
> - Not setting ApplicationVersion (makes it hard to correlate issues with deployments)

## 🧪 Final Tests

Now let's run the application and verify that telemetry is reaching Azure.

1. Run your MerchStore application:

   ```bash
   dotnet run
   ```

   Note the URL where your application is running (typically `https://localhost:5001` or similar).

2. Generate some telemetry by visiting various pages:
   - Navigate to the home page
   - Go to the privacy page
   - Try accessing a non-existent page to generate a 404 error

3. Check telemetry in the Azure Portal:

   a. Navigate to your Application Insights resource in the Azure Portal
   b. Look at the **Overview** page - you should see initial charts after a few minutes
   c. Click on **Live Metrics** in the left menu - this shows real-time telemetry
   d. Click on **Transaction search** in the left menu to see individual requests
   e. Use the **Logs** section to run a simple query:

      ```kusto
      requests
      | order by timestamp desc
      | take 10
      ```

4. Verify different telemetry types:

   a. **Requests**: Should see entries for each page visit
   b. **Dependencies**: May see database calls if Entity Framework is configured
   c. **Exceptions**: Should see entries if you triggered any errors

> 💡 **Information**
>
> - **Live Metrics** shows data with minimal delay (1-2 seconds)
> - **Transaction search** and **Logs** may have 2-5 minute delay
> - Initial data may take a few minutes to appear after first deployment
> - Development mode sends data immediately, so you should see it quickly
>
> ⚠️ **Common Mistakes**
>
> - Not waiting long enough for data to appear (give it 5 minutes initially)
> - Looking in the wrong time range (check "Last hour" or "Last 30 minutes")
> - Having browser ad blockers that might interfere with Application Insights

✅ **Expected Results**

- Application Insights Overview page shows incoming telemetry
- Live Metrics displays real-time data when running in Development mode
- Transaction search shows individual requests with proper URLs
- No errors in the application console related to Application Insights
- Different behavior between Development and Production modes

## 🔧 Troubleshooting

If you encounter issues:

- **No data appearing in Application Insights**:
  - Verify the connection string is correct
  - Check for any errors in the application console
  - Ensure firewall/proxy allows outbound HTTPS traffic to Azure
  - Try using InMemoryChannel even in production temporarily to rule out batching issues

- **Only some telemetry types appearing**:
  - Verify that automatic collectors are enabled
  - Check log levels in configuration
  - Ensure the SDK is properly initialized before other services

- **Performance issues in development**:
  - Verify you're not sending too much debug telemetry
  - Consider using sampling even in development for high-traffic scenarios

- **Configuration not switching between environments**:
  - Check that ASPNETCORE_ENVIRONMENT is set correctly
  - Verify the environment-specific code is being executed (add debug output)

## 🚀 Optional Challenge

Want to take your learning further? Try:

- Add custom telemetry properties to all requests using a TelemetryInitializer
- Create environment-specific Application Insights resources (dev/staging/prod)
- Configure Application Insights using Azure Key Vault for secure connection string storage
- Implement feature flags to control telemetry collection dynamically

## 📚 Further Reading

- [Application Insights for ASP.NET Core Applications](https://docs.microsoft.com/en-us/azure/azure-monitor/app/asp-net-core)
- [Application Insights Channels](https://docs.microsoft.com/en-us/azure/azure-monitor/app/telemetry-channels)
- [Sampling in Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/sampling)
- [Performance Counters in Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/performance-counters)

## Done! 🎉

Great job! You've successfully set up Azure Application Insights for your MerchStore application with proper configuration for both development and production environments. You now have a solid foundation for application monitoring that sends telemetry efficiently based on your environment needs. This setup ensures you can debug issues quickly in development while maintaining performance and controlling costs in production. 🚀

## Appendix: Understanding Telemetry Channels

### What is a Telemetry Channel?

A telemetry channel in Application Insights is responsible for buffering and transmitting telemetry data from your application to Azure. Different channels have different characteristics:

1. **ServerTelemetryChannel** (Default for Production)
   - Batches telemetry items before sending
   - Retries on failures with exponential backoff
   - Persists data to disk if transmission fails
   - Optimized for reliability and performance

2. **InMemoryChannel** (Recommended for Development)
   - Minimal buffering - sends data quickly
   - No persistence - data lost if transmission fails
   - Perfect for immediate feedback during development
   - Can impact performance if used in high-traffic production scenarios

### Developer Mode Explained

Developer Mode is a setting that affects how quickly telemetry is sent:

- **When Enabled**: Telemetry is sent immediately or with minimal delay
- **When Disabled**: Normal batching and transmission rules apply
- **Best Practice**: Only enable in development environments

### Why Different Configurations Matter

1. **Development Environment**:
   - Need immediate feedback for debugging
   - Performance is less critical
   - Want to see all telemetry without sampling

2. **Production Environment**:
   - Performance is critical
   - Cost control through sampling is important
   - Reliability requires proper buffering and retry logic

This configuration approach ensures your monitoring system adapts to the needs of each environment while maintaining the same codebase.
