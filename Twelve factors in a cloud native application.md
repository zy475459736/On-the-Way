Learned and Captured from <Cloud Native Java>

### Core Foundational Ideas

A set of **core foundational ideas** for building applications

* Use **declarative** formats for setup automation, to minimize time and cost for new developers joining the project;
* Have a clean contract with the underlying operation system,offering **maximum portability** between execution environments;
* Are suitable for **deployment** on modern **cloud platforms**, obviating the need for servers and systems administration;
* **Minimize divergence** between development and production, enabling **continuous deployment** for maximum agility;
* And can **scale up** without significant changes to tooling,  architecture, or development practices.

> * 新加入开发人员的学习成本低；
> * 最大的可移植性；
> * 在云平台上可部署
> * 开发环境和生产环境分歧小，可以尽可能敏捷地进行持续开发
> * 伸缩性

### The Twelve-factors

The former part has explicitly state the value proposition of building applications that follow the twelve-factor methodology.These ideas break down further into a set of constraints—twelve indicidual factors that distill core ideas into a collection of opinions for how applications should be build.

* Codebase 

  One codebase tracked in revision control, many deploys.

  > Source code repos for an application should contain a single application with a manifest to its Dependencies.

* Dependencies 

  Explicitly declare and isolate dependencies.

  >  All the dependencies should be available from an artifact repos.

* Config 

  Store Config in the environment

  > The configuration of the apps should be driven by the environment.
  >
  > App settings should be stored as environment variables, making them easy to be changed without deploying config files; any divergence in the app from environment to environment is considered as an environment config, rather than in application.

* Backing services 

  Treat backing services as attached resources

  > A backing service is any service that the twelve-factor appp consumes as a part of its normal operation.
  >
  > Examples:database|api-driven restful web services and so on
  >
  > Backing services are considered to be resources of the app, and are attached to the app for duration of operation.

* Build, release, run 

  Strictly separate build and run stages

  > * Build stage
  >
  >   Compile/Bundles the source code of an app into a package, and the package it created is referred to as a build.
  >
  > * Release stage
  >
  >   This stage take a build and combines it with a config, 
  >
  >   * and then is ready to be operated in an execution environment.
  >   * should have a unique identifier
  >   * should be added to a directory in case of rollback need.
  >
  > * Run stage
  >
  >   Commonly referred to as the runtime, runs the application in the execution environment for a selected release.
  >
  > By seperating each of these stages into separate processes, it is impossible to change an app's code at runtime.The only way to achieve it is to initiate the build stage to create a new release ,or to initiate a rollback to deploy a previous release.

* Processes 

  Execute the app as one or more **stateless** processes

  > The only persistence is through a backing service.
  >
  > Stateless or not: Tear down the app's execution environment and recreated without any loss of data.

* Port Binding 

  Export services via port binding

  > Twelve-factor applications are completely self-contained which mean no webserver is needed at all.
  >
  > Each app will expose access to itself over an HTTP port that is bound to the app in the execution environment , and a routing layer will handle the requests from a public hostname by routing to the app's execution environment and the bound HTTP port.

* Concurrency 

  Scale out via the process model

  > Application should be able to scale out processes or threads for parallel execution of work in an on-demand basis.

  Applications should **distribute work concurrently** <u>deplending on the type of work that is used.</u>

  * Some scenarios that require <u>data processing jobs</u> that are executed as <u>long-running tasks</u> should utilize **executors that are able to asynchronously dispatch concurrent work to an available pool of threads**.

  The twelve-factor app must be able to **scale out horizontally** and **handle requests load-balanced** to multiple identical running instances of an application.

  By **ensuraing applications are designed to be stateless**, it becomes possible to handle heavier workloads by scaling applications horizontally across multiple nodes.

* Disposability 

  Maximize robustness with fast startup and graceful shutdown

  > * Designed to be **disposable**: can be stopped at any time during process execution and gracefully handle the disposal of process.
  >
  > * Minimize the startup time as much as possible.
  >
  >   Short startups reduce the time it takes to scale out application instances to respond to increased load.
  >
  >   If an app's processes take too long to start, there may be reduced availability during a high-volume traffic spike that is capable of overloading all available healthy application instances.By decreasing the startup time of applications to just seconds, new scheduled instances are able to more quickly respond to unpredicted spikes in traffic without decreasing availability or performance.

* Dev/prod parity 

  Keep Development, Staging and Production as similar as possible

  Three types of gaps/divergence need to be prevented.

  * Time gap 

    Development changes should be quickly deployed into production

  * Personal gap

    The developers who make the changes should be closely involved with its deployment into production and closely monitor the behavior afterwards.

  * Tools gap

* Logs

  Treat logs as event streams

  > Logs should be written as <u>an ordered event stream</u> to stout.
  >
  > Application NEVER manage the storage of own log files.
  >
  > INSTEAD, the collection and archival of log output for an application should be handled by the execution environment.

* Admin processes 

  Run admin/management tasks as one-off processes

  > Sometimes developers of the app need to run one-off administrative tasks , which includes <u>database migrations</u> or <u>one-time scripts</u> that have been checked into the app's source code repository.
  >
  > These kinds of tasks are considered to be admin processes.
  >
  > Should be run in the execution environment of the app, with scripts checked into the repos to maintain consistency between environments.