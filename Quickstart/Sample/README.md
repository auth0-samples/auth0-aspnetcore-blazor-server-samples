# Quickstart Sample

This example shows how to add login/logout and extract user profile information from claims.

You can read a quickstart for this sample [here](https://auth0.com/docs/quickstart/webapp/aspnet-core-blazor).

## Requirements

- [.NET SDK](https://dotnet.microsoft.com/download) (.NET 6.0+)

## To run this project

1. Ensure that you have replaced the `appsettings.json` file with the values for your Auth0 account.

2. Run the application from the command line:

```bash
dotnet run
```

3. Go to `http://localhost:3000` (or `https://localhost:7113`) in your web browser to view the website.

## Important Snippets

### 1. Register the Auth0 SDK

```csharp
builder.Services.AddAuth0WebAppAuthentication(options => {
    options.Domain = Configuration["Auth0:Domain"];
    options.ClientId = Configuration["Auth0:ClientId"];
    options.Scope = "openid profile email";
});

```

### 2. Register the Authentication middleware

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

### 3. Login

```csharp
public class LoginModel : PageModel
{
    public async Task OnGet(string redirectUri)
    {
        var authenticationProperties = new LoginAuthenticationPropertiesBuilder()
            .WithRedirectUri(redirectUri)
            .Build();

        await HttpContext.ChallengeAsync(Auth0Constants.AuthenticationScheme, authenticationProperties);
    }
}

```

### 4. User Profile

```csharp
@code {
    [CascadingParameter]
    public Task<AuthenticationState> AuthenticationStateTask { get; set; }
    private string Username = "";
    private string EmailAddress = "";
    private string Picture = "";

    protected override async Task OnInitializedAsync()
    {
        var state = await AuthenticationStateTask;

        Username = state.User.Identity.Name ?? string.Empty;

        EmailAddress = state.User.Claims
            .Where(c => c.Type.Equals(System.Security.Claims.ClaimTypes.Email))
            .Select(c => c.Value)
            .FirstOrDefault() ?? string.Empty;

        Picture = state.User.Claims
            .Where(c => c.Type.Equals("picture"))
            .Select(c => c.Value)
            .FirstOrDefault() ?? string.Empty;

        await base.OnInitializedAsync();
    }
}
```

### 5. Logout

```csharp
[Authorize]
public class LogoutModel : PageModel
{
    public async Task OnGet()
    {
        var authenticationProperties = new LogoutAuthenticationPropertiesBuilder()
                .WithRedirectUri("/")
                .Build();

        await HttpContext.SignOutAsync(Auth0Constants.AuthenticationScheme, authenticationProperties);
        await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
    }
}
```

### 6. Accessing tokens in Blazor components

In order to be able to read the tokens in a blazor component, you will need to find a way to retrieve them from the `HttpContext` without using `HttpContext` directly in the blazor components.

In order to be able to do so, we will need to create two helper classes:

```csharp
public class InitialApplicationState
{
    public string? IdToken { get; set; }
    public string? AccessToken { get; set; }
    public string? RefreshToken { get; set; }
}

public class TokenProvider
{
    public string? IdToken { get; set; }
    public string? AccessToken { get; set; }
    public string? RefreshToken { get; set; }
}
```

- `InitialApplicationState`` will be used to inject the tokens into the applicaiton from the HttpContext
- `TokenProvider` will then be populated and provided to be able to inject in razor compponents without any relation to the `HttpContext`.

Be sure to register the `TokenProvider` in `Program.cs` so it can be injected in the rest of the application:

```csharp
builder.Services.AddScoped<TokenProvider>();
```

With those two files in place, open `_Host.cshtml` and move its content to a new file and call it `_Layout.cshtml`.

- One important things to do in the newly created `_Layout.cshtml` file is to replace `<component type="typeof(App)" render-mode="ServerPrerendered" />` with `@RenderBody()` (we will bring back the component below).

Once that is done, change the content of `_Host.cshtml` to:

```csharp
@page "/"
@namespace Auth0_Blazor.Pages
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
@using Microsoft.AspNetCore.Authentication
@{
    Layout = "_Layout";

    var tokens = new InitialApplicationState
    {
        AccessToken = await HttpContext.GetTokenAsync("access_token"),
        IdToken = await HttpContext.GetTokenAsync("id_token"),
        RefreshToken = await HttpContext.GetTokenAsync("refresh_token"),
    };
}

<component param-InitialState="tokens" type="typeof(App)" render-mode="ServerPrerendered" />
```

With those changes in place, open `App.razor` and set the values on the injected `TokenProvider` instance by using the `InitialState` parameter.

```csharp
@inject TokenProvider TokenProvider

@code {
    [Parameter]
    public InitialApplicationState? InitialState { get; set; }

    protected override Task OnInitializedAsync()
    {
        TokenProvider.AccessToken = InitialState?.AccessToken;
        TokenProvider.IdToken = InitialState?.IdToken;
        TokenProvider.RefreshToken = InitialState?.RefreshToken;

        return base.OnInitializedAsync();
    }
}
```

Once the above is done, you can inject `TokenProvider` in any `razor` component and start using the tokens:

```csharp
@inject TokenProvider tokenProvider
@attribute [Authorize]
@code {
    private string IdToken = "";

    protected override async Task OnInitializedAsync()
    {
        IdToken = tokenProvider.IdToken;

        await base.OnInitializedAsync();
    }
}

```
