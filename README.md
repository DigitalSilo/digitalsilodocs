# Digital Silo

Digital Silo collects stateless tasks, aka **Grains**, and executes them in a scalable serverless environment on the Microsoft Azure Cloud technology. It accelerates the steps of making an application serverless-ready by eliminating the burden of infrastructure-driven implementation and letting developers concentrate on coding their business logic.
## Major components in high level

Digital Silo consists of the following major components:

* Storage or repository to seed the developed grains' DLLs
* Gateway Web API to collect grains' payloads
* Serverless infrastructure to process the submitted grains' payloads
* SignalR Service to report the grains' progress status

## Features

Gateway Web API is the entry point of the system to receive grains' payloads in JSON format. It has no role in running the business logic encapsulated in the grain. The only task that it carries out is to deliver the grains' payloads to the serverless infrastructure for further processing. However, it can manage a virtual queue of submitted grains' payloads as well. Developers can also define the grains' chaining keys in the payloads, and the Gateway can figures out how to line them up in a row to submit to the silo. The subsequent grain's payload in the queue is picked up and processed if its predecessor finishes successfully. Gateway will not send a grain's payload to the serverless part if its predecessor has already failed or terminated.

The serverless infrastructure is the entity that takes care of processing grains statelessly. It can track the progress of grains and resumes their execution from the point where they left off if the grains fail due to any reason. Depending on the deployment pipeline's configuration, the serverless infrastructure can run on a consumption plan or a bit more real-time mode, i.e. premium plan when necessary.

The SignalR Service provides a real-time communication channel between the client application interested in observing grains' status and the backend progress.

### High-Level Architecture diagram
The following diagram depicts the infrastructure and the major components of Digital Silo:

![Digital Silo HLA Diagram](assets/Digital%20Silo%20HLA%20Diagram.svg)

## Developing Grains
### Environment

A development environment is required to be able to develop grains. The development environment is the entire infrastructure of Digital Silo hosted on Azure and provisioned by a Terraform script found here. The service plans and the capacity of Azure services are at their minimal costs in the Terraform script. Should additional horsepower be required, higher-level SKUs can be incorporated in the Terraform script.

### Grain

A grain is a stateless component that encapsulates a specific business logic that runs throughout Digital Silo. A developer has to complete the following steps to prepare a grain to submit it to Digital Silo for processing:

* Add a reference to DigitalSilo.Grains assembly
* Implement `Grain<T>` abstract class to introduce a grain
* Implement `GrainProcessor` abstract class where the logic should reside
* Download the sample Xunit-powered test framework from this GitHub repo to test and debug the grain

### Silo, the processing component

The Terraform script provisions the entire Digital Silo infrastructure just within a few minutes on Azure. Once the infrastructure is ready, a storage account meant to retain the developed grains' DLLs becomes available. The infrastructure knows where to fetch and download the developed grains' DLLs from the storage and make them an integral part of the silo. The grains are ready and fully engaged in executing the business logic after receiving the respective payloads via Gateway.

Optionally a client application in C# or typescript may subscribe to the provisioned signalR Service instance to listen to grains' progress status.