[[_TOC_]]

# Introduction
Our architecture is embracing F# modules, that are typically split into two categories:
* Modules containing api/domain/storage/service specific types
* Modules containing storage/fileIO/service specific functions making use of those types and performing some pipeline specific logic

Because of modules, typical flow of control in F# is top to bottom - at any given point you can reference types/functions that were defined earlier. Additionally, all parameters needed for function must be passed explicitly. In most situations that is ok, however it creates an overhead when dealing with application state.
Example of such interactions are common operations like accessing HttpContext data (logged in user, query parameters, request payload), logging, config from appsettings etc.
This problem is described in details here - https://fsharpforfunandprofit.com/posts/dependency-injection-1/
That leads us to the inversion of control land - we'd like to have some central place to control the lifetime of dependencies functions require and inject them instead of passing via function parameters.

There are many ways to provide such functionality (including hybrid approaches). Some of them are:
* Partial application
* Reader modal
* Composition Root
* Dependency Injection frameworks (composition root conceptually)
* Service Locator

In object oriented languages like C# dependency injection is typically achieved by listing service dependencies in service constructor and storing references to them in properties that methods can access to resolve particular dependency. 
In pure functional languages like Haskell it's typically achieved by reader monad or partial application with composition root - the clear benefit of such approach is explicit mapping of dependencies for function. The overhead however, is maintenance of composition root and a lot of wiring code one need to write.

# AppContext - approach taken in Demetrix Services

F# is a functional first language, but has OO capabilities, - however writing plain modules with functions is certainly a preferred approach compared to classes/objects. Therefore, we've settled on a hybrid approach - static AppContext object that is instantiated and configured during application startup and contains:
* HttpContext with information about request, response, user information from auth cookie
* Appsettings like database connection string, credentials etc
* ASP NET's ServiceProvider - basically a service locator that can be used to retrieve any of the registred dependencies
* Logger instance to provide logging to file and elastic search capabilities within any functions.
* Metrics instance to allow measure some application specific metrics - amount of requests calling given function, time elapsed during execution etc

Exact implementation can be found in Demetrix.Common.Backend project:
![image](uploads/4fdfafbfcf95e725f1d91c744d712f4e/image.png)

AppContext registration can be found startup logic of app that makes use of it. Currently these apps are lims, commercial, warehouse servers and their integration tests. However there's nothing restricting us from using it in CLI apps too.

Below you can find some typical usages of AppContext:

## Connection string injection for storage functions

Connections to database are instantiated in Demetrix.[App].Common project's Db module. Note, that we are using ```open type AppContext``` statement in the open declarations of the module - that's a syntax, that allows to `open` classes similarly to modules and refer to static methods via method name only.

![image](uploads/a0d6d40be1fc8918a55504cece5d0ba7/image.png)

As the result of such dependency injection, storage functions don't need to list connection string as parameter explicitly, - it's injected upon any `Db.openConnection[Async]` call.

## Logging in functions

Logging in any service/storage/fileIO within apps is achieved in a similar way - `open type AppContext` and using `Log.[Debug|Information|Warning|Error]` methods. Following example is from `Demetrix.Lims.Domain.CropDesign` `CropWorkflowPlanner` service:

![image](uploads/073df7562d57ea150b63b6427a2ffd8a/image.png)

This allows us to provide logging to elastic search / file / console as well as support different levels of logging without switching from typical `printfn` way - this is pretty much a drop-in replacement.

## Dependency injection for testing purposes

Sometimes we have a need in mocking dependencies while performing tests - imagine some service, that retrieves a file from aws s3 or google drive - we don't want to call third party every time we are running an integration test. Instead, tests should be able to mock data that is expected to be returned from such call and use prepared data set.

Example of such logic can be found within experiment pipeline. We are importing the excel file from google drive, parsing submission data out of it and perform other pipeline specific logic. In integration test we'd like to test pipeline logic with mocked import file.

### Service module implementation

![image](uploads/b63eeb7ad6fa2f63f823e8ea2d531ea2/image.png)

### Registration of `GDriveFileFetching` dependency in `Demetrix.Lims.Web` `Startup` file

![image](uploads/30f5980b03ce50116b270f381c5ae3de/image.png)

### Mocking of `GDriveFileFetching` dependency in `Demetrix.Lims.Web.Tests` `ApiServer` file

![image](uploads/278670dc41bceb8c11c332a5f7a55790/image.png)

Therefore, while application calls gdrive to retrieve file, - integrations tests are working with locally prepped test file

# Dependency injection via `AppContext` `ServiceLocator`

## Service lifetime

It is critical to understand what is service lifetime. When a component requests another component through dependency injection, whether the instance it receives is unique to that instance of the component or not depends on the lifetime. Setting the lifetime thus decides how many times a component is instantiated, and if a component is shared.

There are 3 options for this with the built-in DI container in ASP.NET Core:
* Singleton
* Scoped
* Transient

Singleton means only a single instance will ever be created. That instance is shared between all components that require it. The same instance is thus used always.

Scoped means an instance is created once per scope. A scope is created on every request to the application, thus any components registered as Scoped will be created once per request. You can also define custom scope:

![image](uploads/b41f69f5a9130e3fc5312698b6699553/image.png)

This can be a useful technique in certain situations - for storing some state between functions without passing dependency in parameters explicitly, however custom scopes must be used with caution and only when needed - in 99% of cases you're good with default scope that is created per request.

Transient components are created every time they are requested and are never shared.

## Function context

Scoped per request lifetime is useful for storing some information needed by multiple functions invoked within pipeline. Typically this involves fetching multiple namespaces from ontology, getting system default user etc. Without it, - we are forced to either fetch related resources within every function that needs them or passing them as dependencies through whole call tree.

![image](uploads/d9324ec40dee51f4ad0ba1452f2a78c2/image.png)

Passing such dependencies to every function can escalate quickly, so scoped `context` is one of the ways to deal with it.

### Registration of context instantiation in Startup.configureServices

![image](uploads/3564849e1420effa777d5558e0652745/image.png)

### Usage in services

![image](uploads/7fa16e63ad30e5b26f43156601730639/image.png)

Dependency injection takes care of invoking instantiation function once per http request and providing context value to each service that requires it. As context value is lazy, - we fetch all required ontology data from db upon first access per request, and then simply using cached value as if dependency was explicitly passed to functions as argument.