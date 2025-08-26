+++
title = "Architecture Note: Use Cases in Modern Android"
[taxonomies]
tags = [ "android" ]
+++

In a layered app architecture there are two types of logic, separating _what_ the app should do from _how_ it should do it.

The data layer contains the majority of the app's business logic, which provides the unique value and key differentiators for your app – it is the implementation of the product requirements for your application's data.

The UI layer (_presentation_), consists of UI elements and UI state. The latter is derived from data entities provided by the data layer, _after_ any application business logic has been applied to it.

What is the optional _domain_ layer then, and what purpose does it serve in a modern app architecture?


<img src="/img/2025-08-24-android-architecture-use-cases.webp" />

#### ViewModel vs UseCase

A ViewModel provides access to the business logic, and produces a state stream consumed by UI logic (where further transformations occur alongside platform-specific implementation details such as the Android `Context` and `Resources`). The state can change because of upstream events, like a network response updating the application's data, or the user performing an action that would create or modify that data in some meaningful way as modeled by the product requirements. 

Although typically each ViewModel is associated with a single app screen, sometimes the ViewModel class can still grow and become harder to manage or even test. The business logic can then be extracted in `UseCase` classes, and that can be especially helpful in two situations:
1. You want to re-use a piece of simple logic.
   For example, if multiple ViewModels perform the same type of user input validation, or access a common resource such as a user preference, those ViewModels can inject the common use case as a dependency.
2. You want to extract complex logic out of the ViewModel.
   For example, if a user flow requires a complex orchestration of multiple data components to process a new UI event or input, this logic can be better managed and tested in isolation if it is in its own UseCase class.

#### Naming
Use the scheme _verb + noun (what) + UseCase_, e.g. `FormatInputUseCase` , or `GetPostHistoryWithAuthorsUseCase`.

#### A Mars Example
Let's _imagine_ a playful scenario in which a scientist, located inside a Martian base, uses an Android device to command and control a fleet of autonomous rovers. These can be deployed anywhere on the planet's surface to perform specific analyses on various environmental samples.

Suppose the operator selected a region, and would like to plan a mission to determine the feasibility of a base expansion in that area.

A complex use case such as this one may require combining different elements in order to perform the task:

```
class PlanBaseExpansionUseCase(
   roverRepository: RoverRepository,
   missionRepository: MissionRepository,
   validateCoordinateUseCase: ValidateCoordinateUseCase,
   makeRegionUseCase: MakeRegionUseCase,
) {
   operator fun invoke(vararg coordinates: Pair<Float, Float>): Flow<MissionPlanResult> {
      // validate the provided coordinates, create a region
      // fetch active rovers compatible with the mission type
      // create mission assignments
      // report live plan feasibility (e.g. success/failure enum or sealed types) back to the UI
   }
}
```

The use case may depend on data layer components as seen above – one or more repositories. To make it easier to test, interfaces should be used:

```
interface RoverRepository {
   suspend fun getAllRoversByCapability(capabilities: Set<RoverCapability>): Flow<List<Rover>>
}

class DefaultRoverRepository : RoverRepository { ... }
```

The main class implementation leverages a planetary network service to produce a state list of online rovers with the given capability.

In the test, the interface allows injecting a _fake_. For example, `FakeRoverRepository` returns a list of rovers compatible with the base expansion mission, but some of them may be running out of battery which would influence the probability of mission success. Fakes allow capturing a variety of stateful behaviors, and decouple the use case code, e.g. from specific network or database implementations.
#### Use Case recommendations
- do name your use case as a verb or function-like name
- do expose a single public member, e.g. allows callers to use the `invoke` operator function `planBaseExpansionUseCase(coordinates)`
- do make your executor function main-safe (the use case is responsible for managing its _concurrency policy_ by switching to the appropriate thread to do the work)
- don't add multiple responsibilities to the same use-case, instead create separate use cases (e.g. `ExecuteMissionUseCase`)
- do depend on interfaces in the use case constructor, instead of concrete implementations
- do return primitive types or domain/data entities from the use case
- don't make the use case dependent on any platform-specific classes (e.g. don't import anything `android.*`)
- do create a use case for simple business logic that is re-used a lot, or for complex logic making viewModels harder to maintain
- consider whether your business logic could more easily be kept in the data layer instead of creating a use case, unless you have a specific need to route everything through the domain layer, perhaps using a framework

Ultimately, use the pattern that most easily helps you implement your application's product requirements and produce value to customers. By separating component responsibilities and promoting testability, issues are easier to isolate and analyze, while you can maintain a high-quality, robust codebase, which scales quickly with the influx of new product requirements. 
