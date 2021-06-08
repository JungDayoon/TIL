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

