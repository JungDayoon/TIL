# Section 6. Pure Dependency Injection

## The Main Benefit of Dependency Injection

- good for stability
- reduce some amount of boilerplate codes in my application
- more readable
- **Non-repetitive definition and exposure of the entire object graph by composition root(both internally and externally) is the main benefit of Dependency Injection.**
  - with DI, you can keep your classes small and focused, but still easily compose them into arbitrary long chains to achieve complex functionality
- single responsibility principle
- reusability



## Context Isolation

**기존 코드**

> ActivityCompositionRoot

```kotlin
class ActivityCompositionRoot(
        private val activity: AppCompatActivity,
        private val appCompositionRoot: AppCompositionRoot
) {

    val screensNavigator by lazy {
        ScreensNavigator(activity)
    }

    private val layoutInflater get() = LayoutInflater.from(activity)

    val viewMvcFactory get() = ViewMvcFactory(layoutInflater)

    private val fragmentManager get() = activity.supportFragmentManager

    val dialogsNavigator get() = DialogsNavigator(fragmentManager)

    private val stackoverflowApi get() = appCompositionRoot.stackoverflowApi

    val fetchQuestionsUseCase get() = FetchQuestionsUseCase(stackoverflowApi)

    val fetchQuestionDetailsUseCase get() = FetchQuestionDetailsUseCase(stackoverflowApi)

  	val application get() = activity.applicationContext
}
```



**변경 코드**

ActivityCompositionRoot에서 `activity.applicationContext`로 얻는 것이 아니라 applicationCompositionRoot에서 주입받는 방법으로 변경

> AppCompositionRoot

```kotlin
class AppCompositionRoot (val application: Application){ // 생성자에 appication 추가

    private val retrofit: Retrofit by lazy {
        Retrofit.Builder()
                .baseUrl(Constants.BASE_URL)
                .addConverterFactory(GsonConverterFactory.create())
                .build()
    }

    val stackoverflowApi: StackoverflowApi by lazy {
        retrofit.create(StackoverflowApi::class.java)
    }

}
```

> ActivityCompositionRoot

```kotlin
class ActivityCompositionRoot(
        private val activity: AppCompatActivity,
        private val appCompositionRoot: AppCompositionRoot
) {

    val screensNavigator by lazy {
        ScreensNavigator(activity)
    }

    private val layoutInflater get() = LayoutInflater.from(activity)

    val viewMvcFactory get() = ViewMvcFactory(layoutInflater)

    private val fragmentManager get() = activity.supportFragmentManager

    val dialogsNavigator get() = DialogsNavigator(fragmentManager)

    private val stackoverflowApi get() = appCompositionRoot.stackoverflowApi

    val fetchQuestionsUseCase get() = FetchQuestionsUseCase(stackoverflowApi)

    val fetchQuestionDetailsUseCase get() = FetchQuestionDetailsUseCase(stackoverflowApi)

    val application get() = appCompositionRoot.application

}
```



-> activity에서 application context를 얻는 방식이 아닌, application 자체에서 context를 얻어와서 사용하는 방식



## Objects vs Data Structures

