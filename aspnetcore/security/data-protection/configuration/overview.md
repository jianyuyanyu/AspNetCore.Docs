---
title: Configure ASP.NET Core Data Protection
author: tdykstra
description: Learn how to configure Data Protection in ASP.NET Core.
monikerRange: '>= aspnetcore-3.1'
ms.author: tdykstra
ms.custom: mvc
ms.date: 06/11/2025
uid: security/data-protection/configuration/overview
---
# Configure ASP.NET Core Data Protection
<!-- ms.sfi.ropc: t -->

:::moniker range=">= aspnetcore-6.0"

When the Data Protection system is initialized, it applies [default settings](xref:security/data-protection/configuration/default-settings) based on the operational environment. These settings are appropriate for apps running on a single machine. However, there are cases where a developer may want to change the default settings:

* The app is spread across multiple machines.
* For compliance reasons.

For these scenarios, the Data Protection system offers a rich configuration API.

> [!WARNING]
> Similar to configuration files, the data protection key ring should be protected using appropriate permissions. You can choose to encrypt keys at rest, but this doesn't prevent cyberattackers from creating new keys. Consequently, your app's security is impacted. The storage location configured with Data Protection should have its access limited to the app itself, similar to the way you would protect configuration files. For example, if you choose to store your key ring on disk, use file system permissions. Ensure only the identity under which your web app runs has read, write, and create access to that directory. If you use Azure Blob Storage, only the web app should have the ability to read, write, or create new entries in the blob store, etc.
>
> The extension method <xref:Microsoft.Extensions.DependencyInjection.DataProtectionServiceCollectionExtensions.AddDataProtection%2A> returns an <xref:Microsoft.AspNetCore.DataProtection.IDataProtectionBuilder>. `IDataProtectionBuilder` exposes extension methods that you can chain together to configure Data Protection options.

> [!NOTE]
> This article was written for an app that runs within a docker container. In a docker container the app always has the same path and, therefore, the same application discriminator. Apps that need to run in multiple environments (for example local and deployed), must set the default application discriminator for the environment.
> Running an app in multiple environments is beyond the scope of this article.

The following NuGet packages are required for the Data Protection extensions used in this article:

* [Azure.Extensions.AspNetCore.DataProtection.Blobs](https://www.nuget.org/packages/Azure.Extensions.AspNetCore.DataProtection.Blobs)
* [Azure.Extensions.AspNetCore.DataProtection.Keys](https://www.nuget.org/packages/Azure.Extensions.AspNetCore.DataProtection.Keys)

## Protect keys with Azure Key Vault (`ProtectKeysWithAzureKeyVault`)

To interact with [Azure Key Vault](https://azure.microsoft.com/services/key-vault/) locally using developer credentials, either sign into your storage account in Visual Studio or sign in with the [Azure CLI](/cli/azure/). If you haven't already installed the Azure CLI, see [How to install the Azure CLI](/cli/azure/install-azure-cli). You can execute the following command in the Developer PowerShell panel in Visual Studio or from a command shell when not using Visual Studio:

```azurecli
az login
```

For more information, see [Sign-in to Azure using developer tooling](/dotnet/azure/sdk/authentication/local-development-dev-accounts#sign-in-to-azure-using-developer-tooling).

When establishing the key vault in the Entra or Azure portal:

* Configure the key vault to use Azure role-based access control (RABC). If you aren't operating on an [Azure Virtual Network](/azure/virtual-network/virtual-networks-overview), including for local development and testing, confirm that public access on the **Networking** step is **enabled** (checked). Enabling public access only exposes the key vault endpoint. Authenticated accounts are still required for access.

* Create an Azure Managed Identity (or add a role to the existing Managed Identity that you plan to use) with the **Key Vault Crypto User** role. Assign the Managed Identity to the Azure App Service that's hosting the deployment: **Settings** > **Identity** > **User assigned** > **Add**.

  > [!NOTE]
  > If you also plan to run an app locally with an authorized user for blob access using the [Azure CLI](/cli/azure/) or Visual Studio's Azure Service Authentication, add your developer Azure user account in **Access Control (IAM)** with the **Key Vault Crypto User** role. If you want to use the Azure CLI through Visual Studio, execute the `az login` command from the Developer PowerShell panel and follow the prompts to authenticate with the tenant.

* When key encryption is active, keys in the key file include the comment, ":::no-loc text="This key is encrypted with Azure Key Vault.":::" After starting the app, select the **View/edit** command from the context menu at the end of the key row to confirm that a key is present with key vault security applied.

* Optionally, you can enable automatic key vault key rotation without concern about decrypting payloads with data protection keys based on expired/rotated key vault keys. Each generated data protection key includes a reference to the key vault key used to encrypted it. Just make sure that you retain expired key vault keys, don't delete them in the key vault. Also, use a versionless key identifier in the app's key vault configuration, where no key GUID is placed at the end of the identifier (example: `https://contoso.vault.azure.net/keys/data-protection`). Use a similar rotation period for both keys with the key vault key rotating more frequently than the data protection key to ensure that a new key vault key is used at the time of data protection key rotation.

Protecting keys with Azure Key Vault implements an <xref:Microsoft.AspNetCore.DataProtection.XmlEncryption.IXmlEncryptor> that disables automatic data protection settings, including the key ring storage location. To configure the Azure Blob Storage provider to store the keys in blob storage, follow the guidance in <xref:security/data-protection/implementation/key-storage-providers#azure-storage> and call one of the <xref:Microsoft.AspNetCore.DataProtection.AzureDataProtectionBuilderExtensions.PersistKeysToAzureBlobStorage%2A> overloads in the app. The following example uses the overload that accepts a blob URI and token credential (<xref:Azure.Core.TokenCredential>), relying on an Azure Managed Identity for role-based access control (RBAC).

To configure the Azure Key Vault provider, call one of the <xref:Microsoft.AspNetCore.DataProtection.AzureDataProtectionKeyVaultKeyBuilderExtensions.ProtectKeysWithAzureKeyVault%2A> overloads. The following example uses the overload that accepts key identifier and token credential (<xref:Azure.Core.TokenCredential>), relying on a Managed Identity for RBAC in production (<xref:Azure.Identity.ManagedIdentityCredential>) or a <xref:Azure.Identity.DefaultAzureCredential> during development and testing. Other overloads accept either a key vault client or an app client ID with client secret. For more information, see <xref:security/data-protection/implementation/key-storage-providers#azure-storage>.

For more information on the Azure SDK's API and authentication, see [Authenticate .NET apps to Azure services using the Azure Identity library](/dotnet/azure/sdk/authentication/) and [Provide access to Key Vault keys, certificates, and secrets with Azure role-based access control](/azure/key-vault/general/rbac-guide?tabs=azure-cli). For logging guidance, see [Logging with the Azure SDK for .NET: Logging without client registration](/dotnet/azure/sdk/logging#logging-without-client-registration). For apps using dependency injection, an app can call <xref:Microsoft.Extensions.Azure.AzureClientServiceCollectionExtensions.AddAzureClientsCore%2A>, passing `true` for `enableLogForwarding`, to create and wire up the logging infrastructure.

In the `Program` file where services are registered:

```csharp
TokenCredential? credential;

if (builder.Environment.IsProduction())
{
    credential = new ManagedIdentityCredential("{MANAGED IDENTITY CLIENT ID}");
}
else
{
    // Local development and testing only
    DefaultAzureCredentialOptions options = new()
    {
        // Specify the tenant ID to use the dev credentials when running the app locally
        // in Visual Studio.
        VisualStudioTenantId = "{TENANT ID}",
        SharedTokenCacheTenantId = "{TENANT ID}"
    };

    credential = new DefaultAzureCredential(options);
}

builder.Services.AddDataProtection()
    .SetApplicationName("{APPLICATION NAME}")
    .PersistKeysToAzureBlobStorage(new Uri("{BLOB URI}"), credential)
    .ProtectKeysWithAzureKeyVault(new Uri("{KEY IDENTIFIER}"), credential);
```

`{MANAGED IDENTITY CLIENT ID}`: The Azure Managed Identity Client ID (GUID).

`{TENANT ID}`: Tenant ID.

`{APPLICATION NAME}`: <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.SetApplicationName%2A> sets the unique name of this app within the data protection system. The value should match across deployments of the app.

`{BLOB URI}`: Full URI to the key file. The URI is generated by Azure Storage when you create the key file. Do not use a SAS.

`{KEY IDENTIFIER}`: Azure Key Vault key identifier used for key encryption. An access policy allows the application to access the key vault with `Get`, `Unwrap Key`, and `Wrap Key` permissions. The version of the key is obtained from the key in the Entra or Azure portal after it's created. If you enable automatic rotation of the key vault key, make sure that you use a versionless key identifier in the app's key vault configuration, where no key GUID is placed at the end of the identifier (example: `https://contoso.vault.azure.net/keys/data-protection`).

For an app to communicate and authorize itself with Azure Key Vault, the [`Azure.Identity` NuGet package](https://www.nuget.org/packages/Azure.Identity/) must be referenced by the app.

[!INCLUDE[](~/includes/package-reference.md)]

> [!NOTE]
> In non-Production environments, the preceding example uses <xref:Azure.Identity.DefaultAzureCredential> to simplify authentication while developing apps that deploy to Azure by combining credentials used in Azure hosting environments with credentials used in local development. For more information, see [Authenticate Azure-hosted .NET apps to Azure resources using a system-assigned managed identity](/dotnet/azure/sdk/authentication/system-assigned-managed-identity).

If the app uses the older Azure packages (`Microsoft.AspNetCore.DataProtection.AzureStorage` and `Microsoft.AspNetCore.DataProtection.AzureKeyVault`), we recommend ***removing*** these references and upgrading to the [`Azure.Extensions.AspNetCore.DataProtection.Blobs`](https://www.nuget.org/packages/Azure.Extensions.AspNetCore.DataProtection.Blobs) and [`Azure.Extensions.AspNetCore.DataProtection.Keys`](https://www.nuget.org/packages/Azure.Extensions.AspNetCore.DataProtection.Keys) packages. The newer packages address key security and stability issues.

**Alternative shared-access signature (SAS) approach**: As an alternative to using a Managed Identity for access to the key blob in Azure Blob Storage, you can call the <xref:Microsoft.AspNetCore.DataProtection.AzureDataProtectionBuilderExtensions.PersistKeysToAzureBlobStorage%2A> overload that accepts a blob URI with a SAS token. The following example continues to use either a <xref:Azure.Identity.ManagedIdentityCredential> (production) or <xref:Azure.Identity.DefaultAzureCredential> (development and testing) for its <xref:Azure.Core.TokenCredential>, as seen in the preceding example:

```csharp
builder.Services.AddDataProtection()
    .SetApplicationName("{APPLICATION NAME}")
    .PersistKeysToAzureBlobStorage(new Uri("{BLOB URI WITH SAS}"))
    .ProtectKeysWithAzureKeyVault(new Uri("{KEY IDENTIFIER}"), credential);
```

`{APPLICATION NAME}`: <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.SetApplicationName%2A> sets the unique name of this app within the data protection system. The value should match across deployments of the app.

`{BLOB URI WITH SAS}`: The full URI where the key file should be stored with the SAS token as a query string parameter. The URI is generated by Azure Storage when you request a SAS for the uploaded key file.

`{KEY IDENTIFIER}`: Azure Key Vault key identifier used for key encryption. An access policy allows the application to access the key vault with `Get`, `Unwrap Key`, and `Wrap Key` permissions. The version of the key is obtained from the key in the Entra or Azure portal after it's created. If you enable automatic rotation of the key vault key, make sure that you use a versionless key identifier in the app's key vault configuration, where no key GUID is placed at the end of the identifier (example: `https://contoso.vault.azure.net/keys/data-protection`).

## Persist keys to the file system (`PersistKeysToFileSystem`)

To store keys on a UNC share instead of at the *%LOCALAPPDATA%* default location, configure the system with <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.PersistKeysToFileSystem%2A>:

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/Snippets/Program.cs" id="snippet_AddDataProtectionPersistKeysToFileSystem":::

> [!WARNING]
> If you change the key persistence location, the system no longer automatically encrypts keys at rest, since it doesn't know whether DPAPI is an appropriate encryption mechanism.

## Persist keys in a database (`PersistKeysToDbContext`)

To store keys in a database using EntityFramework, configure the system with the [Microsoft.AspNetCore.DataProtection.EntityFrameworkCore](https://www.nuget.org/packages/Microsoft.AspNetCore.DataProtection.EntityFrameworkCore/) package:

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/Snippets/Program.cs" id="snippet_AddDataProtectionPersistKeysToDbContext":::

The preceding code stores the keys in the configured database. The database context being used must implement `IDataProtectionKeyContext`.  `IDataProtectionKeyContext` exposes the property `DataProtectionKeys` 

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/Snippets/SampleDbContext.cs" id="snippet_DataProtectionKeys":::

This property represents the table in which the keys are stored. Create the table manually or with `DbContext` Migrations. For more information, see <xref:Microsoft.AspNetCore.DataProtection.EntityFrameworkCore.DataProtectionKey>.

## Protect keys configuration API (`ProtectKeysWith\*`)

You can configure the system to protect keys at rest by calling any of the [`ProtectKeysWith\*`](xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions) configuration APIs. Consider the example below, which stores keys on a UNC share and encrypts those keys at rest with a specific X.509 certificate.

:::moniker-end

:::moniker range=">= aspnetcore-9.0"

You can provide an <xref:System.Security.Cryptography.X509Certificates.X509Certificate2> to <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.ProtectKeysWithCertificate%2A> from a file by calling <xref:System.Security.Cryptography.X509Certificates.X509CertificateLoader.LoadCertificateFromFile%2A?displayProperty=nameWithType>:

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/Snippets/Program.cs" id="snippet_AddDataProtectionProtectKeysWithCertificateX509Certificate2":::

The following code example demonstrates how to load a certificate using a thumbprint:

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/Snippets/Program.cs" id="snippet_AddDataProtectionProtectKeysWithCertificate":::

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-9.0"

You can provide an <xref:System.Security.Cryptography.X509Certificates.X509Certificate2> to <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.ProtectKeysWithCertificate%2A>, such as a certificate loaded from a file:

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/Snippets/Program.cs" id="snippet_AddDataProtectionProtectKeysWithCertificateX509Certificate2":::

The following code example demonstrates how to load a certificate using a thumbprint:

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/Snippets/Program.cs" id="snippet_AddDataProtectionProtectKeysWithCertificate":::

:::moniker-end

:::moniker range=">= aspnetcore-6.0"

For examples and discussion on the built-in key encryption mechanisms, see <xref:security/data-protection/implementation/key-encryption-at-rest>.

## Unprotect keys with any certificate (`UnprotectKeysWithAnyCertificate`)

You can rotate certificates and decrypt keys at rest using an array of <xref:System.Security.Cryptography.X509Certificates.X509Certificate2> certificates with <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.UnprotectKeysWithAnyCertificate%2A>:

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/Snippets/Program.cs" id="snippet_AddDataProtectionUnprotectKeysWithAnyCertificate":::

## Set the default key lifetime (`SetDefaultKeyLifetime`)

To configure the system to use a key lifetime of 14 days instead of the default 90 days, use <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.SetDefaultKeyLifetime%2A>:

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/Snippets/Program.cs" id="snippet_AddDataProtectionSetDefaultKeyLifetime":::

## Set the application name (`SetApplicationName`)

By default, the Data Protection system isolates apps from one another based on their [content root](xref:fundamentals/index#content-root) paths, even if they share the same physical key repository. This isolation prevents the apps from understanding each other's protected payloads.

To share protected payloads among apps:

* Configure <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.SetApplicationName%2A> in each app with the same value.
* Use the same version of the Data Protection API stack across the apps. Perform **either** of the following in the apps' project files:
  * Reference the same shared framework version via the [Microsoft.AspNetCore.App metapackage](xref:fundamentals/metapackage-app).
  * Reference the same [Data Protection package](xref:security/data-protection/introduction#package-layout) version.

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/Snippets/Program.cs" id="snippet_AddDataProtectionSetApplicationName":::

<xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.SetApplicationName%2A> internally sets <xref:Microsoft.AspNetCore.DataProtection.DataProtectionOptions.ApplicationDiscriminator?displayProperty=nameWithType>. For troubleshooting purposes, the value assigned to the discriminator by the framework can be logged with the following code placed after the <xref:Microsoft.AspNetCore.Builder.WebApplication> is built in `Program.cs`:

```csharp
var discriminator = app.Services.GetRequiredService<IOptions<DataProtectionOptions>>()
    .Value.ApplicationDiscriminator;
app.Logger.LogInformation("ApplicationDiscriminator: {ApplicationDiscriminator}", discriminator);
```

For more information on how the discriminator is used, see the following sections later in this article:

* [Per-application isolation](#per-application-isolation)
* [Data Protection and app isolation](#data-protection-and-app-isolation)

> [!WARNING]
> In .NET 6, <xref:Microsoft.AspNetCore.Builder.WebApplicationBuilder> normalizes the content root path to end with a <xref:System.IO.Path.DirectorySeparatorChar>. For example, on Windows the content root path ends in `\` and on Linux `/`. Other hosts don't normalize the path. Most apps migrating from <xref:Microsoft.Extensions.Hosting.HostBuilder> or  <xref:Microsoft.AspNetCore.Hosting.WebHostBuilder> won't share the same app name because they won't have the terminating `DirectorySeparatorChar`. To work around this issue, remove the directory separator character and set the app name manually, as shown in the following code:
>
> ```csharp
> using System.Reflection;
> using Microsoft.AspNetCore.DataProtection;
> 
> var builder = WebApplication.CreateBuilder(args);
>
> var trimmedContentRootPath = 
>     builder.Environment.ContentRootPath.TrimEnd(Path.DirectorySeparatorChar);
>
> builder.Services.AddDataProtection().SetApplicationName(trimmedContentRootPath);
>
> var app = builder.Build();
> 
> app.MapGet("/", () => Assembly.GetEntryAssembly()!.GetName().Name);
> 
> app.Run();
> ```

## Disable automatic key generation (`DisableAutomaticKeyGeneration`)

You may have a scenario where you don't want an app to automatically roll keys (create new keys) as they approach expiration. One example of this scenario might be apps set up in a primary/secondary relationship, where only the primary app is responsible for key management concerns and secondary apps simply have a read-only view of the key ring. The secondary apps can be configured to treat the key ring as read-only by configuring the system with <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.DisableAutomaticKeyGeneration%2A>:

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/Snippets/Program.cs" id="snippet_AddDataProtectionDisableAutomaticKeyGeneration":::

## Per-application isolation

When the Data Protection system is provided by an ASP.NET Core host, it automatically isolates apps from one another, even if those apps are running under the same worker process account and are using the same master keying material. This is similar to the IsolateApps modifier from System.Web's `<machineKey>` element.

The isolation mechanism works by considering each app on the local machine as a unique tenant, thus the <xref:Microsoft.AspNetCore.DataProtection.IDataProtector> rooted for any given app automatically includes the app ID as a discriminator (<xref:Microsoft.AspNetCore.DataProtection.DataProtectionOptions.ApplicationDiscriminator>). The app's unique ID is the app's physical path:

* For apps hosted in IIS, the unique ID is the IIS physical path of the app. If an app is deployed in a web farm environment, this value is stable assuming that the IIS environments are configured similarly across all machines in the web farm.
* For self-hosted apps running on the [Kestrel server](xref:fundamentals/servers/index#kestrel), the unique ID is the physical path to the app on disk.

The unique identifier is designed to survive resets&mdash;both of the individual app and of the machine itself.

This isolation mechanism assumes that the apps aren't malicious. A malicious app can always impact any other app running under the same worker process account. In a shared hosting environment where apps are mutually untrusted, the hosting provider should take steps to ensure OS-level isolation between apps, including separating the apps' underlying key repositories.

If the Data Protection system isn't provided by an ASP.NET Core host (for example, if you instantiate it via the `DataProtectionProvider` concrete type) app isolation is disabled by default. When app isolation is disabled, all apps backed by the same keying material can share payloads as long as they provide the appropriate [purposes](xref:security/data-protection/consumer-apis/purpose-strings). To provide app isolation in this environment, call the [`SetApplicationName`](#set-the-application-name-setapplicationname) method on the configuration object and provide a unique name for each app.

### Data Protection and app isolation

Consider the following points for app isolation:

* When multiple apps are pointed at the same key repository, the intention is that the apps share the same master key material. Data Protection is developed with the assumption that all apps sharing a key ring can access all items in that key ring. The application unique identifier is used to isolate application specific keys derived from the key ring provided keys. It doesn't expect item level permissions, such as those provided by Azure KeyVault to be used to enforce extra isolation. Attempting item level permissions generates application errors. If you don't want to rely on the built-in application isolation, separate key store locations should be used and not shared between applications.

* The application discriminator (<xref:Microsoft.AspNetCore.DataProtection.DataProtectionOptions.ApplicationDiscriminator>) is used to allow different apps to share the same master key material but to keep their cryptographic payloads distinct from one another. <!-- The docs already draw an analogy between this and multi-tenancy.--> For the apps to be able to read each other's cryptographic payloads, they must have the same application discriminator, which can be set by calling [`SetApplicationName`](#set-the-application-name-setapplicationname).

* If an app is compromised (for example, by an RCE attack), all master key material accessible to that app must also be considered compromised, regardless of its protection-at-rest state. This implies that if two apps are pointed at the same repository, even if they use different app discriminators, a compromise of one is functionally equivalent to a compromise of both.

  This "functionally equivalent to a compromise of both" clause holds even if the two apps use different mechanisms for key protection at rest. Typically, this isn't an expected configuration. The protection-at-rest mechanism is intended to provide protection in the event a cyberattacker gains read access to the repository. A cyberattacker who gains write access to the repository (perhaps because they attained code execution permission within an app) can insert malicious keys into storage. The Data Protection system intentionally doesn't provide protection against a cyberattacker who gains write access to the key repository.

* If apps need to remain truly isolated from one another, they should use different key repositories. This naturally falls out of the definition of "isolated". Apps are ***not*** isolated if they all have Read and Write access to each other's data stores.

## Changing algorithms with `UseCryptographicAlgorithms`

The Data Protection stack allows you to change the default algorithm used by newly generated keys. The simplest way to do this is to call <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.UseCryptographicAlgorithms%2A> from the configuration callback:

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/Snippets/Program.cs" id="snippet_AddDataProtectionUseCryptographicAlgorithms":::

The default EncryptionAlgorithm is AES-256-CBC, and the default ValidationAlgorithm is HMACSHA256. The default policy can be set by a system administrator via a [machine-wide policy](xref:security/data-protection/configuration/machine-wide-policy), but an explicit call to `UseCryptographicAlgorithms` overrides the default policy.

Calling `UseCryptographicAlgorithms` allows you to specify the desired algorithm from a predefined built-in list. You don't need to worry about the implementation of the algorithm. In the scenario above, the Data Protection system attempts to use the CNG implementation of AES if running on Windows. Otherwise, it falls back to the managed <xref:System.Security.Cryptography.Aes?displayProperty=fullName> class.

You can manually specify an implementation via a call to <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.UseCustomCryptographicAlgorithms%2A>.

> [!TIP]
> Changing algorithms doesn't affect existing keys in the key ring. It only affects newly-generated keys.

### Specifying custom managed algorithms

To specify custom managed algorithms, create a <xref:Microsoft.AspNetCore.DataProtection.AuthenticatedEncryption.ConfigurationModel.ManagedAuthenticatedEncryptorConfiguration> instance that points to the implementation types:

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/Snippets/Program.cs" id="snippet_AddDataProtectionUseCustomCryptographicAlgorithms":::

Generally the \*Type properties must point to concrete, instantiable (via a public parameterless ctor) implementations of <xref:System.Security.Cryptography.SymmetricAlgorithm> and <xref:System.Security.Cryptography.KeyedHashAlgorithm>, though the system special-cases some values like `typeof(Aes)` for convenience.

> [!NOTE]
> The SymmetricAlgorithm must have a key length of ≥ 128 bits and a block size of ≥ 64 bits, and it must support CBC-mode encryption with PKCS #7 padding. The KeyedHashAlgorithm must have a digest size of >= 128 bits, and it must support keys of length equal to the hash algorithm's digest length. The KeyedHashAlgorithm isn't strictly required to be HMAC.

### Specifying custom Windows CNG algorithms

To specify a custom Windows CNG algorithm using CBC-mode encryption with HMAC validation, create a <xref:Microsoft.AspNetCore.DataProtection.AuthenticatedEncryption.ConfigurationModel.CngCbcAuthenticatedEncryptorConfiguration> instance that contains the algorithmic information:

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/Snippets/Program.cs" id="snippet_AddDataProtectionUseCustomCryptographicAlgorithmsCngCbc":::

> [!NOTE]
> The symmetric block cipher algorithm must have a key length of >= 128 bits, a block size of >= 64 bits, and it must support CBC-mode encryption with PKCS #7 padding. The hash algorithm must have a digest size of >= 128 bits and must support being opened with the BCRYPT\_ALG\_HANDLE\_HMAC\_FLAG flag. The \*Provider properties can be set to null to use the default provider for the specified algorithm. For more information, see the [BCryptOpenAlgorithmProvider](/windows/win32/api/bcrypt/nf-bcrypt-bcryptopenalgorithmprovider) documentation.

To specify a custom Windows CNG algorithm using Galois/Counter Mode encryption with validation, create a <xref:Microsoft.AspNetCore.DataProtection.AuthenticatedEncryption.ConfigurationModel.CngGcmAuthenticatedEncryptorConfiguration> instance that contains the algorithmic information:

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/Snippets/Program.cs" id="snippet_AddDataProtectionUseCustomCryptographicAlgorithmsCngGcm":::

> [!NOTE]
> The symmetric block cipher algorithm must have a key length of >= 128 bits, a block size of exactly 128 bits, and it must support GCM encryption. You can set the <xref:Microsoft.AspNetCore.DataProtection.AuthenticatedEncryption.ConfigurationModel.CngCbcAuthenticatedEncryptorConfiguration.EncryptionAlgorithmProvider> property to null to use the default provider for the specified algorithm. For more information, see the [BCryptOpenAlgorithmProvider](/windows/win32/api/bcrypt/nf-bcrypt-bcryptopenalgorithmprovider) documentation.

### Specifying other custom algorithms

Though not exposed as a first-class API, the Data Protection system is extensible enough to allow specifying almost any kind of algorithm. For example, it's possible to keep all keys contained within a Hardware Security Module (HSM) and to provide a custom implementation of the core encryption and decryption routines. For more information, see <xref:Microsoft.AspNetCore.DataProtection.AuthenticatedEncryption.IAuthenticatedEncryptor> in [Core cryptography extensibility](xref:security/data-protection/extensibility/core-crypto).

## Persisting keys when hosting in a Docker container

When hosting in a [Docker](/dotnet/standard/microservices-architecture/container-docker-introduction/) container, keys should be maintained in either:

* A folder that's a Docker volume that persists beyond the container's lifetime, such as a shared volume or a host-mounted volume.
* An external provider, such as [Azure Blob Storage](/azure/storage/blobs/storage-blobs-introduction) (shown in the [`ProtectKeysWithAzureKeyVault`](#protect-keys-with-azure-key-vault-protectkeyswithazurekeyvault) section) or [Redis](https://redis.io).

## Persisting keys with Redis

Only Redis versions supporting [Redis Data Persistence](/azure/azure-cache-for-redis/cache-how-to-premium-persistence) should be used to store keys. [Azure Blob storage](/azure/storage/blobs/storage-blobs-introduction) is persistent and can be used to store keys. For more information, see [this GitHub issue](https://github.com/dotnet/AspNetCore/issues/13476).

## Logging

Enable the `Information` or lower level of logging to diagnose problems. The following `appsettings.json` file enables information logging of the Data Protection API:

:::code language="csharp" source="samples/6.x/DataProtectionConfigurationSample/appsettings.json" highlight="6":::

For more information on logging, see <xref:fundamentals/logging/index>.

## Additional resources

* <xref:security/data-protection/configuration/non-di-scenarios>
* <xref:security/data-protection/configuration/machine-wide-policy>
* <xref:host-and-deploy/web-farm>
* <xref:security/data-protection/implementation/key-storage-providers>

:::moniker-end

:::moniker range="< aspnetcore-6.0"

When the Data Protection system is initialized, it applies [default settings](xref:security/data-protection/configuration/default-settings) based on the operational environment. These settings are appropriate for apps running on a single machine. However, there are cases where a developer may want to change the default settings:

* The app is spread across multiple machines.
* For compliance reasons.

For these scenarios, the Data Protection system offers a rich configuration API.

> [!WARNING]
> Similar to configuration files, the data protection key ring should be protected using appropriate permissions. You can choose to encrypt keys at rest, but this doesn't prevent cyberattackers from creating new keys. Consequently, your app's security is impacted. The storage location configured with Data Protection should have its access limited to the app itself, similar to the way you would protect configuration files. For example, if you choose to store your key ring on disk, use file system permissions. Ensure only the identity under which your web app runs has read, write, and create access to that directory. If you use Azure Blob Storage, only the web app should have the ability to read, write, or create new entries in the blob store.
>
> The extension method <xref:Microsoft.Extensions.DependencyInjection.DataProtectionServiceCollectionExtensions.AddDataProtection%2A> returns an <xref:Microsoft.AspNetCore.DataProtection.IDataProtectionBuilder>, which exposes extension methods that you can chain together to configure Data Protection options.

The following NuGet packages are required for the Data Protection extensions used in this article:

* [`Azure.Extensions.AspNetCore.DataProtection.Blobs`](https://www.nuget.org/packages/Azure.Extensions.AspNetCore.DataProtection.Blobs)
* [`Azure.Extensions.AspNetCore.DataProtection.Keys`](https://www.nuget.org/packages/Azure.Extensions.AspNetCore.DataProtection.Keys)

[!INCLUDE[](~/includes/package-reference.md)]

## Protect keys with Azure Key Vault (`ProtectKeysWithAzureKeyVault`)

To interact with [Azure Key Vault](https://azure.microsoft.com/services/key-vault/) locally using developer credentials, either sign into your storage account in Visual Studio or sign in with the [Azure CLI](/cli/azure/). If you haven't already installed the Azure CLI, see [How to install the Azure CLI](/cli/azure/install-azure-cli). You can execute the following command in the Developer PowerShell panel in Visual Studio or from a command shell when not using Visual Studio:

```azurecli
az login
```

For more information, see [Sign-in to Azure using developer tooling](/dotnet/azure/sdk/authentication/local-development-dev-accounts#sign-in-to-azure-using-developer-tooling).

When establishing the key vault in the Entra or Azure portal:

* Configure the key vault to use Azure role-based access control (RABC). If you aren't operating on an [Azure Virtual Network](/azure/virtual-network/virtual-networks-overview), including for local development and testing, confirm that public access on the **Networking** step is **enabled** (checked). Enabling public access only exposes the key vault endpoint. Authenticated accounts are still required for access.

* Create an Azure Managed Identity (or add a role to the existing Managed Identity that you plan to use) with the **Key Vault Crypto User** role. Assign the Managed Identity to the Azure App Service that's hosting the deployment: **Settings** > **Identity** > **User assigned** > **Add**.

  > [!NOTE]
  > If you also plan to run an app locally with an authorized user for blob access using the [Azure CLI](/cli/azure/) or Visual Studio's Azure Service Authentication, add your developer Azure user account in **Access Control (IAM)** with the **Key Vault Crypto User** role. If you want to use the Azure CLI through Visual Studio, execute the `az login` command from the Developer PowerShell panel and follow the prompts to authenticate with the tenant.

* When key encryption is active, keys in the key file include the comment, ":::no-loc text="This key is encrypted with Azure Key Vault.":::" After starting the app, select the **View/edit** command from the context menu at the end of the key row to confirm that a key is present with key vault security applied.

* Optionally, you can enable automatic key vault key rotation without concern about decrypting payloads with data protection keys based on expired/rotated key vault keys. Each generated data protection key includes a reference to the key vault key used to encrypted it. Just make sure that you retain expired key vault keys, don't delete them in the key vault. Also, use a versionless key identifier in the app's key vault configuration, where no key GUID is placed at the end of the identifier (example: `https://contoso.vault.azure.net/keys/data-protection`). Use a similar rotation period for both keys with the key vault key rotating more frequently than the data protection key to ensure that a new key vault key is used at the time of data protection key rotation.

Protecting keys with Azure Key Vault implements an <xref:Microsoft.AspNetCore.DataProtection.XmlEncryption.IXmlEncryptor> that disables automatic data protection settings, including the key ring storage location. To configure the Azure Blob Storage provider to store the keys in blob storage, follow the guidance in <xref:security/data-protection/implementation/key-storage-providers#azure-storage> and call one of the <xref:Microsoft.AspNetCore.DataProtection.AzureDataProtectionBuilderExtensions.PersistKeysToAzureBlobStorage%2A> overloads in the app. The following example uses the overload that accepts a blob URI and token credential (<xref:Azure.Core.TokenCredential>), relying on an Azure Managed Identity for role-based access control (RBAC).

To configure the Azure Key Vault provider, call one of the <xref:Microsoft.AspNetCore.DataProtection.AzureDataProtectionKeyVaultKeyBuilderExtensions.ProtectKeysWithAzureKeyVault%2A> overloads. The following example uses the overload that accepts key identifier and token credential (<xref:Azure.Core.TokenCredential>), relying on a Managed Identity for RBAC in production (<xref:Azure.Identity.ManagedIdentityCredential>) or a <xref:Azure.Identity.DefaultAzureCredential> during development and testing. Other overloads accept either a key vault client or an app client ID with client secret. For more information, see <xref:security/data-protection/implementation/key-storage-providers#azure-storage>.

For more information on the Azure SDK's API and authentication, see [Authenticate .NET apps to Azure services using the Azure Identity library](/dotnet/azure/sdk/authentication/) and [Provide access to Key Vault keys, certificates, and secrets with Azure role-based access control](/azure/key-vault/general/rbac-guide?tabs=azure-cli). For logging guidance, see [Logging with the Azure SDK for .NET: Logging without client registration](/dotnet/azure/sdk/logging#logging-without-client-registration). For apps using dependency injection, an app can call <xref:Microsoft.Extensions.Azure.AzureClientServiceCollectionExtensions.AddAzureClientsCore%2A>, passing `true` for `enableLogForwarding`, to create and wire up the logging infrastructure.

In the `Program` file where services are registered:

```csharp
TokenCredential? credential;

if (builder.Environment.IsProduction())
{
    credential = new ManagedIdentityCredential("{MANAGED IDENTITY CLIENT ID}");
}
else
{
    // Local development and testing only
    DefaultAzureCredentialOptions options = new()
    {
        // Specify the tenant ID to use the dev credentials when running the app locally
        // in Visual Studio.
        VisualStudioTenantId = "{TENANT ID}",
        SharedTokenCacheTenantId = "{TENANT ID}"
    };

    credential = new DefaultAzureCredential(options);
}

services.AddDataProtection()
    .SetApplicationName("{APPLICATION NAME}")
    .PersistKeysToAzureBlobStorage(new Uri("{BLOB URI}"), credential)
    .ProtectKeysWithAzureKeyVault(new Uri("{KEY IDENTIFIER}"), credential);
```

`{MANAGED IDENTITY CLIENT ID}`: The Azure Managed Identity Client ID (GUID).

`{TENANT ID}`: Tenant ID.

`{APPLICATION NAME}`: <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.SetApplicationName%2A> sets the unique name of this app within the data protection system. The value should match across deployments of the app.

`{BLOB URI}`: Full URI to the key file. The URI is generated by Azure Storage when you create the key file. Do not use a SAS.

`{KEY IDENTIFIER}`: Azure Key Vault key identifier used for key encryption. An access policy allows the application to access the key vault with `Get`, `Unwrap Key`, and `Wrap Key` permissions. The version of the key is obtained from the key in the Entra or Azure portal after it's created. If you enable automatic rotation of the key vault key, make sure that you use a versionless key identifier in the app's key vault configuration, where no key GUID is placed at the end of the identifier (example: `https://contoso.vault.azure.net/keys/data-protection`).

For an app to communicate and authorize itself with Azure Key Vault, the [`Azure.Identity` NuGet package](https://www.nuget.org/packages/Azure.Identity/) must be referenced by the app.

[!INCLUDE[](~/includes/package-reference.md)]

> [!NOTE]
> In non-Production environments, the preceding example uses <xref:Azure.Identity.DefaultAzureCredential> to simplify authentication while developing apps that deploy to Azure by combining credentials used in Azure hosting environments with credentials used in local development. For more information, see [Authenticate Azure-hosted .NET apps to Azure resources using a system-assigned managed identity](/dotnet/azure/sdk/authentication/system-assigned-managed-identity).

If the app uses the older Azure packages (`Microsoft.AspNetCore.DataProtection.AzureStorage` and `Microsoft.AspNetCore.DataProtection.AzureKeyVault`), we recommend ***removing*** these references and upgrading to the [`Azure.Extensions.AspNetCore.DataProtection.Blobs`](https://www.nuget.org/packages/Azure.Extensions.AspNetCore.DataProtection.Blobs) and [`Azure.Extensions.AspNetCore.DataProtection.Keys`](https://www.nuget.org/packages/Azure.Extensions.AspNetCore.DataProtection.Keys) packages. The newer packages address key security and stability issues.

**Alternative shared-access signature (SAS) approach**: As an alternative to using a Managed Identity for access to the key blob in Azure Blob Storage, you can call the <xref:Microsoft.AspNetCore.DataProtection.AzureDataProtectionBuilderExtensions.PersistKeysToAzureBlobStorage%2A> overload that accepts a blob URI with a SAS token. The following example continues to use either a <xref:Azure.Identity.ManagedIdentityCredential> (production) or <xref:Azure.Identity.DefaultAzureCredential> (development and testing) for its <xref:Azure.Core.TokenCredential>, as seen in the preceding example:

```csharp
services.AddDataProtection()
    .SetApplicationName("{APPLICATION NAME}")
    .PersistKeysToAzureBlobStorage(new Uri("{BLOB URI WITH SAS}"))
    .ProtectKeysWithAzureKeyVault(new Uri("{KEY IDENTIFIER}"), credential);
```

`{APPLICATION NAME}`: <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.SetApplicationName%2A> sets the unique name of this app within the data protection system. The value should match across deployments of the app.

`{BLOB URI WITH SAS}`: The full URI where the key file should be stored with the SAS token as a query string parameter. The URI is generated by Azure Storage when you request a SAS for the uploaded key file.

`{KEY IDENTIFIER}`: Azure Key Vault key identifier used for key encryption. An access policy allows the application to access the key vault with `Get`, `Unwrap Key`, and `Wrap Key` permissions. The version of the key is obtained from the key in the Entra or Azure portal after it's created. If you enable automatic rotation of the key vault key, make sure that you use a versionless key identifier in the app's key vault configuration, where no key GUID is placed at the end of the identifier (example: `https://contoso.vault.azure.net/keys/data-protection`).

## Persist keys to the file system (`PersistKeysToFileSystem`)

To store keys on a UNC share instead of at the *%LOCALAPPDATA%* default location, configure the system with <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.PersistKeysToFileSystem%2A>:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDataProtection()
        .PersistKeysToFileSystem(new DirectoryInfo(@"\\server\share\directory\"));
}
```

> [!WARNING]
> If you change the key persistence location, the system no longer automatically encrypts keys at rest, since it doesn't know whether DPAPI is an appropriate encryption mechanism.

## Persist keys in a database (`PersistKeysToDbContext`)

To store keys in a database using EntityFramework, configure the system with the [Microsoft.AspNetCore.DataProtection.EntityFrameworkCore](https://www.nuget.org/packages/Microsoft.AspNetCore.DataProtection.EntityFrameworkCore/) package:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDataProtection()
        .PersistKeysToDbContext<DbContext>()
}
```

The preceding code stores the keys in the configured database. The database context being used must implement `IDataProtectionKeyContext`.  `IDataProtectionKeyContext` exposes the property `DataProtectionKeys` 

```csharp
public DbSet<DataProtectionKey> DataProtectionKeys { get; set; }
```

This property represents the table in which the keys are stored. Create the table manually or with `DbContext` Migrations. For more information, see <xref:Microsoft.AspNetCore.DataProtection.EntityFrameworkCore.DataProtectionKey>.

## Protect keys configuration API (`ProtectKeysWith\*`)

You can configure the system to protect keys at rest by calling any of the [`ProtectKeysWith\*`](xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions) configuration APIs. Consider the example below, which stores keys on a UNC share and encrypts those keys at rest with a specific X.509 certificate:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDataProtection()
        .PersistKeysToFileSystem(new DirectoryInfo(@"\\server\share\directory\"))
        .ProtectKeysWithCertificate(Configuration["Thumbprint"]);
}
```

You can provide an <xref:System.Security.Cryptography.X509Certificates.X509Certificate2> to <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.ProtectKeysWithCertificate%2A>, such as a certificate loaded from a file:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDataProtection()
        .PersistKeysToFileSystem(new DirectoryInfo(@"\\server\share\directory\"))
        .ProtectKeysWithCertificate(
            new X509Certificate2("certificate.pfx", Configuration["Thumbprint"]));
}
```

For more examples and discussion on the built-in key encryption mechanisms, see <xref:security/data-protection/implementation/key-encryption-at-rest>.

## Unprotect keys with any certificate (`UnprotectKeysWithAnyCertificate`)

You can rotate certificates and decrypt keys at rest using an array of <xref:System.Security.Cryptography.X509Certificates.X509Certificate2> certificates with <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.UnprotectKeysWithAnyCertificate%2A>:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDataProtection()
        .PersistKeysToFileSystem(new DirectoryInfo(@"\\server\share\directory\"))
        .ProtectKeysWithCertificate(
            new X509Certificate2("certificate.pfx", Configuration["MyPasswordKey"));
        .UnprotectKeysWithAnyCertificate(
            new X509Certificate2("certificate_old_1.pfx", Configuration["MyPasswordKey_1"]),
            new X509Certificate2("certificate_old_2.pfx", Configuration["MyPasswordKey_2"]));
}
```

## Set the default key lifetime (`SetDefaultKeyLifetime`)

To configure the system to use a key lifetime of 14 days instead of the default 90 days, use <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.SetDefaultKeyLifetime%2A>:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDataProtection()
        .SetDefaultKeyLifetime(TimeSpan.FromDays(14));
}
```

## Set the application name (`SetApplicationName`)

By default, the Data Protection system isolates apps from one another based on their [content root](xref:fundamentals/index#content-root) paths, even if they share the same physical key repository. This isolation prevents the apps from understanding each other's protected payloads.

To share protected payloads among apps:

* Configure <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.SetApplicationName%2A> in each app with the same value.
* Use the same version of the Data Protection API stack across the apps. Perform **either** of the following in the apps' project files:
  * Reference the same shared framework version via the [Microsoft.AspNetCore.App metapackage](xref:fundamentals/metapackage-app).
  * Reference the same [Data Protection package](xref:security/data-protection/introduction#package-layout) version.

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDataProtection()
        .SetApplicationName("{APPLICATION NAME}");
}
```

<xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.SetApplicationName%2A> internally sets <xref:Microsoft.AspNetCore.DataProtection.DataProtectionOptions.ApplicationDiscriminator?displayProperty=nameWithType>. For more information on how the discriminator is used, see the following sections later in this article:

* [Per-application isolation](#per-application-isolation)
* [Data Protection and app isolation](#data-protection-and-app-isolation)

## Disable automatic key generation (`DisableAutomaticKeyGeneration`)

You may have a scenario where you don't want an app to automatically roll keys (create new keys) as they approach expiration. One example of this scenario might be apps set up in a primary/secondary relationship, where only the primary app is responsible for key management concerns and secondary apps simply have a read-only view of the key ring. The secondary apps can be configured to treat the key ring as read-only by configuring the system with <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.DisableAutomaticKeyGeneration%2A>:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddDataProtection()
        .DisableAutomaticKeyGeneration();
}
```

## Per-application isolation

When the Data Protection system is provided by an ASP.NET Core host, it automatically isolates apps from one another, even if those apps are running under the same worker process account and are using the same master keying material. This is similar to the IsolateApps modifier from System.Web's `<machineKey>` element.

The isolation mechanism works by considering each app on the local machine as a unique tenant, thus the <xref:Microsoft.AspNetCore.DataProtection.IDataProtector> rooted for any given app automatically includes the app ID as a discriminator (<xref:Microsoft.AspNetCore.DataProtection.DataProtectionOptions.ApplicationDiscriminator>). The app's unique ID is the app's physical path:

* For apps hosted in IIS, the unique ID is the IIS physical path of the app. If an app is deployed in a web farm environment, this value is stable assuming that the IIS environments are configured similarly across all machines in the web farm.
* For self-hosted apps running on the [Kestrel server](xref:fundamentals/servers/index#kestrel), the unique ID is the physical path to the app on disk.

The unique identifier is designed to survive resets&mdash;both of the individual app and of the machine itself.

This isolation mechanism assumes that the apps aren't malicious. A malicious app can always impact any other app running under the same worker process account. In a shared hosting environment where apps are mutually untrusted, the hosting provider should take steps to ensure OS-level isolation between apps, including separating the apps' underlying key repositories.

If the Data Protection system isn't provided by an ASP.NET Core host (for example, if you instantiate it via the `DataProtectionProvider` concrete type) app isolation is disabled by default. When app isolation is disabled, all apps backed by the same keying material can share payloads as long as they provide the appropriate [purposes](xref:security/data-protection/consumer-apis/purpose-strings). To provide app isolation in this environment, call the [SetApplicationName](#set-the-application-name-setapplicationname) method on the configuration object and provide a unique name for each app.

### Data Protection and app isolation

Consider the following points for app isolation:

* When multiple apps are pointed at the same key repository, the intention is that the apps share the same master key material. Data Protection is developed with the assumption that all apps sharing a key ring can access all items in that key ring. The application unique identifier is used to isolate application specific keys derived from the key ring provided keys. It doesn't expect item level permissions, such as those provided by Azure KeyVault to be used to enforce extra isolation. Attempting item level permissions generates application errors. If you don't want to rely on the built-in application isolation, separate key store locations should be used and not shared between applications.

* The application discriminator (<xref:Microsoft.AspNetCore.DataProtection.DataProtectionOptions.ApplicationDiscriminator>) is used to allow different apps to share the same master key material but to keep their cryptographic payloads distinct from one another. <!-- The docs already draw an analogy between this and multi-tenancy.--> For the apps to be able to read each other's cryptographic payloads, they must have the same application discriminator, which can be set by calling [`SetApplicationName`](#set-the-application-name-setapplicationname).

* If an app is compromised (for example, by an RCE attack), all master key material accessible to that app must also be considered compromised, regardless of its protection-at-rest state. This implies that if two apps are pointed at the same repository, even if they use different app discriminators, a compromise of one is functionally equivalent to a compromise of both.

  This "functionally equivalent to a compromise of both" clause holds even if the two apps use different mechanisms for key protection at rest. Typically, this isn't an expected configuration. The protection-at-rest mechanism is intended to provide protection in the event a cyberattacker gains read access to the repository. A cyberattacker who gains write access to the repository (perhaps because they attained code execution permission within an app) can insert malicious keys into storage. The Data Protection system intentionally doesn't provide protection against a cyberattacker who gains write access to the key repository.

* If apps need to remain truly isolated from one another, they should use different key repositories. This naturally falls out of the definition of "isolated". Apps are ***not*** isolated if they all have Read and Write access to each other's data stores.

## Changing algorithms with `UseCryptographicAlgorithms`

The Data Protection stack allows you to change the default algorithm used by newly generated keys. The simplest way to do this is to call <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.UseCryptographicAlgorithms%2A> from the configuration callback:

```csharp
services.AddDataProtection()
    .UseCryptographicAlgorithms(
        new AuthenticatedEncryptorConfiguration()
    {
        EncryptionAlgorithm = EncryptionAlgorithm.AES_256_CBC,
        ValidationAlgorithm = ValidationAlgorithm.HMACSHA256
    });
```

The default EncryptionAlgorithm is AES-256-CBC, and the default ValidationAlgorithm is HMACSHA256. The default policy can be set by a system administrator via a [machine-wide policy](xref:security/data-protection/configuration/machine-wide-policy), but an explicit call to `UseCryptographicAlgorithms` overrides the default policy.

Calling `UseCryptographicAlgorithms` allows you to specify the desired algorithm from a predefined built-in list. You don't need to worry about the implementation of the algorithm. In the scenario above, the Data Protection system attempts to use the CNG implementation of AES if running on Windows. Otherwise, it falls back to the managed <xref:System.Security.Cryptography.Aes?displayProperty=fullName> class.

You can manually specify an implementation via a call to <xref:Microsoft.AspNetCore.DataProtection.DataProtectionBuilderExtensions.UseCustomCryptographicAlgorithms%2A>.

> [!TIP]
> Changing algorithms doesn't affect existing keys in the key ring. It only affects newly-generated keys.

### Specifying custom managed algorithms

To specify custom managed algorithms, create a <xref:Microsoft.AspNetCore.DataProtection.AuthenticatedEncryption.ConfigurationModel.ManagedAuthenticatedEncryptorConfiguration> instance that points to the implementation types:

```csharp
serviceCollection.AddDataProtection()
    .UseCustomCryptographicAlgorithms(
        new ManagedAuthenticatedEncryptorConfiguration()
    {
        // A type that subclasses SymmetricAlgorithm
        EncryptionAlgorithmType = typeof(Aes),

        // Specified in bits
        EncryptionAlgorithmKeySize = 256,

        // A type that subclasses KeyedHashAlgorithm
        ValidationAlgorithmType = typeof(HMACSHA256)
    });
```

Generally the \*Type properties must point to concrete, instantiable (via a public parameterless ctor) implementations of <xref:System.Security.Cryptography.SymmetricAlgorithm> and <xref:System.Security.Cryptography.KeyedHashAlgorithm>, though the system special-cases some values like `typeof(Aes)` for convenience.

> [!NOTE]
> The SymmetricAlgorithm must have a key length of ≥ 128 bits and a block size of ≥ 64 bits, and it must support CBC-mode encryption with PKCS #7 padding. The KeyedHashAlgorithm must have a digest size of >= 128 bits, and it must support keys of length equal to the hash algorithm's digest length. The KeyedHashAlgorithm isn't strictly required to be HMAC.

### Specifying custom Windows CNG algorithms

To specify a custom Windows CNG algorithm using CBC-mode encryption with HMAC validation, create a <xref:Microsoft.AspNetCore.DataProtection.AuthenticatedEncryption.ConfigurationModel.CngCbcAuthenticatedEncryptorConfiguration> instance that contains the algorithmic information:

```csharp
services.AddDataProtection()
    .UseCustomCryptographicAlgorithms(
        new CngCbcAuthenticatedEncryptorConfiguration()
    {
        // Passed to BCryptOpenAlgorithmProvider
        EncryptionAlgorithm = "AES",
        EncryptionAlgorithmProvider = null,

        // Specified in bits
        EncryptionAlgorithmKeySize = 256,

        // Passed to BCryptOpenAlgorithmProvider
        HashAlgorithm = "SHA256",
        HashAlgorithmProvider = null
    });
```

> [!NOTE]
> The symmetric block cipher algorithm must have a key length of >= 128 bits, a block size of >= 64 bits, and it must support CBC-mode encryption with PKCS #7 padding. The hash algorithm must have a digest size of >= 128 bits and must support being opened with the BCRYPT\_ALG\_HANDLE\_HMAC\_FLAG flag. The \*Provider properties can be set to null to use the default provider for the specified algorithm. For more information, see the [BCryptOpenAlgorithmProvider](/windows/win32/api/bcrypt/nf-bcrypt-bcryptopenalgorithmprovider) documentation.

To specify a custom Windows CNG algorithm using Galois/Counter Mode encryption with validation, create a <xref:Microsoft.AspNetCore.DataProtection.AuthenticatedEncryption.ConfigurationModel.CngGcmAuthenticatedEncryptorConfiguration> instance that contains the algorithmic information:

```csharp
services.AddDataProtection()
    .UseCustomCryptographicAlgorithms(
        new CngGcmAuthenticatedEncryptorConfiguration()
    {
        // Passed to BCryptOpenAlgorithmProvider
        EncryptionAlgorithm = "AES",
        EncryptionAlgorithmProvider = null,

        // Specified in bits
        EncryptionAlgorithmKeySize = 256
    });
```

> [!NOTE]
> The symmetric block cipher algorithm must have a key length of >= 128 bits, a block size of exactly 128 bits, and it must support GCM encryption. You can set the <xref:Microsoft.AspNetCore.DataProtection.AuthenticatedEncryption.ConfigurationModel.CngCbcAuthenticatedEncryptorConfiguration.EncryptionAlgorithmProvider> property to null to use the default provider for the specified algorithm. For more information, see the [BCryptOpenAlgorithmProvider](/windows/win32/api/bcrypt/nf-bcrypt-bcryptopenalgorithmprovider) documentation.

### Specifying other custom algorithms

Though not exposed as a first-class API, the Data Protection system is extensible enough to allow specifying almost any kind of algorithm. For example, it's possible to keep all keys contained within a Hardware Security Module (HSM) and to provide a custom implementation of the core encryption and decryption routines. For more information, see <xref:Microsoft.AspNetCore.DataProtection.AuthenticatedEncryption.IAuthenticatedEncryptor> in [Core cryptography extensibility](xref:security/data-protection/extensibility/core-crypto).

## Persisting keys when hosting in a Docker container

When hosting in a [Docker](/dotnet/standard/microservices-architecture/container-docker-introduction/) container, keys should be maintained in either:

* A folder that's a Docker volume that persists beyond the container's lifetime, such as a shared volume or a host-mounted volume.
* An external provider, such as [Azure Blob Storage](/azure/storage/blobs/storage-blobs-introduction) (shown in the [Protect keys with Azure Key Vault (`ProtectKeysWithAzureKeyVault`)](#protect-keys-with-azure-key-vault-protectkeyswithazurekeyvault) section) or [Redis](https://redis.io).

## Persisting keys with Redis

Only Redis versions supporting [Redis Data Persistence](/azure/azure-cache-for-redis/cache-how-to-premium-persistence) should be used to store keys. [Azure Blob storage](/azure/storage/blobs/storage-blobs-introduction) is persistent and can be used to store keys. For more information, see [this GitHub issue](https://github.com/dotnet/AspNetCore/issues/13476).

## Logging

Enable the `Information` or lower level of logging to diagnose problems. The following `appsettings.json` file enables information logging of the Data Protection API:

```json
{
  "Logging": {
    "LogLevel": {
      "Microsoft.AspNetCore.DataProtection": "Information"
    }
  }
}
```

For more information on logging, see <xref:fundamentals/logging/index>.

## Additional resources

* <xref:security/data-protection/configuration/non-di-scenarios>
* <xref:security/data-protection/configuration/machine-wide-policy>
* <xref:host-and-deploy/web-farm>
* <xref:security/data-protection/implementation/key-storage-providers>

:::moniker-end
