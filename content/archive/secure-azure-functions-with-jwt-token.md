+++
Description = "How to secure your Azure Function with a jwt token."
Keywords = ["Azure", "Functions", "jwt"]
Tags = ["Azure", "Functions"]
date = "2017-07-20T17:05:31+02:00"
title = "Secure Azure Functions with jwt token"
+++

***Please take a look at the updated post [here](/post/secure-azure-functions-with-jwt-token). DO NOT USE THE CODE FROM THIS POST, WITHOUT ADDITION VALIDATION.***

Out of the box it is only possible to secure your Azure Functions via Function Keys (API-Keys), which sometimes might not fit into your requirements. When using HttpTrigger we luckily have access to the current request and are therefor able to implement our own custom authentication/authorization methods. You could for example use jwt tokens to control authentication/authorization. 

Here is an example code on how to validate jwt tokens and controlling access to your Azure Function. 

**CAUTION: You should not use this code in production. This is just a proof of concept and lacks a lot of validation!**

```csharp
using System;
using System.IdentityModel.Tokens.Jwt;
using System.Linq;
using System.Net;
using System.Net.Http;
using System.Security.Claims;
using System.Threading.Tasks;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.Azure.WebJobs.Host;

namespace FunctionTokenTest
{
    public static class TokenTest1
    {
        [FunctionName("TokenTest")]
        public static async Task<HttpResponseMessage> Run([HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = null)]HttpRequestMessage req, TraceWriter log)
        {
            if (!req.Headers.Contains("Authorization"))
            {
                return new HttpResponseMessage(HttpStatusCode.Forbidden);
            }

            var headerValue = req.Headers.GetValues("Authorization");
            var bearerValue = headerValue.FirstOrDefault(v => v.StartsWith("Bearer ")) ?? String.Empty;
            if (String.IsNullOrWhiteSpace(bearerValue))
            {
                return new HttpResponseMessage(HttpStatusCode.Forbidden);

            }
            var bearerToken = bearerValue.Split(' ')[1];

            var principal = ValidateToken(bearerToken, "MYISSUER", "MYSCOPE");
            if (principal == null)
            {
                return new HttpResponseMessage(HttpStatusCode.Forbidden);
            }

            return new HttpResponseMessage(HttpStatusCode.OK) { Content = new StringContent($"Hello {principal.Identity.Name}") };
        }

        private static ClaimsPrincipal ValidateToken(string jwtToken, string issuer, string requiredScope)
        {
            var handler = new JwtSecurityTokenHandler();
            if (!handler.CanReadToken(jwtToken))
            {
                return null;
            }

            handler.InboundClaimTypeMap.Clear();

            Microsoft.IdentityModel.Tokens.SecurityToken token;
            var principal = handler.ValidateToken(jwtToken, new Microsoft.IdentityModel.Tokens.TokenValidationParameters()
            {
                ValidateAudience = false,
                ValidIssuer = issuer,
                ValidateIssuerSigningKey = false,
                SignatureValidator = (t, param) => new JwtSecurityToken(t),
                NameClaimType = "sub"

            }, out token);

            if (!principal.Claims.Any(c => c.Type == "scope" && c.Value == requiredScope))
            {
                return null;
            }

            return principal;
        }
    }
}
```

You can find a runnable example at https://github.com/BorisWilhelms/FunctionTokenTest

