+++
Description = "How to load certificates stored in Azure"
Keywords = ["Azure", "Certificates", "WEBSITE_LOAD_CERTIFICATES"]
Tags = ["Azure", "Certificates"]
date = "2017-05-03T12:45:42+02:00"
title = "Access certificates in Azure"
+++

Thanks to this [Stackoverflow post](http://stackoverflow.com/questions/23827884/accessing-uploaded-certificates-in-azure-web-sites) for the tip. 

After uploading the certificate using the Azure Portal, you need to add an appsetting called *WEBSITE_LOAD_CERTIFICATES*. The value of the setting is the thumbprint of the certificate. We can also pass multiple thumbprints, delimited by comma or use * to load all available certificates. After this the certificates will be loaded to the *personal* certificate store of the *current user*. 
<!--more-->

The certificate can then be accessed by code 

```csharp
var store = new X509Store(StoreName.My, StoreLocation.CurrentUser);
store.Open(OpenFlags.ReadOnly);

var certs = store.Certificates.Find(X509FindType.FindByThumbprint, YOUR_THUMBPRINT, false);
```