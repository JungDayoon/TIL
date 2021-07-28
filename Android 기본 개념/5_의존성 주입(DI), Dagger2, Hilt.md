# 의존성 주입(DI), Dagger2, Hilt

# 의존성 주입

각종 컴포넌트 간 의존성이 상당히 강한 Android Framework에서 클래스 간 의존도를 낮추는 것이 매우 중요함.

인스턴스를 클래스 외부에서 주입하기 위해서는 인스턴스에 대한 전반적인 생명주기(생성-소멸)의 관리가 필요하다.

# Dagger2

프로젝트의 규모가 커질 수록 의존성 인스턴스들을 manual하게 관리하는 것은 생각보다 많은 리소스가 요구되는데, 이를 전반적으로 관리해주는 것이 대표적으로 Google에서 밀어주고 있는 오픈소스 라이브러리 Dagger2이다.

Dagger2는 자체적으로 Android와 크게 상관관계가 없지만 Android 환경에서 많은 인기를 끌었고, 이를 인지한 Google은 Android 환경에서 사용할 경우 자연스럽게 늘어나는 보일러 플레이트를 줄여주는 Dagger-Android도 함께 지원해주고 있다.

# Dagger Hilt

Dagger Hilt는 2020년 6월 Google에서 오피셜하게 발표한 Android 전용 DI 라이브러리다. Hilt는 Dagger2를 기반으로 Android Framework에서 표준적으로 사용되는 DI Component와 scope를 기본적으로 제공하여, 초기 DI 환경 구축 비용을 크게 절감시키는 것이 가장 큰 목적이다.

Google에서 전격적으로 지원하는 JetPack의 ViewModel에 대한 의존성 주입도 별도의 큰 비용없이 구현할 수 있다. 

![%E1%84%8B%E1%85%B4%E1%84%8C%E1%85%A9%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8C%E1%85%AE%E1%84%8B%E1%85%B5%E1%86%B8(DI),%20Dagger2,%20Hilt%20cf05428cc132401db4e4c971bc5f79ce/_2020-12-04__3.09.11.png](%E1%84%8B%E1%85%B4%E1%84%8C%E1%85%A9%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8C%E1%85%AE%E1%84%8B%E1%85%B5%E1%86%B8(DI),%20Dagger2,%20Hilt%20cf05428cc132401db4e4c971bc5f79ce/_2020-12-04__3.09.11.png)

위와 같은 표준 component/scope들은 Hilt에서 제공하고 있으며, 새로운 component를 정의하고 싶다면 `@DefineComponent` 어노테이션을 사용하여 사용자 정의가 가능하다.

### 사용자 정의 Component

```kotlin
@Scope
@MustBeDocumented
@Retention(value = AnnotationRetention.RUNTIME)
annotation class LoggedUserScope

@LoggedUserScope
@DefineComponent(parent = ApplicationComponent::class)
interface UserComponent {

    // Builder to create instances of UserComponent
    @DefineComponent.Builder
    interface Builder {
        fun setUser(@BindsInstance user: User): UserComponent.Builder
        fun build(): UserComponent
    }
}
```

`LoggedUserScope` 라는 사용자 scope를 정의하고, 해당 scope를 사용하여 `UserComponent` 라는 새로운 component를 만든 예시이다.

`@DefineComponent`에서 예상할 수 있듯이, 사용자 정의되는 component들은 반드시 표준 컴포넌트 중 하나를 부모 컴포넌트로써 상속받아야 한다.

![%E1%84%8B%E1%85%B4%E1%84%8C%E1%85%A9%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8C%E1%85%AE%E1%84%8B%E1%85%B5%E1%86%B8(DI),%20Dagger2,%20Hilt%20cf05428cc132401db4e4c971bc5f79ce/Untitled.png](%E1%84%8B%E1%85%B4%E1%84%8C%E1%85%A9%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8C%E1%85%AE%E1%84%8B%E1%85%B5%E1%86%B8(DI),%20Dagger2,%20Hilt%20cf05428cc132401db4e4c971bc5f79ce/Untitled.png)

사용자 component는 반드시 leaf component로써 표준 component에 추가될 수 있으며, 2개의 layer에 침범하는 형태의 사용자 정의는 불가능하다.

# Hilt Modules

기존의 Dagger2에서는 새로운 module을 생성하면, 사용자가 정의한 component에 해당 module 클래스를 직접 include 해주는 방법이었다.

반면, Hilt는 표준적으로 제공하는 component들이 이미 존재하기 때문에 `@InstallIn` 어노테이션을 사용하여 표준 component들에 module들을 install할 수 있다. Hilt에서 제공하는 기본적인 규칙은 모든 module에 `@InstallIn` 어노테이션을 사용하여 어떤 컴포넌트에 Install 할 지 반드시 정해주어야 한다. 

### 예시

```kotlin
@Module
@InstallIn(ApplicationComponent::class)
object class FooModule {
  // @InstallIn(ApplicationComponent.class) module providers have access to
// the Application binding.@Provides
  fun provideBar(app: Application): Bar {...}
}
```

FooModule 이라는 module을 ApplicationComponent에 install하고, ApplicationComponent에서 제공해주는 Application class를 내부적으로 활용하고 있다.

만약, 하나의 모듈을 다중 컴포넌트에 install하고 싶다면, 

```kotlin
@InstallIn({ViewComponent.class, ViewWithFragmentComponent.class})
```

이처럼 다중 컴포넌트에 하나의 모듈을 install 하는 데는 세 가지 규칙이 있다.

- Provider는 다중 컴포넌트가 모두 동일한 scope에 속해있을 경우에만 scope를 지정할 수 있다. 위의 예시와 같이 ViewComponent와 ViewWithFragmentComponent는 동일한 ViewScoped에 속해있기 때문에 가능하다.
- Provider는 다중 컴포넌트가 서로 간 요소에게 접근이 가능한 경우에만 주입이 가능하다.
- 부모 컴포넌트와 자식 컴포넌트에 동시에 install 될 수 없으며, 자식 컴포넌트는 부모 컴포넌트의 모듈에 대해 접근할 수 있다.

# AndroidEntryPoint

기존의 Dagger2에서는 직접 의존성을 주입해줄 대상을 전부 dependency graph에 지정해주었다면, Hilt에서는 객체를 주입할 대상에게 @AndroidEntryPoint 어노테이션을 추가하는 것만으로도 member injection을 수행할 수 있다.

- Activity
- Fragment
- View
- Service
- BroadcastReceiver

```kotlin
@AndroidEntryPoint
class MyActivity : MyBaseActivity() {
  // Bindings in ApplicationComponent or ActivityComponent
	@Inject lateinit var bar: Bar
	override fun onCreate(savedInstanceState: Bundle?) {
	    // Injection happens in super.onCreate().
			super.onCreate()
	
	    // Do something with bar ...
	}
}
```

MainActivity에 `Bar` 객체 주입하는 예시

# EntryPoint

Hilt의 또 다른 장점은 Dagger에 의해 관리되는 의존성 객체를 injection이 아닌 `EntryPoint` 를 통해서 얻을 수 있다. 모듈과 유사하게 `InstallIn` 어노테이션을 사용하여 Install 하려는 component를 지정하고, `@EntryPoint` 어노테이션을 추가한다.

```kotlin
@EntryPoint
@InstallIn(ApplicationComponent::class)
interface RetrofitInterface {

    fun getRetrofit(): Retrofit
}
```

Retrofit 객체 획득을 위한 EntryPoint interface 작성 예시

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)

    val retrofit = EntryPoints.get(applicationContext, RetrofitInterface::class.java).getRetrofit()
    
    // ... //
}
```

MainActivity에서 Retrofit 객체를 injection이 아닌 EntryPoint를 통해 얻어오는 예시

# ViewModel Injection

Jetpack의 ViewModel은 Android SDK 내부적으로 ViewModel에 대한 lifecycle을 관리하고 있다. 

따라서 ViewModel의 생성 또한 Jetpack에서 제공하는 ViewModelFactory 를 통해서 이루어져야 한다. 

기존에는 각자 ViewModel 환경에 맞는 ViewModelFactory를 따로 작성하였거나, Dagger-Android 유저들은 ViewModel의 constructor injection을 위해 글로벌한 ViewModelFactory를 작성하여 사용했다. 

Hilt에서는 이러한 보일러 플레이트를 줄이기 위한 ViewModelFactory가 이미 내부에 정의되어있고, ActivityComponent와 FragmentComponent에 자동으로 install 된다. 

아래의 @ViewModelInject 어노테이션을 사용하여 constructor injection을 수행한 예시이다. 

```kotlin
class HakunaViewModel @ViewModelInject constructor(
  private val bar: Bar
) : ViewModel() {
  // ... //
}
```

```kotlin
class HakunaViewModel @ViewModelInject constructor(
  private val bar: Bar,
  @Assisted private val savedStateHandle: SavedStateHandle
) : ViewModel() {
  // ... //
}
```

MainActivity에서 사용하는 예시

```kotlin
class HakunaViewModel @ViewModelInject constructor(
  private val bar: Bar,
  @Assisted private val savedStateHandle: SavedStateHandle
) : ViewModel() {
  // ... //
}
```

ViewModel에서 `SavedStateHandle` 를 주입받으려면 아래와 같이 `@Assited` 어노테이션이 사용된다.

## CodeLabs

[Using Hilt in your Android app | Google Codelabs](https://codelabs.developers.google.com/codelabs/android-hilt/?hl=ko#3)

```kotlin
@HiltAndroidApp
class LogApplication : Application() {
    ...
}
```

@HiltAndroidApp은 종속성 주입을 사용할 수 있는 애플리케이션의 기본 클래스를 포함하여 Hilt의 코드 생성을 trigger 한다. Application Container은 app의 상위 컨테이너이다. 즉, 다른 컨테이너가 이 종속성에 접근할 수 있다는 것이다.