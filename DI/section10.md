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

