# Section 2. Fundamental Dependency Injection Techniques

## Comparison Between Fundamental Dependency Injection Techniques

**Constructor injection**

장점

- simple
- constructor signature reflects dependencies
- injected fields can be finalized
- easy to mock services in unit tests
단점
- none



**Method injection**

장점

- method signature reflects dependencies
- can happen after construction

단점

- not as explicit as constructor injection
- can lead to implicit ordering requirements(temporal coupling)



**Field injection(property in Kotlin)**

장점

- can happen after construction

단점

- all cons of method injection(non explicit, temporal coupling)



```kotlin
class Client(private val service1: Service1) { // Constructor injection
	private var service2: Service2? = null
	

	lateinit var service3: Service3? = null // Property injection
	
	fun setService2 (service2: Service2) { // Method injection
		this.service2 = service2
	}

}
```



Constructor injection이 가장 좋지만, 쓸 수 없는 경우도 존재
The service is not available when the client is instantiated
You do not instantiate the client(e.g. Activity)
Framework imposed limitation on constructor(e.g. Fragment)

Dependency injection is just passing services into clients from outside!



## Architectural Patterns

**Design patterns**: general, reusable solution to a commonly occurring problem within a given context in software design

- Observer
- Singleton
- Strategy
- etc..



**Architectural patterns**: broader scope, not as detailed as design patterns
Popular architecture patterns

- Presentation: MVx
- notifications: publish-subscribe architecture
- State change management: event-driven architecture 
- Loosely defined and is a lot of space for customization



## Dependency Injection Architectural Pattern


DIAP(Dependency Injection Architectural Pattern)
Segregation of application's logic into two sets of classes
	functional set: contains classes that encapsulate core application's functionality
	construction set: contains classes that resolve dependencies and instantiate objects from functional set
Functional and construction sets must be disjoint



## Fundamental Dependency Injection Techniques vs DIAP

Dependency injection architectural pattern vs Fundamental techniques in dependency injection 

DIAP
- different levels of abstraction(class vs application)
- fundamental techniques are the low-level implementation details of DIAP

