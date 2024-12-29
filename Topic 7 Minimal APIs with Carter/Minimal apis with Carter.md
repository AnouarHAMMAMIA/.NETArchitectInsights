### Minimal APIs with Carter

**Read time**: 5 minutes.
##

Greetings everyone! Today, we will define some endpoints with minimal boilerplate code using the **Carter library**.

Minimal APIs are great for building small, fast microservices or for prototyping new features quickly. They reduce the amount of boilerplate code you need to write, making your codebase cleaner and easier to maintain compared to Controller-based APIs in some cases.

Carter provides a simple and expressive way to define routes and handle HTTP requests. With Carter, you can focus on writing the core logic of your endpoints without worrying about the underlying infrastructure. It integrates seamlessly with ASP.NET Core and leverages its powerful features to build efficient and maintainable APIs.

Here is a step-by-step guide on how to use Carter with minimal APIs to create the Enroll member endpoint in our project.

First, we need to install the Carter NuGet package as follows:

```c#
dotnet add package Carter
```

Next, we need to add the following code to the `Program.cs` file to configure Carter and define our minimal API endpoints:

```c#
using Carter;
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddCarter();

var app = builder.Build();
app.MapCarter();

app.Run();
```

This code sets up a basic ASP.NET Core application, adds Carter services, and maps Carter routes. Now, let's define our Enroll member endpoint.

```c#
    public class Endpoint : ICarterModule
    {
        public void AddRoutes(IEndpointRouteBuilder app)
        {
            app.MapPost("api/enroll-members", async (Command command, ISender sender) =>
            {
                var result = await sender.Send(command);
                if (result.IsFailure)
                {
                    return Results.BadRequest(result.Error);
                }
                return Results.Ok(result.Value);
            });
        }
    }
```

1. **Class Definition**:
    ```csharp
    public class Endpoint : ICarterModule
    ```
    - This defines a class named `Endpoint` that implements the `ICarterModule` interface. Implementing this interface means the class must provide an implementation for the `AddRoutes` method.

2. **AddRoutes Method**:
    ```csharp
    public void AddRoutes(IEndpointRouteBuilder app)
    ```
    - This method is responsible for defining the routes for the API. It takes an `IEndpointRouteBuilder` object as a parameter, which is used to configure the routes.

3. **Mapping a POST Route**:
    ```csharp
    app.MapPost("api/enroll-members", async (Command command, ISender sender) =>
    ```
    - This line maps a POST request to the route `api/enroll-members`. The route handler is an asynchronous lambda function that takes a `Command` object and an `ISender` object as parameters.

4. **Handling the Request**:
    ```csharp
    var result = await sender.Send(command);
    if (result.IsFailure)
    {
        return Results.BadRequest(result.Error);
    }
    return Results.Ok(result.Value);
    ```
    - The handler sends the `command` to the `sender` and awaits the result.
    - If the result indicates failure (`result.IsFailure`), it returns a `BadRequest` response with the error message.
    - If the result is successful, it returns an `Ok` response with the result value.

 ### Summary

    In conclusion, using minimal APIs with Carter offers a streamlined alternative to traditional controllers by simplifying the development process. This approach enhances efficiency and maintainability, allowing developers to focus more on core logic and less on repetitive setup, especially for small applications or microservices.



Good luck out there, and until next time, keep learning something new!    
