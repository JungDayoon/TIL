# Section 8. Additional Dagger Conventions

## Android Services and Dialogs

## Static Provider Methods and Component Builders

성능 향상을 위해 provider methods를 static 하게 만들기.

kotlin에는 static 개념이 없기 때문에 companion object 내에 method들을 넣어주면 된다.

```kotlin
// 수정 전
@Module
class ActivityModule(
        val activity: AppCompatActivity
) {

    @Provides
    fun activity() = activity

    @Provides
    @ActivityScope
    fun screensNavigator(activity: AppCompatActivity) = ScreensNavigator(activity)

    @Provides
    fun layoutInflater(activity: AppCompatActivity) = LayoutInflater.from(activity)

    @Provides
    fun fragmentManager(activity: AppCompatActivity) = activity.supportFragmentManager

}

// 수정 후
@Module
class ActivityModule(
        val activity: AppCompatActivity
) {

	// Non-static한 property(activity)를 static method 내에 넣을 수 없기 때문에 
	// 해당 method는 companion object로 옮길 수 없음
    @Provides
    fun activity() = activity
	
    companion object {
	@Provides
        @ActivityScope
        fun screensNavigator(activity: AppCompatActivity) = ScreensNavigator(activity)

        @Provides
        fun layoutInflater(activity: AppCompatActivity) = LayoutInflater.from(activity)

        @Provides
        fun fragmentManager(activity: AppCompatActivity) = activity.supportFragmentManager
    }
}
```



`activity()`를 없애려면? -> Bootstrapping dependency (생성자의 property `activity`)를 없애야 함

```kotlin
@ActivityScope
@Subcomponent(modules = [ActivityModule::class])
interface ActivityComponent {

    fun newPresentationComponent(): PresentationComponent

  	// 추가하기
  	@Subcomponent.Builder //-> 해당 클래스가 @Subcomponent이기 때문에 똑같이 맞춰주기
  	interface Builder {
      	@BindsInstance fun activity(activity: AppCompatActivity): Builder
      	fun activityModule(activityModule: ActivityModule): Builder
      	fun build(): ActivityComponent
    }
}
```

```kotlin
open class BaseActivity: AppCompatActivity() {
  
    val activityComponent by lazy {
        appComponent.newActivityComponentBuilder()
                .activity(this) // -> 이로써 ActivityModule의 생성자 property인 activity를 제거할 수 있음
                .activityModule(ActivityModule)
                .build()
    }

}
```

```kotlin
@Module
object ActivityModule {

    @Provides
    @ActivityScope
    fun screensNavigator(activity: AppCompatActivity) = ScreensNavigator(activity)

    @Provides
    fun layoutInflater(activity: AppCompatActivity) = LayoutInflater.from(activity)

    @Provides
    fun fragmentManager(activity: AppCompatActivity) = activity.supportFragmentManager


}
```

-> 모두 static method만 가지기 때문에 object로 변경



### Dagger Conventions (8)

- Dagger generates more performant code for static providers in Modules (use companion object or top-level object in Kotlin)
- `@Component.Builder` (or `@Subcomponent.Builder`) designates inner builder interface for Component
- `@BindInstance` allows for injection of "bootstrapping dependencies" directly into Component builders



## Type Bindings

```kotlin
@Module
object ActivityModule {
	@Provides
	@ActivityScope
	fun screensNavigator(activity: AppCompatActivity): ScreensNavigator = ScreensNavigatorImpl(activity)
}
```

Dagger가 interface와 specific implementation을 어떻게 연결지을지 알려주기 위해서 명시적으로 provider의 return 타입을 기재해주어야 한다.

*`ScreensNavigator`: interface, `ScreensNavigatorImpl`: specific implementation

-> ScreensNavigatorImpl에서 constructor injection을 통해 주입대상임을 Dagger에게 알리고 ActivityModule에서 해당 provider를 지우는 방법은? -> `ScreensNavigator cannot be provided without an @Provides-annotated method.`라는 에러가 뜸

-> Dagger가 ScreensNavigatorImpl라는 object는 찾을 수 있지만, 실제 activity에서 주입받는 객체는 ScreensNavigator이므로 Dagger는 해당 mapping을 스스로 만들 수가 없음

-> 이럴 때 사용하는 annotation이 `Binds`

```kotlin
@Module
abstract class ActivityModule {
	@Binds
  	abstract class screensNavigator(screensNavigatorImpl: ScreensNavigatorImpl): ScreensNavigator
}
```

해당 annotation을 통해, Dagger는 누군가가 ScreensNavigator를 필요로 할 때, ScreensNavigatorImpl을 찾아서 해당 객체를 주게 된다.



**scope 지정** 

- ScreensNavigatorImpl에 한 경우
  - 객체가 필요할 때 마다 Dagger는 특정 activity scope 내에서는 같은 인스턴스를 제공함
- ActivityModule > screensNavigator 함수에 한 경우
  - interface를 activity scope로 제공함. 즉, specific implementation을 직접적으로 제공하는 경우에는 새로운 인스턴스가 만들어지게 됨

-> 다음과 같은 차이로 corner cases가 생겨날 수 있기 때문에 scope 지정에 유의하기



### Dagger Conventions (9)

- `@Binds` allows to map specific provided type to another provided type (e.g. provide implementation of an interface)

- Custom bindings using `@Binds` must be defined as abstract functions in abstract modules
- Abstract `@Binds` functions can't coexist with non-static provider methods in the same module



## Qualifiers

같은 return type을 가진 provider가 여러개 필요한 경우? -> 그냥 실행하면 같은 해당 객체가 여러번 bound 되었다는 error가 뜸

-> `Qualifer` 로 해결 가능

Qualifier은 type의 일부가 됨

Ex) Qualifier로 `@Retrofit1`을 만들고, retrofit provider에 `@Retrofit1` annotation을 추가한다면?

-> return type은 `Retrofit1 Retrofit`이 됨!

따라서, Qualifer를 추가한 provider가 제공하는 객체를 사용하는 곳에서는 반드시 해당 Qualifer를 명시적으로 써주어야 한다. 아니면 type mismatch로 에러가 발생

```kotlin
@Module
abstract class AppModule(val application: Application) {
	@Provides
	@AppScope
	@Retrofit1
	fun retrofit1(): Retrofit {
		Return Retrofit.Builder()
			.baseUrl(Constants.BASE_URL)
			.addConverterFactory(GsonConverterFactory.create())
			.build()
	}
}
```

```kotlin
@Module
abstract class AppModule(val application: Application) {
	@Provides
	@AppScope
	@Named("Retrofit1") // 으로도 사용할 수 있음 (추천 x)
	fun retrofit1(): Retrofit {
		Return Retrofit.Builder()
			.baseUrl(Constants.BASE_URL)
			.addConverterFactory(GsonConverterFactory.create())
			.build()
	}
}
```



### Dagger Conventions (10)

- Quailifers are annotation classes annotated with `@Qualifer`
- From Dagger's standpoint, qualifiers are part of the type (e.g. @Q1 Service and @Q2 Service are different types)
- You can use the standard @Named(String) qualifier



## Providers

```kotlin
class ViewMvcFactory @Inject constructor(
	private val layoutInflater: LayoutInflater,
    	private val imageLoaderProvider: Provider<ImageLoader>
) {
		
	fun newQuestionDetailsViewMvc(parent: ViewGroup?): QuestionDetailsViewMvc {
		val imageLoader1 = imageLoaderProvider.get()
        	val imageLoader2 = imageLoaderProvider.get()
		val imageLoader3 = imageLoaderProvider.get()

		return QuestionsDetailViewMvc(layoutInflater, imageLoaderProvider.get(), parent)
	}
}
```

만약 ImageLoader에 scoping 되어있지 않으면 각 imageLoader id 모두 다르게 나옴 (계속 새로운 인스턴스 생성)

만약 ImageLoader가 ActivityScope에 있으면 각 imageLoader id는 모두 같게 나옴 (같은 activity 내에서는 같은 인스턴스 return)



### Dagger Conventions (11)

- Provider<Type> wrappers are "windows" into Dagger's object graph and allow you to retrieve a single type of services
- Providers are basically "extensions" of composition roots
- You use Providers when you need to perform "late injection"
