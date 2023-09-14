# RDS

RDS is an architectural pattern that stands for Repository, Datastore, Service. Each of these parts work together to offer some functionality that a Component will provide.  The RDS pattern is founded on [SOLID](https://en.wikipedia.org/wiki/SOLID) principles from the very beginning, and offers flexibility in development, future maintenance, and testing.  Further, RDS is simple and easy to understand.  Since it is compartmentalized, it allows multiple developers to work in the same component at the same time without conflict.  Segregation of the logic, storage, and access of resources is a fundamental goal of [Single Responsibility Principle](https://en.wikipedia.org/wiki/Single-responsibility_principle), the 'S' in SOLID.

There are some fundamental rules that should be followed when creating any code using the RDS pattern.

* Any operation in RDS should be assumed to be asynchronous. Therefore, all methods should use platform-specific asynchronous patterns. In iOS, for example, this should be with the async keyword.
* The Repository should always refer to Datastores and Services via an interface ( protocol or interface ), which accomplishes [Liskov Substitution Principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle) and [Interface Segregation Principle](https://en.wikipedia.org/wiki/Interface_segregation_principle).
* The Repository's Datastore and Service should be provided to it, instead of having it construct them itself. This is a fundamental principle of [Depe ndency Inversion Principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle).

![RDS](/images/rds.png "RDS")

In order to construct an RDS, let's look at the individual components, starting with the simplest.

## Service

The Service part of RDS is the mechanism that provides the functionality.  Often it will facilitate communication to a web service, such as [Cat Facts](https://alexwohlbruck.github.io/cat-facts/) or [News](https://newsapi.org/), but is not limited to just consuming remote resources.  It is entirely appropriate to use a Service approach for accessing an on-device source of behavior, such as current location.  In the past, a class that managed communication with a network resource was often very complex, since it had to handle the state of the network, the request, decode the response, and determine whether the response was valid or an error happened.  In RDS, this is no longer the case.  In most cases, the service should return fully serialized models.  However, the responsibility of serialization is on the Model, not the Service. A Service in RDS does one thing, only, and that is shuttle requests from the source to the destination and handle responses from the destination to the source.

Because of the single responsibility, Services should always be stateless.  There are certain cases where some behavior of one network call is predicated on the result of the previous one.  As long as this does not cross the boundary of a single call to the service, this should not violate the stateless mandate. It is the responsibility of the Repository, coordinated with the Datastore, to handle state on behalf of the Service.  

The Service's job is also to insulate the details behind it.  For example, a REST web service that accepts data could allow modification of the data via POST, PATCH, and DELETE.  The Service, however, should not expose as its API terms such as postData  and patchData or have arguments like HTTPMethod. These are implementation details that the Service's consumers should be able to ignore.

To comply with standard practice, components of an RDS triad should be first declared using an interface  ( protocol in iOS).  All methods should be declared with the assumption that they will be asynchronous, and so decorated with the async keyword or returning an appropriate Future/Promise. An example of a Service, written in Swift, that provides a caller with location search information can look like this:

```swift
import Foundation
import CoreLocation

public protocol LocationSearchService {
    func getLocationAt(
        _ coordinate: CLLocationCoordinate2D
        ) async throws -> Location
    
    func findLocationsNamed(
        _ name: String
        ) async throws -> [Location]
}
```

This Service provides two functions: one to get an instance of a Location type for a specific coordinate, and another that will return a list of Location instances starting with the name  passed in. Each declare that they are asynchronous operations and that they could throw an exception or error.
In order to actually use our Service, we'll need an implementation.  This early in development, there is no need to bother with real network code, so we will create a static LocationSearchService  that always returns the same values, which is useful for testing and initial confirmation of behaviors.

```swift
import Foundation
import CoreLocation

class StaticLocationSearchService: LocationSearchService {
    
    func getLocationAt(
        _ coordinate: CLLocationCoordinate2D
        ) async throws -> Location {
        await Task {
            Location(
                city: "Woodstock", 
                state: "Georgia"
            )
        }.value
    }
    
    func findLocationsNamed(
        _ name: String
        ) async throws -> [Location] {
        await Task {
            [
                Location(city: "Woodstock", state: "Georgia"),
                Location(city: "Woodstock", state: "New York")
            ]
        }.value
    }
}
```

## Datastore

The Datastore is the single source of state in an RDS triad. Its job is to serve as a cache of data across calls to the Repository, and as such can not be stateless.  This means that the instance of the Datastore is typically maintained for the life of the Dependency Container that holds it. Therefore, care should be taken in the design of an RDS to ensure that memory and storage of the device is respected.

Since the Datastore will be declared as an interface like all other parts of RDS, the actual mechanism of storage can be deferred to a later time, or changed as the needs of the Package change.  As long as the interface contains the appropriate API for saving and retrieving the data, any conformant classes should just work. Indeed, a composite Datastore that consists of multiple Datastores beneath it using different strategies for often-requested and lesser important, but still vital, data becomes trivial.

Because the underlying storage mechanism of state could change, making assumptions at initial design can be dangerous.  A Datastore that is assumed to always be in memory, and thus have no need to be declared as asynchronous, could one day be changed to store in a cloud environment.  This change would require substantial changes to both the Datastore and the Repository, which could potentially impact the consumers of the Repository, and their consumers as well.  What would have been a simple swap of one class to another becomes a massive effort across multiple parts of the app.

Continuing our previous example, a datastore that is tailor made to retain Location instances could be built to associate the coordinate value of the location to the actual Location instance as returned by the service.  If we declare our Location object definition as

```swift
public struct Location {
    let city: String
    let state: String
}
```

then our datastore interface could be declared as

```swift
public protocol LocationSearchDatastore {
    func get(at location: CLLocationCoordinate2D) async throws -> Location?
    func save(
        _ value: Location,
        at location: CLLocationCoordinate2D
    ) async throws
}
```

Declaring a Datastore that manages only a single type seems rather limiting. Using generics we can create a much more useful Datastore.  Further, if we are going to store models in durable storage, there needs to be some declared way that the type can be serialized and deserialized to a storable format.  In the iOS world, this is most easily accomplished by using the built-in JSON conversion as provided by the Codable protocol.

```swift
extension Location: Codable {}
````

To improve our code, let\'s declare a new Datastore using generics so it can store anything that conforms to Codable as long as the Key conforms to Hashable so we can guarantee the key is unique.

```swift
public protocol KeyedDatastore {
    func get<Key: Hashable, Value: Codable>(with key: Key) async throws -> Value?
    func save<Key: Hashable, Value: Codable>(
        _ value: Value,
        for key: Key
    ) async throws
} 
```

An example of an implementation of this store that retains the data only in memory could easily be accomplished using a Dictionary with the key and values matching any types.  In Swift 5, this would be done using Dictionary<AnyHashable:Any> while in Swift 6, you could use the any keyword to say [any Hashable : Any].

```swift
class KeyedMemoryDatastore : KeyedDatastore {
    func get<Key: Hashable, Value: Codable>(with key: Key) async throws -> Value?  {
        await Task.detached {
            self.map[key] as? Value
        }.value
    }
    
    func save<Key: Hashable, Value: Codable>(_ value: Value, for key: Key) async throws {
        await Task.detached {
            self.map[key] = value
        }.value
    }
    
    private var map: [AnyHashable: Any] = [:]
}
```

We now have a couple of approaches for saving our Location  objects.  One is to have no specifically typed datastore for Locations, and just use an implementation of the generic KeyedDatastore.  For most of our uses, this will be completely appropriate.  Another is to continue to use the LocationSearchDatastore as we declared it above, but for the actual implementation, we can inject an appropriate instance of a datastore for it to use internally.  This is useful when the core storage and fetching needs can be done by a generic store but some extra work on the data is needed on save and retrieve.  For example, if we went with the latter approach, our implementation of the LocationSearchDatastore would look like this:

```swift
class MainLocationSearchDatastore: LocationSearchDatastore {
    init(coreDatastore: KeyedDatastore) {
        self.coreDatastore = coreDatastore
    }
    
    func get(at location: CLLocationCoordinate2D) async throws -> Location? {
        try await coreDatastore.get(with: location)
    }
    
    func save(_ value: Location, at location: CLLocationCoordinate2D) async throws {
        try await coreDatastore.save(value, for: location)
    }
    
    private var coreDatastore: KeyedDatastore
}
```

Then, to use the declared datastore, you would do the following.  This may look rather verbose, but this is intentional.  By breaking the implementations up into logical parts, modification and maintenance become easy.  Managing this complexity is handled in the section on Dependency Containers.

```swift
let coordinate = CLLocationCoordinate2D(latitude: 34.14, longitude: -84.54)
let datastore: LocationSearchDatastore = MainLocationSearchDatastore(coreDatastore: KeyedMemoryDatastore())
try await datastore.save(Location(city: "Woodstock", state: "GA"), at: coordinate)
let saved: Location? = try await datastore.get(at: coordinate)

```

## Two-Level Datastores

One of the advantages of the Datastore approach relying on the DIP is the ability to make fundamental changes to how data is retained without directly impacting the code above.  For example, creating a Datastore that provides an L1/L2 cache approach, where the L1 is fast but expensive and L2 is slow but cheap, can easily be created using existing Datastore approaches.  All that would be needed is to create separate implementations of the KeyedDatastore , one that uses memory and the other using file, tied together with an implementation of a datastore that knows how to coordinate between them.

![two-level-data-store](/images/two-level-datastore.png "two-level-data-store")

We already have a "Fast and Expensive" implementation in the form of the KeyedMemoryDatastore. A "Slow and Cheap" implementation based around a file-based datastore could be as below:

```swift
class KeyedFileDatastore: KeyedDatastore {
    
    init(rootFolder: String) {
        self.rootFolder = rootFolder
    }
    
    func get<Key: Hashable, Value: Codable>(with key: Key) async throws -> Value?  {
        //Note: this code is unsafe!
        let url = FileManager //An extension method to ease path creation
            .folderCacheURL(self.rootFolder)
            .appendingPathComponent("\(key.hashValue)")
        let data = try Data(contentsOf: url)
        return try JSONDecoder().decode(Value.self, from: data)
    }
    
    func save<Key: Hashable, Value: Codable>(_ value: Value, for key: Key) async throws {
        let data = try JSONEncoder().encode(value)
        let valueType = "\(key.hashValue)"
        let filePath = FileManager //An extension method to ease path creation
            .folderCacheURL(self.rootFolder)
            .appendingPathComponent(valueType)
            .absoluteString
        
        let fileUrl = URL(string: filePath)! //Unsafe for production code!
        try data.write(to: fileUrl)
    }
    
    private let rootFolder: String
}

```

Once this is complete, a two-level datastore implementation would look something like below.  As you can see, it is implemented using two Datastores, and as long as each of them conform to the KeyedDatastore interface it doesn't matter about the actual implementation.  We can implement the class following the sequence diagram above like this:

```swift
class KeyedTwoLevelDatastore: KeyedDatastore {
    
    init(l1: KeyedDatastore, l2: KeyedDatastore) {
        self.levelOne = l1
        self.levelTwo = l2
    }
    
    func get<Key: Hashable, Value: Codable>(with key: Key) async throws -> Value?  {
        if let val: Value = try await levelOne.get(with: key) {
            return val
        } else if let val: Value = try await levelTwo.get(with: key) {
            try await levelOne.save(val, for: key)
            return val
        } else {
            return nil
        }
    }
    
    func save<Key: Hashable, Value: Codable>(_ value: Value, for key: Key) async throws {
        try await levelOne.save(value, for: key)
        try await levelTwo.save(value, for: key)
    }
    
    private var levelOne: KeyedDatastore
    private var levelTwo: KeyedDatastore
}
```

Once this is complete, it is trivial to now swap out the core datastore that is backing the `LocationSearchDatastore` by simply switching the implementing class from a memory-based datastore to a two level datastore, using a memory datastore as the level one cache and a filesystem datastore as the level two cache.

```swift
let coordinate = CLLocationCoordinate2D(latitude: 34.14, longitude: -84.54)
let datastore: LocationSearchDatastore = MainLocationSearchDatastore(
    coreDatastore: KeyedTwoLevelDatastore(
        l1: KeyedMemoryDatastore(),
        l2: KeyedFileDatastore(rootFolder: "locations")
    )
)
try await datastore.save(Location(city: "Woodstock", state: "GA"), at: coordinate)
let saved: Location? = try await datastore.get(at: coordinate)
```

## Repository

The final part of the RDS triad is the Repository, which provides the core logic.  It coordinates the requests and manages the fetching, caching, manipulation, and returning of data requested.  Like the Service, the Repository should be stateless and ephemeral, deferring any state needs to the Datastore.  In some cases the signature of the Repository and the Service will be identical.  Even though they share signatures initially, it is important that the pattern of creating separate interfaces be followed, because there is no guarantee that this will always be the case.  
Concrete implementations of Repository types should accept in their constructor instances of their Datastore and Service, and should be referenced in the Repository via their interfaces.  This allows changes to the implementations of the Datastore and Service to minimally impact the Repository. It further aids in testing by allowing mock instances of those dependencies to be passed in.

![location-search-repository](images/location-search-repository.png "location-search-repository")

In a standard use case where the Repository facilitates access to a network resource and caches responses to reduce repeat calls to the remote server, the flow of operation is very straightforward.

* The consuming component requests the Repository for data
* The Repository asks the Datastore if the data is available
* If the Datastore has the data, it returns it and operation completes
* If not, the data is requested from the server.
* If the data is found on the server, it is saved in the Datastore for the next time, and then returned
* If the data is not on the server, some indication is returned instead ( throw  or null response).

![ssd-location-search-repository](images/ssd-location-search-repository.png "ssd-location-search-repository")

Here we have a simplified view of an RDS in a "Request" action where the Datastore is considered the "Source of Truth".  In other words, the data that is in the Datastore is considered the valid data unless it does not exist, and fetches from the server only happen when it is missing.  The opposite approach, where the Service is assumed to have the "Source of Truth", would have the data fetched from the Datastore only when the service was unable to provide this data.  Obviously this doesn't take into account the cases where data could have an age limit.  Determining whether the data is valid based on whatever rules the business case makes is part of the architectural process, and would be reflected in the dynamic diagrams such as the Sequence or State.  Deciding where to put this logic is also an architectural decision. This logic could be built into the Datastore since the datastore is supposed to know what data it has. Alternatively, it could be handled by the Repository, especially when there is more complex business rules around it.  

To continue our previous example, a simple `LocationSearchRepository`  could be created as shown below.  First, we create our interface to declare the operations our Repository provides.  In this case, you can see that the Repository and Service both have identical signatures, but again, resist the urge to use a common interface for both.

```swift
import Foundation
import CoreLocation

public protocol LocationSearchRepository {
    func getLocationAt(_ coordinate: CLLocationCoordinate2D) async throws -> Location
    func findLocationsNamed(_ name: String) async throws -> [Location]
}
```

A concrete implementation of the `LocationSearchRepository`  would use our LocationSearchDatastore  and `LocationSearchService`  interfaces to provide the actual functionality.  Since the getLocationAt method caches the response, its behavior flows exactly as described above.  Based on product decisions, it was decided that there is no need to cache the responses from the `findLocationsNamed`  method, so it is simply a "passthrough" to the Service's method.  

```swift
import Foundation
import CoreLocation

class MainLocationSearchRepository: LocationSearchRepository {
    init(
        datastore: LocationSearchDatastore,
        service: LocationSearchService
    ) {
        self.datastore = datastore
        self.service = service
    }
    
    func getLocationAt(_ coordinate: CLLocationCoordinate2D) async throws -> Location {
        if let location = try await datastore.get(with: coordinate) {
            return location
        }
        
        let location = try await service.getLocationAt(coordinate) 
        try await datastore.save(location, for: coordinate)
        return location
    }
    
    func findLocationsNamed(_ name: String) async throws -> [Location] {
        try await service.findLocationsNamed(name)
    }

    
    private let datastore: LocationSearchDatastore
    private let service: LocationSearchService
}
```

Since we declared the implementation of `LocationSearchRepository`  with injectable lookup and cache, it becomes trivial to test the logic inside it.  We can provide our own test versions of `LocationSearchDatastore`  and `LocationSearchService`  to the Repository to return any data we like, such as a Service that always returns the same value, a Service that always fails, or a Service that returns corrupted or invalid data.  In addition, using Diagnostics, different Service and Datastore implementations can be swapped around to enhance testing.

## Provider

A Provider is simply an interface that declares that it is the source of some behavior.  Its main purpose is to serve as a communication channel between different parts of the app that need it, but not force them to become tightly coupled.  Since they are declared as interfaces, components that are in one logical area of the app can be served with information by parts in other logical locations without them having declared dependencies on each other.  This eliminates the need for things such as Singletons or Event Busses, which are traditional ways of making centralized "repositories" of behavior and data accessible to more remote code.

A Provider can behave in many ways, depending on the case needed. For example, if one of the Service components in a Package needs to communicate with a remote REST service, an API key is often needed.  Several approaches for getting the Service this API key is possible:

* save it as a hard-coded value in the Service implementation (worst)
* have the value contained in some resource file in the Package (not the best)
* have the Application provide the API key in some fashion (better)
* have the Service declare that it needs something to give it an API key, but leave the implementation of that to the user of the Package (best).  

```swift
import Foundation

public protocol LocationSearchAPIKeyProvider {
    func getLocationSearchAPI() async -> String
}
```

In the above case, our Location Search package declares that it needs something that can give it the API key for requesting data.  It doesn't care what that is, just as long as it conforms to the declared interface.  Because of this, the developers of the Location Search package don't have to worry about where the information will come from, leaving them able to focus on the specifics of the package behavior.

Another example of a Provider could be a `LoggingProvider` that can define the behavior that some logging class should implement and then be passed into RDS components. Then, in the application a logging library could be written that decides appropriate log destinations.  As long as that library conforms to the `LoggingProvider` interface, it could be passed to any RDS that declares it can use it.  This means that the app can control log destinations, the RDS components can write to logs as it sees fit, and the two concrete implementations are not coupled in any way.  Instead of one package needing to import another package that contains logging functionality, all the first package need do is declare that it wants some object that conforms to a given Provider interface, and that can be injected from any source.

Another example could be common functionality that, to enhance testability, should be easily mocked.  A NetworkProvider  and concrete implementation will been provided by a foundational package.  This Provider distills the core behavior needed for network communication.  By passing in an instance of this Provider to a Service, networking calls can be standardized across ALL Services.  Enhancements to this Provider then gives all Services the benefits of the new functionality.  Finally, testing the concrete implementations of Service components becomes trivial since a mock instance of the NetworkProvider  can be given to the Service at test time.

#### An Example

To illustrate how Providers would work to eliminate tight coupling, imagine that we need to create a Repository that facilitates fetching weather data. Our application is built to show weather information at the user's "current location", where current location is defined by business rules.  All data calls in our REST weather service require a location coordinate.  We could have a parameter for the coordinate at the request method of our repository.  This means that every time we call for data, we would have to know our "current location".

```swift
import Foundation

protocol WeatherRepository {
    func getCurrentConditions(at location: Location) async throws -> CurrentConditions
}
```

Consumers of this Repository would then be required to have not one dependency, the WeatherRepository but would also have to have something else that would provide them with a current location, doubling the "load" on the logic that this consumer needs.  The logic flow below demonstrates this clearly.

![provider](images/provider.png "provider")
In addition to the consuming component having two dependencies, it also will need to handle two different exceptional states, one being thrown by the current location component and another being thrown by the WeatherRepository . The logic is also more complex, requiring possible early exits from the component if the current location is not available for some reason.

Instead, if we had a Provider that gave callers the current coordinate that the user is viewing, it would mean that calls to get weather data becomes merely "Give me an observation" or "Give me hourly"; The Repository's Datastore and Service layers would be able to manage the request and ensure that if the data was already cached, that data is returned, and if not, the Service can get the appropriate data at the correct location.

```swift
import Foundation

protocol CurrentLocationProvider {
    func getCurrentLocation() async throws -> Location
}

protocol WeatherRepository {
    func getCurrentConditions() async throws -> CurrentConditions
}
```

If we assume that we have a different component to manage the user's selection of their "current location", its implementation could be declared to adopt the interface (conform to the protocol) and be passed in as a dependency to our Location Search repository instance.  Now, these to components are working together, but are not tightly coupled.

```swift
class MainWeatherRepository: WeatherRepository {
    init(
        service: WeatherService,
        locationProvider: CurrentLocationProvider
    ) {
        self.service = service
        self.locationProvider = locationProvider
    }
    
    func getCurrentConditions() async throws -> CurrentConditions {
        let location = try await locationProvider.getCurrentLocation()
        return try await service.getCurrentConditions(at: location)
    }
    
    private let service: WeatherService
    private let locationProvider: CurrentLocationProvider
}
```

At this point, the consumer of weather data cares only about one thing: getting weather data.  Everything else, such as locations, host names, api keys, and logging, have been completely abstracted away but are still completely controllable by the application, which is exactly what we want.
![provider](images/providers-2.png "provider")

This does, however, open up a potential problem.  Because we have now moved the request of all data needed to accomplish a lookup of current conditions, there are multiple places were these calls can fail.  Callers could get unexpected errors that they will not be prepared to handle.  Making this manageable is the subject of the next section.

# Error Handling

Any system with multiple "moving parts" has many points of failure.  Often finding where these happen is a major undertaking on their own.  There are many strategies for isolating faults, from simple and amateur "log everything and sort it later" to sophisticated state management systems that immediately catch when a state transition is illegal.  Striking a balance between robust error handling and writing hundreds of lines of boilerplate error checking is hard.  The best approach here is one that is well-documented and consistently applied throughout an application.

In iOS 8.0, Apple introduced structured error handling with the addition of the try/catch  keywords, much like most modern languages.  With this, error types can be well-defined and explicitly typed.  This makes declaring and identifying specific faults easier.  Further, it enables errors to be handled remotely from where they occured but still have enough information associated with them to be actionable.

Where Kotlin has the ability to declare exactly what exceptions a method could throw, Swift does not.  To ensure that exceptions (from here on referred only as "errors") are handled in a clear way, a hierarchy of error types is needed.  The easiest way to accomplish this is to declare an error type per component.  Then, this error can be subdivided into sub-errors that are specific to the failure reason.  These sub-errors should have properties that adds explicit details to why the error was thrown.

## Error Hierarchies

In order to handle an error thrown deep inside an API at the most appropriate point, enough information must be placed in the error at the site of the fault.  In addition, as the error is thrown up the call stack, the information needs to be retained, but not to the point where an unexpected error type is presented to a caller. To accomplish this, as errors are rethrown, they should be wrapped with an error that is specific to the current domain.

Since any well-formed architecture will have a series of dependencies injected into components, failures can come from a variety of sources.  A component that manages a user's current location could look very simple on the surface but have numerous ways it can fail.  If this component was built in in the standard RDS pattern, errors could be thrown from the Datastore or the Service.  Since we can't make guarantees on how the Datastore will store data, errors could be as varied as a failure to write to a filesystem to a authentication error when fetching from a cloud storage site.  If we simply rethrow any error we receive, error handling code at the Repository level could be presented with an error it isn't expecting and have no idea how to handle.  This means that either the error goes unhandled until it reaches the top of the call stack resulting in an application abort, or it is handled as a generic "just log it and continue" problem, leading to more subtle errors being introduced into the app.

To fix this, errors that are thrown should be prepared to carry with them extra information.  At the very least, the thrown error caught should be included in the new error that is rethrown.  This can be accomplished with the concept of an "inner error".  This way, errors thrown create a "chain of failure" as they are rethrown, and remote error handling sites can probe the error object for this information.  As the error object is unwrapped, like an onion, more and more explicit error conditions are revealed.  This way, detailed error reasons can be retained while keeping the error types that are thrown limited and easy to handle by the caller.
Continuing our example of Location Searching and Weather Data, we could create an error explicitly for the Location Search RDS system.  During development we identify two major points of failure: an error returned from the Location Search Service, and to be safe a "generic" error that we can add extra details if something truly unexpected happens.  To each error we add appropriate details for the problem, and an optional "inner error" type.

```swift
import Foundation

enum LocationSearchError: Error {
    case generic(
        message: String? = nil,
        inner: Error? = nil
    )
    
    case service(
        inner: Error? = nil
    )

    case datastore(
        location: Location? = nil,
        inner: Error? = nil
    )
}
```

We can now update our Location Search Repository to throw our LocationSearchError, or catch any errors and wrap them in the same.

```swift
import Foundation
import CoreLocation

class MainLocationSearchRepository: LocationSearchRepository {
    init(
        datastore: LocationSearchDatastore,
        service: LocationSearchService
    ) {
        self.datastore = datastore
        self.service = service
    }
    
    func getLocationAt(_ coordinate: CLLocationCoordinate2D) async throws -> Location {
        //If we fail to get from the datastore, just assume it was empty
        if let location = try? await datastore.get(at: coordinate) {
            return location
        }
        
        let location: Location
        do {
            location = try await service.getLocationAt(coordinate)
        } catch {
            throw LocationSearchError.service(inner: error)
        }
        
        do {
            try await datastore.save(location, at: coordinate)
        } catch {
            throw LocationSearchError.datastore(location: location, inner: error)
        }
        
        return location
    }
    
    func findLocationsNamed(_ name: String) async throws -> [Location] {
        do {
            return try await service.findLocationsNamed(name)
        } catch {
            throw LocationSearchError.service(inner: error)
        }
    }

    private let datastore: LocationSearchDatastore
    private let service: LocationSearchService
}
```

Here, you can see that there is no way that the Location Search Repository instance can throw any errors that are not LocationSearchErrors. Plus, the callers of these methods can get the specific reason for the failure, if they care, or just handle the issue as needed.  For example, in the do/catch block for saving a fetched location back into the datastore, the error returns the Location  it tried to save along with the reason for the error.  If the caller is ok with the fact that the value was returned from the server successfully, but failed to save in the datastore, they could just simply use the Location returned in the error.  This approach is somewhat flakey and would require extensive documentation to make the callers aware that this could happen.  A better approach might be to simply log the failure of the datastore writes as warnings and continue with the flow, though this risks these warnings being missed (or just ignored) leading to a fault hanging around longer than it should.  Like all approaches to error handling, there are pros and cons.

# Dependency Containers

At this point it should be clear that in this architecture there are many moving parts.  Any system that relies on Inversion of Control will, by design, have numerous dependencies that must be provided whenever an instance of a component is needed.  Having to provide multiple dependencies every time a Repository is needed puts a tremendous burden on the consumers of a package.  This can lead to shortcuts that violate fundamental rules for a Clean Architecture and make the entire system more brittle and harder to maintain.

The solution to this problem is to have a centralized mechanism that knows where all the dependencies it manages are.  Further it should have some means of creating these dependencies and controlling their lifetimes. This way, when a component is needed, a consuming object can request an instance and this central depository can create it and inject all of the dependencies into it.  This is the role of a Dependency Container.
Dependency Containers should be long lived and created as early in the app start-up process as appropriate.  Its tasks are to be a

* factory for Repositories, Services, and Providers
* maintainer of instances of objects that need to retain state(i.e. Datastores)
* manager of the complexity of getting instances of Repositories with many Providers.

Typically there will be a single Dependency Container (DC) per package.  The DC is named  PackageDependencyContainer and will have one or more makeComponent factory methods. Since the consuming component is only interested in the Repositories, they will be public .  All others should be private. Finally, all factory methods must NOT be asynchronous.  In many cases, a Repository will be needed in a place where calling an async method will not be possible.

Let's go back to our previous examples, where we now have two different Repositories: WeatherRepository , and LocationSearchRepository.  At some particular point, the user needs to get weather data at the current location.  They also want to ensure that log messages go to our off-site logging service.  To create an instance of the WeatherRepository  would mean that it must pass in numerous parameters.  This is a very cumbersome ask for the users of the package.  Our simple example above, in reality, requires many more Providers, such as an ApiKeyProvider , a UnitsProvider, and a HostnameProvider. Merely asking for a WeatherRepository  would result in five (or more?) parameters.

This would require you to know where to find all of these Providers at every point where a Repository was needed.  Obviously this is not ideal.  The Dependency Container tames this and creates a much more ergonomic approach to generating providers.  

Instead of passing in Providers when getting a Repository, we pass in external Providers at construction of the Dependency Container, or create them internally. The DC then keeps track of all of the dependencies that it knows about.  It keeps one instance of any Datastores that must be persistent, and as a Repository is asked for it will use these members to give the caller one.  

Ideally, the request for a Repository should have zero parameters.  Obviously this is not always the case but is the goal. Keep in mind that since Repositories and Services lack state, they are supposed to be "ephermeral"; Asking for one and then discarding it when finished should be the norm.  If you make it too cumbersome to make a Repository, you will tempt the user into retaining the instance they created, and thus break the pattern.

```swift
import Foundation

class WeatherDependencyContainer {
    init(
        hostProvoder: HostProvider,
        weatherAPIKeyProvider: WeatherAPIKeyProvider,
        locationSearchAPIKeyProvider: LocationSearchAPIKeyProvider,
        currentLocationProvider: CurrentLocationProvider
    ) {
        self.hostProvoder = hostProvoder
        self.weatherAPIKeyProvider = weatherAPIKeyProvider
        self.locationSearchAPIKeyProvider = locationSearchAPIKeyProvider
        self.currentLocationProvider = currentLocationProvider
        
        self.weatherDatastore = MainWeatherDatastore()
        self.locationSearchDatastore = MainLocationSearchDatastore()
    }
    
    func makeLocationSearchRepository() -> LocationSearchRepository {
        MainLocationSearchRepository(
            datastore: self.locationSearchDatastore,
            service: makeLocationSearchService()
        )
    }
    
    func makeWeatherRepository() -> WeatherRepository {
        MainWeatherRepository(
            datastore: self.weatherDatastore,
            service: makeWeatherService()
        )
    }
    
    private func makeLocationSearchService() -> LocationSearchService {
        MainLocationSearchService(
            hostProvider: hostProvider,
            apiKeyProvider: locationSearchAPIKeyProvider
        )
    }
    
    private func makeWeatherService() -> WeatherService {
        MainWeatherService(
            hostProvider: hostProvider,
            apiKeyProvider: weatherAPIKeyProvider,
            currentLocationProvider: currentLocationProvider
        )
    }
    
    private let hostProvoder: HostProvider
    private let weatherAPIKeyProvider: WeatherAPIKeyProvider
    private let locationSearchAPIKeyProvider: LocationSearchAPIKeyProvider
    private let currentLocationProvider: CurrentLocationProvider
    private let locationSearchDatastore: LocationSearchDatastore
    private let weatherDatastore: WeatherDatastore
}
```

Here is a typical Dependency Container.  As you can see, the external dependencies needed by the components of the Package are passed via the constructor of the DC.  Non-ephemeral objects are also preconstructed at this time, also.  When a LocationSearchRepository is needed, a concrete type is generated and the necessary dependencies are passed in when it is constructed.  The return type is the Protocol or Interface type, NOT the concrete implementation.

This example lacks an important use case, which is when one component managed by the DC is a dependency of another component in the DC.  Imagine if instead of the current location management system being an external RDS component, it was declared and written in this package.  Knowing that the current location's datastore would contain the information we need, you might be tempted to just make a parameter for CurrentLocationDatastore  as part of the WeatherService initializer.  This would be a serious mistake, because you have just made one type directly dependent upon another type.  The correct way here would be to have the CurrentLocationDatastore  also comply with the CurrentLocationProvider  interface.  This way, any object that conforms to the interface would be passable to the service.  In the future, if something needed to be changed, it would be a simple matter of swapping out the implementations instead of rewriting the signature of the weather service.

```swift
class WeatherDependencyContainer {
    init(
        hostProvoder: HostProvider,
        weatherAPIKeyProvider: WeatherAPIKeyProvider,
        locationSearchAPIKeyProvider: LocationSearchAPIKeyProvider
    ) {
        self.hostProvoder = hostProvoder
        self.weatherAPIKeyProvider = weatherAPIKeyProvider
        self.locationSearchAPIKeyProvider = locationSearchAPIKeyProvider
        
        
        self.weatherDatastore = MainWeatherDatastore()
        self.locationSearchDatastore = MainLocationSearchDatastore()
  self.currentLocationDatastore = MainCurrentLocationDatastore()
    }
    
    func makeLocationSearchRepository() -> LocationSearchRepository {}
    
    func makeWeatherRepository() -> WeatherRepository {}
    
    private func makeLocationSearchService() -> LocationSearchService {}
    
    private func makeWeatherService() -> WeatherService {
        return MainWeatherService(
            hostProvider: hostProvider,
            apiKeyProvider: weatherAPIKeyProvider,
            currentLocationProvider: currentLocationDatastore as CurrentLocationProvider
        )

    }
    
    private let hostProvoder: HostProvider
    private let weatherAPIKeyProvider: WeatherAPIKeyProvider
    private let locationSearchAPIKeyProvider: LocationSearchAPIKeyProvider
    private let locationSearchDatastore: LocationSearchDatastore
    private let currentLocationDatastore: CurrentLocationDatastore
    private let weatherDatastore: WeatherDatastore
}

extension MainCurrentLocationDatastore: CurrentLocationProvider {
    func getCurrentLocation() async throws -> Location {
        ...
    }
}
```

## Organizing Dependency Containers

Because Dependency Containers are the core mechanism for fetching Repositories, it is necessary to maintain them in some organized fashion.  To that end, Dependency Containers can contain references to other Dependency Containers.  Further, since DCs have somewhat limited logic, making unit and integration testing unnecessary, it is acceptable that one DC create another DC directly without insisting that they use Dependency Injection.

It is likely that the Application being created will have RDS triads build for its own purpose.  Therefore, it too will have Providers, and will need some mechanism for handling the resultant complexity.  Because of that, an app-level Dependency Container will be needed.  The AppDependencyContainer  can then be an excellent point where Package-level DCs can be constructed for use in the rest of the app.

Given our previous example, our App could create an `AppDependencyContainer`  and have as a member property a reference to a constructed `WeatherDependencyContainer`.  This container can be constructed lazily, leaving the initialization until needed.  Then, instead of numerous Dependency Containers having to be located in the application code, only a single instance of the AppDependencyContainer  will need to be.  Any components that need a Repository will be able to then ask that single DC for a child DC and be able to fetch the Repository that they need.

Finally, it is **vitally important** to have *only a single instance of any particular DC*. Since DCs contain the state of the various RDS parts, having multiple instances means multiple instances of datastores.  This WILL lead to state mismanagement and subtle infuriating errors.  

# Putting It All Together

Now that we have explained the basic moving parts needed to create a Package, we can start the process of creating one.  The initial analysis our specific needs should make some determination of the behavior that the Package should provide, the data that will be passed back and forth, and the potential lifespan of the data.  With these data in hand, we can start defining the various parts of our Package.

## Models

The arguments passed back and forth from the Services dictate how to build the models.  For each model, some mechanism is needed to convert the object into something that can be passed to the Service.  In most cases, this will be JSON.  Because of this, you should ensure that these models use the platform specific mechanisms needed to serialize and deserialize the contents to this format.

Care should be given in creating the easist implementation of Models for use by the client. In some cases it is necessary to create two different "implementations" of a Model: one for Client use and one for Service use.  The notable reason here might be because the Service needs extra information that is provided by a Provider that the Client use of the Model doesn't care about.  Breaking the implementation of the Model into internal and external instances will improve the user experience of the Package and lead to less misuse of the Model.

## Service

Since the Service is the means by which behaviors are provided, this is the next place to work on.  Like all components in a Package, start creating a protocol  or interface  to serve as an interface.  Then, provide method declarations that will facilitate the functionality.  Remember to declare the return types of all methods are asynchronous in the platform correct manner, even those that return is void, so that the calls can be async  in their implementation classes.

When creating the signatures of the Service interface, remember to abstract the actual details of the mechanism you are connecting to.  Don't use terms such as "put", "post", and "patch", for example, when the Service is abstracting a REST service.  Prefer more generic terms such as "save", "update", or "delete".

## Datastore

Once a Service and its Models have been defined, it is time to move to defining the Datastore.  Recall that the purpose of the Datastore is to limit the need for calls to the Service, or just to retain data for future use in those RDS implementations that lack a Service.  Understanding the lifespan of the Model is key to creating the Datastore.

For weather data, as an example, the lifespan of an object can be measured by the Cache-Control  header's max-age  value.  Models can either declare a "lifespan" parameter or perhaps be wrapped by a generic class that provides a "time to live" property and methods to handle it. This model could retain both the Weather object and a date object for when the weather object has expired.  Instead of saving weather data, the Datastore implementations are designed to save instances of this type of object.  When a record is requested from the WeatherDatastore , the record requested is checked both for the presence of the record AND for the expiration.  If it is expired, null is returned, exactly as if the record was not there.  In the normal RDS flow, this then triggers the Repository to request for this record from the Service, thus getting the most up-to-date data available.

For data where the Service contains the "source of truth", the datastore would only be fetched if the service call fails for some reason.  In this case, care must be taken to ensure that server failures not become "log and forget," since long-term faults in the service will lead to stale data being used by the app continuously.

## Repository

The Repository is the next part that you will create.  In cases where the Repository and Service have the same signatures, defining the interface is simply a matter of cutting and pasting.  Do not reuse the same interface for both, though, because there is no guarantee that they will remain the same in the future.  Ideally, methods in a repository should have as few parameters as possible.  The goal is to make calling the methods of a Repository as friction-free as you can.  If you can inject a Provider when a Repository is created, instead of passing in the same value for a call every time it is needed, do so.  Finally, try to build your Repositories as if the caller doesn't even know there is a cache or a network service behind it.  If something happens in the app that causes the cache for an RDS to no longer be valid, consider injecting a cache clear mechanism into the Datastore as a Provider when the Datastore is constructed.  Then, when the app behavior requires the cache being flushed, it just happens, completely behind the scenes.

When you create the implementation of your Repository, the arguments for the constructor should be of the type of your Datastore and Service interfaces, and accept your implementations of the Datastore and Service; Repositories should never create their own Datastores or Services.  Indeed, NOTHING in this architecture should create its own dependencies. Also, any Providers that will pass in functionality should also be injected at construction.

It is important to remember that Repositories do not have any state in them, and should be treated as ephemeral.  All state that needs to happen during interaction with a Service should be maintained by the Datastore, or Providers that facilitate two-way communication.  In any respect, avoid creating multiple sources of truth.  In our previous example, we had a Weather RDS and a Current Location RDS, with a CurrentLocationProvider  that gave the Weather Service the gps coordinates to use for weather lookups.  Since the Current Location RDS provides the location coordinates, whenever the coordinates are needed, use the Provider to get the information.  You should NOT get the coordinates, save them in a local variable, and then use that.  If you did this, you have now created two sources of "truth", one of which can be out of step with the original.

## Providers

When creating the various components in an RDS set, it will become obvious where extra data will be needed.  Extra functionality that will be controlled by outside entities will need to be inserted into the various parts of each Package.  In order to accomplish this in a SOLID fashion, you will define Providers that will give the information and functionality needed.

Early on in developing the Package, identify these needed Providers and create interfaces for them.  Then, to keep development of the package moving, create conformant classes and use them.  When it comes time to integrate the Package into the app as a whole, those Provider implementations can be swapped out for real ones.  

## Errors

When creating the parts of the Package, create errors that provide enough information to callers why a particular request failed.  The caller may not have the ability to fix the problem, but having the reason why can assist in diagnosing it for the future.  Errors should be granular enough to explain what happened at the type level, but not so granular that the caller has to potentially handle dozens of possible fault reasons.  When in doubt fewer error types with more explicit information contained in them is better than more errors with less information.

## Dependency Container

The final part that must be created to make a functional Package is the Dependency Container.  When defining a DC, think about the Providers that will be given to it from outside and those that will be generated internally.  Pass in those external Providers, and make factory methods for the internal ones.  Remember that since state is only ever retained in a Datastore, creating a Repository that declares that it can be a Provider for data, and then passing that Provider to another Repository that needs it, is completely appropriate, as long as it is passed AS the Provider.

Do not make the factory methods that return components asynchronous.  This will make it hard to request a component from a synchronous method.

# Final Thoughts

This document has attempted to explain both the *why* and the *how* of constructing a Package for any application. That being said, no guide is useful unless it is understood and followed.  Therefore, if any part of this is unclear, please reach out to the Architect for a more detailed explanation.  Guessing and then creating something that falls outside the boundaries of the guide will just mean wasted work, since this architecture should be followed exactly.  At no point, though, should this guide be seen as inviolate.  If there are areas where deviating from this becomes necessary, let's discuss it and come up with solutions together; adapting this guide will be part of the overall process.
