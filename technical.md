# 목차
## 코틀린 코루틴의 구성 요소와 특징
### 코루틴이란?
#### 코루틴을 사용하는 이유
#### 코루틴의 동작 방식
## 안드로이드에서 코루틴 사용하기
### 목표 : 코루틴을 이용해 사진 첨부 과정을 비동기 처리해보자
### 1. 생명주기에 따라 코루틴 제어하기
#### CoroutineScope
### 2. 각 사진마다 독립적인 코루틴 만들기
#### 구조화된 동시성
### 3. 코루틴 내부에 필요한 작업 정의하기
#### Coroutine Builder
### 4. 스레드 설정하기
#### Dispatcher

---

# **코틀린 코루틴의 구성 요소와 특징**

## **코루틴이란?**

### 코루틴을 사용하는 이유

안드로이드는 UI 작업을 메인 스레드에서만 수행하는 싱글 스레드 패러다임을 채택하고 있습니다.<br>
이러한 패러다임을 채택한 이유는 UI를 그리는 뷰의 구조적 특성 때문입니다.<br>
뷰들은 계층적인 구조를 가지며 이를 제대로 표시하기 위해서는 각 요소를 그리는 순서가 매우 중요합니다.<br>

만약 메인 스레드 외에 다른 스레드를 뷰를 그리기 위해 사용한다면, 동기화 문제로 예상치 못한 동작이 발생할 수 있습니다.<br>
그래서 안드로이드에서 메인 스레드는 **UI 스레드**라고도 불리며 일정 주기(약 16ms)마다 UI를 렌더링 합니다.<br>

<img width="770" alt="image" src="https://github.com/user-attachments/assets/e420ec4e-abca-4bf4-9c7a-83fbfcaadf89">

위 사진처럼 메인 스레드가 블록되어 업데이트가 적절한 타이밍에 일어나지 못한다면 프레임 드랍이 생기며 앱이 사용자와 매끄럽게 상호작용할 수 없어집니다.<br>
또한 blocking 상태가 5초 이상 유지되면 아래처럼 ANR(Application Not Responding) 오류가 발생하며 앱이 종료됩니다.<br>

이런 상황을 막기 위해선 네트워크 요청이나 파일 I/O 처럼 오랜 시간이 걸리는 작업을 시작한 후,<br>
결과를 기다리지 않고 다른 작업을 수행하다가 기존의 작업이 완료되면 그 결과를 처리하는 비동기 방식을 도입하는 게 중요합니다.<br>


### 코루틴의 작동 방식

<img width="414" alt="image" src="https://github.com/user-attachments/assets/d4dd6b58-f0fb-4d41-9428-20669d880f6a">

루틴이란 ‘특정한 일을 처리하기 위한 일련의 명령’입니다.<br>
서브 루틴은 루틴의 하위에서 실행되는 또 다른 루틴, 즉 함수 내부에서 호출되는 함수를 뜻합니다.<br>

이 서브 루틴은 하나의 진입점을 가지며 한번 호출되면 끝날 때까지 멈추지 않고 쭉 실행됩니다.<br>
따라서 어떤 루틴에 의해 서브 루틴이 호출되면, 루틴이 실행 되던 스레드가 다른 작업을 할 수 없습니다.<br>

<img width="414" alt="image" src="https://github.com/user-attachments/assets/90ec1163-1954-4abb-b554-2755e0a895dd">


반면 코루틴은 일시 중단 가능한 작업 단위로 특정 지점에서 작업을 멈추고 필요할 때 재개할 수 있습니다.<br>
자신이 작업을 하지 않을 때는 실행을 멈춘 후 다른 코루틴이 스레드를 사용하며 작업할 수 있도록 스레드 사용 권한을 양보합니다.<br>
코루틴은 일반적인 루틴과 달리 여러개의 진입점을 가지는데 함수가 처음 호출될 때와 중단 이후 재개할 때입니다.<br>

<img width="532" alt="image" src="https://github.com/user-attachments/assets/4b3f88f7-a7a3-4809-9eaa-42eb93333d44">

코루틴은 실행 정보를 저장하고 전달하기 위해 CPS(continuationPassingStyle) 방식을 사용합니다.<br>
함수를 호출할 때마다 Continuation을 전달한다는 뜻으로, Continuation 객체는 중단 시점마다 현재 상태들을 기억하고 다음에 무슨 일을 해야 할지를 담고 있는 확장된 콜백 역할을 합니다.<br>

- context : dispatcher, Job 등 CoroutineContext를 저장하는 변수
- resumeWith() : 실행을 재개하기 위한 메서드


# **안드로이드에서 코루틴 사용하기**

### 목표 : 코루틴을 이용해 사진 첨부 과정을 비동기 처리해보자!

스타카토 생성/수정 화면에는 갤러리에서 사진 여러 장을 선택해 첨부하는 기능이 있습니다.<br>
해당 기능의 요구사항 및 실행 흐름은 아래와 같습니다.

|사진 첨부 전|사진 업로딩 중|사진 업로드 완료|
|---|---|---|
|<img width="532" alt="1empty" src="https://github.com/user-attachments/assets/62f040b7-4a6d-4baf-9fe0-a77c20308b44">|<img width="532" alt="2loading" src="https://github.com/user-attachments/assets/8dac959a-1496-4a52-a6ef-02981e1e26cc">|<img width="532" alt="3finish" src="https://github.com/user-attachments/assets/4eef8c17-4fdf-4f26-bac6-1b9c55df7a44">|

1. 각 사진의 URI를 얻어와 MultiBody.Part로 변환한 뒤 Url을 얻기 위한 네트워크 요청을 보냅니다.
2. 요청에 대한 응답을 기다리는 동안 이미지 업로딩 애니메이션을 보여주고, 응답이 도착하면 UI를 업데이트 합니다.
3. 만약 이미지 업로딩 도중 X 버튼이 클릭되면 작업을 취소하고 리스트에서 사진을 삭제합니다.
4. 업로딩에 실패한 사진은 실패 및 재 업로드 버튼을 보여줍니다.

위 기능에 코루틴을 적용한 내용을 관련 개념들과 함께 흐름 순서대로 정리해 보았습니다.

## **1. 생명주기에 따라 코루틴 제어하기**

### CoroutineScope

코루틴은 실행 범위 및 생명주기를 관리하기 위해 CoroutineScope의 내부에서 실행되어야 합니다.<br>
현재, 사진 업로드 로직 관련 데이터는 안드로이드 Jetpack AAC ViewModel에서 관리하고 있습니다.<br>
따라서 메모리 누수를 방지하기 위해 코루틴이 뷰모델 생명주기를 따르도록 해야 합니다.

androidx.lifecycle은 ViewModel의 생명주기를 따르는 CoroutineScope인 `ViewModelScope`를 제공합니다.<br>
```kotlin
public val ViewModel.viewModelScope: CoroutineScope
    get() = synchronized(VIEW_MODEL_SCOPE_LOCK) {
        getCloseable(VIEW_MODEL_SCOPE_KEY)
            ?: createViewModelScope().also { scope -> addCloseable(VIEW_MODEL_SCOPE_KEY, scope) }
    }
```
`viewModelScope` 속성으로 접근할 수 있으며, ViewModel이 지워질 때 람다 내부에서 시작된 모든 코루틴을 자동으로 취소합니다.<br>
현 단계에서 구현하고자 하는 사진 별 네크워크 요청은 ViewModel이 활성화된 경우에만 수행해야 하는 작업이므로 이를 사용해주었습니다.

이 외에도 액티비티나 프래그먼트 등 안드로이드의 lifeCycleAware 컴포넌트에서 사용할 수 있는 `lifecycleScope`도 제공합니다.

## **2. 각 사진마다 독립적인 코루틴 만들기**

### 구조화된 동시성

코루틴은 ‘부모 코루틴’이 ‘자식 코루틴’의 실행을 관리하고 제어하는 계층 구조를 따릅니다.<br>
이를 코루틴의 **구조화된 동시성**이라고 하며, 이러한 특징 덕분에 코루틴들 간의 실행 흐름을 쉽게 관리하고 예외 및 취소 동작을 일관되게 처리할 수 있습니다.

**1. 실행 관리**

- **실행 범위 설정** : 명시적인 **스코프** 제공을 통해 어떤 코루틴이 실행될 수 있는지를 명확히 정의할 수 있습니다.<br>
`coroutineScope`나 `supervisorScope`를 사용해 코루틴이 구조화된 스코프 안에서 실행되도록 할 수 있습니다.
- **부모-자식 관계 :** 기본적으로 부모 코루틴은 자식 코루틴이 완료될 때까지 기다립니다.
- **동시성의 제어**: 특정 스코프(혹은 부모 코루틴)에 속한 여러 코루틴들은 서로 중단과 재개를 반복하며 실행됩니다.

```kotlin
// 두 코루틴이 모두 끝나기 전까지 coroutineScope는 완료되지 않음
coroutineScope {
    launch { /* 첫 번째 코루틴 */ }
    launch { /* 두 번째 코루틴 */ }
}
```

**2. 예외**

- **예외 전파**: 자식 코루틴에서 발생한 예외가 부모 코루틴으로 전파되며 이로 인해 예외가 발생하면 코루틴 계층 전체에 영향을 줍니다.
- **일관된 에러 핸들링**: 자식 코루틴에서 예외가 발생하면 부모 코루틴에서 적절한 조치를 취해 상위 레벨에서 예외 처리를 중앙화 할 수 있습니다.

```kotlin
// 하나의 자식 코루틴에서 예외가 발생하면 같은 스코프에 있는 다른 자식 코루틴들도 모두 취소됨
coroutineScope {
    launch { throw Exception("Error in coroutine") } // 자식 코루틴에서 예외 발생 시
    launch { /* 다른 자식 코루틴도 취소됨 */ }
}  // 부모 코루틴도 취소됨
```

**3. 취소**

- **부모가 자식을 취소**: 부모 코루틴이 취소되면, 그 하위의 모든 자식 코루틴도 자동으로 취소됩니다.
- **자식에서 취소가 발생해도 부모는 끝까지 관리**: 자식 코루틴이 취소되면, 그 코루틴이 속한 부모 코루틴도 취소되지만, 부모 코루틴은 필요한 경우 자식 코루틴의 모든 정리 작업을 수행하고 나서 종료될 수 있습니다.
- **협력적인 취소**: 코루틴은 취소 가능 상태일 때 스스로 취소를 협력적으로 처리하며, 부모 코루틴이 자식에게 취소 신호를 보낼 수 있습니다. 자식 코루틴이 취소될 때 `finally` 블록이나 `withContext(NonCancellable)` 같은 방식으로 정리 작업을 수행할 수 있습니다.

## 3. **코루틴 내부에 필요한 작업 정의하기**

## Coroutine Builder

코루틴을 만들고 시작하는 역할을 하는 함수들로, 수행하고자 하는 동작을 람다 함수로 정의할 수 있습니다.<br>
코틀린에서 제공하는 주요 코루틴 빌더로는 launch, async 등이 있습니다.

### **1. launch**

```kotlin
fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job

```

- 값을 반환하지 않는 코루틴을 생성/실행할 수 있습니다.
- `Job`을 반환하며, `cancel()`로 실행 중인 작업을 중단시킬 수 있습니다.

### **2. async**

```kotlin
fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T>
```

- 값을 반환하는 코루틴을 생성/실행할 수 있습니다.
- `Job`을 상속하는 `Deferred`를 반환하며, 이 때 T는 수행 결과 반환되는 값의 타입입니다.
- `await()`로 값이 반환될 때까지 기다린 후 결과값을 얻어 낼 수 있습니다.

위의 두 빌더로 만들어진 코루틴은 기본적으로 즉시 실행됩니다. (`CoroutineStart.DEFAULT`)

```kotlin
    fun fetchPhotosUrlsByUris(context: Context) {
        pendingPhotos.getValue()?.forEach { photo ->
            val job = createPhotoUploadJob(context, photo)
            job.invokeOnCompletion { _ ->
                photoJobs.remove(photo.uri.toString())
            }
            photoJobs[photo.uri.toString()] = job
        }
    }
```

- `photoJobs`라는 `Map`에 `photo.uri`를 Key로, 생성한 `job`을 Value로 저장해 작업 상태를 관리합니다.
- `invokeOnCompletion`을 사용하여 `job`이 완료되면(성공/실패 여부와 관계없이) 해당 `photo.uri`에 해당하는 `job`을 `photoJobs`에서 제거합니다.

```kotlin
    private fun createPhotoUploadJob(
        context: Context,
        photo: AttachedPhotoUiModel,
    ) = viewModelScope.launch(buildCoroutineExceptionHandler()) {
        val multiPartBody = convertExcretaFile(context, photo.uri, FORM_DATA_NAME)
        imageRepository.convertImageFileToUrl(multiPartBody)
            .onSuccess {
                updatePhotoWithUrl(photo, it.imageUrl)
            }
            .onException { _, message ->
                _errorMessage.postValue(message)
            }
    }
```

- `viewModelScope`에서  `launch`로 코루틴을 시작합니다.
- 사진 파일을 `convertExcretaFile`로 변환하고, 이를 서버로 업로드하여 URL로 변환하는 작업을 실행합니다.
- `imageRepository.convertImageFileToUrl`은 네트워크 통신을 수행하여 사진 파일을 URL로 변환하고, 성공 시 `updatePhotoWithUrl`로 변환된 URL을 저장합니다.
- 실패 시, `_errorMessage`에 에러 메시지를 게시합니다.

## **4. 스레드 설정하기**

### Dispatcher

`Dispatcher`는 코루틴이 실행될 스레드나 **스레드 풀을** 결정하는 역할을 합니다.<br>
각각은 특정 작업에 최적화되어 있으므로, 이를 고려한 적절한 선택을 통해 성능을 최적화하는 것이 중요합니다.<br>
주요 Dispatcher는 다음과 같습니다:

1. **`Dispatchers.Main`**:
    - **UI 스레드**에서 코루틴을 실행합니다.
    안드로이드의 경우 UI 업데이트는 항상 Main 스레드에서만 가능하므로, UI 관련 작업은 이 Dispatcher에서 실행해야 합니다.
2. **`Dispatchers.IO`**:
    - I/O 작업 등 블로킹 작업에 최적화된 스레드 풀에서 코루틴을 실행합니다.
    ex) 파일 입출력, 네트워크 요청, 데이터베이스 작업 등
    - 많은 스레드를 이용해서 비동기 작업을 병렬로 수행할 수 있습니다.
3. **`Dispatchers.Default`**:
    - CPU 집약적인 작업을 처리할 때 사용합니다. ex) 복잡한 알고리즘, 데이터 처리 작업
    `IO`와는 달리 계산량이 많은 작업이나 데이터를 처리할 때 사용하며 적절한 스레드 수를 유지하면서 CPU 성능을 최적화합니다.
4. **`Dispatchers.Unconfined`**:
    - 특정 스레드에 구애받지 않아 어떤 스레드로 옮겨갈지 보장되지 않습니다.
    - 일반적으로는 잘 사용하지 않으며 테스트 용도나 특수한 상황에서 사용됩니다.

### 코루틴 지원을 위한 설정

메서드에 suspend 키워드만 추가하면 Retrofit2가 내부적으로 코루틴을 통해 API 호출을 비동기로 수행합니다.<br>
이미지 전송을 위한 ImageApiService 코드는 아래와 같습니다.

```kotlin
interface ImageApiService {
    @Multipart
    @POST(IMAGE_PATH)
    suspend fun postImage(
        @Part imageFile: MultipartBody.Part,
    ): Response<ImageResponse>
}
```

`viewModelScope.launch`는 UI 스레드를 이용하지만, 위 `suspend fun postImage()`의 경우 Retrofit2가 내부적으로 IO 스레드를 사용하도록 처리하고 있기 때문에 별도로 Dispacher를 설정해 줄 필요는 없었습니다.
