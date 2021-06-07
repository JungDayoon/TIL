# Section 7. Dagger 2 Tutorial

## Dagger 2

Dagger 2 is the official dependency injection framework for Android



## Gradle Configuration

```
// Add Dagger dependencies
dependencies {
  implementation 'com.google.dagger:dagger:2.x'
  annotationProcessor 'com.google.dagger:dagger-compiler:2.x'
}
```



## Components and Modules

기존에 CompositionRoot를 Dagger module로 변경 (`@module` annotation 추가)

CompositionRoot에서 제공하던 property들을 Dagger provides로 변경(`@Provides` annotation은 property type에서 사용할 수 없기 때문에 해당 property를 function으로 수정)

<PresentationCompositionRoot 수정 -> PresentationModule>

```kotlin
// 이전
val screensNavigator get() = activityCompositionRoot.screensNavigator

// 이후
@Provides
fun screensNavigator() = activityCompositionRoot.screensNavigator
```

private modifiers 제거

```kotlin
// 이전
private val layoutInflater get() = activityCompositionRoot.layoutInflater
@Provides
fun viewMvcFactory() = ViewMvcFactory(layoutInflater)

// 이후
// layoutInflater property 제거
@Provides
fun layoutInflater() = activityComposition
@provides
fun viewMvcFactory(layoutInflater: LayoutInflater) = viewMvcFactory(layoutInflater)

// who will provide layoutInflater? -> dagger will !
// dagger will look for another provider method which provides an object of "LayoutInflater"

```

-> 끝이 아님. 왜냐하면 Dagger에서는 Composition Root(Module)을 직접적으로 사용하지 않기 때문

-> 새로운 interface 생성(PresentationComponent)

what is component?

- Module의 wrapper 역할을 함

```kotlin
@Component(modules = [PresentationModule::class])
interface PresentationComponent {
  // Dagger Component는 Module에서 provides하는 object에 대한 access를 가지고, expose 할 수 있음
  
  fun viewMvcFactory(): ViewMvcFactory
}
```



Injector에서 presentationModule을 통해 접근했던 부분들을 presentationComponent를 통해 접근하도록 수정

(Dagger에서는 module에 직접적으로 접근하지 않기 때문. convention)

```kotlin
// 이전
class Injector(private val module: PresentationModule) {
  private fun getServiceForClass(type: Class<*>): Any {
    when (type) {
      DialogsNavigator::class.java -> {
        return module.dialogNavigator
      }
      ScreensNavigator::class.java -> {
        return module.screensNavigator
      }
      else -> {
        throw exception("unsupported service type: $type")
      }
    }
  }
}
```

```kotlin
// 이후
class Injector(private val component: PresentationComponent) {
  private fun getServiceForClass(type: Class<*>): Any {
    when (type) {
      DialogsNavigator::class.java -> {
        return component.dialogNavigator()
      }
      ScreensNavigator::class.java -> {
        return component.screensNavigator()
      }
      else -> {
        throw exception("unsupported service type: $type")
      }
    }
  }
}
```

BaseActivity와 BaseFragment에서 injector 객체를 주입받는데, 생성자가 바뀌었으므로 수정필요

```kotlin
open class BaseActivity: AppCompatActivity() {
  private appCompositionRoot get() = (application as MyApplication).appCompositionRoot
  
  val activityCompositionRoot by lazy {
    ActivityCompositionRoot(this, appCompositionRoot)
  }
  
  private val compositionRoot by lazy {
    PresentationModule(activityCompositionRoot)
  }
  
  // 이전
  protected val injector get() = Injector(compositionRoot)
  
  // 이후
  private val presentationComponent by lazy {
    DaggerPresentationComponent.builder()
    	.presentationModule(PresentationModule(activityCompostionRoot))
    	.build()
    
  }
  protected val injector get() = Injector(presentationComponent)
}
```

what is DaggerPresentationComponent?

- Dagger가 PresentationModule과 PresentationComponent definition을 보고 만들어낸 class (dagger's convention)



*Refactoring 전

![image-20210602011934363](https://user-images.githubusercontent.com/45536712/120359038-333fa900-c342-11eb-93ff-a343a46a755c.png)



*Refactoring 후

![스크린샷 2021-06-02 오전 1.20.27](https://user-images.githubusercontent.com/45536712/120359516-b82ac280-c342-11eb-9b89-213f329da324.png)



### Dagger Convention (1)

- Components are interfaces annotated with `@Component`

- Modules are classes annotated with `@Module`

- Methods in modules that provides services are annotated with `@Provides`

- Provided services can be used as method arguments in other provider methods



## Scopes

```kotlin
@Singleton
@Provides
fun stackoverflowApi() = retrofit.create(StackoverflowApi::clss.java)
```

-> Dagger가 stackoverflowApi에 처음 접근할 때 해당 instance를 cache 해두었다가 다음에 여러번 접근하더라도 동일한 instance를 return 하게 됨



case 1) AppModule에서 제공하는 객체들을 singleton으로 만들기

```kotlin
@Singleton
@Provides
fun retrofit(): Retrofit =
  Retrofit.Builder()
    .baseUrl(Constants.BASE_URL)
    .addConverterFactory(GsonConverterFactory.create())
    .build()


@Singleton
@Provides
fun stackoverflowApi(): StackoverflowApi = retrofit().create(StackoverflowApi::class.java)
// stackoverflowApi에서 retrofit()함수를 직접적으로 호출하면, stackoverflowApi에 접근할 때 마다 retrofit 객체가 새로 생겨나게 됨. 
// 지금은 다행히 stackoverflowApi도 singleton이므로 문제가 생기지 않지만 다른 scope에서 생성된다면 stackoverflowApi 객체가 새로 생성될 때마다 retrofit 객체도 새로 생성되게 됨 -> bug

// 수정 -> retrofit 객체를 method 인자로 넘겨주기!
@Singleton
@Provides
fun stackoverflowApi(retrofit: Retrofit): StackoverflowApi = retrofit.create(StackoverflowApi::class.java)
```



case 2) Singleton annotation을 AppScope annotation으로 바꾸기

Application이 살아있는 동안 유지되는 scope이므로 singleton과 의미는 동일

```kotlin
@Scope
annotation class AppScope {
}

// -> @Singleton을 @AppScope로 대체

```



*Refactoring 후

![_2021-05-31__12.55.14](https://user-images.githubusercontent.com/45536712/120433777-9b2fd700-c3b6-11eb-925c-3783032ca0d5.png)



### Dagger Conventions (2)

- Scopes are annotations, annotated with `@Scope`

- Components that provide scoped services must be scoped

- All clients get the same instance of a scoped service from the same instance of a Component



## Component as Injector

PresentationComponent를 injector로 사용하기

```kotlin
@Component(modules = [PresentationModule::class])
interface PresentationComponent {

    fun inject(fragment: QuestionsListFragment) 
  // interface지만 구현할 필요 없음. Dagger가 알아서 해줌
  // Dagger에서 component는 기본적으로 injector임 (Module은 compositionRoot)

    fun inject(activity: QuestionDetailsActivity)

}
```



QuestionsListFragment, QuestionDetailsActivity에서 주입 받는 property들에 `@Inject` annotation 추가(must not be private - Dagger's convention)

```kotlin
class QuestionDetailsActivity : BaseActivity(), QuestionDetailsViewMvc.Listener {

    @Inject lateinit var fetchQuestionDetailsUseCase: FetchQuestionDetailsUseCase
    @Inject lateinit var dialogsNavigator: DialogsNavigator
    @Inject lateinit var screensNavigator: ScreensNavigator
    @Inject lateinit var viewMvcFactory: ViewMvcFactory

    override fun onCreate(savedInstanceState: Bundle?) {
        injector.inject(this) // presentationComponent의 inject 함수
        super.onCreate(savedInstanceState)
        viewMvc = viewMvcFactory.newQuestionDetailsViewMvc(null)
        setContentView(viewMvc.rootView)

        questionId = intent.extras!!.getString(EXTRA_QUESTION_ID)!!
    }
}
```

![_2021-05-31__1.09.23](https://user-images.githubusercontent.com/45536712/120433758-979c5000-c3b6-11eb-8c57-b516eb8ded46.png)



### Dagger Conventions (3)

- Void methods with single argument defined on components generate injectors for the type of the argument
- Client's non-private non-final properties annotated with `@Inject` designate injection targets



## Dependent Components

### Dagger Conventions (4)

- Component inter-dependencies are specified as part of `@Component` annotation
- Component B that depends on Component A has implicit access to all services exposed by Component A
  - Services from A can be injected by B
  - Services from A can be consumed inside modules of B



## Subcomponents

```kotlin
@ActivityScope
@Component(dependencies = [AppComponent::class], modules = [ActivityModule::class])
interface ActivityComponent {

    fun activity(): AppCompatActivity

    fun layoutInflater(): LayoutInflater

    fun fragmentManager(): FragmentManager

    fun stackoverflowApi(): StackoverflowApi

    fun screensNavigator(): ScreensNavigator

}

```

위의 코드 처럼 explicit declaration을 없앨 수 있는 방법은 없을까? -> Dagger will do!

Dagger는 stackoverflowApi가 AppModule로 부터 제공 받는다는 것을 알고 있음. 또한 stackoverflowApi가 어디서 사용되는 지도 알고 있음(in PresentationModule)

-> PresentationComponent를 Subcomponent로 만들자

<PresentationComponent>

```kotlin
// 수정 전
@PresentationScope
@Component(dependencies = [ActivityModule::class], modules = [PresentationModule::class])
interface PresentationComponent {
    fun inject(fragment: QuestionsListFragment)
    fun inject(activity: QuestionDetailsActivity)
}

// 수정 후
@PresentationScope
@Subcomponent(modules = [PresentationModule::class])
interface PresentationComponent {
    fun inject(fragment: QuestionsListFragment)
    fun inject(activity: QuestionDetailsActivity)
}

```

<ActivityComponent>

```kotlin
// 수정 전 코드는 위에 있음
// 수정 후 
@ActivityScope
@Component(dependencies = [AppComponent::class], modules = [ActivityModule::class])
interface ActivityComponent {
    fun newPresentationComponent(presentationModule: PresentationModule): PresentationComponent

}
```

<BaseActivity>

```kotlin
// 수정 전
private val presentationComponent by lazy {
  	DaggerPresentationComponent.builder()
      .activityComponent(activityComponent)
      .presentationModule(PresentationModule())
      .build()
}

// 수정 후
private val presentationComponent by lazy {
  	activityComponent.newPresentationComponent(PresentationModule())
}
```



### Dagger Conventions (5)

- Subcomponents specifed by `@Subcomponent` annotation
- Parent Component exposes factory method which returns Subcomponent
- The argument of the factory method are Subcomponent's modules
- Subcomponents get access to all services provided by parent (provided, not just exposed)



## Multi-Module Components

여러 module에 의존하는 single component를 만들 수 있음

언제? -> if modules don't have any bootstrapping. 즉, module에 생성자가 없는 경우

```kotlin
@PresentationScope
// 하나의 component가 두개의 module에 의존
@Subcomponent(modules = [PresentationModule::class, UseCasesModule::class])
interface PresentationComponent {
    fun inject(fragment: QuestionsListFragment)
    fun inject(activity: QuestionDetailsActivity)
}

```

```kotlin
// in BaseActivity, newPresentationComponent의 생성자로 모듈을 넘겨주지 않아도 됨 - 두 모듈 모두 생성자가 없기 때문
// 만약 생성자가 있는 경우에는 명시해주어야함. 대거가 해당 생성자를 어디서 가져와야 하는지 모르기 때문)
private val presentationComponent by lazy {
    activityComponent.newPresentationComponent()
}
```

주의점: 해당 Dagger convention 사용 시 어떤 모듈에 의존하는지 component definition을 봐야지만 확인 가능. code maintainence에 더 많은 effort가 들 수도 있음



### Dagger Conventions (6)

- Components can use more than one module
- Modules of a single Component share the same object graph
- Dagger automatically instantiates modules with no-argument constructors



