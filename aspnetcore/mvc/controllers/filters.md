---
title: Filters in ASP.NET Core
author: ardalis
description: Learn how filters work and how to use them in ASP.NET Core.
ms.author: riande
ms.custom: mvc
ms.date: 05/08/2019
uid: mvc/controllers/filters
---
# Filters in ASP.NET Core

By [Kirk Larkin](https://github.com/serpent5), [Rick Anderson](https://twitter.com/RickAndMSFT), [Tom Dykstra](https://github.com/tdykstra/), and [Steve Smith](https://ardalis.com/)

*Filters* in ASP.NET Core allow code to be run before or after specific stages in the request processing pipeline.

Built-in filters handle tasks such as:

* Authorization (preventing access to resources a user isn't authorized for).
* Response caching (short-circuiting the request pipeline to return a cached response).

Custom filters can be created to handle cross-cutting concerns. Examples of cross-cutting concerns include error handling, caching, configuration, authorization, and logging.  Filters avoid duplicating code. For example, an error handling exception filter could consolidate error handling.

This document applies to Razor Pages, API controllers, and controllers with views.

[View or download sample](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/mvc/controllers/filters/sample) ([how to download](xref:index#how-to-download-a-sample)).

## How filters work

Filters run within the *ASP.NET Core action invocation pipeline*, sometimes referred to as the *filter pipeline*.  The filter pipeline runs after ASP.NET Core selects the action to execute.

![The request is processed through Other Middleware, Routing Middleware, Action Selection, and the ASP.NET Core Action Invocation Pipeline. The request processing continues back through Action Selection, Routing Middleware, and various Other Middleware before becoming a response sent to the client.](filters/_static/filter-pipeline-1.png)

### Filter types

Each filter type is executed at a different stage in the filter pipeline:

* [Authorization filters](#authorization-filters) run first and are used to determine whether the user is authorized for the request. Authorization filters short-circuit the pipeline if the request is unauthorized.

* [Resource filters](#resource-filters):

  * Run after authorization.  
  * <xref:Microsoft.AspNetCore.Mvc.Filters.IResourceFilter.OnResourceExecuting*> can run code before the rest of the filter pipeline. For example, `OnResourceExecuting` can run code before model binding.
  * <xref:Microsoft.AspNetCore.Mvc.Filters.IResourceFilter.OnResourceExecuted*> can run code after the rest of the pipeline has completed.

* [Action filters](#action-filters) can run code immediately before and after an individual action method is called. They can be used to manipulate the arguments passed into an action and the result returned from the action. Action filters are **not** supported in Razor Pages.

* [Exception filters](#exception-filters) are used to apply global policies to unhandled exceptions that occur before anything has been written to the response body.

* [Result filters](#result-filters) can run code immediately before and after the execution of individual action results. They run only when the action method has executed successfully. They are useful for logic that must surround view or formatter execution.

The following diagram shows how filter types interact in the filter pipeline.

![The request is processed through Authorization Filters, Resource Filters, Model Binding, Action Filters, Action Execution and Action Result Conversion, Exception Filters, Result Filters, and Result Execution. On the way out, the request is only processed by Result Filters and Resource Filters before becoming a response sent to the client.](filters/_static/filter-pipeline-2.png)

## Implementation

Filters support both synchronous and asynchronous implementations through different interface definitions.

Synchronous filters can run code before (`On-Stage-Executing`) and after (`On-Stage-Executed`) their pipeline stage. For example, `OnActionExecuting` is called before the action method is called. `OnActionExecuted` is called after the action method returns.

[!code-csharp[](./filters/sample/FiltersSample/Filters/MySampleActionFilter.cs?name=snippet_ActionFilter)]

Asynchronous filters define an `On-Stage-ExecutionAsync` method:

[!code-csharp[](./filters/sample/FiltersSample/Filters/SampleAsyncActionFilter.cs?name=snippet)]

In the preceding code, the `SampleAsyncActionFilter` has an <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutionDelegate> (`next`)  that executes the action method.  Each of the `On-Stage-ExecutionAsync` methods take a `FilterType-ExecutionDelegate` that executes the filter's pipeline stage.

### Multiple filter stages

Interfaces for multiple filter stages can be implemented in a single class. For example, the <xref:Microsoft.AspNetCore.Mvc.Filters.ActionFilterAttribute> class implements `IActionFilter`, `IResultFilter`, and their async equivalents.

Implement **either** the synchronous or the async version of a filter interface, **not** both. The runtime checks first to see if the filter implements the async interface, and if so, it calls that. If not, it calls the synchronous interface's method(s). If both asynchronous and synchronous interfaces are implemented in one class, only the async method is called. When using abstract classes like <xref:Microsoft.AspNetCore.Mvc.Filters.ActionFilterAttribute> override only the synchronous methods or the async method for each filter type.

### Built-in filter attributes

ASP.NET Core includes built-in attribute-based filters that can be subclassed and customized. For example, the following result filter adds a header to the response:

<a name="add-header-attribute"></a>

[!code-csharp[](./filters/sample/FiltersSample/Filters/AddHeaderAttribute.cs?name=snippet)]

Attributes allow filters to accept arguments, as shown in the preceding example. Apply the `AddHeaderAttribute` to a controller or action method and specify the name and value of the HTTP header:

[!code-csharp[](./filters/sample/FiltersSample/Controllers/SampleController.cs?name=snippet_AddHeader&highlight=1)]

<!-- `https://localhost:5001/Sample` -->

Several of the filter interfaces have corresponding attributes that can be used as base classes for custom implementations.

Filter attributes:

* <xref:Microsoft.AspNetCore.Mvc.Filters.ActionFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.Filters.ExceptionFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.Filters.ResultFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.FormatFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.TypeFilterAttribute>

## Filter scopes and order of execution

A filter can be added to the pipeline at one of three *scopes*:

* Using an attribute on an action.
* Using an attribute on a controller.
* Globally for all controllers and actions as shown in the following code:

[!code-csharp[](./filters/sample/FiltersSample/StartupGF.cs?name=snippet_ConfigureServices)]

The preceding code adds three filters globally using the [MvcOptions.Filters](xref:Microsoft.AspNetCore.Mvc.MvcOptions.Filters) collection.

### Default order of execution

When there are multiple filters for a particular stage of the pipeline, scope determines the default order of filter execution.  Global filters surround class filters, which in turn surround method filters.

As a result of filter nesting, the *after* code of filters runs in the reverse order of the *before* code. The filter sequence:

* The *before* code of global filters.
  * The *before* code of controller filters.
    * The *before* code of action method filters.
    * The *after* code of action method filters.
  * The *after* code of controller filters.
* The *after* code of global filters.
  
The following example that illustrates the order in which filter methods are called for synchronous action filters.

| Sequence | Filter scope | Filter method |
|:--------:|:------------:|:-------------:|
| 1 | Global | `OnActionExecuting` |
| 2 | Controller | `OnActionExecuting` |
| 3 | Method | `OnActionExecuting` |
| 4 | Method | `OnActionExecuted` |
| 5 | Controller | `OnActionExecuted` |
| 6 | Global | `OnActionExecuted` |

This sequence shows:

* The method filter is nested within the controller filter.
* The controller filter is nested within the global filter.

### Controller and Razor Page level filters

Every controller that inherits from the <xref:Microsoft.AspNetCore.Mvc.Controller> base class includes [Controller.OnActionExecuting](xref:Microsoft.AspNetCore.Mvc.Controller.OnActionExecuting*),  [Controller.OnActionExecutionAsync](xref:Microsoft.AspNetCore.Mvc.Controller.OnActionExecutionAsync*), and [Controller.OnActionExecuted](xref:Microsoft.AspNetCore.Mvc.Controller.OnActionExecuted*)
`OnActionExecuted` methods. These methods:

* Wrap the filters that run for a given action.
* `OnActionExecuting` is called before any of the action's filters.
* `OnActionExecuted` is called after all of the action filters.
* `OnActionExecutionAsync` is called before any of the action's filters. Code in the filter after `next` runs after the action method.

For example, in the download sample, `MySampleActionFilter` is applied globally in startup.

The `TestController`:

* Applies the `SampleActionFilterAttribute` (`[SampleActionFilter]`) to the `FilterTest2` action:
* Overrides `OnActionExecuting` and `OnActionExecuted`.

[!code-csharp[](./filters/sample/FiltersSample/Controllers/TestController.cs?name=snippet)]

Navigating to `https://localhost:5001/Test/FilterTest2` runs the following code:

* `TestController.OnActionExecuting`
  * `MySampleActionFilter.OnActionExecuting`
    * `SampleActionFilterAttribute.OnActionExecuting`
      * `TestController.FilterTest2`
    * `SampleActionFilterAttribute.OnActionExecuted`
  * `MySampleActionFilter.OnActionExecuted`
* `TestController.OnActionExecuted`

For Razor Pages, see [Implement Razor Page filters by overriding filter methods](xref:razor-pages/filter#implement-razor-page-filters-by-overriding-filter-methods).

### Overriding the default order

The default sequence of execution can be overridden by implementing <xref:Microsoft.AspNetCore.Mvc.Filters.IOrderedFilter>. `IOrderedFilter` exposes the <xref:Microsoft.AspNetCore.Mvc.Filters.IOrderedFilter.Order> property that takes precedence over scope to determine the order of execution. A filter with a lower `Order` value:

* Runs the *before* code before that of a filter with a higher value of `Order`.
* Runs the *after* code after that of a filter with a higher `Order` value.

The `Order` property can be set with a constructor parameter:

```csharp
[MyFilter(Name = "Controller Level Attribute", Order=1)]
```

Consider the same 3 action filters shown in the preceding example. If the `Order` property of the controller and global filters is set to 1 and 2 respectively, the order of execution is reversed.

| Sequence | Filter scope | `Order` property | Filter method |
|:--------:|:------------:|:-----------------:|:-------------:|
| 1 | Method | 0 | `OnActionExecuting` |
| 2 | Controller | 1  | `OnActionExecuting` |
| 3 | Global | 2  | `OnActionExecuting` |
| 4 | Global | 2  | `OnActionExecuted` |
| 5 | Controller | 1  | `OnActionExecuted` |
| 6 | Method | 0  | `OnActionExecuted` |

The `Order` property overrides scope when determining the order in which filters run. Filters are sorted first by order, then scope is used to break ties. All of the built-in filters implement `IOrderedFilter` and set the default `Order` value to 0. For built-in filters, scope determines order unless `Order` is set to a non-zero value.

## Cancellation and short-circuiting

The filter pipeline can be short-circuited by setting the <xref:Microsoft.AspNetCore.Mvc.Filters.ResourceExecutingContext.Result> property on the <xref:Microsoft.AspNetCore.Mvc.Filters.ResourceExecutingContext> parameter provided to the filter method. For instance, the following Resource filter prevents the rest of the pipeline from executing:

<a name="short-circuiting-resource-filter"></a>

[!code-csharp[](./filters/sample/FiltersSample/Filters/ShortCircuitingResourceFilterAttribute.cs?name=snippet)]

In the following code, both the `ShortCircuitingResourceFilter` and the `AddHeader` filter target the `SomeResource` action method. The `ShortCircuitingResourceFilter`:

* Runs first, because it's a Resource Filter and `AddHeader` is an Action Filter.
* Short-circuits the rest of the pipeline.

Therefore the `AddHeader` filter never runs for the `SomeResource` action. This behavior would be the same if both filters were applied at the action method level, provided the `ShortCircuitingResourceFilter` ran first. The `ShortCircuitingResourceFilter` runs first because of its filter type, or by explicit use of `Order` property.

[!code-csharp[](./filters/sample/FiltersSample/Controllers/SampleController.cs?name=snippet_AddHeader&highlight=1,9)]

## Dependency injection

Filters can be added by type or by instance. If an instance is added, that instance is used for every request. If a type is added, it's type-activated. A type-activated filter means:

* An instance is created for each request.
* Any constructor dependencies are populated by [dependency injection](xref:fundamentals/dependency-injection) (DI).

Filters that are implemented as attributes and added directly to controller classes or action methods cannot have constructor dependencies provided by [dependency injection](xref:fundamentals/dependency-injection) (DI). Constructor dependencies cannot be provided by DI because:

* Attributes must have their constructor parameters supplied where they're applied. 
* This is a limitation of how attributes work.

The following filters support constructor dependencies provided from DI:

* <xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.TypeFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory> implemented on the attribute.

The preceding filters can be applied to a controller or action method:

Loggers are available from DI. However, avoid creating and using filters purely for logging purposes. The [built-in framework logging](xref:fundamentals/logging/index) typically provides what's needed for logging. Logging added to filters:

* Should focus on business domain concerns or behavior specific to the filter.
* Should **not** log actions or other framework events. The built in filters log actions and framework events.

### ServiceFilterAttribute

Service filter implementation types are registered in `ConfigureServices`. A <xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute> retrieves an instance of the filter from DI.

The following code shows the `AddHeaderResultServiceFilter`:

[!code-csharp[](./filters/sample/FiltersSample/Filters/LoggingAddHeaderFilter.cs?name=snippet_ResultFilter)]

In the following code, `AddHeaderResultServiceFilter` is added to the DI container:

[!code-csharp[](./filters/sample/FiltersSample/Startup.cs?name=snippet_ConfigureServices&highlight=4)]

In the following code, the `ServiceFilter` attribute retrieves an instance of the `AddHeaderResultServiceFilter` filter from DI:

[!code-csharp[](./filters/sample/FiltersSample/Controllers/HomeController.cs?name=snippet_ServiceFilter&highlight=1)]

[ServiceFilterAttribute.IsReusable](xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute.IsReusable):

* Provides a hint that the filter instance *may* be reused outside of the request scope it was created within. The ASP.NET Core runtime doesn't guarantee:

  * That a single instance of the filter will be created.
  * The filter will not be re-requested from the DI container at some later point.

* Should not be used with a filter that depends on services with a lifetime other than singleton.

 <xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute> implements <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory>. `IFilterFactory` exposes the <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory.CreateInstance*> method for creating an <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterMetadata> instance. `CreateInstance` loads the specified type from DI.

### TypeFilterAttribute

<xref:Microsoft.AspNetCore.Mvc.TypeFilterAttribute> is similar to <xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute>, but its type isn't resolved directly from the DI container. It instantiates the type by using <xref:Microsoft.Extensions.DependencyInjection.ObjectFactory?displayProperty=fullName>.

Because `TypeFilterAttribute` types aren't resolved directly from the DI container:

* Types that are referenced using the `TypeFilterAttribute` don't need to be registered with the DI container.  They do have their dependencies fulfilled by the DI container.
* `TypeFilterAttribute` can optionally accept constructor arguments for the type.

When using `TypeFilterAttribute`, setting `IsReusable` is a hint that the filter instance *may* be reused outside of the request scope it was created within. The ASP.NET Core runtime provides no guarantees that a single instance of the filter will be created. `IsReusable` should not be used with a filter that depends on services with a lifetime other than singleton.

The following example shows how to pass arguments to a type using `TypeFilterAttribute`:

[!code-csharp[](../../mvc/controllers/filters/sample/FiltersSample/Controllers/HomeController.cs?name=snippet_TypeFilter&highlight=1,2)]
[!code-csharp[](../../mvc/controllers/filters/sample/FiltersSample/Filters/LogConstantFilter.cs?name=snippet_TypeFilter_Implementation&highlight=6)]

<!-- 
https://localhost:5001/home/hi?name=joe
VS debug window shows 
FiltersSample.Filters.LogConstantFilter:Information: Method 'Hi' called
-->

## Authorization filters

Authorization filters:

* Are the first filters run in the filter pipeline.
* Control access to action methods.
* Have a before method, but no after method.

Custom authorization filters require a custom authorization framework. Prefer configuring the authorization policies or writing a custom authorization policy over writing a custom filter. The built-in authorization filter:

* Calls the authorization system.
* Does not authorize requests.

Do **not** throw exceptions within authorization filters:

* The exception will not be handled.
* Exception filters will not handle the exception.

Consider issuing a challenge when an exception occurs in an authorization filter.

Learn more about [Authorization](xref:security/authorization/introduction).

## Resource filters

Resource filters:

* Implement either the <xref:Microsoft.AspNetCore.Mvc.Filters.IResourceFilter> or <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncResourceFilter> interface.
* Execution wraps most of the filter pipeline.
* Only [Authorization filters](#authorization-filters) run before resource filters.

Resource filters are useful to short-circuit most of the pipeline. For example, a caching filter can avoid the rest of the pipeline on a cache hit.

Resource filter examples:

* [The short-circuiting resource filter](#short-circuiting-resource-filter) shown previously.
* [DisableFormValueModelBindingAttribute](https://github.com/aspnet/Entropy/blob/rel/2.0.0-preview2/samples/Mvc.FileUpload/Filters/DisableFormValueModelBindingAttribute.cs):

  * Prevents model binding from accessing the form data.
  * Used for large file uploads to prevent the form data from being read into memory.

## Action filters

> [!IMPORTANT]
> Action filters do **not** apply to Razor Pages. Razor Pages supports <xref:Microsoft.AspNetCore.Mvc.Filters.IPageFilter> and <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncPageFilter> . For more information, see [Filter methods for Razor Pages](xref:razor-pages/filter).

Action filters:

* Implement either the <xref:Microsoft.AspNetCore.Mvc.Filters.IActionFilter> or <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncActionFilter> interface.
* Their execution surrounds the execution of action methods.

The following code shows a sample action filter:

[!code-csharp[](./filters/sample/FiltersSample/Filters/MySampleActionFilter.cs?name=snippet_ActionFilter)]

The <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutingContext> provides the following properties:

* <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutingContext.ActionArguments> - enables the inputs to an action method be read.
* <xref:Microsoft.AspNetCore.Mvc.Controller> - enables manipulating the controller instance.
* <xref:System.Web.Mvc.ActionExecutingContext.Result> - setting `Result` short-circuits execution of the action method and subsequent action filters.

Throwing an exception in an action method:

* Prevents running of subsequent filters.
* Unlike setting `Result`, is treated as a failure instead of a successful result.

The <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutedContext> provides `Controller` and `Result` plus the following properties:

* <xref:System.Web.Mvc.ActionExecutedContext.Canceled> - True if the action execution was short-circuited by another filter.
* <xref:System.Web.Mvc.ActionExecutedContext.Exception> - Non-null if the action or a previously run action filter threw an exception. Setting this property to null:

  * Effectively handles the exception.
  * `Result` is executed as if it was returned from the action method.

For an `IAsyncActionFilter`, a call to the <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutionDelegate>:

* Executes any subsequent action filters and the action method.
* Returns `ActionExecutedContext`.

To short-circuit, assign <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutingContext.Result?displayProperty=fullName> to a result instance and don't call `next` (the `ActionExecutionDelegate`).

The framework provides an abstract <xref:Microsoft.AspNetCore.Mvc.Filters.ActionFilterAttribute> that can be subclassed.

The `OnActionExecuting` action filter can be used to:

* Validate model state.
* Return an error if the state is invalid.

[!code-csharp[](./filters/sample/FiltersSample/Filters/ValidateModelAttribute.cs?name=snippet)]

The `OnActionExecuted` method runs after the action method:

* And can see and manipulate the results of the action through the <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutedContext.Result> property.
* <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutedContext.Canceled> is set to true if the action execution was short-circuited by another filter.
* <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutedContext.Exception> is set to a non-null value if the action or a subsequent action filter threw an exception. Setting `Exception` to null:

  * Effectively handles an exception.
  * `ActionExecutedContext.Result` is executed as if it were returned normally from the action method.

[!code-csharp[](./filters/sample/FiltersSample/Filters/ValidateModelAttribute.cs?name=snippet2&higlight=12-99)]

## Exception filters

Exception filters:

* Implement <xref:Microsoft.AspNetCore.Mvc.Filters.IExceptionFilter> or <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncExceptionFilter>. 
* Can be used to implement common error handling policies.

The following sample exception filter uses a custom error view to display details about exceptions that occur when the app is in development:

[!code-csharp[](./filters/sample/FiltersSample/Filters/CustomExceptionFilterAttribute.cs?name=snippet_ExceptionFilter&highlight=16-19)]

Exception filters:

* Don't have before and after events.
* Implement <xref:Microsoft.AspNetCore.Mvc.Filters.IExceptionFilter.OnException*> or <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncExceptionFilter.OnExceptionAsync*>.
* Handle unhandled exceptions that occur in Razor Page or controller creation, [model binding](xref:mvc/models/model-binding), action filters, or action methods.
* Do **not** catch exceptions that occur in resource filters, result filters, or MVC result execution.

To handle an exception, set the <xref:System.Web.Mvc.ExceptionContext.ExceptionHandled> property to `true` or write a response. This stops propagation of the exception. An exception filter can't turn an exception into a "success". Only an action filter can do that.

Exception filters:

* Are good for trapping exceptions that occur within actions.
* Are not as flexible as error handling middleware.

Prefer middleware for exception handling. Use exception filters only where error handling *differs* based on which action method is called. For example, an app might have action methods for both API endpoints and for views/HTML. The API endpoints could return error information as JSON, while the view-based actions could return an error page as HTML.

## Result filters

Result filters:

* Implement an interface:
  * <xref:Microsoft.AspNetCore.Mvc.Filters.IResultFilter> or <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncResultFilter>
  * <xref:Microsoft.AspNetCore.Mvc.Filters.IAlwaysRunResultFilter> or <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncAlwaysRunResultFilter>
* Their execution surrounds the execution of action results.

### IResultFilter and IAsyncResultFilter

The following code shows a result filter that adds an HTTP header:

[!code-csharp[](./filters/sample/FiltersSample/Filters/LoggingAddHeaderFilter.cs?name=snippet_ResultFilter)]

The kind of result being executed depends on the action. An action returning a view would include all razor processing as part of the <xref:Microsoft.AspNetCore.Mvc.ViewResult> being executed. An API method might perform some serialization as part of the execution of the result. Learn more about [action results](xref:mvc/controllers/actions)

Result filters are only executed for successful results - when the action or action filters produce an action result. Result filters are not executed when exception filters handle an exception.

The <xref:Microsoft.AspNetCore.Mvc.Filters.IResultFilter.OnResultExecuting*?displayProperty=fullName> method can short-circuit execution of the action result and subsequent result filters by setting <xref:Microsoft.AspNetCore.Mvc.Filters.ResultExecutingContext.Cancel?displayProperty=fullName> to `true`. Write to the response object when short-circuiting to avoid generating an empty response. Throwing an exception in `IResultFilter.OnResultExecuting` will:

* Prevent execution of the action result and subsequent filters.
* Be treated as a failure instead of a successful result.

When the <xref:Microsoft.AspNetCore.Mvc.Filters.IResultFilter.OnResultExecuted*?displayProperty=fullName> method runs:

* The response has likely been sent to the client and cannot be changed.
* If an exception was thrown, the response body is not sent.

<!-- Review preceding "If an exception was thrown: Original 
When the OnResultExecuted method runs, the response has likely been sent to the client and cannot be changed further (unless an exception was thrown).

SHould that be , 
If an exception was thrown **IN THE RESULT FILTER**, the response body is not sent.

 -->

`ResultExecutedContext.Canceled` is set to `true` if the action result execution was short-circuited by another filter.

`ResultExecutedContext.Exception` is set to a non-null value if the action result or a subsequent result filter threw an exception. Setting `Exception` to null effectively handles an exception and prevents the exception from being rethrown by ASP.NET Core later in the pipeline. There is no reliable way to write data to a response when handling an exception in a result filter. If the headers have been flushed to the client when an action result throws an exception, there's no reliable mechanism to send a failure code.

For an <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncResultFilter>, a call to `await next` on the <xref:Microsoft.AspNetCore.Mvc.Filters.ResultExecutionDelegate> executes any subsequent result filters and the action result. To short-circuit, set [ResultExecutingContext.Cancel](xref:Microsoft.AspNetCore.Mvc.Filters.ResultExecutingContext.Cancel) to `true` and don't call the `ResultExecutionDelegate`:

[!code-csharp[](./filters/sample/FiltersSample/Filters/MyAsyncResponseFilter.cs?name=snippet)]

The framework provides an abstract `ResultFilterAttribute` that can be subclassed. The [AddHeaderAttribute](#add-header-attribute) class shown previously is an example of a result filter attribute.

### IAlwaysRunResultFilter and IAsyncAlwaysRunResultFilter

The <xref:Microsoft.AspNetCore.Mvc.Filters.IAlwaysRunResultFilter> and <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncAlwaysRunResultFilter> interfaces declare an <xref:Microsoft.AspNetCore.Mvc.Filters.IResultFilter> implementation that runs for all action results. The filter is applied to all action results unless:

* An <xref:Microsoft.AspNetCore.Mvc.Filters.IExceptionFilter> or <xref:Microsoft.AspNetCore.Mvc.Filters.IAuthorizationFilter> applies and short-circuits the response.
* An exception filter handles an exception by producing an action result.

Filters other than `IExceptionFilter` and `IAuthorizationFilter` don't short-circuit `IAlwaysRunResultFilter` and `IAsyncAlwaysRunResultFilter`.

For example, the following filter always runs and sets an action result (<xref:Microsoft.AspNetCore.Mvc.ObjectResult>) with a *422 Unprocessable Entity* status code when content negotiation fails:

[!code-csharp[](./filters/sample/FiltersSample/Filters/UnprocessableResultFilter.cs?name=snippet)]

### IFilterFactory

<xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory> implements <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterMetadata>. Therefore, an `IFilterFactory` instance can be used as an `IFilterMetadata` instance anywhere in the filter pipeline. When the runtime prepares to invoke the filter, it attempts to cast it to an `IFilterFactory`. If that cast succeeds, the <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory.CreateInstance*> method is called to create the `IFilterMetadata` instance that is invoked. This provides a flexible design, since the precise filter pipeline doesn't need to be set explicitly when the app starts.

`IFilterFactory` can be implemented using custom attribute implementations as another approach to creating filters:

[!code-csharp[](./filters/sample/FiltersSample/Filters/AddHeaderWithFactoryAttribute.cs?name=snippet_IFilterFactory&highlight=1,4,5,6,7)]

The preceding code can be tested by running the [download sample](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/mvc/controllers/filters/sample):

* Invoke the F12 developer tools.
* Navigate to `https://localhost:5001/Sample/HeaderWithFactory`

The F12 developer tools display the following response headers added by the sample code:

* **author:** `Joe Smith`
* **globaladdheader:** `Result filter added to MvcOptions.Filters`
* **internal:** `My header`

The preceding code creates the **internal:** `My header` response header.

### IFilterFactory implemented on an attribute

<!-- Review 
This section needs to be rewritten.
What's a non-named attribute?
-->

Filters that implement `IFilterFactory` are useful for filters that:

* Don't require passing parameters.
* Have constructor dependencies that need to be filled by DI.

<xref:Microsoft.AspNetCore.Mvc.TypeFilterAttribute> implements <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory>. `IFilterFactory` exposes the <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory.CreateInstance*> method for creating an <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterMetadata> instance. `CreateInstance` loads the specified type from the services container (DI).

[!code-csharp[](./filters/sample/FiltersSample/Filters/SampleActionFilterAttribute.cs?name=snippet_TypeFilterAttribute&highlight=1,3,7)]

The following code shows three approaches to applying the `[SampleActionFilter]`:

[!code-csharp[](./filters/sample/FiltersSample/Controllers/HomeController.cs?name=snippet&highlight=1)]

In the preceding code, decorating the method with `[SampleActionFilter]` is the preferred approach to applying the `SampleActionFilter`.

## Using middleware in the filter pipeline

Resource filters work like [middleware](xref:fundamentals/middleware/index) in that they surround the execution of everything that comes later in the pipeline. But filters differ from middleware in that they're part of the ASP.NET Core runtime, which means that they have access to ASP.NET Core context and constructs.

To use middleware as a filter, create a type with a `Configure` method that specifies the middleware to inject into the filter pipeline. The following example uses the localization middleware to establish the current culture for a request:

[!code-csharp[](./filters/sample/FiltersSample/Filters/LocalizationPipeline.cs?name=snippet_MiddlewareFilter&highlight=3,21)]

Use the <xref:Microsoft.AspNetCore.Mvc.MiddlewareFilterAttribute> to run the middleware:

[!code-csharp[](./filters/sample/FiltersSample/Controllers/HomeController.cs?name=snippet_MiddlewareFilter&highlight=2)]

Middleware filters run at the same stage of the filter pipeline as Resource filters, before model binding and after the rest of the pipeline.

## Next actions

* See [Filter methods for Razor Pages](xref:razor-pages/filter)
* To experiment with filters, [download, test, and modify the GitHub sample](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/mvc/controllers/filters/sample).
