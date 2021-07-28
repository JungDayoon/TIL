# Android Architecture Samples

# todo-mvp

**기능**

- Tasks - task 리스트를 관리
- TaskDetails - task 1개를 읽거나 지우기
- AddEditTask - task를 만들거나 수정(isNewTask 변수를 통해 add/edit 구분)
- Statistics - task의 통계를 보여줌

**Presenter를 구분하는 방식 - Google Architecture 사용**

- Contract - View와 Presenter에 대한 interface 작성
- Presenter - Contract.Presenter를 상속받아서 구현
    - 본질적으로는 MVC의 컨트롤러와 유사하지만, 뷰에 연결되는 것이 아니라 인터페이스로 연결된다는 점이 다르다.
    - 즉, Model과 View 사이의 자료전달 역할을 맡고있다.
- View - Contract.View를 상속받아서 구현

# todo-mvp-rxjava

```java
Flowable<List<Task>> getTasks();

Flowable<Optional<Task>> getTask(@NonNull String taskId);
```

데이터 모델 레이어는 task를 검색하는 방법으로 RxJava2 Flowable 스트림을 노출한다.

Presenter는 TasksRepository의 Flowable을 구독하고 조작한 다음, .subscribe(..) 방식으로 어떠한 view를 표시할 것인지 결정한다.

예를 들어서, StatisticsPresenter에서 active와 completed된 task의 계산을 수행할 스레드와 이 계산이 끝난 후에 수행해야 할 일들을 결정한다. → 모두 정상인 경우에는 그 통계를 보여주고, 필요할 경우에는 loading statistics error 표시.

```java
...
Disposable disposable = Flowable
    .zip(completedTasks, activeTasks, new BiFunction<Long, Long, Pair<Long, Long>>() {
        @Override
        public Pair<Long, Long> apply(Long completed, Long active) throws Exception {
            return Pair.create(active, completed);
        }
     })
     .subscribeOn(mSchedulerProvider.computation())
     .observeOn(mSchedulerProvider.ui())
     .subscribe(
         new Consumer<Pair<Long, Long>>() {
             @Override
             public void accept(Pair<Long, Long> stats) throws Exception {
                 mStatisticsView.showStatistics(stats.first, stats.second),
             }
         },
         new Consumer<Throwable>() {
             @Override
             public void accept(Throwable throwable) throws Exception {
                 mStatisticsView.showLoadingStatisticsError();
             }
         },
         new Action() {
             @Override
             public void run() throws Exception {
                 mStatisticsView.setProgressIndicator(false);
             }
     });
```

# Disposable

앞서 살펴본 대로, 우리는. subscribe(Observer)를 이용해서 데이터 스트림을 구독했습니다.

그런데 그걸로 끝인가요?

특정 시점까지만 구독하고 싶은 경우는 어떤가요?

이를 해결하기 위해서 RxJava에서는. subsribe(Observer)를 호출하면 Disposable이라는 객체를 반환합니다.

프로그래머는 이 객체의 인터페이스인 **dispose를 호출하여 구독을 취소**하고 Observable로부터 데이터를 받지 않을 수 있습니다.

- zip: 여러개의 Observable을 합쳐서 전송하게 된다. 특정 item끼리 합쳐서 두 개의 발행을 합쳐서 내려준다.

    ![Android%20Architecture%20Samples%2041529512b2b445c2a82262f40fd61e9c/_2020-12-02__3.06.41.png](Android%20Architecture%20Samples%2041529512b2b445c2a82262f40fd61e9c/_2020-12-02__3.06.41.png)

working thread의 handling은 RxJava의 스케줄러의 도움을 받아서 한다. 

예를 들어, 모든 데이터베이스 쿼리와 함께 데이터베이스 생성은 IO 스레드에서 수행된다.

subscribeOn과 observeOn 메소드들은 Presenter 클래스 내에서 사용된다. → Flowables가 computation 스레드에서 작동하며 observing은 메인 스레드에서 한다.

BiFunction<T, U, R> → 파라미터 타입이 T와 U이고, 리턴타입이 R인 함수이다. 

```java
@Override
    public void populateTask() {
        if (isNewTask()) {
            throw new RuntimeException("populateTask() was called but task is new.");
        }
        mCompositeDisposable.add(mTasksRepository
                .getTask(mTaskId)
                .subscribeOn(mSchedulerProvider.computation())
                .observeOn(mSchedulerProvider.ui())
                .subscribe(
                        // onNext
                        taskOptional -> {
                            if (taskOptional.isPresent()) {
                                Task task = taskOptional.get();
                                if (mAddTaskView.isActive()) {
                                    mAddTaskView.setTitle(task.getTitle());
                                    mAddTaskView.setDescription(task.getDescription());

                                    mIsDataMissing = false;
                                }
                            } else {
                                if (mAddTaskView.isActive()) {
                                    mAddTaskView.showEmptyTaskError();
                                }
                            }
                        },
                        // onError
                        throwable -> {
                            if (mAddTaskView.isActive()) {
                                mAddTaskView.showEmptyTaskError();
                            }
                        }));
    }
```

```java
@Override
    public Flowable<Optional<Task>> getTask(@NonNull final String taskId) {
        checkNotNull(taskId);

        final Task cachedTask = getTaskWithId(taskId);

        // Respond immediately with cache if available
        if (cachedTask != null) {
            return Flowable.just(Optional.of(cachedTask));
        }

        // Load from server/persisted if needed.

        // Do in memory cache update to keep the app UI up to date
        if (mCachedTasks == null) {
            mCachedTasks = new LinkedHashMap<>();
        }

        // Is the task in the local data source? If not, query the network.
        Flowable<Optional<Task>> localTask = getTaskWithIdFromLocalRepository(taskId);
        Flowable<Optional<Task>> remoteTask = mTasksRemoteDataSource
                .getTask(taskId)
                .doOnNext(taskOptional -> {
                    if (taskOptional.isPresent()) {
                        Task task = taskOptional.get();
                        mTasksLocalDataSource.saveTask(task);
                        mCachedTasks.put(task.getId(), task);
                    }
                });

        return Flowable.concat(localTask, remoteTask)
                .firstElement()
                .toFlowable();
    }
```

Edit시 저장되었던 Task를 띄우는 코드

mRepository에서 mTaskId를 통해 Task를 가져온다.(이 작업을 구독함) → 이 task는 io 작업이기 때문에 computation 스레드에서 진행

subscribe 수행이 끝나면 다시 ui 스레드로 돌아와서 작업을 마무리해야한다. → 다시 어느 thread를 사용할 것인지 정하는 것을 observeOn에서 함

다시 ui 스레드로 바뀐 상황에서 밑의 코드들을 진행 → 아까 받아온 task 가 존재하면, mAddTaskView가 active인지 확인. 

active라면 그 task의 제목과 내용을 설정해준다.

기존 코드와 비교

```java
@Override
    public void populateTask() {
        if (isNewTask()) {
            throw new RuntimeException("populateTask() was called but task is new.");
        }
        mTasksRepository.getTask(mTaskId, this);
    }
```

```java
@Override
    public void getTask(@NonNull final String taskId, @NonNull final GetTaskCallback callback) {
        checkNotNull(taskId);
        checkNotNull(callback);

        Task cachedTask = getTaskWithId(taskId);

        // Respond immediately with cache if available
        if (cachedTask != null) {
            callback.onTaskLoaded(cachedTask);
            return;
        }

        // Load from server/persisted if needed.

        // Is the task in the local data source? If not, query the network.
        mTasksLocalDataSource.getTask(taskId, new GetTaskCallback() {
            @Override
            public void onTaskLoaded(Task task) {
                // Do in memory cache update to keep the app UI up to date
                if (mCachedTasks == null) {
                    mCachedTasks = new LinkedHashMap<>();
                }
                mCachedTasks.put(task.getId(), task);
                callback.onTaskLoaded(task);
            }

            @Override
            public void onDataNotAvailable() {
                mTasksRemoteDataSource.getTask(taskId, new GetTaskCallback() {
                    @Override
                    public void onTaskLoaded(Task task) {
                        // Do in memory cache update to keep the app UI up to date
                        if (mCachedTasks == null) {
                            mCachedTasks = new LinkedHashMap<>();
                        }
                        mCachedTasks.put(task.getId(), task);
                        callback.onTaskLoaded(task);
                    }

                    @Override
                    public void onDataNotAvailable() {
                        callback.onDataNotAvailable();
                    }
                });
            }
        });
    }
```

mvp에서는 fragment내에 status 관리 코드가 필요

mvvm에서 가장 중요한 부분 - data binding 

viewmodel에서 마치 ui component처럼 사용하는 것 !

test 가능한 코드를 만들기 위해 - unit, ui, scenario test

—> 그 중에서도 unit test

ui component에 적혀있는지 확인하지 않아도 가능

sub component?