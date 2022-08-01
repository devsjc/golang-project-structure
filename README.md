# Golang Project Structure

Information on how to structure a go project, based on the Hexagonal Architecture pattern.

## Layout pattern

The layout revolves around three main high-level concepts:

- *Handlers* are the point at which services will begin processing a request. These may be REST endpoints or responsing to events from a messaging service such as Kafka or PubSub
- *Services* are where the business logic resides
- *Repository* is where the service will write or retrieve data from. This can be a database such as MongoDB or a Kafka topic. If we send data to it, it is considered a repository.

This is inspired by the [hexagonal architecture pattern](alistair.cockburn.us/hexagonal-architecture), that is to say it is built upon *ports* and *adapters*. Events from the outside world will arrive at a port. The core applicaiton is unaware of the source or format of the input. When the service needs to send something to the outside world it does so using an outbound *port*, which will use an *adapter* to transform the message into the appropriate external format. We call inbound ports *handlers* and outbound ports *repositories*. They will handle converting to and from transport and storage formats so the actual service code will only work with custom Go types that we define.

These concepts are transformed into the following structure for a service's source code:

```yaml
myservice: # sources root
  cmd:
    myservice:
      - main.go # the entrypoint to the service
  internal:
    handler:
      rest:
        - api.go # defines an adaptor for reading from a rest endpoint
      kafka:
        - kafka.go # defines an adaptor for reading from kafka
    repository:
      cosmos:
        - cosmos.go # defines an adaptor for writing to cosmos
    service:
      - service.go # the business logic of the application
    - myservice.go # global structs and type definitions for the internal logic
  vendor:
  mock:
  - .gitignore
  - go.mod
  - README.md
```

The first folder is the `cmd` folder. This folder has a subfolder that matches the name of the service and contains a `main.go` file. This is the main application for the service and where all dependencies are wired up and handlers started for listening to requests.

Next comes `internal`, which holds special meaning for Go tools: the `go` command will not allow packages located under the `internal` folder to be imported outside of the source subtree. By using this, casual dependance on the service in other services is prevented. When looking to reuse some functionality from existing services, the code in question should either become a shared library or just be copied into the new service (the [Go Proverbs] https://go-proverbs.github.io/) recommend a little copying).

Within `internal` folder there are three subfolders, `handler`, `repository` and `service`, which are the main components making up the application. In the root of the `internal` folder there is also a Go file, which holds the types and interfaces used throughout the codebase. By having these types declared at the top level, other developers have a reasonble starting place to look and see what the service is about, and what data it is operating on. Also by having types at the top the flow of dependencies is downward, avoiding circular dependencies between the subpackages. For more on this, see Brian Ketelsen's [talk on Go best practices](https://www.youtube.com/watch?v=MzTcsI6tn-0).

The `handler` package will contain subpackages based on what it responds to. The idea being that when a service exposes an interface, it has the appropriate handler subpackage so in future the technology that is being listened to can be swapped out without affecting the core of the service.

The `repository` folder is intended for packages that interact with a sink for a service's data. This could mean writing to a database or sending an event to Kafka.

Finally under `internal` we see the `service` folder. This is for the service package which contains all the business logic of the application. This package will have dependencies such as repositories *dependency injected* at startup by the main function under the `cmd` folder. The service should be coded to only accept types based upon interfaces so that it can be easily unit tested.

Back at the root level we see two folders `mock` and `vendor`. The `mock` folder holds code generated via the Go mock tool to facilitate unit testing the service, and should not be committed to VCS - instead opting for inline `go generate` commands to generate the code each time and prevent manual altering. The `vendor` folder holds all the source code of the packages the service consumes. Vendoring offers the strongest guarantee of repeatable builds, provides eay debugging into dependant code and provides a clear history of what changes were made and when. It also enables the use of private go libraries in locations that do not have read permissions on the VCS repository containing the private library. See [Bill Kennedy's blog post on vendoring](https://www.ardanlabs.com/blog/2020/04/modules-06-vendoring.html).
