---
title: Get Started
description: A light-weight REST Api framework for ASP.Net 6 that implements REPR (Request-Endpoint-Response) Pattern.
---

# {$frontmatter.title}

Follow the steps below to create your first endpoint that will handle an http post request and send a response back to the client.

## Create a new project

```sh |copy
  dotnet new web -n MyWebApp
```

## Install Nuget Package

Install the latest library version using the following cli command:

```sh |copy
  dotnet add package FastEndpoints
```

or with nuget package manager:

```sh |copy
  Install-Package FastEndpoints
```

## Prepare Setup

Replace the contents of **Program.cs** file with the following:

```cs |copy|title=Program.cs
global using FastEndpoints;

var builder = WebApplication.CreateBuilder();
builder.Services.AddFastEndpoints();

var app = builder.Build();
app.UseAuthorization();
app.UseFastEndpoints();
app.Run();
```

## Add a Request DTO

Create a file called **MyRequest.cs** and add the following:

```cs |copy|title=MyRequest.cs
public class MyRequest
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public int Age { get; set; }
}
```

## Add a Response DTO

Create a file called **MyResponse.cs** and add the following:

```cs |copy|title=MyResponse.cs
public class MyResponse
{
    public string FullName { get; set; }
    public bool IsOver18 { get; set; }
}
```

## Add an endpoint definition

create a file called **MyEndpoint.cs** and add the following:

```cs |copy|title=MyEndpoint.cs
public class MyEndpoint : Endpoint<MyRequest>
{
    public override void Configure()
    {
        Verbs(Http.POST);
        Routes("/api/user/create");
        AllowAnonymous();
    }

    public override async Task HandleAsync(MyRequest req, CancellationToken ct)
    {
        var response = new MyResponse()
        {
            FullName = req.FirstName + " " + req.LastName,
            IsOver18 = req.Age > 18
        };

        await SendAsync(response);
    }
}
```

Now run your web app and send a POST request to the **/api/user/create** endpoint using a REST client such as postman with the following request body:

```json
{
	"FirstName": "marlon",
	"LastName": "brando",
	"Age": 40
}
```

You should then get a response back such as this:

```json
{
	"FullName": "marlon brando",
	"IsOver18": true
}
```

That's all there's to it. you simply configure how the endpoint should be listening to incoming requests from clients in the **Configure()** section calling methods such as **Verbs()**, **Routes()**, **AllowAnonymous()**, etc. then you override the HandleAsync() method in order to specify your handling logic. the request dto is automatically populated from the json body of your http request and passed in to the handler. when you're done processing, you call the **SendAsync()** method with a new response dto to be sent to the requesting client.

## Endpoint types

There are 4 different endpoint base types you can inherit from.

1. **Endpoint&lt;TRequest&gt;** - use this type if there's only a request dto. you can however send any object to the client that can be serialized as a response with this generic overload.
2. **Endpoint&lt;TRequest,TResponse&gt;** - use this type if you have both request and response dtos. the benefit of this generic overload is that you get strongly-typed access to properties of the dto when doing integration testing and validations.
3. **EndpointWithoutRequest** - use this type if there's no request nor response dto. you can send any serializable object as a response here also.
4. **EndpointWithoutRequest&lt;TResponse&gt;** - use this type if there's no request dto but there is a response dto.

It is also possible to define endpoints with **EmptyRequest** and **EmptyResponse** if needed like so:

```cs
public class MyEndpoint : Endpoint<EmptyRequest,EmptyResponse> { }
```

## Sending responses

There are multiple response sending methods you can use. it is also possible to simply populate the Response property of the endpoint and get a 200 ok response with the value of the Response property serialized in the body automatically. for ex:

**Response DTO**:

```cs
public class MyResponse
{
    public string FullName { get; set; }
    public int Age { get; set; }
}
```

**Endpoint definition**:

```cs
public class MyEndpoint : EndpointWithoutRequest<MyResponse>
{
    public override void Configure()
    {
        Get("/api/person");
        AllowAnonymous();
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        var person = await dbContext.GetFirstPersonAsync();

        Response.FullName = person.FullName;
        Response.Age = person.Age;
    }
}
```

Assigning a new instance to the Response property also has the same effect:

```cs
public override Task HandleAsync(CancellationToken ct)
{
    Response = new()
    {
        FullName = "john doe",
        Age = 124
    };
    return Task.CompletedTask;
}
```

## Configuring endpoints using attributes

Instead of overriding the Configure() method, endpoint classes can be annotated with attributes:

- **[HttpGet(...)]**
- **[AllowAnonymous]**
- **[Authorize(...)]**

Advanced usage however does require overriding **Configure()**. You can only use one of these strategies for configuring endpoints.

:::admonition type="warning"
An exception will be thrown if you use both or none at all.
:::

```cs
[HttpPost("/my-endpoint")]
[Authorize(Roles = "Admin,Manager")]
public class UpdateAddress : Endpoint<MyRequest, MyResponse>
{
    public override async Task HandleAsync(MyRequest req, CancellationToken ct)
    {
        await SendAsync(new MyResponse { });
    }
}
```

## Cancellation token

The HandleAsync method of the endpoint is supplied a CancellationToken which you can pass down to your own async methods within the handler that requires a token.

The Send\*Async methods of the endpoint also optionally accepts a CancellationToken. i.e. you can either pass down the same token supplied to the HandleAsync method or you may create/use a different token with these response sending methods depending on your requirement.

However, do note that it is not required to supply a CancellationToken to the Send\*Async methods, and there's no real need to dirty up your code like the following:

```cs
  await SendAsync(response, cancellation: ct);
```

Because if you do not supply the token to the Send\*Async methods, the library automatically supplies the same token that is supplied to the HandleAsync method internally, and your code can remain cleaner.

The analyzer hint/warning can be turned off by adding the following to your csproj file:

```xml
  <NoWarn>CA2016</NoWarn>
```