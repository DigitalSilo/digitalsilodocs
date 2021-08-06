# Digital Silo

Digital Silo collects stateless tasks, aka **Grains**, and executes them in a scalable serverless environment on the Microsoft Azure Cloud technology. It accelerates the steps of making an application serverless-ready by eliminating the burden of infrastructure-driven implementation and letting developers concentrate on coding their business logic.

## Major components in high level

Digital Silo consists of the following major components:

* Storage or repository to seed the developed grains' DLLs
* Gateway Web API to collect grains' payloads
* Serverless infrastructure to process the submitted grains' payloads
* SignalR Service to report the grains' progress status

## Features

Gateway Web API is the entry point of the system to receive grains' payloads in JSON format. It has no role in running the business logic encapsulated in the grain. The only task it carries out is to deliver the grains' payloads to the serverless infrastructure for further processing. However, it can manage a virtual queue of submitted grains' payloads as well. Developers can also define the grains' chaining keys in the payloads, and the Gateway can figures out how to line them up in a row to submit to the silo. The subsequent grain's payload in the queue is picked up and processed if its predecessor finishes successfully. Gateway will not send a grain's payload to the serverless part if its predecessor has already failed or terminated.

The serverless infrastructure is the entity that takes care of processing grains statelessly. It can track the progress of grains and resumes their execution from the point where they left off if the grains fail due to any reason. Depending on the deployment pipeline's configuration, the serverless infrastructure can run on a consumption plan or a bit more real-time mode, i.e. premium plan when necessary.

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

Let's imagine that we want to have the silo compute the perimeter of a rectangular. We will need a grain to define a rectangular and a respective response class where the computed perimeter is stored.

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

The initial definition of `RectangularGrain` class must be augmented per the following code snippet to accommodate validation:

```cs
public class RectangularGrain : Grain<PerimeterResponse>, IValidatorProvider<RectangularGrain>
{
    public Rectangular Rectangular { get; set; }
    public bool EnableAsyncValidation { get;  set; } = false;
    public IValidator<T> GetValidator() => new RectangularGrainValidator();
}
```