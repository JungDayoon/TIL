# Section 9. Dagger and ViewModel 

## Incorrect ViewModel Integration

일반적인 Constructor Injection 방법을 통해 ViewModel을 주입하게 되면, 화면 전환 등 Activity가 recreated될 때마다 viewModel 또한 다시 생성된다. -> **Bad way !!**



## Dedicated Factories for ViewModel

해결방안: ViewModelFactory를 만들기

```kotlin
class MyViewModel @Inject constructor(
		private val fetchQuestionsUseCase: FetchQuestionsUseCase
): ViewModel() {
  	private val _questions = MutableLiveData<List<Question>>()
  	val questions: LiveData<List<Question>> = _questions
  
  	init {
      	viewModelScope.launch {
          	val result = fetchQuestionsUseCase.fetchLatestQuestions()
          	if (result is FetchQuestionsUseCase.Result.Success) {
              	_questions.value = result.questions
            } else {
              throw RuntimeException("fetch failed")
            }
        }
    }
}

class Factory @Inject constructor(
  	private val fetchQuestionsUseCaseProvider: Provider<FetchQuestionsUseCase>
): ViewModelProvider.Factory {
	override fun <T: ViewModel?> create(modelClass: Class<T>): T {
		return MyViewModel(fetchQuestionsUseCaseProvider.get()) as T
	}
}
```

```kotlin
// 수정 전
class ViewModelActivity: BaseActivity() {
  	@Inject lateinit var myViewModel: MyViewModel
}

// 수정 후
class ViewModelActivity: BaseActivity() {
  	@Inject lateinit var myViewModelFactory: MyViewModel.Factory
  	lateinit var myViewModel: MyViewModel
		
  	override fun onCreate(savedInstanceState: Bundle?) {
      	myViewModel = ViewModelProvider(this, myViewModelFactory).get(MyViewModel::class.java)
    }
}

```



## Refactoring ViewModel Factories According to the Law of Demeter

ViewModel constructor에 property가 추가될 때마다 Factory의 constructor에도 추가해주어야하고, 이를 binding하기 위한 function의 parameter에도 추가해주어야 한다.

-> 더 나은 방법은 없을까?

```kotlin
class MyViewModel @Inject constructor(
		private val fetchQuestionsUseCase: FetchQuestionsUseCase
): ViewModel() {
  	private val _questions = MutableLiveData<List<Question>>()
  	val questions: LiveData<List<Question>> = _questions
  
  	init {
      	viewModelScope.launch {
          	val result = fetchQuestionsUseCase.fetchLatestQuestions()
          	if (result is FetchQuestionsUseCase.Result.Success) {
              	_questions.value = result.questions
            } else {
              throw RuntimeException("fetch failed")
            }
        }
    }
}

class Factory @Inject constructor(
		private val myViewModelProvider: Provider<MyViewModel>
): ViewModelProvider.Factory {
	override fun <T: ViewModel?> create(modelClass: Class<T>): T {
		return myViewModelProvider.get() as T
	}
}
```



## Centralized Factory for ViewModel

ViewModel이 여러 개 있을 때 하나의 ViewModelFactory로 여러 ViewModel들을 instantiate 하는 방법이 있을까?

```kotlin
class ViewModelFactory @Inject constructor(
		private val myViewModelProvider: Provider<MyViewModel>,
  	private val myViewModel2Provider: Provider<MyViewModel2>
): ViewModelProvider.Factory {
	override fun <T: ViewModel?> create(modelClass: Class<T>): T {
		return when (modelClass) {
      	MyViewModel::class.java -> myViewModelProvider.get() as T
      	MyViewModel2::class.java -> myViewModel2Provider.get() as T
      	else -> throw RuntimeException("unsupported viewmodel type: $modelClass")
    }
	}
}
```



## Multibinding

```kotlin
@Module
abstract class ViewModelsModule {
		@Binds
  	@IntoMap 
  	// take all the providers for the specific type, ViewModel
  	// Dagger will construct map data structure for us and push all this viewModels
  	@ViewModelKey(MyViewModel::class)
		abstract fun myViewModel(myViewModel: MyViewModel): ViewModel
  	
  	@Binds
  	@IntoMap
  	@ViewModelKey(MyViewModel2::class)
		abstract fun myViewModel2(myViewModel2: MyViewModel2): ViewModel

}
```

```kotlin
@MapKey
annotation class ViewModelKey(val value: KClass<out ViewModel>) {
}
```

```kotlin
class ViewModelFactory @Inject constructor(
  	private val providers: Map<Class<out ViewModel>, @JvmSuppressWildCards Provider<ViewModel>>
): ViewModelProvider.Factory {
	override fun <T: ViewModel?> create(modelClass: Class<T>): T {
    val provider = providers[modelClass]
    return provider?.get() as T ?: throw RuntimeException("unsupported viewmodel type: $modelClass")
	}
}
```

-> 이렇게 해두면 ViewModel이 50개가 되든, 100개가 되든 ViewModelFactory는 변하지 않게 된다.

-> 변화는 오직 ViewModelsModule에서만 생김



### Dagger Conventions (multibinding)

- `@IntoMap` annotation can be used to bind multiple services of the same type into Map data structure

- Keys of individual services in the Map are defined with a custom annotation, annotated with `@MapKey` annotation

- Dagger will automatically provide the following Map: 

  `Map<key_type, Provider<service_type>>`

- Use `@JvmSuppressWildcards` at injection site to make it work in Kotlin



## ViewModel with SavedState

```kotlin
class MyViewModel @Inject constructor(
		private val fetchQuestionsUseCase: FetchQuestionsUseCase,
		private val fetchQuestionDetailsUseCase: FetchQuestionDetailsUseCase,
		private val savedStateHandle: SavedStateHandle //-> runtime parameter
): ViewModel() {
  	private val _questions: MutableLiveData<List<Question>> = savedStateHandle.getLiveData("questions")
  	val questions: LiveData<List<Question>> = _questions
  
  	init {
      	viewModelScope.launch {
          	val result = fetchQuestionsUseCase.fetchLatestQuestions()
          	if (result is FetchQuestionsUseCase.Result.Success) {
              	_questions.value = result.questions
            } else {
              throw RuntimeException("fetch failed")
            }
        }
    }
}
```

