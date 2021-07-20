# Section 10. Hilt

## Hilt's Fundamental Assumptions

Component Hierarchy

![스크린샷 2021-05-06 오후 10.59.14](https://user-images.githubusercontent.com/45536712/122946747-96a67f00-d3b4-11eb-9a46-3164f31a7b62.png)

Component Lifetimes

![스크린샷 2021-05-06 오후 10.59.14](https://user-images.githubusercontent.com/45536712/122946877-b2aa2080-d3b4-11eb-9e8c-aa76fa4a1a78.png)

Component default bindings

![스크린샷 2021-05-06 오후 10.59.14](https://user-images.githubusercontent.com/45536712/122946981-c3f32d00-d3b4-11eb-8586-b21c03e62ae5.png)



## Removing Dagger Component

Hilt를 사용하기 위해서는 MyApplication에 `@HiltAndroidApp` annotation이 필요하다.

```kotlin
@HiltAndroidApp
class MyApplication: Application() {
	override fun onCreate() {
			super.onCreate()
	}
}
```

Component는 모두 삭제

Activity에 `@AndroidEntryPoint` annotation 붙이기



## Hilt Injection Mechanics

## Installing Modules into Components

Hilt가 Component를 자동으로 생성하는데, 그 component와 Module을 연결해주기 위해서 `@InstallIn` annotation이 필요하다.

```kotlin
@Module
@InstallIn(ActivityComponent::class)
abstract class ActivityModule {
		@ActivityScope
		@Binds
		abstract fun screensNavigator(screensNavigatorImpl: ScreensNavigatorImpl): ScreensNavigator
		
		companion object {
				@Provides
				fun layoutInflater(activity: AppCompatActivity) = LayoutInflater.from(activity)
				
				@Provides
				fun fragmentManager(activity: AppCompatActivity) = activity.supportFragmentManager
		}
}
```

```kotlin
@Module
@InstallIn(SingletonComponent::class)
class AppModule {
		@Provides
		@AppScope
		fun retrofit(urlProvider: UrlProvider): Retrofit {
				return Retrofit.Builder()
									.baseUrl(urlProvider.getBaseUrl())
									.addConverterFactory(GsonConverterFactory.create())
									.build()
		}
}
```

AppModule을 empty constructor로 변경

-> SingletonComponent는 자동으로 Application을 bind한다. (Hilt's Convention)



## Hilt Scopes

기존에 사용하던 `@ActivityScope`, `@AppScope`를 제거하고 Hilt에서 제공하는 `@ActivityScoped`, `@Singleton` 으로 대체할 수 있음



## Providing AppCompatActivity

Hilt의 ActivityComponent가 Activity를 자동으로 제공하지만, 해당 앱에서는 모두 AppCompatActivity를 사용하고 있음. -> 자동으로 제공 불가

ActivityModule에서 AppCompatActivity를 제공하도록 추가해주어야 함

```kotlin
@Module
@InstallIn(ActivityComponent::class)
abstract class ActivityModule {
		@ActivityScope
		@Binds
		abstract fun screensNavigator(screensNavigatorImpl: ScreensNavigatorImpl): ScreensNavigator
		
		companion object {
      	@Provides
      	fun appCompatActivity(activity: Activity): AppCompatActivity = activity as AppCompatActivity
      
				@Provides
				fun layoutInflater(activity: AppCompatActivity) = LayoutInflater.from(activity)
				
				@Provides
				fun fragmentManager(activity: AppCompatActivity) = activity.supportFragmentManager
		}
}
```



## Hilt and ViewModel

ViewModel 클래스에 `@HiltViewModel` annotation 추가하기

ViewModelFactory 제거하기

기존에는

```kotlin
myViewModel = ViewModelProvider(this).get(MyViewModel::class.java)
```

다음과 같이 하려면 ViewModel이 empty constructor일 때만 가능했음

-> Hilt를 사용하여 `@HiltViewModel` 를 붙여주면 viewModel을 쉽게 provide 할 수 있다.

Hilt는 savedStateHandle도 inject해준다. -> constructor argument에 넣어주기만 하면 됨



```
implementation "androidx.activity:activity-ktx:1.2.3"
```

을 build.gradle에 추가해주면

```kotlin
@AndroidEntryPoint
class ViewModelActivity: BaseActivity() {
		private val myViewModel: MyViewModel by viewModels()
}
```

로 viewModel을 사용할 수 있다.



## Hilt Summary

### Main Features of Hilt

- Use predefined set of Implicit Components
- Each predefined Component is associated with a predefined Scope
- Automatically provides specific bootstrapping services (e.g. Application)
- Modules specify in which Components they should be installed
- Simplified ViewModel handling

