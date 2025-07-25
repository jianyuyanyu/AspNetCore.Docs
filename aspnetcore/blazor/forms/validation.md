---
title: ASP.NET Core Blazor forms validation
author: guardrex
description: Learn how to use validation in Blazor forms.
monikerRange: '>= aspnetcore-3.1'
ms.author: wpickett
ms.custom: mvc
ms.date: 11/12/2024
uid: blazor/forms/validation
---
# ASP.NET Core Blazor forms validation

[!INCLUDE[](~/includes/not-latest-version.md)]

This article explains how to use validation in Blazor forms.

## Form validation

In basic form validation scenarios, an <xref:Microsoft.AspNetCore.Components.Forms.EditForm> instance can use declared <xref:Microsoft.AspNetCore.Components.Forms.EditContext> and <xref:Microsoft.AspNetCore.Components.Forms.ValidationMessageStore> instances to validate form fields. A handler for the <xref:Microsoft.AspNetCore.Components.Forms.EditContext.OnValidationRequested> event of the <xref:Microsoft.AspNetCore.Components.Forms.EditContext> executes custom validation logic. The handler's result updates the <xref:Microsoft.AspNetCore.Components.Forms.ValidationMessageStore> instance.

Basic form validation is useful in cases where the form's model is defined within the component hosting the form, either as members directly on the component or in a subclass. Use of a [validator component](#validator-components) is recommended where an independent model class is used across several components.

:::moniker range=">= aspnetcore-8.0"

In Blazor Web Apps, client-side validation requires an active Blazor SignalR circuit. Client-side validation isn't available to forms in components that have adopted static server-side rendering (static SSR). Forms that adopt static SSR are validated on the server after the form is submitted.

:::moniker-end

In the following component, the `HandleValidationRequested` handler method clears any existing validation messages by calling <xref:Microsoft.AspNetCore.Components.Forms.ValidationMessageStore.Clear%2A?displayProperty=nameWithType> before validating the form.

`Starship8.razor`:

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/Components/Pages/Starship8.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/Starship8.razor":::

:::moniker-end

:::moniker range="< aspnetcore-8.0"

```razor
@page "/starship-8"
@implements IDisposable
@inject ILogger<Starship8> Logger

<h2>Holodeck Configuration</h2>

<EditForm EditContext="editContext" OnValidSubmit="Submit">
    <div>
        <label>
            <InputCheckbox @bind-Value="Model!.Subsystem1" />
            Safety Subsystem
        </label>
    </div>
    <div>
        <label>
            <InputCheckbox @bind-Value="Model!.Subsystem2" />
            Emergency Shutdown Subsystem
        </label>
    </div>
    <div>
        <ValidationMessage For="() => Model!.Options" />
    </div>
    <div>
        <button type="submit">Update</button>
    </div>
</EditForm>

@code {
    private EditContext? editContext;

    public Holodeck? Model { get; set; }

    private ValidationMessageStore? messageStore;

    protected override void OnInitialized()
    {
        Model ??= new();
        editContext = new(Model);
        editContext.OnValidationRequested += HandleValidationRequested;
        messageStore = new(editContext);
    }

    private void HandleValidationRequested(object? sender,
        ValidationRequestedEventArgs args)
    {
        messageStore?.Clear();

        // Custom validation logic
        if (!Model!.Options)
        {
            messageStore?.Add(() => Model.Options, "Select at least one.");
        }
    }

    private void Submit()
    {
        Logger.LogInformation("Submit called: Processing the form");
    }

    public class Holodeck
    {
        public bool Subsystem1 { get; set; }
        public bool Subsystem2 { get; set; }
        public bool Options => Subsystem1 || Subsystem2;
    }

    public void Dispose()
    {
        if (editContext is not null)
        {
            editContext.OnValidationRequested -= HandleValidationRequested;
        }
    }
}
```

<!--
:::code language="razor" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/Pages/forms-and-validation/Starship8.razor":::
-->

:::moniker-end

## Data Annotations Validator component and custom validation

The <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component attaches data annotations validation to a cascaded <xref:Microsoft.AspNetCore.Components.Forms.EditContext>. Enabling data annotations validation requires the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component. To use a different validation system than data annotations, use a custom implementation instead of the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component. The framework implementations for <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> are available for inspection in the reference source:

* [`DataAnnotationsValidator`](https://github.com/dotnet/AspNetCore/blob/main/src/Components/Forms/src/DataAnnotationsValidator.cs)
* [`EnableDataAnnotationsValidation`](https://github.com/dotnet/AspNetCore/blob/main/src/Components/Forms/src/EditContextDataAnnotationsExtensions.cs)

If you need to enable data annotations validation support for an <xref:Microsoft.AspNetCore.Components.Forms.EditContext> in code, call <xref:Microsoft.AspNetCore.Components.Forms.EditContextDataAnnotationsExtensions.EnableDataAnnotationsValidation%2A> with an injected <xref:System.IServiceProvider> (`@inject IServiceProvider ServiceProvider`) on the <xref:Microsoft.AspNetCore.Components.Forms.EditContext>. For an advanced example, see the [`NotifyPropertyChangedValidationComponent` component in the ASP.NET Core Blazor framework's `BasicTestApp` (`dotnet/aspnetcore` GitHub repository)](https://github.com/dotnet/aspnetcore/blob/main/src/Components/test/testassets/BasicTestApp/FormsTest/NotifyPropertyChangedValidationComponent.razor). In a production version of the example, replace the `new TestServiceProvider()` argument for the service provider with an injected <xref:System.IServiceProvider>.

[!INCLUDE[](~/includes/aspnetcore-repo-ref-source-links.md)]

Blazor performs two types of validation:

* *Field validation* is performed when the user tabs out of a field. During field validation, the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component associates all reported validation results with the field.
* *Model validation* is performed when the user submits the form. During model validation, the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component attempts to determine the field based on the member name that the validation result reports. Validation results that aren't associated with an individual member are associated with the model rather than a field.

In custom validation scenarios:

* Validation manages a <xref:Microsoft.AspNetCore.Components.Forms.ValidationMessageStore> for a form's <xref:Microsoft.AspNetCore.Components.Forms.EditContext>.
* The <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component is used to attach validation support to forms based on [validation attributes (data annotations)](xref:mvc/models/validation#validation-attributes).

There are two general approaches for achieving custom validation, which are described in the next two sections of this article:

* [Manual validation using the `OnValidationRequested` event](#manual-validation-using-the-onvalidationrequested-event): Manually validate a form's fields with data annotations validation and custom code for field checks when validation is requested via an event handler assigned to the <xref:Microsoft.AspNetCore.Components.Forms.EditContext.OnValidationRequested%2A> event.
* [Validator components](#validator-components): One or more custom validator components can be used to process validation for different forms on the same page or the same form at different steps of form processing (for example, client validation followed by server validation).

## Manual validation using the `OnValidationRequested` event

You can manually validate a form with a custom event handler assigned to the <xref:Microsoft.AspNetCore.Components.Forms.EditContext.OnValidationRequested%2A?displayProperty=nameWithType> event to manage a <xref:Microsoft.AspNetCore.Components.Forms.ValidationMessageStore>.

The Blazor framework provides the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component to attach additional validation support to forms based on [validation attributes (data annotations)](xref:mvc/models/validation#validation-attributes). 

Recalling the earlier `Starship8` component example, the `HandleValidationRequested` method is assigned to <xref:Microsoft.AspNetCore.Components.Forms.EditContext.OnValidationRequested%2A>, where you can perform manual validation in C# code. A few changes demonstrate combining the existing manual validation with data annotations validation via a <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> and a validation attribute applied to the `Holodeck` model.

Reference the <xref:System.ComponentModel.DataAnnotations?displayProperty=fullName> namespace in the component's Razor directives at the top of the component definition file:

```razor
@using System.ComponentModel.DataAnnotations
```

Add an `Id` property to the `Holodeck` model with a validation attribute to limit the string's length to six characters:

```csharp
[StringLength(6)]
public string? Id { get; set; }
```

Add a <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component (`<DataAnnotationsValidator />`) to the form. Typically, the component is placed immediately under the `<EditForm>` tag, but you can place it anywhere in the form:

```razor
<DataAnnotationsValidator />
```

Change the form's submit behavior in the `<EditForm>` tag from <xref:Microsoft.AspNetCore.Components.Forms.EditForm.OnSubmit> to <xref:Microsoft.AspNetCore.Components.Forms.EditForm.OnValidSubmit>, which ensures that the form is valid before executing the assigned event handler method:

```diff
- OnSubmit="Submit"
+ OnValidSubmit="Submit"
```

In the `<EditForm>`, add a field for the `Id` property:

```razor
<div>
    <label>
        <InputText @bind-Value="Model!.Id" />
        ID (6 characters max)
    </label>
    <ValidationMessage For="() => Model!.Id" />
</div>
```

After making the preceding changes, the form's behavior matches the following specification:

* The data annotations validation on the `Id` property doesn't trigger a validation failure when the `Id` field merely loses focus. The validation executes when the user selects the **`Update`** button.
* Any manual validation that you want to perform in the `HandleValidationRequested` method assigned to the form's <xref:Microsoft.AspNetCore.Components.Forms.EditContext.OnValidationRequested%2A> event executes when the user selects the form's **`Update`** button. In the existing code of the `Starship8` component example, the user must select either or both of the checkboxes to validate the form.
* The form doesn't process the `Submit` method until both the data annotations and manual validation pass.

## Validator components

Validator components support form validation by managing a <xref:Microsoft.AspNetCore.Components.Forms.ValidationMessageStore> for a form's <xref:Microsoft.AspNetCore.Components.Forms.EditContext>.

The Blazor framework provides the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component to attach validation support to forms based on [validation attributes (data annotations)](xref:mvc/models/validation#validation-attributes). You can create custom validator components to process validation messages for different forms on the same page or the same form at different steps of form processing (for example, client validation followed by server validation). The validator component example shown in this section, `CustomValidation`, is used in the following sections of this article:

* [Business logic validation with a validator component](#business-logic-validation-with-a-validator-component)
* [Server validation with a validator component](#server-validation-with-a-validator-component)

Of the [data annotation built-in validators](xref:mvc/models/validation#built-in-attributes), only the [`[Remote]` validation attribute](xref:mvc/models/validation#remote-attribute) isn't supported in Blazor.

> [!NOTE]
> Custom data annotation validation attributes can be used instead of custom validator components in many cases. Custom attributes applied to the form's model activate with the use of the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component. When used with server validation, any custom attributes applied to the model must be executable on the server. For more information, see the [Custom validation attributes](#custom-validation-attributes) section.

Create a validator component from <xref:Microsoft.AspNetCore.Components.ComponentBase>:

* The form's <xref:Microsoft.AspNetCore.Components.Forms.EditContext> is a [cascading parameter](xref:blazor/components/cascading-values-and-parameters) of the component.
* When the validator component is initialized, a new <xref:Microsoft.AspNetCore.Components.Forms.ValidationMessageStore> is created to maintain a current list of form errors.
* The message store receives errors when developer code in the form's component calls the `DisplayErrors` method. The errors are passed to the `DisplayErrors` method in a [`Dictionary<string, List<string>>`](xref:System.Collections.Generic.Dictionary%602). In the dictionary, the key is the name of the form field that has one or more errors. The value is the error list.
* Messages are cleared when any of the following have occurred:
  * Validation is requested on the <xref:Microsoft.AspNetCore.Components.Forms.EditContext> when the <xref:Microsoft.AspNetCore.Components.Forms.EditContext.OnValidationRequested> event is raised. All of the errors are cleared.
  * A field changes in the form when the <xref:Microsoft.AspNetCore.Components.Forms.EditContext.OnFieldChanged> event is raised. Only the errors for the field are cleared.
  * The `ClearErrors` method is called by developer code. All of the errors are cleared.

Update the namespace in the following class to match your app's namespace.

`CustomValidation.cs`:

:::code language="csharp" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/CustomValidation.cs":::

> [!IMPORTANT]
> Specifying a namespace is **required** when deriving from <xref:Microsoft.AspNetCore.Components.ComponentBase>. Failing to specify a namespace results in a build error:
>
> > :::no-loc text="Tag helpers cannot target tag name '\<global namespace>.{CLASS NAME}' because it contains a ' ' character.":::
>
> The `{CLASS NAME}` placeholder is the name of the component class. The custom validator example in this section specifies the example namespace `BlazorSample`.

> [!NOTE]
> Anonymous lambda expressions are registered event handlers for <xref:Microsoft.AspNetCore.Components.Forms.EditContext.OnValidationRequested> and <xref:Microsoft.AspNetCore.Components.Forms.EditContext.OnFieldChanged> in the preceding example. It isn't necessary to implement <xref:System.IDisposable> and unsubscribe the event delegates in this scenario. For more information, see <xref:blazor/components/component-disposal>.

## Business logic validation with a validator component

For general business logic validation, use a [validator component](#validator-components) that receives form errors in a dictionary.

Basic validation is useful in cases where the form's model is defined within the component hosting the form, either as members directly on the component or in a subclass. Use of a validator component is recommended where an independent model class is used across several components.

In the following example:

* A shortened version of the `Starfleet Starship Database` form (`Starship3` component) of the [Example form](xref:blazor/forms/input-components#example-form) section of the *Input components* article is used that only accepts the starship's classification and description. Data annotation validation isn't triggered on form submission because the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component isn't included in the form.
* The `CustomValidation` component from the [Validator components](#validator-components) section of this article is used.
* The validation requires a value for the ship's description (`Description`) if the user selects the "`Defense`" ship classification (`Classification`).

When validation messages are set in the component, they're added to the validator's <xref:Microsoft.AspNetCore.Components.Forms.ValidationMessageStore> and shown in the <xref:Microsoft.AspNetCore.Components.Forms.EditForm>'s validation summary.

`Starship9.razor`:

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/Components/Pages/Starship9.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/Starship9.razor":::

:::moniker-end

:::moniker range="< aspnetcore-8.0"

```razor
@page "/starship-9"
@inject ILogger<Starship9> Logger

<h1>Starfleet Starship Database</h1>

<h2>New Ship Entry Form</h2>

<EditForm Model="Model" OnValidSubmit="Submit">
    <CustomValidation @ref="customValidation" />
    <ValidationSummary />
    <div>
        <label>
            Primary Classification:
            <InputSelect @bind-Value="Model!.Classification">
                <option value="">Select classification ...</option>
                <option value="Exploration">Exploration</option>
                <option value="Diplomacy">Diplomacy</option>
                <option value="Defense">Defense</option>
            </InputSelect>
        </label>
    </div>
    <div>
        <label>
            Description (optional):
            <InputTextArea @bind-Value="Model!.Description" />
        </label>
    </div>
    <div>
        <button type="submit">Submit</button>
    </div>
</EditForm>

@code {
    private CustomValidation? customValidation;

    public Starship? Model { get; set; }

    protected override void OnInitialized() =>
        Model ??= new() { ProductionDate = DateTime.UtcNow };

    private void Submit()
    {
        customValidation?.ClearErrors();

        var errors = new Dictionary<string, List<string>>();

        if (Model!.Classification == "Defense" &&
                string.IsNullOrEmpty(Model.Description))
        {
            errors.Add(nameof(Model.Description),
                new() { "For a 'Defense' ship classification, " +
                "'Description' is required." });
        }

        if (errors.Any())
        {
            customValidation?.DisplayErrors(errors);
        }
        else
        {
            Logger.LogInformation("Submit called: Processing the form");
        }
    }
}
```

<!--
:::code language="razor" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/Pages/forms-and-validation/Starship9.razor":::
-->

:::moniker-end

> [!NOTE]
> As an alternative to using [validation components](#validator-components), data annotation validation attributes can be used. Custom attributes applied to the form's model activate with the use of the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component. When used with server validation, the attributes must be executable on the server. For more information, see the [Custom validation attributes](#custom-validation-attributes) section.

## Server validation with a validator component

:::moniker range=">= aspnetcore-8.0"

*This section is focused on Blazor Web App scenarios, but the approach for any type of app that uses server validation with web API adopts the same general approach.*

:::moniker-end

:::moniker range="< aspnetcore-8.0"

*This section is focused on hosted Blazor WebAssembly scenarios, but the approach for any type of app that uses server validation with web API adopts the same general approach.*

:::moniker-end

Server validation is supported in addition to client validation:

* Process client validation in the form with the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component.
* When the form passes client validation (<xref:Microsoft.AspNetCore.Components.Forms.EditForm.OnValidSubmit> is called), send the <xref:Microsoft.AspNetCore.Components.Forms.EditContext.Model?displayProperty=nameWithType> to a backend server API for form processing.
* Process model validation on the server.
* The server API includes both the built-in framework data annotations validation and custom validation logic supplied by the developer. If validation passes on the server, process the form and send back a success status code ([`200 - OK`](https://developer.mozilla.org/docs/Web/HTTP/Status/200)). If validation fails, return a failure status code ([`400 - Bad Request`](https://developer.mozilla.org/docs/Web/HTTP/Status/400)) and the field validation errors.
* Either disable the form on success or display the errors.

Basic validation is useful in cases where the form's model is defined within the component hosting the form, either as members directly on the component or in a subclass. Use of a validator component is recommended where an independent model class is used across several components.

The following example is based on:

:::moniker range=">= aspnetcore-8.0"

* A Blazor Web App with Interactive WebAssembly components created from the [Blazor Web App project template](xref:blazor/project-structure).
* The `Starship` model  (`Starship.cs`) of the [Example form](xref:blazor/forms/input-components#example-form) section of the *Input components* article.
* The `CustomValidation` component shown in the [Validator components](#validator-components) section.

Place the `Starship` model (`Starship.cs`) into a shared class library project so that both the client and server projects can use the model. Add or update the namespace to match the namespace of the shared app (for example, `namespace BlazorSample.Shared`). Since the model requires data annotations, confirm that the shared class library uses the shared framework or add the [`System.ComponentModel.Annotations`](https://www.nuget.org/packages/System.ComponentModel.Annotations) package to the shared project.

[!INCLUDE[](~/includes/package-reference.md)]

In the main project of the Blazor Web App, add a controller to process starship validation requests and return failed validation messages. Update the namespaces in the last `using` statement for the shared class library project and the `namespace` for the controller class. In addition to client and server data annotations validation, the controller validates that a value is provided for the ship's description (`Description`) if the user selects the `Defense` ship classification (`Classification`).

:::moniker-end

:::moniker range="< aspnetcore-8.0"

* A hosted Blazor WebAssembly [solution](xref:blazor/tooling#visual-studio-solution-file-sln) created from the [Blazor WebAssembly project template](xref:blazor/project-structure). The approach is supported for any of the secure hosted Blazor solutions described in the [hosted Blazor WebAssembly security documentation](xref:blazor/security/webassembly/index#implementation-guidance).
* The `Starship` model  (`Starship.cs`) of the [Example form](xref:blazor/forms/input-components#example-form) section of the *Input components* article.
* The `CustomValidation` component shown in the [Validator components](#validator-components) section.

Place the `Starship` model (`Starship.cs`) into the solution's **`Shared`** project so that both the client and server apps can use the model. Add or update the namespace to match the namespace of the shared app (for example, `namespace BlazorSample.Shared`). Since the model requires data annotations, add the [`System.ComponentModel.Annotations`](https://www.nuget.org/packages/System.ComponentModel.Annotations) package to the **`Shared`** project.

[!INCLUDE[](~/includes/package-reference.md)]

In the **:::no-loc text="Server":::** project, add a controller to process starship validation requests and return failed validation messages. Update the namespaces in the last `using` statement for the **`Shared`** project and the `namespace` for the controller class. In addition to client and server data annotations validation, the controller validates that a value is provided for the ship's description (`Description`) if the user selects the `Defense` ship classification (`Classification`).

:::moniker-end

The validation for the `Defense` ship classification only occurs on the server in the controller because the upcoming form doesn't perform the same validation client-side when the form is submitted to the server. Server validation without client validation is common in apps that require private business logic validation of user input on the server. For example, private information from data stored for a user might be required to validate user input. Private data obviously can't be sent to the client for client validation.

> [!NOTE]
> The `StarshipValidation` controller in this section uses Microsoft Identity 2.0. The Web API only accepts tokens for users that have the "`API.Access`" scope for this API. Additional customization is required if the API's scope name is different from `API.Access`.
>
> For more information on security, see:
>
> * <xref:blazor/security/index> (and the other articles in the Blazor *Security and Identity* node)
> * [Microsoft identity platform documentation](/entra/identity-platform/)

`Controllers/StarshipValidation.cs`:

:::moniker range=">= aspnetcore-8.0"

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using BlazorSample.Shared;

namespace BlazorSample.Server.Controllers;

[Authorize]
[ApiController]
[Route("[controller]")]
public class StarshipValidationController(
    ILogger<StarshipValidationController> logger) 
    : ControllerBase
{
    static readonly string[] scopeRequiredByApi = [ "API.Access" ];

    [HttpPost]
    public async Task<IActionResult> Post(Starship model)
    {
        HttpContext.VerifyUserHasAnyAcceptedScope(scopeRequiredByApi);

        try
        {
            if (model.Classification == "Defense" && 
                string.IsNullOrEmpty(model.Description))
            {
                ModelState.AddModelError(nameof(model.Description),
                    "For a 'Defense' ship " +
                    "classification, 'Description' is required.");
            }
            else
            {
                logger.LogInformation("Processing the form asynchronously");

                // async ...

                return Ok(ModelState);
            }
        }
        catch (Exception ex)
        {
            logger.LogError("Validation Error: {Message}", ex.Message);
        }

        return BadRequest(ModelState);
    }
}
```

:::moniker-end

:::moniker range="< aspnetcore-8.0"

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using BlazorSample.Shared;

namespace BlazorSample.Server.Controllers;

[Authorize]
[ApiController]
[Route("[controller]")]
public class StarshipValidationController(
    ILogger<StarshipValidationController> logger) 
    : ControllerBase
{
    static readonly string[] scopeRequiredByApi = new[] { "API.Access" };

    [HttpPost]
    public async Task<IActionResult> Post(Starship model)
    {
        HttpContext.VerifyUserHasAnyAcceptedScope(scopeRequiredByApi);

        try
        {
            if (model.Classification == "Defense" && 
                string.IsNullOrEmpty(model.Description))
            {
                ModelState.AddModelError(nameof(model.Description),
                    "For a 'Defense' ship " +
                    "classification, 'Description' is required.");
            }
            else
            {
                logger.LogInformation("Processing the form asynchronously");

                // async ...

                return Ok(ModelState);
            }
        }
        catch (Exception ex)
        {
            logger.LogError("Validation Error: {Message}", ex.Message);
        }

        return BadRequest(ModelState);
    }
}
```

:::moniker-end

Confirm or update the namespace of the preceding controller (`BlazorSample.Server.Controllers`) to match the app's controllers' namespace.

When a model binding validation error occurs on the server, an [`ApiController`](xref:web-api/index) (<xref:Microsoft.AspNetCore.Mvc.ApiControllerAttribute>) normally returns a [default bad request response](xref:web-api/index#default-badrequest-response) with a <xref:Microsoft.AspNetCore.Mvc.ValidationProblemDetails>. The response contains more data than just the validation errors, as shown in the following example when all of the fields of the `Starfleet Starship Database` form aren't submitted and the form fails validation:

```json
{
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Id": [ "The Id field is required." ],
    "Classification": [ "The Classification field is required." ],
    "IsValidatedDesign": [ "This form disallows unapproved ships." ],
    "MaximumAccommodation": [ "Accommodation invalid (1-100000)." ]
  }
}
```

> [!NOTE]
> To demonstrate the preceding JSON response, you must either disable the form's client validation to permit empty field form submission or use a tool to send a request directly to the server API, such as [Firefox Browser Developer](https://www.mozilla.org/firefox/developer/).

If the server API returns the preceding default JSON response, it's possible for the client to parse the response in developer code to obtain the children of the `errors` node for forms validation error processing. It's inconvenient to write developer code to parse the file. Parsing the JSON manually requires producing a [`Dictionary<string, List<string>>`](xref:System.Collections.Generic.Dictionary%602) of errors after calling <xref:System.Net.Http.Json.HttpContentJsonExtensions.ReadFromJsonAsync%2A>. Ideally, the server API should only return the validation errors, as the following example shows:

```json
{
  "Id": [ "The Id field is required." ],
  "Classification": [ "The Classification field is required." ],
  "IsValidatedDesign": [ "This form disallows unapproved ships." ],
  "MaximumAccommodation": [ "Accommodation invalid (1-100000)." ]
}
```

To modify the server API's response to make it only return the validation errors, change the delegate that's invoked on actions that are annotated with <xref:Microsoft.AspNetCore.Mvc.ApiControllerAttribute> in the `Program` file. For the API endpoint (`/StarshipValidation`), return a <xref:Microsoft.AspNetCore.Mvc.BadRequestObjectResult> with the <xref:Microsoft.AspNetCore.Mvc.ModelBinding.ModelStateDictionary>. For any other API endpoints, preserve the default behavior by returning the object result with a new <xref:Microsoft.AspNetCore.Mvc.ValidationProblemDetails>.

:::moniker range=">= aspnetcore-8.0"

Add the <xref:Microsoft.AspNetCore.Mvc?displayProperty=fullName> namespace to the top of the `Program` file in the main project of the Blazor Web App:

```csharp
using Microsoft.AspNetCore.Mvc;
```

In the `Program` file, add or update the following <xref:Microsoft.Extensions.DependencyInjection.MvcServiceCollectionExtensions.AddControllersWithViews%2A> extension method and add the following call to <xref:Microsoft.Extensions.DependencyInjection.MvcCoreMvcBuilderExtensions.ConfigureApiBehaviorOptions%2A>:

```csharp
builder.Services.AddControllersWithViews()
    .ConfigureApiBehaviorOptions(options =>
    {
        options.InvalidModelStateResponseFactory = context =>
        {
            if (context.HttpContext.Request.Path == "/StarshipValidation")
            {
                return new BadRequestObjectResult(context.ModelState);
            }
            else
            {
                return new BadRequestObjectResult(
                    new ValidationProblemDetails(context.ModelState));
            }
        };
    });
```

If you're adding controllers to the main project of the Blazor Web App for the first time, map controller endpoints when you place the preceding code that registers services for controllers. The following example uses default controller routes:

```csharp
app.MapDefaultControllerRoute();
```

> [!NOTE]
> The preceding example explicitly registers controller services by calling <xref:Microsoft.Extensions.DependencyInjection.MvcServiceCollectionExtensions.AddControllersWithViews%2A> to automatically [mitigate Cross-Site Request Forgery (XSRF/CSRF) attacks](xref:security/anti-request-forgery). If you merely use <xref:Microsoft.Extensions.DependencyInjection.MvcServiceCollectionExtensions.AddControllers%2A>, antiforgery isn't enabled automatically.

For more information on controller routing and validation failure error responses, see the following resources:

* <xref:mvc/controllers/routing>
* <xref:web-api/handle-errors#validation-failure-error-response>

In the `.Client` project, add the `CustomValidation` component shown in the [Validator components](#validator-components) section. Update the namespace to match the app (for example, `namespace BlazorSample.Client`).

In the `.Client` project, the `Starfleet Starship Database` form is updated to show server validation errors with help of the `CustomValidation` component. When the server API returns validation messages, they're added to the `CustomValidation` component's <xref:Microsoft.AspNetCore.Components.Forms.ValidationMessageStore>. The errors are available in the form's <xref:Microsoft.AspNetCore.Components.Forms.EditContext> for display by the form's validation summary.

In the following component, update the namespace of the shared project (`@using BlazorSample.Shared`) to the shared project's namespace. Note that the form requires authorization, so the user must be signed into the app to navigate to the form.

:::moniker-end

:::moniker range="< aspnetcore-8.0"

Add the <xref:Microsoft.AspNetCore.Mvc?displayProperty=fullName> namespace to the top of the `Program` file in the **:::no-loc text="Server":::** app:

```csharp
using Microsoft.AspNetCore.Mvc;
```

In the `Program` file, locate the <xref:Microsoft.Extensions.DependencyInjection.MvcServiceCollectionExtensions.AddControllersWithViews%2A> extension method and add the following call to <xref:Microsoft.Extensions.DependencyInjection.MvcCoreMvcBuilderExtensions.ConfigureApiBehaviorOptions%2A>:

```csharp
builder.Services.AddControllersWithViews()
    .ConfigureApiBehaviorOptions(options =>
    {
        options.InvalidModelStateResponseFactory = context =>
        {
            if (context.HttpContext.Request.Path == "/StarshipValidation")
            {
                return new BadRequestObjectResult(context.ModelState);
            }
            else
            {
                return new BadRequestObjectResult(
                    new ValidationProblemDetails(context.ModelState));
            }
        };
    });
```

> [!NOTE]
> The preceding example explicitly registers controller services by calling <xref:Microsoft.Extensions.DependencyInjection.MvcServiceCollectionExtensions.AddControllersWithViews%2A> to automatically [mitigate Cross-Site Request Forgery (XSRF/CSRF) attacks](xref:security/anti-request-forgery). If you merely use <xref:Microsoft.Extensions.DependencyInjection.MvcServiceCollectionExtensions.AddControllers%2A>, antiforgery isn't enabled automatically.

In the **:::no-loc text="Client":::** project, add the `CustomValidation` component shown in the [Validator components](#validator-components) section. Update the namespace to match the app (for example, `namespace BlazorSample.Client`).

In the **:::no-loc text="Client":::** project, the `Starfleet Starship Database` form is updated to show server validation errors with help of the `CustomValidation` component. When the server API returns validation messages, they're added to the `CustomValidation` component's <xref:Microsoft.AspNetCore.Components.Forms.ValidationMessageStore>. The errors are available in the form's <xref:Microsoft.AspNetCore.Components.Forms.EditContext> for display by the form's validation summary.

In the following component, update the namespace of the **`Shared`** project (`@using BlazorSample.Shared`) to the shared project's namespace. Note that the form requires authorization, so the user must be signed into the app to navigate to the form.

:::moniker-end

`Starship10.razor`:

:::moniker range=">= aspnetcore-8.0"

> [!NOTE]
> Forms based on <xref:Microsoft.AspNetCore.Components.Forms.EditForm> automatically enable [antiforgery support](xref:blazor/forms/index#antiforgery-support). The controller should use <xref:Microsoft.Extensions.DependencyInjection.MvcServiceCollectionExtensions.AddControllersWithViews%2A> to register controller services and automatically enable antiforgery support for the web API.

```razor
@page "/starship-10"
@rendermode InteractiveWebAssembly
@using System.Net
@using System.Net.Http.Json
@using Microsoft.AspNetCore.Authorization
@using Microsoft.AspNetCore.Components.WebAssembly.Authentication
@using BlazorSample.Shared
@attribute [Authorize]
@inject HttpClient Http
@inject ILogger<Starship10> Logger

<h1>Starfleet Starship Database</h1>

<h2>New Ship Entry Form</h2>

<EditForm FormName="Starship10" Model="Model" OnValidSubmit="Submit">
    <DataAnnotationsValidator />
    <CustomValidation @ref="customValidation" />
    <ValidationSummary />
    <div>
        <label>
            Identifier: 
            <InputText @bind-Value="Model!.Id" disabled="@disabled" />
        </label>
    </div>
    <div>
        <label>
            Description (optional):
            <InputTextArea @bind-Value="Model!.Description" 
                disabled="@disabled" />
        </label>
    </div>
    <div>
        <label>
            Primary Classification:
            <InputSelect @bind-Value="Model!.Classification" disabled="@disabled">
                <option value="">Select classification ...</option>
                <option value="Exploration">Exploration</option>
                <option value="Diplomacy">Diplomacy</option>
                <option value="Defense">Defense</option>
            </InputSelect>
        </label>
    </div>
    <div>
        <label>
            Maximum Accommodation:
            <InputNumber @bind-Value="Model!.MaximumAccommodation" 
                disabled="@disabled" />
        </label>
    </div>
    <div>
        <label>
            Engineering Approval:
            <InputCheckbox @bind-Value="Model!.IsValidatedDesign" 
                disabled="@disabled" />
        </label>
    </div>
    <div>
        <label>
            Production Date:
            <InputDate @bind-Value="Model!.ProductionDate" disabled="@disabled" />
        </label>
    </div>
    <div>
        <button type="submit" disabled="@disabled">Submit</button>
    </div>
    <div style="@messageStyles">
        @message
    </div>
</EditForm>

@code {
    private CustomValidation? customValidation;
    private bool disabled;
    private string? message;
    private string messageStyles = "visibility:hidden";

    [SupplyParameterFromForm]
    private Starship? Model { get; set; }

    protected override void OnInitialized() => 
        Model ??= new() { ProductionDate = DateTime.UtcNow };

    private async Task Submit(EditContext editContext)
    {
        customValidation?.ClearErrors();

        try
        {
            using var response = await Http.PostAsJsonAsync<Starship>(
                "StarshipValidation", (Starship)editContext.Model);

            var errors = await response.Content
                .ReadFromJsonAsync<Dictionary<string, List<string>>>() ?? 
                new Dictionary<string, List<string>>();

            if (response.StatusCode == HttpStatusCode.BadRequest && 
                errors.Any())
            {
                customValidation?.DisplayErrors(errors);
            }
            else if (!response.IsSuccessStatusCode)
            {
                throw new HttpRequestException(
                    $"Validation failed. Status Code: {response.StatusCode}");
            }
            else
            {
                disabled = true;
                messageStyles = "color:green";
                message = "The form has been processed.";
            }
        }
        catch (AccessTokenNotAvailableException ex)
        {
            ex.Redirect();
        }
        catch (Exception ex)
        {
            Logger.LogError("Form processing error: {Message}", ex.Message);
            disabled = true;
            messageStyles = "color:red";
            message = "There was an error processing the form.";
        }
    }
}
```

The `.Client` project of a Blazor Web App must also register an <xref:System.Net.Http.HttpClient> for HTTP POST requests to a backend web API controller. Confirm or add the following to the `.Client` project's `Program` file:

```csharp
builder.Services.AddScoped(sp => 
    new HttpClient { BaseAddress = new Uri(builder.HostEnvironment.BaseAddress) });
```

The preceding example sets the base address with `builder.HostEnvironment.BaseAddress` (<xref:Microsoft.AspNetCore.Components.WebAssembly.Hosting.IWebAssemblyHostEnvironment.BaseAddress%2A?displayProperty=nameWithType>), which gets the base address for the app and is typically derived from the `<base>` tag's `href` value in the host page.

<!--
:::code language="razor" source="~/../blazor-samples/8.0/BlazorWebAppSample/Components/Pages/Starship10.razor":::
-->

:::moniker-end

:::moniker range="< aspnetcore-8.0"

```razor
@page "/starship-10"
@using System.Net
@using System.Net.Http.Json
@using Microsoft.AspNetCore.Authorization
@using Microsoft.AspNetCore.Components.WebAssembly.Authentication
@using BlazorSample.Shared
@attribute [Authorize]
@inject HttpClient Http
@inject ILogger<Starship10> Logger

<h1>Starfleet Starship Database</h1>

<h2>New Ship Entry Form</h2>

<EditForm Model="Model" OnValidSubmit="Submit">
    <DataAnnotationsValidator />
    <CustomValidation @ref="customValidation" />
    <ValidationSummary />
    <div>
        <label>
            Identifier: 
            <InputText @bind-Value="Model!.Id" disabled="@disabled" />
        </label>
    </div>
    <div>
        <label>
            Description (optional):
            <InputTextArea @bind-Value="Model!.Description" 
                disabled="@disabled" />
        </label>
    </div>
    <div>
        <label>
            Primary Classification:
            <InputSelect @bind-Value="Model!.Classification" disabled="@disabled">
                <option value="">Select classification ...</option>
                <option value="Exploration">Exploration</option>
                <option value="Diplomacy">Diplomacy</option>
                <option value="Defense">Defense</option>
            </InputSelect>
        </label>
    </div>
    <div>
        <label>
            Maximum Accommodation:
            <InputNumber @bind-Value="Model!.MaximumAccommodation" 
                disabled="@disabled" />
        </label>
    </div>
    <div>
        <label>
            Engineering Approval:
            <InputCheckbox @bind-Value="Model!.IsValidatedDesign" 
                disabled="@disabled" />
        </label>
    </div>
    <div>
        <label>
            Production Date:
            <InputDate @bind-Value="Model!.ProductionDate" disabled="@disabled" />
        </label>
    </div>
    <div>
        <button type="submit" disabled="@disabled">Submit</button>
    </div>
    <div style="@messageStyles">
        @message
    </div>
</EditForm>

@code {
    private CustomValidation? customValidation;
    private bool disabled;
    private string? message;
    private string messageStyles = "visibility:hidden";
    
    public Starship? Model { get; set; }

    protected override void OnInitialized() => 
        Model ??= new() { ProductionDate = DateTime.UtcNow };

    private async Task Submit(EditContext editContext)
    {
        customValidation?.ClearErrors();

        try
        {
            using var response = await Http.PostAsJsonAsync<Starship>(
                "StarshipValidation", (Starship)editContext.Model);

            var errors = await response.Content
                .ReadFromJsonAsync<Dictionary<string, List<string>>>() ?? 
                new Dictionary<string, List<string>>();

            if (response.StatusCode == HttpStatusCode.BadRequest && 
                errors.Any())
            {
                customValidation?.DisplayErrors(errors);
            }
            else if (!response.IsSuccessStatusCode)
            {
                throw new HttpRequestException(
                    $"Validation failed. Status Code: {response.StatusCode}");
            }
            else
            {
                disabled = true;
                messageStyles = "color:green";
                message = "The form has been processed.";
            }
        }
        catch (AccessTokenNotAvailableException ex)
        {
            ex.Redirect();
        }
        catch (Exception ex)
        {
            Logger.LogError("Form processing error: {Message}", ex.Message);
            disabled = true;
            messageStyles = "color:red";
            message = "There was an error processing the form.";
        }
    }
}
```

:::moniker-end

<!--
:::code language="razor" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/Pages/forms-and-validation/Starship10.razor":::
-->

> [!NOTE]
> As an alternative to the use of a [validation component](#validator-components), data annotation validation attributes can be used. Custom attributes applied to the form's model activate with the use of the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component. When used with server validation, the attributes must be executable on the server. For more information, see the [Custom validation attributes](#custom-validation-attributes) section.

:::moniker range="< aspnetcore-8.0"

> [!NOTE]
> The server validation approach in this section is suitable for any of the hosted Blazor WebAssembly solution examples in this documentation set:
>
> * [Microsoft Entra ID (ME-ID)](xref:blazor/security/webassembly/hosted-with-microsoft-entra-id)
> * [Azure Active Directory (AAD) B2C](xref:blazor/security/webassembly/hosted-with-azure-active-directory-b2c)
> * [Identity Server](xref:blazor/security/webassembly/hosted-with-identity-server)

:::moniker-end

## `InputText` based on the input event

Use the <xref:Microsoft.AspNetCore.Components.Forms.InputText> component to create a custom component that uses the `oninput` event ([`input`](https://developer.mozilla.org/docs/Web/API/HTMLElement/input_event)) instead of the `onchange` event ([`change`](https://developer.mozilla.org/docs/Web/API/HTMLElement/change_event)). Use of the `input` event triggers field validation on each keystroke.

The following `CustomInputText` component inherits the framework's `InputText` component and sets event binding to the `oninput` event ([`input`](https://developer.mozilla.org/docs/Web/API/HTMLElement/input_event)).

`CustomInputText.razor`:

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/CustomInputText.razor":::

The `CustomInputText` component can be used anywhere <xref:Microsoft.AspNetCore.Components.Forms.InputText> is used. The following  component uses the shared `CustomInputText` component.

`Starship11.razor`:

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/Components/Pages/Starship11.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/Starship11.razor":::

:::moniker-end

:::moniker range="< aspnetcore-8.0"

```razor
@page "/starship-11"
@using System.ComponentModel.DataAnnotations
@inject ILogger<Starship11> Logger

<EditForm Model="Model" OnValidSubmit="Submit">
    <DataAnnotationsValidator />
    <ValidationSummary />
    <CustomInputText @bind-Value="Model!.Id" />
    <button type="submit">Submit</button>
</EditForm>

<div>
    CurrentValue: @Model?.Id
</div>

@code {
    public Starship? Model { get; set; }

    protected override void OnInitialized() => Model ??= new();

    private void Submit()
    {
        Logger.LogInformation("Submit called: Processing the form");
    }

    public class Starship
    {
        [Required]
        [StringLength(10, ErrorMessage = "Id is too long.")]
        public string? Id { get; set; }
    }
}
```

<!--
:::code language="razor" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/Pages/forms-and-validation/Starship11.razor":::
-->

:::moniker-end

## Validation Summary and Validation Message components

The <xref:Microsoft.AspNetCore.Components.Forms.ValidationSummary> component summarizes all validation messages, which is similar to the [Validation Summary Tag Helper](xref:mvc/views/working-with-forms#the-validation-summary-tag-helper):

```razor
<ValidationSummary />
```

Output validation messages for a specific model with the `Model` parameter:
  
```razor
<ValidationSummary Model="Model" />
```

The <xref:Microsoft.AspNetCore.Components.Forms.ValidationMessage%601> component displays validation messages for a specific field, which is similar to the [Validation Message Tag Helper](xref:mvc/views/working-with-forms#the-validation-message-tag-helper). Specify the field for validation with the <xref:Microsoft.AspNetCore.Components.Forms.ValidationMessage%601.For%2A> attribute and a lambda expression naming the model property:

```razor
<ValidationMessage For="@(() => Model!.MaximumAccommodation)" />
```

The <xref:Microsoft.AspNetCore.Components.Forms.ValidationMessage%601> and <xref:Microsoft.AspNetCore.Components.Forms.ValidationSummary> components support arbitrary attributes. Any attribute that doesn't match a component parameter is added to the generated `<div>` or `<ul>` element.

Control the style of validation messages in the app's stylesheet (`wwwroot/css/app.css` or `wwwroot/css/site.css`). The default `validation-message` class sets the text color of validation messages to red:

```css
.validation-message {
    color: red;
}
```

:::moniker range=">= aspnetcore-8.0"

## Determine if a form field is valid

Use <xref:Microsoft.AspNetCore.Components.Forms.EditContext.IsValid%2A?displayProperty=nameWithType> to determine if a field is valid without obtaining validation messages.

<span aria-hidden="true">❌</span> Supported, but not recommended:

```csharp
var isValid = !editContext.GetValidationMessages(fieldIdentifier).Any();
```

<span aria-hidden="true">✔️</span> Recommended:

```csharp
var isValid = editContext.IsValid(fieldIdentifier);
```

:::moniker-end

## Custom validation attributes

To ensure that a validation result is correctly associated with a field when using a [custom validation attribute](xref:mvc/models/validation#custom-attributes), pass the validation context's <xref:System.ComponentModel.DataAnnotations.ValidationContext.MemberName> when creating the <xref:System.ComponentModel.DataAnnotations.ValidationResult>.

`CustomValidator.cs`:

:::moniker range=">= aspnetcore-8.0"

```csharp
using System;
using System.ComponentModel.DataAnnotations;

public class CustomValidator : ValidationAttribute
{
    protected override ValidationResult IsValid(object? value, 
        ValidationContext validationContext)
    {
        ...

        return new ValidationResult("Validation message to user.",
            [ validationContext.MemberName! ]);
    }
}
```

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-8.0"

```csharp
using System;
using System.ComponentModel.DataAnnotations;

public class CustomValidator : ValidationAttribute
{
    protected override ValidationResult IsValid(object? value, 
        ValidationContext validationContext)
    {
        ...

        return new ValidationResult("Validation message to user.",
            new[] { validationContext.MemberName! });
    }
}
```

:::moniker-end

:::moniker range="< aspnetcore-6.0"

```csharp
using System;
using System.ComponentModel.DataAnnotations;

public class CustomValidator : ValidationAttribute
{
    protected override ValidationResult IsValid(object value, 
        ValidationContext validationContext)
    {
        ...

        return new ValidationResult("Validation message to user.",
            new[] { validationContext.MemberName });
    }
}
```

:::moniker-end

Inject services into custom validation attributes through the <xref:System.ComponentModel.DataAnnotations.ValidationContext>. The following example demonstrates a salad chef form that validates user input with dependency injection (DI).

The `SaladChef` class indicates the approved starship ingredient list for a Ten Forward salad.

`SaladChef.cs`:

:::code language="csharp" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/SaladChef.cs":::

Register `SaladChef` in the app's DI container in the `Program` file:

```csharp
builder.Services.AddTransient<SaladChef>();
```

The `IsValid` method of the following `SaladChefValidatorAttribute` class obtains the `SaladChef` service from DI to check the user's input.

`SaladChefValidatorAttribute.cs`:

:::code language="csharp" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/SaladChefValidatorAttribute.cs":::

The following component validates user input by applying the `SaladChefValidatorAttribute` (`[SaladChefValidator]`) to the salad ingredient string (`SaladIngredient`).

`Starship12.razor`:

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/Components/Pages/Starship12.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/Starship12.razor":::

:::moniker-end

:::moniker range="< aspnetcore-8.0"

```razor
@page "/starship-12"
@inject SaladChef SaladChef

<EditForm Model="this" autocomplete="off">
    <DataAnnotationsValidator />
    <p>
        <label>
            Salad topper (@saladToppers):
            <input @bind="SaladIngredient" />
        </label>
    </p>
    <button type="submit">Submit</button>
    <ul>
        @foreach (var message in context.GetValidationMessages())
        {
            <li class="validation-message">@message</li>
        }
    </ul>
</EditForm>

@code {
    private string? saladToppers;

    [SaladChefValidator]
    public string? SaladIngredient { get; set; }

    protected override void OnInitialized() => 
        saladToppers ??= string.Join(", ", SaladChef.SaladToppers);
}
```

:::moniker-end

:::moniker range=">= aspnetcore-7.0"

## Custom validation CSS class attributes

Custom validation CSS class attributes are useful when integrating with CSS frameworks, such as [Bootstrap](https://getbootstrap.com/).

To specify custom validation CSS class attributes, start by providing CSS styles for custom validation. In the following example, valid (`validField`) and invalid (`invalidField`) styles are specified.

Add the following CSS classes to the app's stylesheet:

```css
.validField {
    border-color: lawngreen;
}

.invalidField {
    background-color: tomato;
}
```

Create a class derived from <xref:Microsoft.AspNetCore.Components.Forms.FieldCssClassProvider> that checks for field validation messages and applies the appropriate valid or invalid style.

`CustomFieldClassProvider.cs`:

:::moniker-end

:::moniker range=">= aspnetcore-8.0"

```csharp
using Microsoft.AspNetCore.Components.Forms;

public class CustomFieldClassProvider : FieldCssClassProvider
{
    public override string GetFieldCssClass(EditContext editContext, 
        in FieldIdentifier fieldIdentifier)
    {
        var isValid = editContext.IsValid(fieldIdentifier);

        return isValid ? "validField" : "invalidField";
    }
}
```

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

```csharp
using Microsoft.AspNetCore.Components.Forms;

public class CustomFieldClassProvider : FieldCssClassProvider
{
    public override string GetFieldCssClass(EditContext editContext, 
        in FieldIdentifier fieldIdentifier)
    {
        var isValid = !editContext.GetValidationMessages(fieldIdentifier).Any();

        return isValid ? "validField" : "invalidField";
    }
}
```

:::moniker-end

:::moniker range=">= aspnetcore-7.0"

<!--
:::code language="csharp" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/CustomFieldClassProvider.cs":::
-->

Set the `CustomFieldClassProvider` class as the Field CSS Class Provider on the form's <xref:Microsoft.AspNetCore.Components.Forms.EditContext> instance with <xref:Microsoft.AspNetCore.Components.Forms.EditContextFieldClassExtensions.SetFieldCssClassProvider%2A>.

`Starship13.razor`:

:::moniker-end

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/Components/Pages/Starship13.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/Starship13.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

```razor
@page "/starship-13"
@using System.ComponentModel.DataAnnotations
@inject ILogger<Starship13> Logger

<EditForm EditContext="editContext" OnValidSubmit="Submit">
    <DataAnnotationsValidator />
    <ValidationSummary />
    <InputText @bind-Value="Model!.Id" />
    <button type="submit">Submit</button>
</EditForm>

@code {
    private EditContext? editContext;

    public Starship? Model { get; set; }

    protected override void OnInitialized()
    {
        Model ??= new();
        editContext = new(Model);
        editContext.SetFieldCssClassProvider(new CustomFieldClassProvider());
    }

    private void Submit()
    {
        Logger.LogInformation("Submit called: Processing the form");
    }

    public class Starship
    {
        [Required]
        [StringLength(10, ErrorMessage = "Id is too long.")]
        public string? Id { get; set; }
    }
}
```

<!--
:::code language="razor" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/Pages/forms-and-validation/Starship13.razor":::
-->

:::moniker-end

:::moniker range=">= aspnetcore-7.0"

The preceding example checks the validity of all form fields and applies a style to each field. If the form should only apply custom styles to a subset of the fields, make `CustomFieldClassProvider` apply styles conditionally. The following `CustomFieldClassProvider2` example only applies a style to the `Name` field. For any fields with names not matching `Name`, `string.Empty` is returned, and no style is applied. Using [reflection](/dotnet/csharp/advanced-topics/reflection-and-attributes/), the field is matched to the model member's property or field name, not an `id` assigned to the HTML entity.

`CustomFieldClassProvider2.cs`:

:::moniker-end

:::moniker range=">= aspnetcore-8.0"

```csharp
using Microsoft.AspNetCore.Components.Forms;

public class CustomFieldClassProvider2 : FieldCssClassProvider
{
    public override string GetFieldCssClass(EditContext editContext,
        in FieldIdentifier fieldIdentifier)
    {
        if (fieldIdentifier.FieldName == "Name")
        {
            var isValid = editContext.IsValid(fieldIdentifier);

            return isValid ? "validField" : "invalidField";
        }

        return string.Empty;
    }
}
```

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

```csharp
using Microsoft.AspNetCore.Components.Forms;

public class CustomFieldClassProvider2 : FieldCssClassProvider
{
    public override string GetFieldCssClass(EditContext editContext,
        in FieldIdentifier fieldIdentifier)
    {
        if (fieldIdentifier.FieldName == "Name")
        {
            var isValid = !editContext.GetValidationMessages(fieldIdentifier).Any();

            return isValid ? "validField" : "invalidField";
        }

        return string.Empty;
    }
}
```

:::moniker-end

:::moniker range=">= aspnetcore-7.0"

<!--
:::code language="csharp" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/CustomFieldClassProvider2.cs":::
-->

> [!NOTE]
> Matching the field name in the preceding example is case sensitive, so a model property member designated "`Name`" must match a conditional check on "`Name`":
>
> * <span aria-hidden="true">✔️</span><span class="visually-hidden">Correctly matches:</span> `fieldId.FieldName == "Name"`
> * <span aria-hidden="true">❌</span><span class="visually-hidden">Fails to match:</span> `fieldId.FieldName == "name"`
> * <span aria-hidden="true">❌</span><span class="visually-hidden">Fails to match:</span> `fieldId.FieldName == "NAME"`
> * <span aria-hidden="true">❌</span><span class="visually-hidden">Fails to match:</span> `fieldId.FieldName == "nAmE"`

Add an additional property to `Model`, for example:

```csharp
[StringLength(10, ErrorMessage = "Description is too long.")]
public string? Description { get; set; } 
```

Add the `Description` to the `CustomValidationForm` component's form:

```razor
<InputText @bind-Value="Model!.Description" />
```

Update the <xref:Microsoft.AspNetCore.Components.Forms.EditForm.EditContext%2A> instance in the component's `OnInitialized` method to use the new Field CSS Class Provider:

```csharp
editContext?.SetFieldCssClassProvider(new CustomFieldClassProvider2());
```

Because a CSS validation class isn't applied to the `Description` field, it isn't styled. However, field validation runs normally. If more than 10 characters are provided, the validation summary indicates the error:

> Description is too long.

In the following example:

* The custom CSS style is applied to the `Name` field.
* Any other fields apply logic similar to Blazor's default logic and using Blazor's default field CSS validation styles, `modified` with `valid` or `invalid`. Note that for the default styles, you don't need to add them to the app's stylesheet if the app is based on a Blazor project template. For apps not based on a Blazor project template, the default styles can be added to the app's stylesheet:

  ```css
  .valid.modified:not([type=checkbox]) {
      outline: 1px solid #26b050;
  }

  .invalid {
      outline: 1px solid red;
  }
  ```

`CustomFieldClassProvider3.cs`:

:::moniker-end

:::moniker range=">= aspnetcore-8.0"

```csharp
using Microsoft.AspNetCore.Components.Forms;

public class CustomFieldClassProvider3 : FieldCssClassProvider
{
    public override string GetFieldCssClass(EditContext editContext,
        in FieldIdentifier fieldIdentifier)
    {
        var isValid = editContext.IsValid(fieldIdentifier);

        if (fieldIdentifier.FieldName == "Name")
        {
            return isValid ? "validField" : "invalidField";
        }
        else
        {
            if (editContext.IsModified(fieldIdentifier))
            {
                return isValid ? "modified valid" : "modified invalid";
            }
            else
            {
                return isValid ? "valid" : "invalid";
            }
        }
    }
}
```

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

```csharp
using Microsoft.AspNetCore.Components.Forms;

public class CustomFieldClassProvider3 : FieldCssClassProvider
{
    public override string GetFieldCssClass(EditContext editContext,
        in FieldIdentifier fieldIdentifier)
    {
        var isValid = !editContext.GetValidationMessages(fieldIdentifier).Any();

        if (fieldIdentifier.FieldName == "Name")
        {
            return isValid ? "validField" : "invalidField";
        }
        else
        {
            if (editContext.IsModified(fieldIdentifier))
            {
                return isValid ? "modified valid" : "modified invalid";
            }
            else
            {
                return isValid ? "valid" : "invalid";
            }
        }
    }
}
```

:::moniker-end

:::moniker range=">= aspnetcore-7.0"

<!--
:::code language="csharp" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/CustomFieldClassProvider3.cs":::
-->

Update the <xref:Microsoft.AspNetCore.Components.Forms.EditForm.EditContext%2A> instance in the component's `OnInitialized` method to use the preceding Field CSS Class Provider:

```csharp
editContext.SetFieldCssClassProvider(new CustomFieldClassProvider3());
```

Using `CustomFieldClassProvider3`:

* The `Name` field uses the app's custom validation CSS styles.
* The `Description` field uses logic similar to Blazor's logic and Blazor's default field CSS validation styles.

:::moniker-end

## Class-level validation with `IValidatableObject`

[Class-level validation with `IValidatableObject`](xref:mvc/models/validation#ivalidatableobject) ([API documentation](xref:System.ComponentModel.DataAnnotations.IValidatableObject)) is supported for Blazor form models. <xref:System.ComponentModel.DataAnnotations.IValidatableObject> validation only executes when the form is submitted and only if all other validation succeeds.

## Blazor data annotations validation package

The [`Microsoft.AspNetCore.Components.DataAnnotations.Validation`](https://www.nuget.org/packages/Microsoft.AspNetCore.Components.DataAnnotations.Validation) is a package that fills validation experience gaps using the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component. The package is currently *experimental*.

> [!WARNING]
> The [`Microsoft.AspNetCore.Components.DataAnnotations.Validation`](https://www.nuget.org/packages/Microsoft.AspNetCore.Components.DataAnnotations.Validation) package has a latest version of *release candidate* at [NuGet.org](https://www.nuget.org/packages/Microsoft.AspNetCore.Components.DataAnnotations.Validation). Continue to use the *experimental* release candidate package at this time. Experimental features are provided for the purpose of exploring feature viability and may not ship in a stable version. Watch the [Announcements GitHub repository](https://github.com/aspnet/Announcements), the [`dotnet/aspnetcore` GitHub repository](https://github.com/dotnet/aspnetcore), or this topic section for further updates.

:::moniker range="< aspnetcore-6.0"

## `[CompareProperty]` attribute

The <xref:System.ComponentModel.DataAnnotations.CompareAttribute> doesn't work well with the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component because the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> doesn't associate the validation result with a specific member. This can result in inconsistent behavior between field-level validation and when the entire model is validated on a submit. The [`Microsoft.AspNetCore.Components.DataAnnotations.Validation`](https://www.nuget.org/packages/Microsoft.AspNetCore.Components.DataAnnotations.Validation) *experimental* package introduces an additional validation attribute, `ComparePropertyAttribute`, that works around these limitations. In a Blazor app, `[CompareProperty]` is a direct replacement for the [`[Compare]` attribute](xref:System.ComponentModel.DataAnnotations.CompareAttribute).

:::moniker-end

:::moniker range=">= aspnetcore-10.0"

## Nested objects and collection types

Blazor form validation includes support for validating properties of nested objects and collection items with the built-in <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator>.

To create a validated form, use a <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component inside an <xref:Microsoft.AspNetCore.Components.Forms.EditForm> component, just as before.

To opt into the nested objects and collection types validation feature:

<!-- UPDATE 10.0 - API cross-links -->

1. Call the `AddValidation` extension method in the `Program` file where services are registered.
2. Declare the form model types in a C# class file, not in a Razor component (`.razor`).
3. Annotate the root form model type with the `[ValidatableType]` attribute.

Without following the preceding steps, form validation behavior doesn't include nested model and collection type validation.

<!-- UPDATE 10.0 - Replace with a fully working, cut-'n-paste example -->

The following example demonstrates customer orders with the improved form validation (details omitted for brevity):

In `Program.cs`, call `AddValidation` on the service collection:

```csharp
builder.Services.AddValidation();
```

In the following `Order` class, the `[ValidatableType]` attribute is required on the top-level model type. The other types are discovered automatically. `OrderItem` and `ShippingAddress` aren't shown for brevity, but nested and collection validation works the same way in those types if they were shown.

`Order.cs`:

```csharp
[ValidatableType]
public class Order
{
    public Customer Customer { get; set; } = new();
    public List<OrderItem> OrderItems { get; set; } = [];
}

public class Customer
{
    [Required(ErrorMessage = "Name is required.")]
    public string? FullName { get; set; }

    [Required(ErrorMessage = "Email is required.")]
    public string? Email { get; set; }

    public ShippingAddress ShippingAddress { get; set; } = new();
}
```

In the following `OrderPage` component, the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> component is present in the <xref:Microsoft.AspNetCore.Components.Forms.EditForm> component.

`OrderPage.razor`:

```razor
<EditForm Model="Model">
    <DataAnnotationsValidator />

    <h3>Customer Details</h3>
    <div class="mb-3">
        <label>
            Full Name
            <InputText @bind-Value="Model!.Customer.FullName" />
        </label>
        <ValidationMessage For="@(() => Model!.Customer.FullName)" />
    </div>

    // ... form continues ...
</EditForm>

@code {
    public Order? Model { get; set; }

    protected override void OnInitialized() => Model ??= new();
}
```

The requirement to declare the model types outside of Razor components (`.razor` files) is due to the fact that both the new validation feature and the Razor compiler itself are using a source generator. Currently, output of one source generator can't be used as an input for another source generator.

## Complex types

Blazor provides support for validating form input using data annotations with the built-in <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator>. However, the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> only validates top-level properties of the model bound to the form that aren't complex-type properties.

To validate the bound model's entire object graph, including complex-type properties, use the `ObjectGraphDataAnnotationsValidator` provided by the *experimental* [`Microsoft.AspNetCore.Components.DataAnnotations.Validation`](https://www.nuget.org/packages/Microsoft.AspNetCore.Components.DataAnnotations.Validation) package.

> [!NOTE]
> The `ObjectGraphDataAnnotationsValidator` isn't compatible with [nested objects and collection types validation](#nested-objects-and-collection-types), but it's capable of validating nested objects and collection types on its own.

:::moniker-end

:::moniker range="< aspnetcore-10.0"

## Nested objects, collection types, and complex types

Blazor provides support for validating form input using data annotations with the built-in <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator>. However, the <xref:Microsoft.AspNetCore.Components.Forms.DataAnnotationsValidator> only validates top-level properties of the model bound to the form that aren't collection- or complex-type properties.

To validate the bound model's entire object graph, including collection- and complex-type properties, use the `ObjectGraphDataAnnotationsValidator` provided by the *experimental* [`Microsoft.AspNetCore.Components.DataAnnotations.Validation`](https://www.nuget.org/packages/Microsoft.AspNetCore.Components.DataAnnotations.Validation) package:

:::moniker-end

```razor
<EditForm ...>
    <ObjectGraphDataAnnotationsValidator />
    ...
</EditForm>
```

Annotate model properties with `[ValidateComplexType]`. In the following model classes, the `ShipDescription` class contains additional data annotations to validate when the model is bound to the form:

`Starship.cs`:

```csharp
using System;
using System.ComponentModel.DataAnnotations;

public class Starship
{
    ...

    [ValidateComplexType]
    public ShipDescription ShipDescription { get; set; } = new();

    ...
}
```

`ShipDescription.cs`:

```csharp
using System;
using System.ComponentModel.DataAnnotations;

public class ShipDescription
{
    [Required]
    [StringLength(40, ErrorMessage = "Description too long (40 char).")]
    public string? ShortDescription { get; set; }

    [Required]
    [StringLength(240, ErrorMessage = "Description too long (240 char).")]
    public string? LongDescription { get; set; }
}
```

## Enable the submit button based on form validation

To enable and disable the submit button based on form validation, the following example:

* Uses a shortened version of the earlier `Starfleet Starship Database` form (`Starship3` component) of the [Example form](xref:blazor/forms/input-components#example-form) section of the *Input components* article that only accepts a value for the ship's Id. The other `Starship` properties receive valid default values when an instance of the `Starship` type is created.
* Uses the form's <xref:Microsoft.AspNetCore.Components.Forms.EditContext> to assign the model when the component is initialized.
* Validates the form in the context's <xref:Microsoft.AspNetCore.Components.Forms.EditContext.OnFieldChanged> callback to enable and disable the submit button.
* Implements <xref:System.IDisposable> and unsubscribes the event handler in the `Dispose` method. For more information, see <xref:blazor/components/component-disposal>.

> [!NOTE]
> When assigning to the <xref:Microsoft.AspNetCore.Components.Forms.EditForm.EditContext?displayProperty=nameWithType>, don't also assign an <xref:Microsoft.AspNetCore.Components.Forms.EditForm.Model?displayProperty=nameWithType> to the <xref:Microsoft.AspNetCore.Components.Forms.EditForm>.

`Starship14.razor`:

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorSample_BlazorWebApp/Components/Pages/Starship14.razor":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorSample_BlazorWebApp/Components/Pages/Starship14.razor":::

:::moniker-end

:::moniker range="< aspnetcore-8.0"

```razor
@page "/starship-14"
@implements IDisposable
@inject ILogger<Starship14> Logger

<EditForm EditContext="editContext" OnValidSubmit="Submit">
    <DataAnnotationsValidator />
    <ValidationSummary />
    <div>
        <label>
            Identifier: 
            <InputText @bind-Value="Model!.Id" />
        </label>
    </div>
    <div>
        <button type="submit" disabled="@formInvalid">Submit</button>
    </div>
</EditForm>

@code {
    private bool formInvalid = false;
    private EditContext? editContext;

    private Starship? Model { get; set; }

    protected override void OnInitialized()
    {
        Model ??=
            new()
            {
                Id = "NCC-1701",
                Classification = "Exploration",
                MaximumAccommodation = 150,
                IsValidatedDesign = true,
                ProductionDate = new DateTime(2245, 4, 11)
            };
        editContext = new(Model);
        editContext.OnFieldChanged += HandleFieldChanged;
    }

    private void HandleFieldChanged(object? sender, FieldChangedEventArgs e)
    {
        if (editContext is not null)
        {
            formInvalid = !editContext.Validate();
            StateHasChanged();
        }
    }

    private void Submit()
    {
        Logger.LogInformation("Submit called: Processing the form");
    }

    public void Dispose()
    {
        if (editContext is not null)
        {
            editContext.OnFieldChanged -= HandleFieldChanged;
        }
    }
}
```

<!--
:::code language="razor" source="~/../blazor-samples/7.0/BlazorSample_WebAssembly/Pages/forms-and-validation/Starship14.razor":::
-->

:::moniker-end

If a form isn't preloaded with valid values and you wish to disable the **`Submit`** button on form load, set `formInvalid` to `true`.

A side effect of the preceding approach is that a validation summary (<xref:Microsoft.AspNetCore.Components.Forms.ValidationSummary> component) is populated with invalid fields after the user interacts with any one field. Address this scenario in either of the following ways:

* Don't use a <xref:Microsoft.AspNetCore.Components.Forms.ValidationSummary> component on the form.
* Make the <xref:Microsoft.AspNetCore.Components.Forms.ValidationSummary> component visible when the submit button is selected (for example, in a `Submit` method).

```razor
<EditForm ... EditContext="editContext" OnValidSubmit="Submit" ...>
    <DataAnnotationsValidator />
    <ValidationSummary style="@displaySummary" />

    ...

    <button type="submit" disabled="@formInvalid">Submit</button>
</EditForm>

@code {
    private string displaySummary = "display:none";

    ...

    private void Submit()
    {
        displaySummary = "display:block";
    }
}
```
