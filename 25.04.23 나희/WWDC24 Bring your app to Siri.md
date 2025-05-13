# WWDC24: Bring your app to Siri

## **기존 프레임워크**

### **SiriKit (iOS 10 도입)**

<img src="./WWDC24 Bring your app to Siri/스크린샷_2025-04-23_오후_7.59.22.png">

- Apple이 제공하는 시스템 인텐트를 활용해 앱에 Siri 기능을 연동할 수 있음
- 예시: 음악 재생, 문자 보내기 등

### **App Intents (iOS 16 도입)**

<img src="./WWDC24 Bring your app to Siri/스크린샷_2025-04-23_오후_7.59.36.png">

- Siri뿐 아니라 Shortcuts(단축어), Spotlight 등 다양한 시스템 기능과의 통합을 지원하는 최신 프레임워크
- SiriKit에 해당 도메인이 없다면, App Intents가 기본 선택지

## **Apple Intelligence와 함께 개선된 Siri**

<img src="./WWDC24 Bring your app to Siri/스크린샷_2025-04-23_오후_7.59.50.png">

- 대규모 언어 모델 기반으로 Siri가 더욱 똑똑해짐
- 주요 개선사항 3가지
    1. 자연스럽고 사람 같은 말투
    2. 화면 인식 기반의 문맥 대응 → 사용자가 보고 있는 화면을 이해하고 거기에 맞춰 행동
    3. 더 풍부한 언어 이해 → 더듬거나 꼬인 말도 의도를 파악함

## App Intents + Apple Intelligence: 새로운 통합 방식

<img src="./WWDC24 Bring your app to Siri/스크린샷_2025-04-23_오후_8.00.33.png">

- App Intents를 기반으로 Siri와 앱을 연결하는 **App Intent Domains** 도입
    - 기능 유형별로 정리된 API 그룹 (예: Books, Camera, Photos, Spreadsheets 등)
    - iOS 18 기준, 총 12개 도메인 제공 (Mail, Photos 도메인은 현재 사용 가능)
    - 예시: "어제 Mary 찍은 사진에 시네마틱 프리셋 적용해줘" → 사진 편집 앱에서 `SetFilter` 인텐트 실행
    - Siri는 이 도메인 기반으로 **100개 이상의 액션** 지원
    - 앱 기능이 도메인과 겹친다면 바로 앱에 적용 가능

---

## 이걸 사용하기 위해 새로 도입된 아이

### **Assistant Schema (어시스턴트 스키마)**

<img src="./WWDC24 Bring your app to Siri/스크린샷_2025-04-23_오후_8.01.44.png">

<img src="./WWDC24 Bring your app to Siri/스크린샷_2025-04-23_오후_8.01.53.png">

- Siri가 이해할 수 있도록 **의도를 특정 형태**로 구조화함
- 개발자가 해야 할 일은 오직 `perform()` 메서드만 구현하는 것
→ 자연어 해석, 문맥 추론 등은 Apple Intelligence가 처리

### Schema란?

- Siri가 이해할 수 있도록 학습된 인텐트의 구조화된 형태
- Apple은 인텐트를 "스키마"에 맞춰 정형화하고, 이를 기반으로 음성 명령을 이해함
- 즉, 올바른 스키마에 맞춰 Intent를 만들면 Siri가 나머지를 처리함

## Siri 요청이 처리되는 흐름 (Lifecycle)

<img src="./WWDC24 Bring your app to Siri/스크린샷_2025-04-23_오후_8.02.32.png">

1. 사용자가 Siri에게 요청을 보냄
2. Apple Intelligence 모델이 요청을 처리하고, 어떤 도메인/스키마에 해당하는지 추론
    - Apple Intelligence는 **Siri가 명령을 이해하고 행동하도록** 학습된 언어 모델(LLM)을 사용
    - Siri가 명령을 해석하고 **적절한 액션으로 연결**해줄 수 있도록 학습
    - Siri는 "앱이 어떤 기능을 갖고 있는지" 정확히 알아야함 → 그래서 스키마 사용하자~
3. 기기 내 AppIntent 모음(toolbox)에서 해당 스키마에 매칭되는 intent를 찾음
    - 이 툴박스에는 기기에 설치된 모든 앱의 App Intent들이 **스키마별로 그룹화되어 저장**
    - 인텐트를 스키마에 맞게 정의하면, 모델은 그 인텐트를 **이해하고 추론할 수 있는 능력**을 얻게됨
4. 해당 AppIntent의 perform() 실행
5. 결과는 Siri 응답 UI로 보여짐 (또는 음성으로 읽힘)

## 그래서 우리 이거 어떻게 사용하는데?

이러한 스키마에 맞게 구현하기 위해, 아래 세개의 도구를 활용할 수 있음

| 역할 | 비유 |
| --- | --- |
| Assistant Schema | "이 버튼을 누르면 뭘 해야 하는가"라는 사양서 |
| AssistantIntent | 그 사양서대로 작동하는 버튼의 기능 구현 |
| AssistantEntity | 그 버튼이 다룰 데이터(예: 앨범, 사진)의 설계도 |
| AssistantEnum | 선택지 (예: 색상, 종류)를 정의한 메뉴 |

### **기존 AppIntent 방식**

AppIntents는 기본적으로 꽤 많은 메타데이터를 요구

<img src="./WWDC24 Bring your app to Siri/스크린샷_2025-04-23_오후_8.02.42.png">

### **Assistant Intent 방식**

<img src="./WWDC24 Bring your app to Siri/스크린샷_2025-04-23_오후_8.05.41.png">

- `@AssistantIntent`라는 Swift 매크로를 붙이면, Siri는 이 인텐트가 무엇을 하는 것인지 바로 이해할 수 있음
    - 툴박스에서 시리는 이걸 보고 찾음
- title, description 모두 생략 가능
- **스키마**를 기반으로 만들면, 이미 앨범 생성에는 title과 color가 들어간다는 걸 Siri가 알고 있기 때문에,
일일이 정의하지 않아도 됨.
- 그래서 우린 perform 함수만 작성하면 됨 → 해당 결과 siri에게 반환

## App Entity

<img src="./WWDC24 Bring your app to Siri/스크린샷_2025-04-23_오후_8.19.10.png">

- AppIntents 프레임워크는 단순히 액션을 만들기 위한 인텐트만 포함하는 게 아님
    - 앱 내부의 개념을 모델링하는 엔티티(entity)도 함께 포함
    - 인텐트가 반환하는 AlbumEntity와 같은 것
- AppEntity를 Siri에 노출하기 위한 **새로운 매크로** 추가
    - 인텐트처럼, 이 새 타입을 사용하는 것도 **AppEntity 선언에 코드 한 줄만 추가**
    - AssistantEntity도 사전에 정의된 스키마(shape)의 혜택을 받을 수 있음
    - 그 결과 **더 간결한 AppEntity 구현**이 가능
        - 필요하다면, **추가적인 옵셔널 프로퍼티를 선언**함으로써
        - AssistantEntity도 **기존 스키마를 넘어 확장**할 수 있음
            - 예를 들어 이 앨범 엔티티에 있는 앨범 색상(Album Color) 속성

## App Enum

- AppEnum을 Siri에 노출하는 것도 **Entity나 Intent만큼 간단**
    - enum 선언에 AssistantEnum 매크로만 추가하면 됨
        - 하지만 Entity나 Intent와 달리, **AssistantEnum은 enum 케이스에 어떤 스키마 형태도 강제하지 않음**
        - 즉, **enum을 완전히 자유롭게 구현**
    - @AssistantEnum(schema:)를 쓰면 Apple이 미리 정의한 스키마 덕분에 typeDisplayRepresentation은 **불필요하게 중복**되기 때문에 생략 가능

<img src="./WWDC24 Bring your app to Siri/스크린샷_2025-04-23_오후_8.21.35.png">

<img src="./WWDC24 Bring your app to Siri/스크린샷_2025-04-23_오후_8.24.29.png">

해당 도구들을 기기의 사진(Asset)을 열거나 즐겨찾기하는 기능을 **AppIntent**로 구현하고,
이를 **Siri**와 **Shortcuts 앱** 에서 동작하도록 구현하는 예시로 확인해보자

### 앱 내 Asset 모델 정의

```swift
struct Asset {
    let id: UUID
    let title: String
    let creationDate: Date
    var isFavorite: Bool
}
```

### AppEntity 정의

사진(Asset)을 Siri가 이해할 수 있도록 모델링한 타입

```swift

@AssistantEntity(domain: .photos, schema: .asset)
struct AssetEntity: AppEntity {
    var id: UUID
    var title: String
    var creationDate: Date
    var isFavorite: Bool
    var hasSuggestedEdits: Bool  // 스키마에 요구되는 속성

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(title)")
    }

    var persistentIdentifier: UUID { id }

    static var typeDisplayName: String = "Photo"

    static func query(for identifiers: [UUID]) -> [AssetEntity] {
        // 해당 ID들을 기반으로 실제 Entity 조회
    }

    static func allEntities() -> [AssetEntity] {
        // 예시: 전체 AssetEntity 리스트 반환
    }
}

```

### AppIntent 1: 특정 사진 열기 (`OpenAssetIntent`)

- @AssistantIntent 매크로에 domain과 schema를 지정하면, Siri가 이 Intent를 Apple Intelligence 기반으로 처리할 수 있게 됨
- perform 실행
    - 여기서는 해당 사진 오픈

```swift
@AssistantIntent(domain: .photos, schema: .openAsset)
struct OpenAssetIntent: AppIntent {
    @Parameter(title: "Photo")
    var asset: AssetEntity

    func perform() async throws -> some IntentResult {
        NavigationManager.shared.open(assetID: asset.id)
        return .result()
    }
}
```

### AppIntent 2: 즐겨찾기 추가 (`FavoriteAssetIntent`)

```swift
@AssistantIntent(domain: .photos, schema: .favoriteAsset)
struct FavoriteAssetIntent: AppIntent {
    @Parameter(title: "Photo")
    var asset: AssetEntity

    func perform() async throws -> some IntentResult {
        AssetStore.shared.markFavorite(id: asset.id)
        return .result()
    }
}
```

이렇게 기능을 추가하면 Shortcuts(단축어)에서 확인이 가능하다.

### 어떤 프레임워크를 써야 할까?

| 조건 | 사용 프레임워크 |
| --- | --- |
| Apple이 정의한 도메인에 포함됨 | SiriKit |
| 커스텀 기능 또는 새로운 영역 | App Intents |

## 퀴즈

1. Siri와 앱을 통합하는 데 사용되는 두 가지 주요 프레임워크는 무엇인가요?
2. App Intents와 함께 사용되며, Siri가 의도를 이해하고 실행할 수 있게 도와주는 구조화된 설명 단위는?