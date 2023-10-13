# 상태 홀더 및 UI 상태

- [상태 홀더 및 UI 상태](#상태-홀더-및-ui-상태)
  - [UI 상태 생성 파이프라인의 요소](#ui-상태-생성-파이프라인의-요소)
    - [UI 상태](#ui-상태)
    - [로직](#로직)
  - [Android 수명 주기와 UI 상태 및 로직 유형](#android-수명-주기와-ui-상태-및-로직-유형)
    - [UI 상태 생성 파이프라인](#ui-상태-생성-파이프라인)
  - [상태 홀더 및 책임](#상태-홀더-및-책임)
    - [상태 홀더 유형](#상태-홀더-유형)
  - [비즈니스 로직 및 상태 홀더](#비즈니스-로직-및-상태-홀더)
    - [비즈니스 로직 상태 홀더로서의 ViewModel](#비즈니스-로직-상태-홀더로서의-viewmodel)
  - [UI 로직 및 상태 홀더](#ui-로직-및-상태-홀더)
  - [상태 홀더의 ViewModel과 일반 클래스 중에서 선택](#상태-홀더의-viewmodel과-일반-클래스-중에서-선택)
  - [상태 홀더 혼합 가능](#상태-홀더-혼합-가능)
  - [래퍼런스](#래퍼런스)

---

이전 UI 레이어 가이드에서 UDF(단방향 데이터 흐름, Unidirectional Data Flow) 관리를 상태 홀더라고 하는 특수 클래스에 위임하여 여러 이점을 말했었음.  
이 문서에서는 상태 홀더, UI 레이어에서 상태 홀더의 역할을 자세히 살펴 볼 것임

이 문서를 통해 UI 레이어에서 앱 상태를 관리하는 방법, 즉 UI 상태 생성 파이프라인을 이해할 수 있음. 다음 사항을 이해하고 파악 할 수 있음.

- UI 레이어에 있는 UI 상태 유형 이해
- UI 레이어의 이러한 UI 상태에서 작동하는 로직 유형 이해
- 상태 홀더의 적절한 구현 (예: ViewModel 또는 간단한 클래스)을 선택하는 방법 파악

## UI 상태 생성 파이프라인의 요소

UI 상태와 이를 생성하는 로직이 UI 레이어를 정의함

### UI 상태

UI 상태는 UI를 설명하는 속성. UI 상태에는 두 가지 유형이 있음

- **화면 UI 상태** : 화면에 표시해야하는 항목.  
  예를 들어 `NewsUiState` 클래스에는 UI를 렌더링하는 데 필요한 뉴스 기사와 기타 정보가 포함될 수 있음. 이 상태는 앱 데이터를 포함하므로 대개 계층 구조의 다른 레이어에 연결 됨

- **UI 요소 상태** : 렌더링 방식에 영향을 주는 UI 요소에 고유한 속성을 나타냄.  
  UI 요소는 표시하거나 숨길 수 있으며 특정 글꼴이나 글꼴 크기, 글꼴 색상을 적용할 수 있음.  
  Android View에서 View는 기본적으로 스테이트풀(Stateful) 이므로 이 상태 자체를 관리하여 상태를 수정하거나 쿼리하는 메서드를 노출함. 텍스트에 관한 `TextView` 클래스의 `get` 및 `set` 메서드를 예로 들 수 있음.  
  Compose 에서 상태는 Composable 외부에 있으며 Composable 아주 가까이에서 호출 구성 가능한 함수나 상태 홀더로 승급할 수 도 있음. `Scaffold` Composable의 `ScaffoldState`를 예로 들 수 있음.

### 로직

UI 상태는 정적 속성이 아님. 시간이 지남에 따라 앱 데이터와 사용자 이벤트로 인해 UI 상태가 변경되기 때문. 로직은 변경된 UI 상태 부분, 변경 이유, 변경해야 하는 시점 등 구체적인 변경사항을 결정함.

![UI 상태 생산자인 로직](images/image07.png)

앱 로직은 비즈니스 로직 또는 UI 로직일 수 있음

- **비즈니스 로직** : 앱데이터에 대한 제품 요구사항의 구현.  
  예를 들어 사용자가 버튼을 탭할 때 뉴스 리더 앱에서 기사를 북마크에 추가함. 북마크를 파일이나 데이터베이스에 저장하는 이 로직은 일반적으로 도메인 또는 데이터 레이어에 배치됨. 상태 홀더는 일반적으로 노출되는 메서드를 호출하여 이 로직을 이러한 레이어에 위임함

- **UI 로직** : 화면에 UI 상태를 표시하는 방법과 관련이 있음.  
  사용자가 카테고리를 선택했을 때 올바른 검색창 힌트를 가져오는 것, 목록의 특정 항목으로 스크롤 하는 것, 또는 사용자가 버튼을 클릭할 때 특정 화면으로의 Navigation 로직을 예로 들 수 있음

---

## Android 수명 주기와 UI 상태 및 로직 유형

UI 레이어는 두 부분으로 구성 됨.  
하나는 UI 수명 주기에 종속되고 다른 하나는 UI 수명 주기와 무관함. 이렇게 분리하면 각 부분에 사용할 수 있는 데이터 소스가 결정되므로 다른 유형의 UI 상태와 로직이 필요함

- **UI 수명 주기와 무관** : UI 레이어의 이 부분은 앱의 데이터 생성 레이어 (데이터 또는 도메인 레이어)를 처리하고 비즈니스 로직으로 정의 됨.  
  UI의 수명 주기, 구성 변경 `Activity` 재생성은 UI 상태 생성 파이프라인의 활성화 여부에 영향을 줄 수 있지만 생성된 데이터 유효성에는 영향을 미치지 않음

- **UI 수명 주기에 종속** : UI 레이어의 이 부분은 UI 로직을 처리하며 수명 주기나 구성 변경사항의 직접적인 영향을 받음.  
  이러한 변경사항은 내부에서 읽은 데이터 소스의 유효성에 직접 영향을 미치므로 상태는 수명 주기가 활성 상태일 때만 변경될 수 있음.  
  런타임 권한과 구성 종속 리소스(예: 현지화된 문자열) 가져오기를 예로 들 수 있음

정리하자면 다음 표와 같음

|UI 수명주기와 무관|UI 수명 주기에 종속|
|-----|---|
|비즈니스 로직|UI 로직|
|화면 UI 상태||

### UI 상태 생성 파이프라인

UI 상태 생성 파이프라인은 UI 상태를 생성하기 위해 실행하는 단계를 나타냄.  
이러한 단계는 이전에 정의된 로직 유형을 적용하는 것으로 구성되며 UI의 요구사항에 완전히 종속됨.  
일부 UI는 파이프라인의 UI 수명 주기와 무관한 부분과 UI 수명 주기에 종속된 부분 모두에서 또는 둘 중 하나에서 이점을 얻을 수 있고 아무런 이득을 얻지 못할 수도 있음.

즉, UI 레이어 파이프라인의 다음과 같은 순열은 유효함

- UI 자체에서 생성 및 관리하는 UI 상태. 예를 들어 간단하고 재사용 가능한 기본 카운터는 다음과 같음

```kotlin
@Composable
fun Counter() {
    // UI 자체에서 UI 상태를 관리
    var count by remember { mutableStateOf(0) }
    Row {
        Button(onClick = { ++count }) {
            Text(text = "증가하기")
        }
        Button(onClick = { --count }) {
            Text(text = "감소하기")
        }
    }
}
```

- UI 로직 → UI
  사용자가 목록 상단으로 이동할 수 있는 버튼을 표시/숨기기

```kotlin
@Composable
fun ContactList(contacts: List<Contact>) {
    val listState = rememberLazyListState()
    val isAtTopOfList by rememver {
        derivedStateOf {
            listState.firstVisibleItemIndex < 3
        }
    }

    // LazyColumn 과 lazyListState 생성 
    ...

    // 스크롤 위치에 따라 리스트 상단으로 스크롤하는 버튼 표시하거나 숨기기 (UI 로직)
    AnimatedVisibility(visible = !isAtTopOfList) {
        ScrollToTopButton()
    }
}
```

- 비즈니스 로직 → UI
  화면에 현재 사용자의 사진을 표시하는 UI 요소

```kotlin
@Composable
fun UserProfileScreen(viewModel: UserProfileViewModel = hiltViewModel()) {
    // 비즈니스 로직인 상태 홀더에서 화면 UI 상태 읽기
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    // 사용자 아바타 사진 표시하는 Composable 호출
    UserAvatar(picture = uiState.profilePicture)
}
```

- 비즈니스 로직 → UI 로직 → UI  
  특정 UI 상태에 관해 화면에 올바른 정보를 표시하기 위해 스크롤하는 UI 요소

```kotlin
@Composable
fun ContactsList(viewModel: ContactsViewModel = hiltViewModel()) {
    // 비즈니스 로직인 상태 홀더에서 화면 UI 상태 읽기
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val contacts = uiState.contacts
    val deepLinkedContact = uiState.deepLinkedContact

    val listState = rememberLazyListState()

    // LazyColumn 과 lazyListSate 생성
    ...

    // 비즈니스 로직의 정보에 의존하는 UI 로직 수행
    if (deepLinkedContact != null && contacts.isNotEmpty()) {
        LaunchedEffect(listState, deepLinkedContact, contacts) {
            val deepLinkedContactIndex = contacts.indexOf(deepLinkedContact)
            if (deepLinkedContactIndex >= 0) {
                // 딥링크 아이템으로 스크롤
                listState.animateScrollToItem(deepLinkedContactIndex)
            }
        }
    }
}
```

두 가지 종류의 로직이 모두 UI 상태 생성 파이프라인에 적용되는 경우 **비즈니스 로직이 항상 UI 로직보다 먼저 적용되어야 함**  
UI 로직 다음에 비즈니스 로직을 적용하려고 하면 비즈니스 로직이 UI 로직에 종속된다는 것을 의미함.  
다음 섹션에서는 다양한 로직 유형과 상태 홀더를 자세히 살펴보며 이것이 문제가 되는 이유를 설명함.

![UI 레이어의 로직 적용](images/image08.png)

---

## 상태 홀더 및 책임

상태 홀더의 책임은 앱이 읽을 수 있도록 상태를 저장하는 것.  
로직이 필요한 경우 상태 홀더는 중개자 역할을 하며 필요한 로직을 호스팅하는 데이터 소스에 대한 액세스 권한을 제공함. 이러한 방식으로 상태 홀더는 로직을 적절한 데이터 소스에 위임함.

여기에는 다음과 같은 이점이 있음

- **간단한 UI** : UI가 상태를 바인딩 함
- **유지관리** : 상태 홀더에 정의된 로직을 UI 자체를 변경하지 않고도 반복 가능
- **테스트 가능성** : UI 및 상태 생성 로직을 독립적으로 테스트 가능
- **가독성** : 코드 리더가 UI 표시 코드와 UI 상태 생성 코드 간의 차이점을 명확하게 알아 볼 수 있음

크기나 범위와 관계 없이 모든 UI 요소는 상응하는 상태 홀더와 1:1 관계를 가짐. 또한 상태 홀더는 UI 상태 변경을 야기할 수 있는 모든 사용자 작업을 수락하고 처리할 수 있어야 하고 후속 상태 변경을 생성해야 함

> 상태 홀더가 반드시 필요한 것은 아님. 간단한 UI는 표시 코드와 함께 로직을 인라인으로 호스팅할 수 있음

### 상태 홀더 유형

UI 상태 및 로직의 종류와 마찬가지로 UI 레이어에는 UI 수명 주기와의 관계에 따라 정의되는 두 가지 유형의 상태 홀더가 있음

- 비즈니스 로직 상태 홀더
- UI 로직 상태 홀더

> UI 로직 상태 홀더가 데이터 또는 도메인 레이어의 정보에 종속되는 경우, 비즈니스 로직 상태 홀더에서 UI 로직 상태 홀더로 이 정보를 전달해야 함.  
> 이는 비즈니스 로직 상태 홀더가 UI 수명 주기와 무관하므로 UI 로직 상태 홀더보다 수명이 길기 때문임

---

## 비즈니스 로직 및 상태 홀더

비즈니스 로직 상태 홀더는 사용자 이벤트를 처리하고 데이터 또는 도메인 레이어에서 화면 UI 상태로 데이터를 변환 함.  
Android 수명 주기와 앱 구성 변경사항을 고려할 때 최적의 사용자 환경을 제공하려면 비즈니스 로직을 활용하는 상태 홀더에 다음 속성이 있어야 함

|속성|세부정보|
|------------|-----------------------------|
|UI 상태 생성|비즈니스 로직 상태 홀더는 UI의 UI 상태를 생성해야 함. 이 UI 상태는 종종 사용자 이벤틀르 처리하고 도메인 및 데이터 레이어에서 데이터를 읽은 결과임|
|Activity 재생성을 통해 유지됨|비즈니스 로직 상태 홀더는 Activity 재생성 전반에 걸쳐 상태 및 상태 처리 파이프라인을 유지하여 원활한 사용자 환경을 제공할 수 있도록 함. 상태 홀더를 유지할 수 없어 다시 만드는 경우(일반적으로 프로세스 종료 후) 상태 홀더는 일관된 사용자 환경을 보장하기 위해 마지막 상태를 쉽게 재생성할 수 있어야 함|
|장기 지속 상태 보유|비즈니스 로직 상태 홀더는 종종 탐색 대상의 상태를 관리하는 데 사용함. 따라서 Navigation 그래프에서 삭제될 때까지 Navigation 변경 후에도 상태를 유지하는 경우가 많음|
|UI에 고유하며 재사용할 수 없음|비즈니스 로직 상태 홀더는 일반적으로 특정 앱 기능(예: TaskEditViewModel 또는 TaskListViewModel)의 상태를 생성하므로 이 앱 기능에만 적용 됨. 동일한 상태 홀더가 다양한 폼팩터에서 이러한 앱 기능을 지원할 수 있음. 예를 들어 모바일, TV, 태블릿 버전의 앱은 동일한 비즈니스 로직 상태 홀더를 재사용할 수 있음|

> 비즈니스 로직 상태 홀더는 일반적으로 ViewModel 인스턴스로 구현됨. ViewModel 인스턴스가 위에서 설명한 여러 기능, 특히 Activity 재생성 시 유지 기능을 지원하기 때문임

예시로 [Now in Android](https://github.com/android/nowinandroid) 앱의 작성자 Navigation 대상을 살펴보자

비즈니스 로직 상태 홀더 역할을 하는 [`AuthorViewModel`](https://github.com/android/nowinandroid/blob/main/feature-author/src/main/java/com/google/samples/apps/nowinandroid/feature/author/AuthorViewModel.kt) 은 다음 사례에서 UI 상태를 생성함

```kotlin
@HiltViewModel
class AuthorViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val authorsRepository: AutorsRepository,
    newsRepository: NewsRepository
): ViewModel() {

    val uiState: StateFlow<AuthorScreenUiState> = ...

    // 비즈니스 로직
    fun followAuthor(followed: Boolean) {
        ...
    }
}
```

`AuthorViewModel`에는 앞에서 설명한 속성이 있음

**AuthorScreenUiState 생성**  
AuthorViewModel은 AuthorsRepository 및 NewsRepository에서 데이터를 읽고 AuthorScreenUiState를 생성하는 데 이 데이터를 사용함.  
또한 사용자가 AuthorsRepository에 위임하여 Author를 팔로우하거나 팔로우 해제하려고 할 때 비즈니스 로직을 적용함

**데이터 영역에 대한 액세스 권한 보유**  
AuthorsRepository및 NewsRepository 인스턴스가 생성자에서 전달되므로 Author를 따르는 비즈니스 로직을 구현할 수 있음

**Activity 재생성 시 유지**  
ViewModel로 구현되므로 빠른 Activity 재생성ㅅ에도 유지됨. 프로세스 종료의 경우 SavedStateHandle 객체를 읽어 데이터 레이어에서 UI 상태를 복원하는 데 필요한 최소한의 정보를 제공할 수 있음

**장기 지속 상태 보유**  
ViewModel의 범위가 Navigation 그래프로 지정되므로 작성자 대상이 Navigation 그래프에서 삭제되지 않는 한 uiState인 StateFlow의 UI 상태는 메모리에 유지됨.  
StateFlow를 사용하면 UI 상태의 수집기가 있는 경우에만 상태가 생성되므로 상태를 생성하는 비즈니스 로직의 적용을 지연시킬 수 있다는 이점도 추가 됨

**UI에 고유함**
AuthorViewModel은 작성자 Navigation 대상에만 적용되며 다른 곳에서는 재사용할 수 없음.  
Navigation 대상에서 재사용되는 비즈니스 로직이 있는 경우 해당 비즈니스 로직은 데이터 또는 도메인 레이어 범위 구성요소에 캡슐화 되어야 함

> ViewModel은 대상 수준 UI에서만 사용해야 함. 검색창이나 칩 그룹과 같이 UI의 재사용 가능한 부분에서 사용하면 안 됨. 이러한 경우에는 일반 클래스가 더 적합함
>
> 경고 : ViewModel 인스턴스를 다른 구성 가능한 함수로 전달 금지.  
> 전달하면 구성 가능한 함수가 ViewModel 유형과 결합되어 재사용성이 떨어지고 테스트와 미리보기가 더 어려워짐. 또한 ViewModel 인스턴스를 관리하는 명확한 단일 소스 저장소(SSOT)도 없음.  
> ViewModel을 전달하면 여러 Composable이 ViewModel 함수를 호출하고 상태를 수정할 수 있으므로 버그를 디버그하기가 더 어려워짐  
> 대신 UDF 권장사항을 따라 필요한 상태만 전달함. 마찬가지로 ViewModel의 Composable SSOT에 도달할 때 까지 전파 이벤트를 위로 전달함. 이벤트를 처리하고 상응하는 ViewModel 메서드를 호출하는 SSOT임

### 비즈니스 로직 상태 홀더로서의 ViewModel

Android 개발에서 **ViewModel이 가진 이점**이 있으므로, 비즈니스 로직에 대한 액세스 권한을 제공하고 화면에 표시하기 위한 앱 데이터를 준비하는 데는 ViewModel이 적합함. 이점은 다음과 같음

- ViewModel에 의해 트리거된 작업이 구성 변경에도 그대로 유지 됨
- Navigation 과의 통합
  - 화면이 백스택에 있는 동안 Navigation이 ViewModel을 캐시함. 이는 개발자가 대상으로 돌아갈 때 이전에 로드한 데이터를 즉시 사용할 수 있도록 하는 데 중요함. 이 작업은 Composable 화면의 수명 주기를 따르는 상태 홀더를 사용할 경우 더 어려워 짐
  - 또한 대상이 백스택에서 사라질 때 ViewModel도 삭제되기 때문에, 상태가 자동으로 정리됨. 이는 구성 변경으로 인한 새 화면으로의 이동 등 여러 이유로 발생 할 수 있는 Composable 폐기에 관한 리스닝과는 다름
- Hilt와 같은 다른 Jetpack 라이브러리와의 통합

---

## UI 로직 및 상태 홀더

UI 로직은 UI 자체에서 제공하는 데이터에 작동하는 로직임  
이는 UI 요소의 상태 또는 UI 데이터 소스(예: 권한 API나 Resources)에 있을 수 있음. UI 로직을 활용하는 상태 홀더에는 일반적으로 다음 속성이 있음

- **UI 상태 생성 및 UI 요소 상태 관리**

- **Activity 재생성 시 유지되지 않음**
  - UI 로직에서 호스팅 되는 상태 홀더는 종종 UI 자체의 데이터 소스에 종속되므로 구성 변경 시 이 정보를 유지하려고 하면 메모리 누수를 일으키는 경우가 많음.  
  상태 홀더가 구성 변경 시 데이터를 유지하려는 경우 Activity 재생성 시 유지되기에 더 적합한 다른 구성요소에 위임해야 함.  
  예를 들어 Compose 에서 `remembered` 함수로 만든 Composable UI 요소 상태는 Activity 재생성 전반에 걸쳐 상태를 유지하기 위해 `rememberSaveable`에 위임되는 경우가 많음. 이러한 함수의 예로는 `rememberScaffoldState()`와 `rememberLazyListState()`가 있음

- **UI 범위 데이터 소스 참조가 있음**
  - UI 로직 상태 홀더가 UI와 동일한 수명 주기를 가지므로 수명 주기 API 및 리소스와 같은 데이터 소스를 안전하게 참조하고 읽을 수 있음

- **여러 UI에서 재사용 가능**
  - 동일한 UI 로직 상태 홀더의 다양한 인스턴스가 앱의 여러 부분에서 재사용될 수 있음.  
  예를 들어 칩 그룹의 사용자 입력 이벤트를 관리하는 상태 홀더를 필터 칩의 검색 페이지와 이메일 수신자의 'to' 필드에 사용할 수 있음

일반적으로 UI 로직 상태 홀더는 일반 클래스로 구현됨. 이는 UI 자체가 UI 로직 상태 홀더 생성을 담당하고 UI 로직 상태 홀더의 수명 주기가 UI 자체의 수명 주기와 동일하기 떄문임.  
예를 들어 Compose 에서 상태 홀더는 컴포지션의 일부이며 컴포지션의 수명 주기를 따름

> 일반 클래스 상태 홀더는 UI 로직이 너무 복잡해서 UI 밖으로 이동할 때 사용 됨. 그 외의 경우 UI 로직은 UI에서 인라인으로 구현 가능

[Now in Android](https://github.com/android/nowinandroid) 샘플 앱을 통해 예시를 들어 보자

Now in Android 샘플 앱은 기기의 화면 크기에 따라 Navigation 시에 하단 앱 바 또는 Navigation 레일을 표시함. 작은 홤녀에서는 하단 앱 바를, 큰 화면에서는 탐색 레일을 사용함

구성 가능한 `NiaApp` 함수에서 사용되는 적절한 탐색 UI 요소를 결정하는 로직은 비즈니스 로직에 종속되지 않으므로 [`NiaAppState`](https://github.com/android/nowinandroid/blob/main/app/src/main/java/com/google/samples/apps/nowinandroid/ui/NiaAppState.kt) 이라는 일반 클래스 상태 홀더로 관리 할 수 있음

```kotlin
@Stable
class NiaAppState(
    val navController: NavHostController,
    val windowSizeClass: WindowSizeClass
) {
    // UI 로직
    val shouldShowBottomBar: Boolean
        get() = windowSizeClass.widthSizeClass == WindowWidthSizeclass.Compact || 
            windowSizeClass.heightSizeClass == WindowHeightSizeClass.Compact
    
    // UI 로직
    val shouldShowNavRail: Boolean
        get() = !shouldShowBottomBar
    
    // UI 상태
    val currentDestination: NavDestination?
        @Composable get() = navControlle
            .currentBackStackEntryAsState().value?.destination

    // UI 로직
    fun navigate(destination: NiaNavigationDestination, route: String? = null) { /* ... */ }
}
```

위 예에서 `NiaAppState`에 관한 다음 세부정보가 중요함

- **Activity 재생성시 유지되지 않음** : `NiaAppState`는 Compose 이름 지정 규칙을 따르는 구성 가능한 함수 `rememberNiaAppState`를 사용하여 만들어 컴포지션에서 `remembered` 됨.  
  Activity 가 재생성되면 이전 인스턴스가 손실되고, 재생성된 Activity의 새 구성에 적합한 모든 종속 항목이 전달된 상태로 새 인스턴스가 생성됨.  
  이러한 종속 항목은 새로운 것이거나 이전 구성에서 복원된 것일 수 있음. 예를 들어 `rememberNavController()`는 `NiaAppState` 생성자에서 사용되며, `rememberSaveable`에 위임하여 Activity 재생성 전반에 걸쳐 상태를 유지함

- **UI 범위 데이터 소스 참조가 있음** : `navigationController`, `Resources`, 기타 유사한 수명 주기 범위 유형에 관한 참조가 `NiaAppState`에 안전하게 보관될 수 있음. 동일한 수명 주기 범위를 공유하기 때문임

> 일반 상태 홀더 클래스는 검색창이나 칩 그룹과 같은 재사용 가능한 UI 부분에 사용하는 것이 좋음  
> 이 경우 VieWModel을 사용하면 안 됨. ViewMod디은 Navigation 대상의 상태를 관리하고 비즈니스 로직에 액세스 하는 데 사용하게 가장 적합하기 때문임

---

## 상태 홀더의 ViewModel과 일반 클래스 중에서 선택

위 섹션에서 ViewModel과 일반 클래스 상태 홀더 중 하나를 선택하는 일은 UI 상태에 적용되는 로직과 로직이 작동하는 데이터 소스에 따라 결정됨

> 대부분의 애플리케이션은 다른 경우라면 일반 클래스 상태 홀더에 배치될 수 있는 UI 자체에서 UI 로직을 인라인으로 실행하도록 선택함. 이는 간단한 사례에서는 괜찮지만 다른 상황에서는 로직을 일반 클래스 상태 홀더로 가져와 가독성을 개선할 수 있음

요약하면 아래 다이어그램은 UI 상태 생성 파이프라인에서 상태 홀더의 위치를 보여줌

![UI 상태 생성 파이프라인의 상태 홀더. 화살표는 데이터 흐름을 의미](images/image09.png)

**결과적으로 소비되는 위치와 가장 가까운 상태 홀더를 사용하여 UI 상태를 생성해야 함**  
좀 더 쉽게 말하면 적절한 소유권을 유지하면서 상태를 가능한 한 낮게 유지해야 함.  
비즈니스 로직에 액세스해야 하며 화면이 이동될 수 있는 한(Activity 재생성 시에도) UI 상태를 유지해야 하는 경우 비즈니스 로직 상태 홀더 구현에 ViewModel을 사용하는 것이 좋음. 단기 지속 UI 상태 및 UI 로직의 경우 수명 주기가 UI 에만 종속되는 일반 클래스로 충분함

---

## 상태 홀더 혼합 가능

상태 홀더는 종속 항목의 전체 기간이 같거나 더 짧다면 다른 상태 홀더에 종속될 수 있음. 예를 들면 다음과 같음

- UI 로직 상태 홀더는 다른 UI 로직 상태 홀더에 종속될 수 있음
- 화면 수준의 상태 홀더는 UI 로직 상태 홀더에 종속될 수 있음

다음 코드는 Compose의 `DrawerState`가 다른 내부 상태 홀더 `SwipeableState`에 종속되는 방식과 앱의 UI 로직 상태 홀더가 `DrawerState`에 종속될 수 있는 방식을 보여줌

```kotlin
@Stable
class DrawerState(/* ... */) {
    internal val swipeableState = SwipeableState(/* ... */)
    // ...
}

@Statble
class MyAppState(
    private val drawerState: DrawerState,
    private val navController: NavHostController
) { /* ... */ }

@Composable
fun rememberMyAppState(
    drawerState: DrawerState = rememberDrawerState(DrawerValue.Closed),
    navController: NavHostControler = rememberNavController()
): MyAppState = rememnber(drawerState, navController) { 
    MyAppState(drawerState, navContoller)
}
```

> 주의 : 화면 수준의 상태 홀더가 화면 또는 화면의 일부에 대한 비즈니스 로직 복잡성을 관리한다는 점을 고려할 때 화면 수준의 상태 홀더가 다른 화면 수준의 상태 홀더에 종속된느 것은 적절하지 않음  
> 이 시나리오에 해당한다면 화면과 상태 홀더에 관해 다시 생각해보고 필요한 것이 맞는지 확인 할 것

상태 홀더 보다 오래 지속되는 종속 항목의 예로는 화면 수준의 상태 홀더에 종속되는 UI 로직 상태 홀더가 있습니다. 이렇게 하면 단기 지속 상태 홀더의 재사용성이 저하되고 실제로 필요한 것보다 더 많은 로직과 상태에 액세스할 수 있음.

단기 지속 상태 홀더에 더 높은 범위의 상태 홀더에 관한 특정 정보가 필요한 경우 상태 홀더 인스턴스를 전달하는 대신 필요한 정보만 매개변수로 전달할 것.  
예를 들어 다음 코드는 UI 로직 상태 홀더 클래스가 전체 ViewModel 인스턴스를 종속 항목으로 전달하는 대신 VieModel에서 필요한 항목만 매개변수로 수신함

```kotlin
class MyScreenViewModel(/* ... */) {
    val uiState: StateFlow<MyScreenUiState> = /* ... */
    fun doSomething() { /* ... */ }
    fun doAnotherTing() { /* ... */ }
}

@Stable
class MyScreenState(
    // 이 클래스와 같은 일반 클래스에 ViewModel 인스턴스를 절대로 전달 금지
    // private val viewModel: MyScreenViewModel <- 금지

    // 대신에 필요한 종속 항목만 전달할 것
    private val someState: StateFlow<SomeState>,
    private val doSomething: () -> Unit,

    // 다른 UI 범위 타입
    private val scaffoldState: ScaffoldState
) { /* ... */ }

@Composable
fun rememberMyScreenState(
    someState: StateFlow<SomeState>,
    doSomething: () -> Unit,
    scaffoldState: ScaffoldState = rememberScaffoldState()
): MyScreenState = remember(someState, doSomething, scaffoldState) {
    MyScreenState(someState, doSomething, scaffoldState)
}

@Composable
fun MyScreen(
    modifier: Modifier = Modifier,
    viewMdoel: MyScreenViewMdoel = viewModel(),
    state: MyScreenState = rememberMyScreenState(
        someState = viewModel.uiState.map { it.toSomeState() },
        doSomething = viewModel::doSomething
    ),
    // ...
) { /* ... */ }
```

다음 다이어그램은 위 코드의 UI 와 다양한 상태 홀더 간의 종속 관계를 보여줌

![다양한 상태 홀더에 종속된 UI. 화살표는 종속 항목을 의미](images/image10.png)

---

> UI 에서 상태관리를 중점적으로 알아보는 파트였다. ViewModel 뿐만 아니라 UI 자체에서도 일반 클래스를 통해 상태를 관리해도 좋다는걸 알아갔다.
> 이로 인해 비즈니스 로직과 UI 로직을 더 구별지어 정의할 수 있어서 코드 가독성, 유지관리성, 테스트 가능성이 모두 좋아진다는 것이다.  
> 해당 가이드에서 모든 예시가 Compose로 나와있지만 iOS의 SwiftUI로 선언형 UI에 학습이 돼있어서 이해하는데에 어려움이 없었다. 확실히 구글에서 지양하는 앱 아키텍처와 이에 따른 Jetpack 라이브러리들은 Compose 사용을 엄두에 두고 있는 것 같다. 현재 내 지식으로는 아직 Compose와 xml을 통한 View의 차이점을 명확하게 잘 모르겠다. 선언형이나 명령형이냐 차이를 말하는게 아니라 View에서도 데이터 바인딩을 이용하면 선언형과 비슷하거나 같게 사용할 수 있는게 아닌가 싶은 생각이다. 이 부분에 대해서는 내가 직접 찾아봐서 좀 더 알아봐야 겠다.

## 래퍼런스

[안드로이드 앱 아키텍처 가이드/UI 레이어/상태 홀더 및 UI 상태](https://developer.android.com/topic/architecture/ui-layer/stateholders?hl=ko)
