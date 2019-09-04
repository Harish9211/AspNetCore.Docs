---
title: Controller action return types in ASP.NET Core web API
author: scottaddie
description: Learn about using the various controller action method return types in an ASP.NET Core web API.
ms.author: scaddie
ms.custom: mvc
ms.date: 09/03/2019
uid: web-api/action-return-types
---
# Controller action return types in ASP.NET Core web API

By [Scott Addie](https://github.com/scottaddie)

[View or download sample code](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/web-api/action-return-types/samples) ([how to download](xref:index#how-to-download-a-sample))

ASP.NET Core offers the following options for web API controller action return types:

::: moniker range=">= aspnetcore-2.1"

* [Specific type](#specific-type)
* [IActionResult](#iactionresult-type)
* [ActionResult\<T>](#actionresultt-type)

::: moniker-end

::: moniker range="<= aspnetcore-2.0"

* [Specific type](#specific-type)
* [IActionResult](#iactionresult-type)

::: moniker-end

This document explains when it's most appropriate to use each return type.

## Specific type

The simplest action returns a primitive or complex data type (for example, `string` or a custom object type). Consider the following action, which returns a collection of custom `Product` objects:

[!code-csharp[](../web-api/action-return-types/samples/2x/WebApiSample.Api.21/Controllers/ProductsController.cs?name=snippet_Get)]

Without known conditions to safeguard against during action execution, returning a specific type could suffice. The preceding action accepts no parameters, so parameter constraints validation isn't needed.

When known conditions need to be accounted for in an action, multiple return paths are introduced. In such a case, it's common to mix an <xref:Microsoft.AspNetCore.Mvc.ActionResult> return type with the primitive or complex return type. Either [IActionResult](#iactionresult-type) or [ActionResult\<T>](#actionresultt-type) are necessary to accommodate this type of action.

::: moniker range=">= aspnetcore-3.0"

### Asynchronous stream

Memory consumption on the server becomes a performance consideration when returning large data sets. Imagine an action that calls <xref:System.Linq.Enumerable.ToList*> on a LINQ query. The query returns an entire product catalog, consisting of hundreds of thousands of products. Each product record is stored in server memory. An asynchronous stream can relieve the burden on the server.

In ASP.NET Core 3.0 or later, a web API action can return an [asynchronous stream](/dotnet/csharp/whats-new/csharp-8#asynchronous-streams). Consider the following action, which uses <xref:System.Collections.Generic.IAsyncEnumerable%601> to stream paged product data to the client:

[!code-csharp[](../web-api/action-return-types/samples/3x/WebApiSample.Api.30/Controllers/ProductsController.cs?name=snippet_GetByPage)]

Consider the following `GetByPageChunk` action as an illustration of how the client's *time-to-first-byte* (TTFB) is dramatically reduced over its synchronous counterpart. TTFB refers to the number of milliseconds the client spends awaiting the action's initial response. The client requests a single page of product data. Each page consists of 10 product records. The action is triggered with the URI `https://localhost:<port>/Products/chunk/1/10`. The action's `pageNumber` and `pageSize` parameters assume values of `1` and `10`, respectively.

[!code-csharp[](../web-api/action-return-types/samples/3x/WebApiSample.Api.30/Controllers/ProductsController.cs?name=snippet_GetByPageChunk)]

The `yield return product;` statement sends the first product to the client, and execution continues for the second product. Product records are returned individually until all 10 products have been sent to the client. The client can begin its processing upon receiving the first product record. The synchronous form of this action would have blocked the client until all 10 records had been retrieved from the server.

::: moniker-end

## IActionResult type

The <xref:Microsoft.AspNetCore.Mvc.IActionResult> return type is appropriate when multiple `ActionResult` return types are possible in an action. The `ActionResult` types represent various HTTP status codes. Any non-abstract class deriving from `ActionResult` qualifies as a valid return type. Some common return types in this category are <xref:Microsoft.AspNetCore.Mvc.BadRequestResult> (400), <xref:Microsoft.AspNetCore.Mvc.NotFoundResult> (404), and <xref:Microsoft.AspNetCore.Mvc.OkObjectResult> (200). Alternatively, convenience methods in the <xref:Microsoft.AspNetCore.Mvc.ControllerBase> class can be used to return `ActionResult` types from an action. For example, `return BadRequest();` is a shorthand form of `return new BadRequestResult();`.

Because there are multiple return types and paths in this type of action, liberal use of the [[ProducesResponseType]](xref:Microsoft.AspNetCore.Mvc.ProducesResponseTypeAttribute) attribute is necessary. This attribute produces more descriptive response details for web API help pages generated by tools like [Swagger](xref:tutorials/web-api-help-pages-using-swagger). `[ProducesResponseType]` indicates the known types and HTTP status codes to be returned by the action.

### Synchronous action

Consider the following synchronous action in which there are two possible return types:

::: moniker range=">= aspnetcore-2.1"

[!code-csharp[](../web-api/action-return-types/samples/2x/WebApiSample.Api.21/Controllers/ProductsController.cs?name=snippet_GetById&highlight=8,11)]

::: moniker-end

::: moniker range="<= aspnetcore-2.0"

[!code-csharp[](../web-api/action-return-types/samples/2x/WebApiSample.Api.Pre21/Controllers/ProductsController.cs?name=snippet_GetById&highlight=8,11)]

::: moniker-end

In the preceding action:

* A 404 status code is returned when the product represented by `id` doesn't exist in the underlying data store. The <xref:Microsoft.AspNetCore.Mvc.ControllerBase.NotFound*> convenience method is invoked as shorthand for `return new NotFoundResult();`.
* A 200 status code is returned with the `Product` object when the product does exist. The <xref:Microsoft.AspNetCore.Mvc.ControllerBase.Ok*> convenience method is invoked as shorthand for `return new OkObjectResult(product);`.

### Asynchronous action

Consider the following asynchronous action in which there are two possible return types:

::: moniker range=">= aspnetcore-2.1"

[!code-csharp[](../web-api/action-return-types/samples/2x/WebApiSample.Api.21/Controllers/ProductsController.cs?name=snippet_CreateAsync&highlight=8,13)]

::: moniker-end

::: moniker range="<= aspnetcore-2.0"

[!code-csharp[](../web-api/action-return-types/samples/2x/WebApiSample.Api.Pre21/Controllers/ProductsController.cs?name=snippet_CreateAsync&highlight=8,13)]

::: moniker-end

In the preceding action:

* A 400 status code is returned when the product description contains "XYZ Widget". The <xref:Microsoft.AspNetCore.Mvc.ControllerBase.BadRequest*> convenience method is invoked as shorthand for `return new BadRequestResult();`.
* A 201 status code is generated by the <xref:Microsoft.AspNetCore.Mvc.ControllerBase.CreatedAtAction*> convenience method when a product is created. An alternative to calling `CreatedAtAction` is `return new CreatedAtActionResult(nameof(GetById), "Products", new { id = product.Id }, product);`. In this code path, the `Product` object is provided in the response body. A `Location` response header containing the newly created product's URL is provided.

For example, the following model indicates that requests must include the `Name` and `Description` properties. Failure to provide `Name` and `Description` in the request causes model validation to fail.

[!code-csharp[](../web-api/action-return-types/samples/2x/WebApiSample.DataAccess/Models/Product.cs?name=snippet_ProductClass&highlight=5-6,8-9)]

::: moniker range=">= aspnetcore-2.1"

If the [[ApiController]](xref:Microsoft.AspNetCore.Mvc.ApiControllerAttribute) attribute in ASP.NET Core 2.1 or later is applied, model validation errors result in a 400 status code. For more information, see [Automatic HTTP 400 responses](xref:web-api/index#automatic-http-400-responses).

## ActionResult\<T> type

ASP.NET Core 2.1 introduced the [ActionResult\<T>](xref:Microsoft.AspNetCore.Mvc.ActionResult`1) return type for web API controller actions. It enables you to return a type deriving from <xref:Microsoft.AspNetCore.Mvc.ActionResult> or return a [specific type](#specific-type). `ActionResult<T>` offers the following benefits over the [IActionResult type](#iactionresult-type):

* The [[ProducesResponseType]](xref:Microsoft.AspNetCore.Mvc.ProducesResponseTypeAttribute) attribute's `Type` property can be excluded. For example, `[ProducesResponseType(200, Type = typeof(Product))]` is simplified to `[ProducesResponseType(200)]`. The action's expected return type is instead inferred from the `T` in `ActionResult<T>`.
* [Implicit cast operators](/dotnet/csharp/language-reference/keywords/implicit) support the conversion of both `T` and `ActionResult` to `ActionResult<T>`. `T` converts to <xref:Microsoft.AspNetCore.Mvc.ObjectResult>, which means `return new ObjectResult(T);` is simplified to `return T;`.

C# doesn't support implicit cast operators on interfaces. Consequently, conversion of the interface to a concrete type is necessary to use `ActionResult<T>`. For example, use of `IEnumerable` in the following example doesn't work:

```csharp
[HttpGet]
public ActionResult<IEnumerable<Product>> Get() =>
    _repository.GetProducts();
```

One option to fix the preceding code is to return `_repository.GetProducts().ToList();`.

Most actions have a specific return type. Unexpected conditions can occur during action execution, in which case the specific type isn't returned. For example, an action's input parameter may fail model validation. In such a case, it's common to return the appropriate `ActionResult` type instead of the specific type.

### Synchronous action

Consider a synchronous action in which there are two possible return types:

[!code-csharp[](../web-api/action-return-types/samples/2x/WebApiSample.Api.21/Controllers/ProductsController.cs?name=snippet_GetById&highlight=7,10)]

In the preceding action:

* A 404 status code is returned when the product doesn't exist in the database.
* A 200 status code is returned with the corresponding `Product` object when the product does exist. Before ASP.NET Core 2.1, the `return product;` line had to be `return Ok(product);`.

### Asynchronous action

Consider an asynchronous action in which there are two possible return types:

[!code-csharp[](../web-api/action-return-types/samples/2x/WebApiSample.Api.21/Controllers/ProductsController.cs?name=snippet_CreateAsync&highlight=8,13)]

In the preceding action:

* A 400 status code (<xref:Microsoft.AspNetCore.Mvc.ControllerBase.BadRequest*>) is returned by the ASP.NET Core runtime when:
  * The [[ApiController]](xref:Microsoft.AspNetCore.Mvc.ApiControllerAttribute) attribute has been applied and model validation fails.
  * The product description contains "XYZ Widget".
* A 201 status code is generated by the <xref:Microsoft.AspNetCore.Mvc.ControllerBase.CreatedAtAction*> method when a product is created. In this code path, the `Product` object is provided in the response body. A `Location` response header containing the newly created product's URL is provided.

::: moniker-end

## Additional resources

* <xref:mvc/controllers/actions>
* <xref:mvc/models/validation>
* <xref:tutorials/web-api-help-pages-using-swagger>
