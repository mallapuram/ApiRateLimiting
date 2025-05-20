# ApiRateLimiting 
To implement rate limiting, throttling, and rejected request handling for HTTP requests in the WebApi project instead o. Let me create a middleware solution for that purpose.

Here's an implementation for HTTP request rate limiting in the WebApi project:


```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.RateLimiting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using Serilog;
using System;
using System.Collections.Generic;
using System.Net;
using System.Text.Json;
using System.Threading;
using System.Threading.RateLimiting;
using System.Threading.Tasks;

namespace GemsService.WebApi.Infrastructure.Middleware
{
    /// <summary>
    /// Rate limiting options configuration class
    /// </summary>
    public class RateLimitOptions
    {
        public const string SectionName = "RateLimiting";
        
        // Global rate limiting (requests per window)
        public int GlobalLimit { get; set; } = 100;
        public int GlobalLimitPerWindow { get; set; } = 10;
        public int GlobalWindowSeconds { get; set; } = 60;
        
        // Per-client rate limiting
        public int ClientLimit { get; set; } = 20;
        public int ClientLimitPerWindow { get; set; } = 5;
        public int ClientWindowSeconds { get; set; } = 30;
        
        // Throttling settings
        public int MaxConcurrentRequests { get; set; } = 25;
        
        // Queue settings
        public int QueueLimit { get; set; } = 50;
    }
    
    /// <summary>
    /// Static extension methods for configuring rate limiting in the application
    /// </summary>
    public static class RateLimitingExtensions
    {
        /// <summary>
        /// Configures rate limiting for the application
        /// </summary>
        public static IServiceCollection AddGemsRateLimiting(this IServiceCollection services, RateLimitOptions options)
        {
            // Register rate limit options
            services.AddSingleton(Options.Create(options));
            
            // Add core rate limiting services
            services.AddRateLimiter(limiterOptions =>
            {
                // Add a named limiter for global rate limiting
                limiterOptions.AddTokenBucketLimiter("global", globalOptions =>
                {
                    globalOptions.TokenLimit = options.GlobalLimit;
                    globalOptions.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
                    globalOptions.QueueLimit = options.QueueLimit;
                    globalOptions.ReplenishmentPeriod = TimeSpan.FromSeconds(options.GlobalWindowSeconds);
                    globalOptions.TokensPerPeriod = options.GlobalLimitPerWindow;
                    globalOptions.AutoReplenishment = true;
                });
                
                // Add a named limiter based on client IP
                limiterOptions.AddConcurrencyLimiter("throttle", throttleOptions =>
                {
                    throttleOptions.PermitLimit = options.MaxConcurrentRequests;
                    throttleOptions.QueueLimit = options.QueueLimit;
                    throttleOptions.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
                });
                
                // Add a named limiter for per-client rate limiting
                limiterOptions.AddPolicy("clientPolicy", context =>
                {
                    // Get client identifier (IP address, API key, user ID, etc.)
                    string clientId = GetClientIdentifier(context);
                    
                    return RateLimitPartition.GetTokenBucketLimiter(clientId, key => new TokenBucketRateLimiterOptions
                    {
                        TokenLimit = options.ClientLimit,
                        QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                        QueueLimit = options.QueueLimit,
                        ReplenishmentPeriod = TimeSpan.FromSeconds(options.ClientWindowSeconds),
                        TokensPerPeriod = options.ClientLimitPerWindow,
                        AutoReplenishment = true
                    });
                });
                
                // Global configuration
                limiterOptions.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
                
                // Override the default rejection handler
                limiterOptions.OnRejected = async (context, token) =>
                {
                    // Create a detailed error response
                    var response = new
                    {
                        Status = (int)HttpStatusCode.TooManyRequests,
                        Title = "Too many requests",
                        Detail = "You have exceeded the allowed rate limit. Please try again later.",
                        RetryAfter = GetRetryAfterHeaderValue(context)
                    };
                    
                    // Log the rate limit rejection
                    var logger = context.HttpContext.RequestServices.GetRequiredService<ILogger<RateLimitOptions>>();
                    logger.LogWarning("Rate limit exceeded for client {ClientId}. Policy: {Policy}",
                        GetClientIdentifier(context.HttpContext), context.Lease.MetadataAs<string>() ?? "unknown");
                    
                    context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
                    context.HttpContext.Response.ContentType = "application/json";
                    
                    // Set retry-after header if available
                    if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
                    {
                        context.HttpContext.Response.Headers["Retry-After"] = retryAfter.ToString();
                    }
                    
                    await context.HttpContext.Response.WriteAsync(JsonSerializer.Serialize(response), token);
                };
            });
            
            return services;
        }
        
        /// <summary>
        /// Gets a client identifier from the request
        /// </summary>
        private static string GetClientIdentifier(HttpContext context)
        {
            // Get client IP address
            string? clientIp = context.Connection.RemoteIpAddress?.ToString();
            
            // API key from header (if available)
            string? apiKey = context.Request.Headers["X-API-Key"].ToString();
            
            // User ID from claims (if authenticated)
            string? userId = context.User?.Identity?.Name;
            
            // Use whichever identifier is available, with priority
            return userId ?? apiKey ?? clientIp ?? "unknown";
        }
        
        /// <summary>
        /// Gets the value for the Retry-After header
        /// </summary>
        private static int GetRetryAfterHeaderValue(RateLimitLease lease)
        {
            if (lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
            {
                return (int)retryAfter.TotalSeconds;
            }
            
            return 60; // Default retry time
        }
    }
    
    /// <summary>
    /// Optional middleware for advanced rate limiting scenarios
    /// </summary>
    public class GemsRateLimitingMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly ILogger<GemsRateLimitingMiddleware> _logger;
        private readonly RateLimitOptions _options;
        
        public GemsRateLimitingMiddleware(
            RequestDelegate next,
            ILogger<GemsRateLimitingMiddleware> logger,
            IOptions<RateLimitOptions> options)
        {
            _next = next;
            _logger = logger;
            _options = options.Value;
        }
        
        public async Task InvokeAsync(HttpContext context)
        {
            // Custom pre-processing logic (if needed)
            
            // Continue to the next middleware
            await _next(context);
            
            // Custom post-processing logic (if needed)
        }
    }
}

```

Now, to integrate this into the application, you'll need to update the `Program.cs` or `Startup.cs` file in the WebApi project. Here's how you would do that:


```csharp
using GemsService.WebApi.Infrastructure.Middleware;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.RateLimiting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

// In Program.cs or Startup.ConfigureServices
public void ConfigureServices(IServiceCollection services)
{
    // Get rate limit options from configuration
    var rateLimitOptions = Configuration.GetSection(RateLimitOptions.SectionName).Get<RateLimitOptions>() ?? new RateLimitOptions();
    
    // Add rate limiting services
    services.AddGemsRateLimiting(rateLimitOptions);
    
    // Other service registrations...
}

// In Program.cs or Startup.Configure
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // Other middleware...
    
    // Apply rate limiting middleware to all routes
    app.UseRateLimiter();
    
    // Optional: Add custom rate limiting middleware for advanced scenarios
    // app.UseMiddleware<GemsRateLimitingMiddleware>();
    
    // Other middleware...
}

```

For this to work, you'll also need to add configuration settings to your `appsettings.json`:


```json
{
  "RateLimiting": {
    "GlobalLimit": 100,
    "GlobalLimitPerWindow": 10,
    "GlobalWindowSeconds": 60,
    "ClientLimit": 20,
    "ClientLimitPerWindow": 5,
    "ClientWindowSeconds": 30,
    "MaxConcurrentRequests": 25,
    "QueueLimit": 50
  }
}

```

## How to Apply Rate Limiting to Specific Endpoints

For more granular control, you can apply rate limiting to specific controllers or endpoints using attributes:


```csharp
[EnableRateLimiting("global")]
public class HighVolumeController : ControllerBase
{
    [EnableRateLimiting("clientPolicy")]
    public IActionResult Get()
    {
        // This endpoint is protected by both global and client-specific rate limits
        return Ok();
    }
    
    [DisableRateLimiting]
    public IActionResult Priority()
    {
        // This endpoint bypasses rate limiting
        return Ok();
    }
}

```

## Features of this Implementation

1. **Multiple Rate Limiting Strategies**:
   - Global rate limiting for the entire API
   - Per-client rate limiting based on IP, API key, or user ID
   - Concurrency throttling to limit simultaneous requests

2. **Request Queuing**:
   - Requests exceeding limits are queued (up to a configurable limit)
   - Oldest-first processing ensures fairness

3. **Rejected Request Handling**:
   - Detailed error responses with proper HTTP status codes
   - Retry-After headers to inform clients when to retry
   - Comprehensive logging of rejections

4. **Flexibility**:
   - Configuration-driven for easy adjustment
   - Support for different client identification methods
   - Ability to disable rate limiting for specific endpoints

5. **Scalability**:
   - Built on .NET's official rate limiting APIs
   - Efficient token bucket algorithm
   - Minimal overhead for request processing

This implementation provides a comprehensive solution for handling high traffic while maintaining service stability and responsiveness. It gracefully handles requests when capacity limits are reached, providing clear feedback to clients and maintaining system integrity.
