# Digital Silo

Digital Silo collects stateless tasks, aka **Grains**, and executes them in a scalable serverless environment on the Microsoft Azure Cloud technology. It accelerates the steps of making an application serverless-ready by eliminating the burden of infrastructure-driven implementation and letting developers concentrate on coding their business logic.

## Major components in Digital Silo

Digital Silo consists of the following major components:

* Storage or repository to seed the developed grains' DLLs
* Gateway Web API to collect grains' payloads
* Serverless infrastructure to process the submitted grains' payloads
* SignalR Service to report the grains' progress status

## Features

Gateway Web API is the entry point of the system to receive grains' payloads in JSON format. It has no role in running the business logic encapsulated in the grain. The only task it carries out is to deliver the grains' payloads to the serverless infrastructure for further processing. However, it can manage a virtual queue of submitted grains' payloads as well. Developers can also define the grains' chaining keys in the payloads, and the Gateway can figure out how to line them up in a row to submit to the silo. The subsequent grain's payload in the queue is picked up and processed if its predecessor finishes successfully. Gateway will not send a grain's payload to the serverless part if its predecessor has already failed or terminated.

The serverless infrastructure is the entity that takes care of processing grains statelessly. It can track the progress of grains and resumes their execution from the point where they left off if the grains fail due to any reason. It can also execute grains with delays should such a request is made through the grain's payload.

Depending on the deployment pipeline's configuration, the serverless infrastructure can run on a consumption plan or a bit more real-time mode, i.e. premium plan when necessary.

The SignalR Service provides a real-time communication channel between the client application interested in observing grains' status and the backend progress.

### High-Level Architecture diagram

The following diagram depicts the infrastructure and the major components of Digital Silo:

![Digital Silo HLA Diagram](assets/Digital%20Silo%20HLA%20Diagram.svg)

## Developing Grains

### Environment

A development environment is required to be able to develop and test the grains. The development environment is the entire infrastructure of Digital Silo hosted on Azure and provisioned by a Terraform script found [here](https://github.com/DigitalSilo/digitalsilo/tree/master/infrastructure). The service plans and the capacity of Azure services are at their minimal costs in the Terraform script. Should additional horsepower is required, one can incorporate higher-level SKUs in the Terraform script.

#### Silo, the processing component

The Terraform script provisions the entire Digital Silo infrastructure just within a few minutes on Azure. Once the infrastructure is ready, a storage account to retain the developed grains' DLLs becomes available to the outside world. The infrastructure knows where to fetch and download the developed grains' DLLs from the storage and turn them into an integral part of the silo's infrastructure. At this point, the grains are ready and fully engaged in executing the business logic after receiving their respective payloads via Gateway.

Optionally a client application in C# or typescript may subscribe to the provisioned signalR Service instance to listen to grains' progress status.

#### Grain

A grain is a stateless component that encapsulates a specific business logic that runs throughout Digital Silo. Please consult the following section to learn about the details of grain development steps.

### Development steps

#### Prerequisites

The following tools are required to start developing grains:
Visual Studio 2019 (for Mac or Windows), or Visual Studio Code
.net core 3.1 and its latest SDK
Digital Silo instance running on Azure that the provided Terraform script can provision
Digital Silo's `DigitalSilo.Grain` assembly is available to download from [this artifacts feed](https://pkgs.dev.azure.com/umplify/Grain/_packaging/DigitalSilo/nuget/v3/index.json)
Xunit framework to compose unit tests and integration tests
An optional [Xunit extension library](https://www.nuget.org/packages/Xunit.Microsoft.DependencyInjection/) is available here to help leverage dependency injection capability in writing Xunit integration tests

#### Introducing a grain

After adding a reference to the `DigitalSilo.Grain` package, the following class that defines the basics of a grain becomes available to inherit from:

```cs
public class Grain<TResponse> : Grain, IRequest<TResponse>
where TResponse : Response, new()
```
`TResponse` has to be an instance of the `Response` abstract class defined in the `DigitalSilo.Grain.Abstracts` namespace.

Let's imagine that we want to have the silo compute the perimeter of a rectangular. We will need grain to define a rectangular and a respective response class where the computed perimeter is stored.

```cs
using DigitalSilo.Grain.Abstracts;
using DigitalSilo.Grain;

public class PerimeterResponse : Response
{
    public int Perimeter { get; set; }
}

public class RectangularGrain : Grain<PerimeterResponse>
{
    public int Length { get; set; }
    public int Width { get; set; }
}
```

We can even define a `record` to represent a rectangular object and use that object's instance in the grain definition per the following slightly-refactored rectangular grain definition:

```cs
public record Rectangular
{
    public int Length { get; set; }
    public int Width { get; set; }
}

public class RectangularGrain : Grain<PerimeterResponse>
{
    public Rectangular Rectangular { get; set; }
}
```

#### Implementing grain processor

Processing a grain occurs in a separate class from the grain definition. To implement a processor class, one must derive the processor class from the following abstract class available in `DigitalSilo.Grain.Abstracts.Grain` namespace:

```cs
public abstract class GrainProcessor<TGrain, TResponse> : 
where TGrain : Grain<TResponse>, new()
where TResponse : Response, new()
```

So, the declaration of perimeter calculator class would look like the following example:

```cs
using DigitalSilo.Grain.Abstracts.Grain;

public class RectangularPerimeterCalculator : GrainProcessor<RectangularGrain, PerimeterResponse>
{
}
```

The following two abstract methods have to be overridden in the derived grain processor class:

```cs
protected abstract Task<TResponse> ProcessAsync(TGrain request, CancellationToken cancellationToken);
protected abstract Task FinalizeAsync();
```

`ProcessAsync` is the method that usually carries the business logic, and `FinalizeAsync` is a method that can optionally be implemented. Usually, tasks like closing a network connection, removing unnecessary files, cleanups, etc., would happen in `FinalizeAsync()` method. So, the implementation of `ProcessAsync` method to calculate the rectangular's perimeter would look like the following code snippet:

```cs
protected override Task<PerimeterResponse> ProcessAsync(RectangularGrain request, CancellationToken cancellationToken)
{
    var response = new PerimeterResponse
    {
        ResultCode = DigitalSilo.Grain.ResultCode.Success.GetCode()
        Perimeter = 2 * request.Rectangular.Width + 2 * request.Rectangular.Length
    };
    return Task.FromResult(response);
}
```

That's it! This pattern has illustrated how simple it is to define a grain and its associated processor. This pattern can easily be applied to more complex scenarios, e.g. database operations, sending emails, etc.

#### Validating grains

Digital Silo's validation mechanism has been constructed based on the [FluentValidation](https://fluentvalidation.net) library. One must complete the following two steps to have Digital Silo validate a grain's data:

Step 1) The grain's validator class should inherit the following abstract class:

```cs
using DigitalSilo.Grain.Abstracts.Grain.Validators;
public abstract class GrainValidator<TGrain>{ ... }
```
Step 2) The grain class should implement the following interface:

```cs
using FluentValidation;

namespace DigitalSilo.Grain.Abstracts.Validators
{
    public interface IValidatorProvider<T>
    where T : DigitalSilo.Grain.Grain
    {
        bool EnableAsyncValidation { get;  set; }
        IValidator<T> GetValidator();
    }
}
```

So, the validator class of `RectangularGrain` will look like the following code snippet:

```cs
using FluentValidation;

public class RectangularGrainValidator : GrainValidator<RectangularGrain>
{
    public RectangularGrainValidator()
    : base()
    {
        RuleFor(grain => grain.Rectangular).NotNull();
        RuleFor(grain => grain.Recyangular.Length).GreaterThan(0).When(grain => grain.Rectangular != default);
        RuleFor(grain => grain.Recyangular.Width).GreaterThan(0).When(grain => grain.Rectangular != default);
    }
}
```

A developer must augment the initial definition of `RectangularGrain` class per the following code snippet to accommodate validation:

```cs
public class RectangularGrain : Grain<PerimeterResponse>, IValidatorProvider<RectangularGrain>
{
    public Rectangular Rectangular { get; set; }
    public bool EnableAsyncValidation { get;  set; } = false;
    public IValidator<T> GetValidator() => new RectangularGrainValidator();
}
```

#### A real-world example

[This Github repository](https://github.com/DigitalSilo/digitalsiloexamples) contains three working Digital Silo grain examples whose source codes are available within the ["src"](https://github.com/DigitalSilo/digitalsiloexamples/tree/main/src) folder in the repo. 

`FibonacciGrain` computes Fibonacci Sequence, `ReverseFibbonacciGrain` computes Fibonacci Sequence in the reverse order, and `WorkerGrain` uses [Bogus library](https://github.com/bchavez/Bogus) to generate some fictitious users' data in the example repo. Please note the associated responses and validators in the examples.

The C# compiler produces three DLLs out of these examples' projects that can be uploaded to Digital Silo's storage for integrating them with the system.

So far, we have learned how to introduce a grain, its processor and validator. Once uploaded to the designated Digital Silo storage account, the grain becomes an integral part of Digital Silo. In the rest of this section, we will show you how to:

* Configure a grain
* Have Digital Silo process or run a configured grain

### Configure a grain

A developer must familiarize themselves with a few grain's properties to configure them properly to put the grain on work. So, let's learn about those properties by taking a look at the base grain definition in the `DigitalSilo.Grain` package:

```cs
namespace DigitalSilo.Grain
{
    public class Grain : IUnique
    {
        protected readonly Subject<Grain> _progressSubject;
        
        public Grain() => _progressSubject = new Subject<Grain>();

        public string UId { get; set; }
        public string ClientKey { get; set; }
        public IEnumerable<string> DependentGrainsUIds { get; set; }
        public Metadata Metadata { get; set; }
        public ProcessAttributes ProcessAttributes { get; set; } = new ProcessAttributes();
        [JsonIgnore]
        public IObservable<Grain> Progress => _progressSubject.AsObservable();
    }

    public class Grain<TResponse> : Grain, IRequest<TResponse>
        where TResponse : Response, new()
    {
        public Grain()
        : base() => ProcessAttributes.Stage = ExecutionStage.Seeded;

        public void OnFinishedProcessing()
            => _progressSubject.OnNext(this);

        public void OnFinalized()
            => _progressSubject.OnCompleted();

        public void OnError(Exception exception)
            => _progressSubject.OnError(exception);
    }
}

public class ProcessAttributes
{
    public ExecutionStage Stage { get; set; }
    public string TypeName { get; set; }
    public bool IsDurable { get; set; } = false;
    public double ExecutionDelayInMilliseconds { get; set; } = 0;
    public ResilienceSettings ResilienceSettings { get; set; } = new ResilienceSettings();
}
```

The following `Grain` class properties are the most significant to initialize upon submitting a grain's payload.

```cs
public string UId { get; set; }
public string ClientKey { get; set; }
public IEnumerable<string> DependentGrainsUIds { get; set; }
public ProcessAttributes ProcessAttributes { get; set; } 
```

#### UId

`UId` is A unique string, e.g. a `Guid` dedicated to one and only one grain upon its payload submission.

#### ClientKey

`ClientKey` is a string representing the client application's unique identity. This entry becomes useful when listening to the client's submitted grains' status reported by the backend.

#### DependentGrainsUIds

`DependentGrainsUIds` is a collection (array) of other grains' `UId`s that depend on the submitted grain. This property is valuable when developers want to line up grains in a queue.

#### ProcessAttributes

`ProcessAttributes` property is an instance of `ProcessAttributes` with the following three significant properties:

```cs
public string TypeName { get; set; }
public bool IsDurable { get; set; } = false;
public double ExecutionDelayInMilliseconds { get; set; } = 0;
```

##### TypeName

`TypeName` is the implemented grain's type name.

##### IsDurable

When set to true, it pushes the grain's execution to the durable context of Digital Silo, and it is useful when the longevity of a task is a mandate.

##### ExecutionDelayInMilliseconds

`ExecutionDelayInMilliseconds`, if set to a value greater than zero, delays the grain's execution for the given time.

#### A practical example of a configured grain

One of our grain examples in the examples repo is [Fibonacci sequence number generator](https://github.com/DigitalSilo/digitalsiloexamples/tree/main/src/DigitalSilo.Fibonacci). To have Digital Silo run this grain and generate 10 numbers, one should submit the following JSON payload to Digital Silo's Gateway Web API by tools like POSTMAN:

```json
 {
    "uId" : "83900c8cc4a77",
    "isDurable": "false",
    "MaxNumberOfTerms":10,
    "clientKey":"postman",
    "processAttributes": {
        "typeName" : "FibonacciGrain"
    }
 }
```

`MaxNumberOfTerms` is the input value of `FibonacciGrain` instructing Digital Silo to produce a maximum of 10 sequence numbers.

## Integration with Digital Silo

Integration with Digital Silo consists of the following three steps:

1. Acquiring a daemon Azure AD B2C token
2. Negotiation and listening to signalR
3. Making Gateway Web API calls

### Acquiring a daemon Azure AD B2C token

Please get in touch with Digital Silo for details.

### Negotiation and listening to signalR

The signalR element in Digital Silo, aka Watchdog, is the crucial component to report grains' responses and statuses to the subscribing client applications. The following URL is available to receive GET requests to have the client app negotiate and subscribe to signalR events:

```url
https://dsdemowatchdog.azurewebsites.net/api/negotiate
```

Digital Silo's Azure AD B2C protects the mentioned URL, and developers must also provide the following headers in their GET requests:

```url
x-ms-signalr-userid
x-functions-key
```

**Note 1:** Negotiation URL varies for every deployed instance of the system by Terraform.

**Note 2:** Please acquire the Digital Silo team about using the Azure AD B2C. Therefore a `Bearer` token is required in association with the `Authorization` header.

#### Populating x-ms-signalr-userid

`x-ms-signalr-userid` is the same as the application's client key discussed earlier.

#### Populating x-functions-key

`x-functions-key` will be available after running the deployment Terraform script.

### Responses received via signalR

The signalR client application receives signalR responses in JSON format, consistent with the grain's response object specified by the grain's developer *(derived from the `Response` abstract class mentioned earlier)*.

The events that the signalR reports are either of the following entries:

* onBegin: fired when processing a grain is about to begin
* onNext: fired when the grain is processed and the system is about to process the next one
* onError: fired when the grain resulted in an error
* onCompleted: fired when the grain completes

### Gateway Web API

Digital Silo Gateway Web API is the entry port to the system that collects grains' payloads to submit them to the serverless infrastructure to execute the grains. [Digital Silo Gateway API's Open API documentation](https://dsdemogatewayapp.azurewebsites.net/index.html) shows that only a handful of APIs is available, making the system integration easy and fun. Developers can choose their favourite programming languages to integrate with Digital Silo Gateway Web API.

### submitGrain POST action

This action collects a JSON payload representing one grain only.

#### Example

```json
{
    "uId" : "8900cc8c",
    "container":"test",
    "isDurable": "false",
    "MaxNumberOfTerms":10,
    "clientKey":"postman",
    "processAttributes":{
        "typeName" : "ReverseFibbonacciGrain"
    }
 }
```

### submitGrains POST action

This action collects a JSON payload representing a collection of grains. This POST action becomes handy when submitting a batch of grains is required.

#### Example

```json
[
    {
        "uId" : "12345",
        "container":"test",
        "typeName" : "WorkerGrain",
        "isDurable": "false",
        "input":10001
    },
    {
        "uId" : "90099",
        "container":"test",
        "typeName" : "WorkerGrain",
        "isDurable": "false",
        "input":10002
    },
    {
        "uId" : "904444",
        "container":"test",
        "typeName" : "WorkerGrain",
        "isDurable": "false",
        "input":10003
    },
    {
        "uId" : "18900",
        "container":"test",
        "typeName" : "WorkerGrain",
        "isDurable": "false",
        "input":10004
    }
]
```

### terminate/{grainUId} GET action

Invoke this GET action to have a running grain terminated. `{grainUId}` is the terminating grain's `UId`. If the system is unable to locate the running grain, it will report an error message.

#### Example

```
https://dsdemogatewayapp.azurewebsites.net/api/gateway/terminate/18900
```

### terminate POST action

Invoke this POST action to have a batch of running grains terminated. The request body should represent a string array comprising the UIds of the grains to terminate.

#### Example

```json
[
    "12345",
    "90099",
    "904444",
    "18900"
]
```

### next/{grainUId} GET action

This action has been reserved for system use only, and its invocation results in adverse situations. Developers must avoid calling this action even though it is available and publicly exposed.

### Gateway API's actions' Responses

Please consult Swagger's document to learn about each Web API call's respective response object. The response object returns a result code which may have one of the values per the following enum:

```cs
public enum ResultCode : short
{
    Unknown = -1,
    Success,
    Warning,
    InvalidObject,
    Error,
    Failed,
    Canceled,
    Unauthorized
}
```

**Important note:** Please note that the response received from Gateway API is not the processing result of the submitted grains as signalR is responsible for communicating such results. Gateway APIs' responses only reflect the results of the operations within the context of Gateway only.
