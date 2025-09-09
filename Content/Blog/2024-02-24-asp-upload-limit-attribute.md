---
title:  "Increasing the ASP.NET Core upload limit for a specific endpoint via attribute"
published:   2024-02-24
tags: [C#, ASP.NET Core, Programming]
---

By default, ASP.NET Core [limits the max request body size to 30 MB](https://github.com/aspnet/Announcements/issues/267). The limit can be increased by using a middleware, or global Kestrel configuration, as described in the linked issue. What happens if you don't want to globally increase the limit, but just for a single endpoint? The same configuration can be packaged into an attribute, and the size limit can even be made dependent of an `IConfiguration` value.

```csharp
using Microsoft.AspNetCore.Http.Features;
using Microsoft.AspNetCore.Mvc.Filters;

namespace IncreaseUploadLimitDemo;

[AttributeUsage(AttributeTargets.Method)]
public class UploadSizeLimitFilter(string appSettingsConfigName) : Attribute, IAuthorizationFilter
{
    public void OnAuthorization(AuthorizationFilterContext context)
    {
        var configuration = context.HttpContext.RequestServices.GetService<IConfiguration>();

        if (configuration?.GetValue<long?>(appSettingsConfigName) is not { } maxRequestBodySize) return;

        if (context.HttpContext.Features.Get<IHttpMaxRequestBodySizeFeature>() is { } maxRequestBodySizeFeature)
        {
            maxRequestBodySizeFeature.MaxRequestBodySize = maxRequestBodySize;
        }
    }
}
```

An `IAuthorizationFilter` is used because `IHttpMaxRequestBodySizeFeature.MaxRequestBodySize` can only be set as long as the request has not been read yet, which is the case for a regular `IActionFilter` for example.

Assuming the configuration value name `MaxRequestBodySize` has been set in `appsettings.json` similar to this:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "MaxRequestBodySize": 500000000
}

```

an endpoint can now be annotated with the previously implemented attribute:

```csharp
 [HttpGet(Name = "GetWeatherForecast")]
 [UploadSizeLimitFilter("MaxRequestBodySize")]
 public IEnumerable<WeatherForecast> Get()
 {
     return Enumerable.Range(1, 5).Select(index => new WeatherForecast
         {
             Date = DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
             TemperatureC = Random.Shared.Next(-20, 55),
             Summary = Summaries[Random.Shared.Next(Summaries.Length)]
         })
         .ToArray();
 }
 ```
