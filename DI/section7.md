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

기존에 CompositionRoot를 Dagger module로 변경 (@module annotation 추가)

CompositionRoot에서 제공하던 property들을 Dagger provides로 변경(@Provides annotation은 property type에서 사용할 수 없기 때문에 해당 property를 function으로 수정)

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

1. Components are interfaces annotated with @Component
2. Modules are classes annotated with @Module
3. Methods in modules that provides services are annotated with @Provides
4. Provided services can be used as method arguments in other provider methods

