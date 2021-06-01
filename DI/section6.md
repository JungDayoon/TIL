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

- Objects expose behavior

- Data Structures expose data

Data structures which should expose well structured data, shouldn't know anything about objects

Dependency Injection Architectural Pattern deals with objects, not data structures

-> You should not expose data structures on your object graph either internally or externally. Composition rules only provide objects.





## Injecting Services "from Outside"

> QuestionsListFragment

```kotlin
class QuestionsListFragment : BaseFragment(), QuestionsListViewMvc.Listener {

    private var isDataLoaded = false
    private lateinit var viewMvc: QuestionsListViewMvc
    private lateinit var fetchQuestionsUseCase: FetchQuestionsUseCase
    private lateinit var dialogsNavigator: DialogsNavigator
    private lateinit var screensNavigator: ScreensNavigator

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        fetchQuestionsUseCase = compositionRoot.fetchQuestionsUseCase
        dialogsNavigator = compositionRoot.dialogsNavigator
        screensNavigator = compositionRoot.screensNavigator
    }
}
```

onCreate 에서 compositionRoot를 통해 각자의 dependency를 주입받고 있는 상황

-> 바깥에서 주입받도록 수정



> Injector

```kotlin
class Injector(private val compositionRoot: PresentationCompositionRoot) {

    fun inject(fragment: QuestionsListFragment) {
        fragment.dialogsNavigator = compositionRoot.dialogsNavigator
        fragment.fetchQuestionsUseCase = compositionRoot.fetchQuestionsUseCase
        fragment.screensNavigator = compositionRoot.screensNavigator
        fragment.viewMvcFactory = compositionRoot.viewMvcFactory
    }

    fun inject(activity: QuestionDetailsActivity) {
        activity.screensNavigator = compositionRoot.screensNavigator
        activity.dialogsNavigator = compositionRoot.dialogsNavigator
        activity.fetchQuestionDetailsUseCase = compositionRoot.fetchQuestionDetailsUseCase
        activity.viewMvcFactory = compositionRoot.viewMvcFactory
    }
}
```

> QuestionsListFragment

```kotlin
class QuestionsListFragment : BaseFragment(), QuestionsListViewMvc.Listener {

    lateinit var fetchQuestionsUseCase: FetchQuestionsUseCase
    lateinit var dialogsNavigator: DialogsNavigator
    lateinit var screensNavigator: ScreensNavigator
    lateinit var viewMvcFactory: ViewMvcFactory

    private lateinit var viewMvc: QuestionsListViewMvc

    private var isDataLoaded = false

    override fun onCreate(savedInstanceState: Bundle?) {
        injector.inject(this)
        super.onCreate(savedInstanceState)
    }
}
```





## Dependency Injection Frameworks

- External library
- Provides specific template for Construction Set implementation
- Provides set of pre-defined conventions

