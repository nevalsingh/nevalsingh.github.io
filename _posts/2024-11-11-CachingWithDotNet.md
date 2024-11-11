---
layout: post
title: "Setting Up External Caching in .Net 8 with Dependency Injection"
date: 2024-11-11
tags: C# .Net8 Redis Valkey Caching
author: Nev Singh
---
In this post, we'll explore a code snippet that configures Redis or Valkey caching for an .Net 8 application using the dependency injection (DI) container. This setup uses the `StackExchange.Redis.Extensions` library to streamline Redis integration, allowing our application to handle cache connections and configuration effortlessly. Let's walk through the code and see how each part works.

## Code Example
### Cache DI Configuration
```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

using StackExchange.Redis.Extensions.Core;
using StackExchange.Redis.Extensions.Core.Abstractions;
using StackExchange.Redis.Extensions.Core.Configuration;
using StackExchange.Redis.Extensions.Core.Implementations;
using StackExchange.Redis.Extensions.Newtonsoft;

namespace Tools.Core.Configuration;

/// <summary>
/// Caching Configuration
/// </summary>
public static class CacheConfiguration
{
    /// <summary>
    /// Configures and registers Redis caching services in the dependency injection container.
    /// Sets up Redis connection settings, including host, port, password, and various connection parameters.
    /// </summary>
    /// <param name="services"></param>
    /// <param name="config"></param>
    /// <param name="environment"></param>
    /// <exception cref="NotImplementedException"></exception>
    public static void AddCacheConfiguration(this IServiceCollection services, IConfiguration config,
        IWebHostEnvironment environment)
    {
        CacheOptions cacheOptions = config.GetSection(CacheOptions.Section).Get<CacheOptions>() 
            ?? throw new NotImplementedException("Configuration for Cache Not Found");

        RedisConfiguration redisConfiguration = new RedisConfiguration
        {
            AbortOnConnectFail = true,
            Hosts = new RedisHost[]
            {
                new RedisHost
                {
                    Host = cacheOptions.Host,
                    Port = cacheOptions.Port,
                },
            },
            AllowAdmin = true,
            Password = cacheOptions.Password ?? "",
            User = cacheOptions.Username ?? "",
            ServerEnumerationStrategy = new ServerEnumerationStrategy
            {
                Mode = ServerEnumerationStrategy.ModeOptions.All,
                TargetRole = ServerEnumerationStrategy.TargetRoleOptions.Any,
                UnreachableServerAction = ServerEnumerationStrategy.UnreachableServerActionOptions.Throw,
            },
            Ssl = false // You can conditionally set this based on your environments
        };

        services.AddSingleton(redisConfiguration);
        services.AddSingleton<IRedisClient, RedisClient>();
        services.AddSingleton<IRedisConnectionPoolManager, RedisConnectionPoolManager>();
        services.AddSingleton<ISerializer, NewtonsoftSerializer>();

        services.AddSingleton(provider =>
        {
            var redisClient = provider.GetRequiredService<IRedisClient>();
            var db = int.TryParse(cacheOptions.DbKey, out var dbKey) ? dbKey : 0;
            return redisClient.GetDb(db);
        });
    }
}
```

### CacheOptions Class for Redis/Valkey Configuration

The `CacheOptions` class is a configuration model that defines settings for connecting to a Redis or Valkey cache. This class includes properties for host, port, authentication credentials, and the target database key. Using data annotations, certain properties are validated to ensure required values are provided.

```csharp
using System.ComponentModel.DataAnnotations;

namespace Tools.Core.Model.Options;

/// <summary>
/// Config Options for Redis/Valkey Cache
/// </summary>
public class CacheOptions
{
    public const string Section = "Cache";

    [Required]
    public required string Host { get; init; }

    public string? Username { get; init; }

    public int Port { get; init; } = 6379;

    public string? Password { get; init; }

    public string? DbKey { get; init; }
}
```
## Explanation
### Overview
This CacheConfiguration class is a static helper class for configuring Redis caching services. It defines the AddCacheConfiguration method, which sets up the necessary Redis services in the .Net 8 dependency injection container. The code utilizes the StackExchange.Redis.Extensions library to manage the Redis connection, configuration, and serialization.

### Key Components

#### CacheOptions:

The CacheOptions object is fetched from the application’s configuration (usually from appsettings.json). It includes settings such as the Redis host, port, username, password, and database key.
If CacheOptions is not found, a Exception is thrown.

```
{
  "Cache": {
    "Host": "localhost",
    "Port": 6379,
    "Username": "myusername",
    "Password": "mypassword",
    "DbKey": "0"
  }
}
```

#### RedisConfiguration:

This object defines how the application connects to Redis, including:
Hosts: Specifies the Redis server’s host and port.
Password and User: Provides optional authentication details if Redis requires credentials.
Server Enumeration Strategy: Defines how the client should discover Redis servers and handle unreachable servers.
SSL Setting: SSL is enabled if the environment is not local, enhancing security for production deployments.

#### Service Registrations:

The code registers several key Redis services into the DI container:
IRedisClient: The primary interface for interacting with Redis.
IRedisConnectionPoolManager: Manages Redis connections for performance and reliability.
ISerializer: Uses NewtonsoftSerializer to handle JSON serialization for Redis operations.

#### Redis Database Selection:

The code configures which Redis database to use by retrieving the database key from CacheOptions. If DbKey isn’t set, it defaults to database 0.
It then retrieves the specified database using redisClient.GetDb(db), allowing the application to target a particular Redis database for caching.

### Usage
To apply this caching configuration in your .Net 8 application, call AddCacheConfiguration during service configuration, passing in the IServiceCollection, IConfiguration, and IWebHostEnvironment instances.

```csharp
// In Program.cs
builder.Services.AddCacheConfiguration(builder.Configuration, builder.Environment);
```
```csharp
// In YourImpl.cs
private readonly IRedisDatabase _cache;
    
public OfferController(IRedisDatabase cache)
{
  _cache = cache;
}

public async Task InsertCache ()
{
  await _cache.AddAsync("Foo", "Bar");
}
```

## Conclusion
This configuration class provides a clean and organized way to manage Redis caching in an .Net 8 application. By centralizing the configuration in CacheConfiguration, the application ensures consistent, reliable Redis connectivity with SSL support for production, pooling for performance, and JSON serialization for easy data handling.
