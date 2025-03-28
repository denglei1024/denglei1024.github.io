---
title: "在 asp.net core 项目中动态获取运行时的服务地址"
---

> 版本信息：.net9

我在使用 asp.net core web api 搭建项目实践 consul 的服务发现功能，想动态获取当前运行实例的服务地址，查阅了相关资料发现可以使用`ServerFeatures`来获取。

关键代码是：

```csharp

// 获得服务地址  
var features = app.ServerFeatures;  
var addresses = features.GetRequiredFeature<IServerAddressesFeature>();  
var address = addresses.Addresses.First();

```

此时，需要提醒你，由于要获取的是运行的服务地址，因此，要等服务启动后执行代码才可以。

建议可以把这段代码放在`ApplicationStarted`事件里执行。


```csharp
// 获取生命周期管理  
var lifetime = app.ApplicationServices.GetRequiredService<IHostApplicationLifetime>();  
lifetime.ApplicationStarted.Register(() =>  
{  
    // 获得服务地址  
    var features = app.ServerFeatures;  
    var addresses = features.GetRequiredFeature<IServerAddressesFeature>();  
    var address = addresses.Addresses.First();  
});

```
完整的示例代码如下：

```csharp

public static void UseConsul(this IApplicationBuilder app, IConfiguration configuration)
{
    // 获取生命周期管理
    var lifetime = app.ApplicationServices.GetRequiredService<IHostApplicationLifetime>();
    lifetime.ApplicationStarted.Register(() =>
    {
        var consulClient = app.ApplicationServices
            .GetRequiredService<IConsulClient>();
        // 获得服务地址
        var features = app.ServerFeatures;
        var addresses = features.GetRequiredFeature<IServerAddressesFeature>();
        var address = addresses.Addresses.First();
        var serviceIdProvider = app.ApplicationServices.GetRequiredService<ServiceIdProvider>();
        // 注册服务
        var uri = new Uri(address);
        var registration = new AgentServiceRegistration()
        {
            ID = serviceIdProvider.ServiceId,
            Address = $"{uri.Scheme}:{uri.Host}",
            Name = configuration["Consul:ServiceName"],
            Port = uri.Port,
            Tags = ["order"],
            Checks =
            [
                new AgentServiceCheck()
                {
                    CheckID = serviceIdProvider.CheckId,
                    TTL = TimeSpan.FromSeconds(10),
                    DeregisterCriticalServiceAfter = TimeSpan.FromSeconds(60)
                }
            ]
        };
        consulClient.Agent.ServiceDeregister(registration.ID).Wait();
        consulClient.Agent.ServiceRegister(registration).Wait();
    });
}

```