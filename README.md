# .NET Core - Request Limit Control

Reports are very common in any system and some of them can be very heavy to process. Therefore, in this case, the application may be degraded during the generation of the report, affecting other users. But, how can we avoid this? The first thought would be to block the bottom, right? Um, but it is so easy to hack, the user can just edit the html props and enable it again. Therefore, the best option is to block it on the server side, using some request limit control strategy.


## Solution - AspNetCoreRateLimit

`AspNetCoreRateLimit` is a library that can help us with requisition control. You can define rules to decide how many times a customer can call the resource over a period of time.


But how does it work?

AspNetCoreRateLimit uses the MemoryCache solution to save information about customer requests. For example, a customer can only make 10 requests in a 5-second interval for a specific endpoint. Thus, each request will be saved in memory cache and if the client exceeds this limit, the application will stop the request and return an http error status.


## Implementation

First of all, we need to install the library AspNetCoreRateLimit


### Configuring


The best option to configure AspNetCoreRateLimit is to define all the information within appsettings.json. So, we will create a block like this:

```
"IpRateLimiting": {
    "EnableEndpointRateLimiting": true,
    "StackBlockedRequests": false,
    "RealIpHeader": "X-Real-IP",
    "ClientIdHeader": "X-ClientId",
    "HttpStatusCode": 429, 
    "GeneralRules": [
      {
        "Endpoint": "*/api/test", 
        "Period": "2s", //interval
        "Limit": 2 //limit of request in the interval
      }
    ]
  },
```

With our rules configurated, we need to add few lines in `Startup.cs`:

```
 public void ConfigureServices(IServiceCollection services)
 {
      services.AddOptions();
      //AspNetCoreRateLimit uses MemoryCache to control the numbers of request
      services.AddMemoryCache();

       //Adding AspNetCoreRateLimit rules
      services.Configure<IpRateLimitOptions>(Configuration.GetSection("IpRateLimiting"));

      //Adding the store
      services.AddSingleton<IIpPolicyStore, MemoryCacheIpPolicyStore>();

      //Adding the request counter
      services.AddSingleton<IRateLimitCounterStore, MemoryCacheRateLimitCounterStore>();
      services.AddSingleton<IRateLimitConfiguration, RateLimitConfiguration>();
      services.AddHttpContextAccessor();
      services.AddControllers();
      services.AddSingleton<IRateLimitConfiguration, RateLimitConfiguration>();
      services.AddSwaggerGen(c =>
      {
          c.SwaggerDoc("v1", new OpenApiInfo { Title = "RequestLimit", Version = "v1" });
      });
}

```
        
And the final step is to activate the service:

```
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
        app.UseSwagger();
        app.UseSwaggerUI(c => c.SwaggerEndpoint("/swagger/v1/swagger.json", "RequestLimit v1"));
    }

    app.UseHttpsRedirection();

    app.UseRouting();
    app.UseIpRateLimiting() //Adding this block;
    app.UseAuthorization();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```
And it's done, now our api has a request control. You can customize your configuration to follow specific rules, all options are listed here:
https://github.com/stefanprodan/AspNetCoreRateLimit
