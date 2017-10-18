+++
Description = "How to secure your Azure Function with a jwt token."
Keywords = ["Azure", "Functions", "jwt", "security"]
Tags = ["Azure", "Functions"]
date = "2017-10-17"
title = "Secure Azure Functions with JWT token"
+++
***I have completely rewritten this post. You can find the original post [here](/archive/secure-azure-functions-with-jwt-token).***

Out of the box it is only possible to secure your Azure Functions via Function Keys (API-Keys), which sometimes might not fit into your requirements. When using HttpTrigger we luckily have access to the current request and are therefor able to implement our own custom authentication/authorization methods. You could for example use JWT tokens issued by an OpenID provider to control authentication/authorization. 

In this post you learn how to validate OpenID JWT tokens and controlling access to your Azure Function. 

First lets see the function.

```csharp
public static class HelloWorldFunction
{
    [FunctionName("HelloWorld")]
    public static async Task<HttpResponseMessage> Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = null)]HttpRequestMessage req)
    {
        // Authentication boilerplate code start
        ClaimsPrincipal principal;
        if ((principal = await Security.ValidateTokenAsync(req.Headers.Authorization)) == null)
        {
            return req.CreateResponse(HttpStatusCode.Unauthorized);
        }
        // Authentication boilerplate code end

        return req.CreateResponse(HttpStatusCode.OK, "Hello " + principal.Identity.Name);
    }
}
```
It is very important that you set the authorization level to anonymous, since we want to skip all checks done by Azure Functions. Then we need to add the "authentication boilerplate code" to every function, we want to protect with OpenID JWT tokens. Unfortunately there is currently no generic way to add this, e.g. via attributes.

Now we need to implement the validation method. 


```csharp
using Microsoft.IdentityModel.Protocols;
using Microsoft.IdentityModel.Protocols.OpenIdConnect;
using Microsoft.IdentityModel.Tokens;
using System;
using System.IdentityModel.Tokens.Jwt;
using System.Net.Http.Headers;
using System.Security.Claims;
using System.Threading;
using System.Threading.Tasks;

namespace FunctionApp5
{
    public static class Security
    {
        private static readonly IConfigurationManager<OpenIdConnectConfiguration> _configurationManager;

        static Security()
        {
            var issuer = Environment.GetEnvironmentVariable("ISSUER");

            var documentRetriever = new HttpDocumentRetriever();
            documentRetriever.RequireHttps = issuer.StartsWith("https://");

            _configurationManager = new ConfigurationManager<OpenIdConnectConfiguration>(
                $"{issuer}/.well-known/openid-configuration",
                new OpenIdConnectConfigurationRetriever(),
                documentRetriever
            );
        }

        public static async Task<ClaimsPrincipal> ValidateTokenAsync(AuthenticationHeaderValue value)
        {
            if (value?.Scheme != "Bearer")
            {
                return null;
            }

            var config = await _configurationManager.GetConfigurationAsync(CancellationToken.None);
            var issuer = Environment.GetEnvironmentVariable("ISSUER");
            var audience = Environment.GetEnvironmentVariable("AUDIENCE");

            var validationParameter = new TokenValidationParameters()
            {
                RequireSignedTokens = true,
                ValidAudience = audience,
                ValidateAudience = true,
                ValidIssuer = issuer,
                ValidateIssuer = true,
                ValidateIssuerSigningKey = true,
                ValidateLifetime = true,
                IssuerSigningKeys = config.SigningKeys
            };

            ClaimsPrincipal result = null;
            var tries = 0;

            while (result == null && tries <= 1)
            {
                try
                {
                    var handler = new JwtSecurityTokenHandler();
                    result = handler.ValidateToken(value.Parameter, validationParameter, out var token);
                }
                catch (SecurityTokenSignatureKeyNotFoundException)
                {
                    // This exception is thrown if the signature key of the JWT could not be found.
                    // This could be the case when the issuer changed its signing keys, so we trigger a 
                    // refresh and retry validation.
                    _configurationManager.RequestRefresh();
                    tries++;
                }
                catch (SecurityTokenException)
                {
                    return null;
                }
            }

            return result;
        }
    }
}
```
Here we use a `ConfigurationManager` in order to retrieve the signing keys from the OpenID provider. It is very important to have the `ConfigurationManager` as a static, since it is caching the configuration in order to reduce HTTP requests to the OpenID provider. The `ConfigurationManager` refreshes the configuration after configurable timeouts. Next in the `ValidateTokenAsync` method we setup TokenValidationParameter. Here we want to enable as much validation as possible. 

Once we configured the validation parameters, we call `ValidateToken`. If the token can not be validated a `SecurityTokenException` is thrown. There are several derived exceptions, but in this case we just care for the `SecurityTokenSignatureKeyNotFoundException` exception, which is thrown if the `JwtSecurityTokenHandler` can not find the signature keys, that are specified in the JWT. This could be the case, when the issuer changed its signature keys, after the `ConfigurationManager` fetched the configuration. Therefore we trigger a refresh on the `ConfigurationManger` and retry to validate the JWT.  

If the validation is successful we return a `ClaimsPrincipal` which contains the claims provided by the token.

If you are not using OpenID you need to change the `ConfigurationManager` options. Alternatively you can remove the `ConfigurationManger` and provide static signing keys via the `TokenValidationParameters`.