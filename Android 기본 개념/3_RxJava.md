# RxJava

## **RxJava란?**

자바로 리액티브 프로그래밍을 할 수 있는 라이브러리이며 비동기 프로그래밍과 함수형 프로그래밍 기법을 함께 활용한다.

리액티브 프로그래밍은 복잡한 비동기 프로그램을 쉽게 만들 수 있게 해준다. 또한 비동기에서 처리하기 힘든 에러 처리나 데이터 가공을 쉽게 할 수 있도록 돕는다. 이벤트를 콜백이 아니라 데이터의 모음으로 모델링하기 때문이다.

**어떤 기능이 직접 실행되는 것이 아니라 시스템에 어떤 이벤트가 발생했을 때 처리한다.**

## 리액티브 프로그래밍

- 기존 pull 방식의 프로그래밍 개념을 **push 방식**의 프로그래밍 개념으로 바꾼다.
    - 예를 들어, 전국 매장의 매출액 정보를 실시간으로 집계한다고 할 때, 기존에는 각 매장의 변화 상황을 데이터베이스에서 가져(pull)와야한다. 하지만, 리액티브 프로그래밍서는 데이터의 변화가 발생했을 때 변경이 발생한 곳에서 새로운 데이터를 보내(push 방식)준다.
- **함수형 프로그래밍**의 지원을 받는다.
    - 우리가 아는 콜백이나 옵서버 패턴을 넘어 RxJava기반의 리액티브 프로그래밍이 되려면 함수형 프로그래밍이 필요
    - 콜백이나 옵서퍼 패턴은 옵서버가 1개이거나 단일 스레드 환경에서는 문제가 없지만 멀티 스레드 환경에서는 많은 주의가 필요하다. (대표적 예 : 데드락, 동기화 문제)
    - 함수형 프로그램은 부수 효과가 없다. (부수효과: 같은 자원에 여러 스레드가 경쟁조건에 빠지게 되었을 때 예측할 수 없는 잘못된 결과가 나오는 현상.)
        - 한 두 개의 스레드가 있을 때는 정상 동작하다가 수십 수백 개의 스레드가 동시에 단일 자원에 접근하면 계산 결과가 꼬이고 디버깅하다가 매우 어렵다.
    - 함수형 프로그래밍은 부수 효과가 없는 순수 함수를 지향한다. 따라서, 멀티스레드 환경에서도 안전하다.

## **RxJava 변수**

**1. Observable**

observer 패턴을 구현. 옵서버 패턴은 객체의 상태 변화를 관찰하는 옵서버 목록을 객체에 등록.

직관적으로, **관찰자(Observer)가 관찰하는 대상 !!**

- onNext: Observable이 데이터의 발행을 알린다.
- onComplete: 모든 데이터의 발행이 완료했음을 알린다. 단 한번만 발생하며, 발생한 후에는 더이상 onNext 이벤트가 발생할 수 없다.
- onError: 에러 발생을 알림. onError 이벤트가 발생하면 이후에 onNext 및 onComplete 이벤트가 발생하지 않는다. 즉, Observable의 실행을 종료한다.

**차가운 Observable**

Observable을 선언하고 just() 함수를 호출해도 Observer가 subscribe() 함수를 호출하여 구독하지 않으면 데이터를 발행하지 않는다 -> 게으른 접근법

구독자가 구독하면 준비된 데이터를 처음부터 발행.

별도의 언급이 없으면 차가운 Observable이라고 생각하면 된다.

ex) 웹 요청, 데이터베이스 쿼리/읽기, 내가 원하는 URL이나 데이터를 지정하면 그때부터 서버나 데이터베이스에 요청을 보내고 결과를 받아옴

**뜨거운 Observable**

구독자가 존재여부와 관계없이 데이터를 발행함 -> 여러 구독자를 고려할 수 있음

단, 구독자로서는 Observable에서 발행하는 데이터를 모두 수신할 것으로 보장할 수 없다.

구독한 시점부터 Observable에서 발행한 값을 받는다.

ex) 마우스 이벤트, 키보드 이벤트, 시스템 이벤트, 센서 데이터, 주식 가격 등

**주의점** 배압을 고려해야함

- 배압은 Observable에서 데이터를 발행하는 속도와 구독자가 처리하는 속도의 차이가 클 때 발생한다.
- Flowable이라는 특화 클래스에서 배압을 처리할 수 있음

**2. Flowable**

Flowable은 **연속적으로 데이터가 흐르는 스트림**을 생성한다. 한 번 구독하면, 데이터가 흐름에 따라 화면을 갱신시키는 코드에 사용할 수 있다.

ex) 사용자의 click event, viewModel의 데이터를 구독해서 view를 계속 변경

**3. Single**

Single은 **일회성 데이터 흐름**에 사용한다. 구독 시점에, 해당 이벤트가 성공했는지 실패했는지 여부에 따라서 구독하는 코드가 단 한번만 실행된다. 비동기 코드 작성에 매우 유용하다.

ex) restful api를 호출하고 해당 결과를 받아올 때, viewModel의 현재 시점에 상태를 받아와서 다른 작업을 수행할 때, 사용자 이벤트를 받고 DB를 업데이트할 때

**4. Maybe 5. Completable**

Single과 유사하게 단발성 이벤트를 상징하는 데이터 스트림.

차이점은 구독한 값을 받아오는 방식에 있다.

**Single**: onSucess, onError

**Maybe**: onSuccess, onComplete, onError

**Completable**: onComplete, onError

## **RxJava 함수**

**1. just**

파라미터를 통해 받은 데이터로 `Observable` 변수를 생성하는 함수. 모든 데이터가 수신되면 `onComplete()`가 수신된다.

**2. timer**

`timer` 함수는 호출된 시간부터 일정한 시간 동안 대기하고, long 타입 0을 송신 및 종료하는 `Observable`변수를 생성한다.

<-> `interval`은 조건까지 반복적으로 송신

```java
Observable
                .timer(2000, TimeUnit.MILLISECONDS)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe{
                    //2초 후 task 수행
                }

```

**3. map**

`Observable`에서 송신하는 데이터를 변환하고, 변환된 데이터를 송신하는 연산자이다. 하나의 데이터만 송신할 수 있으며, 반드시 데이터를 송신해야한다.

```java
//email check 
        val disposableEmail = RxTextView.textChanges(inputTextField[0])
            .map { t -> t.isEmpty() || Pattern.matches(emailRule, t) } //-> 송신하는 데이터를 true/false로 변환하여 subscribe로 넘김
            .subscribe({
                reactiveInputViewData(inputTextField[0].text.toString(), 0, it)
            }){

            }

```

**4. subscribe**

subscriber가 이벤트를 전달받기 위해 하는 행위

- **subscribeOn과 observeOn의 차이점**
    - **subscribeOn**: subscribe에서 사용할 스레드를 지정. 도중에 observeOn이 호출되어도 subscribeOn의 스레드 지정에는 영향을 끼치지 않음
    - **observeOn**: observable이 다음 처리를 진행할 때 사용할 스레드를 지정. observeOn이 선언된 후 처리가 끝나면 observeOn에서 선언한 스레드로 변경되어 이후의 처리를 진행한다.

    예를 들어서,

    ```java
    // 사용자의 데이터가 DB에 존재하는지 확인
    UserDatabase.getInstance(applicationContext)!!
            .getUserDao()
            .getUser(loginEmail, loginPWD)
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe({
                Log.d("TAG_login", "로그인 가능")
                Toast.makeText(this@MainActivity, "안녕하세요, " + it.nickname + "님", Toast.LENGTH_LONG).show()
                showInfo(it)
            },
               {
                   Log.d("TAG_login", "로그인 불가능")
                   showError()
               })

    ```

    으로 사용했다면, DB 접근이 필요한 getUser()가 io thread 안에서 실행되고, subscribe() 코드가 mainThread 안에서 실행된다.

    만약 observeOn을 지정하지 않는다면,

    `Can't toast on a thread that has not called Looper.prepare()`

    라는 오류가 발생한다. io thread에서는 toast를 할 수 없기 때문이다.