# WWDC23 - Meet mergeable libraries

## Static / Dynamic Library

<img src="./WWDC23 - Meet mergeable libraries/스크린샷_2025-04-06_오후_7.40.16.png">

- 여러 개의 오브젝트 파일(`.o` )을 하나로 묶은 파일
- 정적 라이브러리를 앱에 링크하면, **해당 라이브러리의 코드가 앱 바이너리에 복사**
    - 빌드할 때마다 코드가 복사되므로 빌드 시간이 길어짐
    - 정적 라이브러리의 코드가 바뀌거나 더 많은 라이브러리를 사용할수록 빌드 성능 **↓**
    - 런타임에 외부 라이브러리를 로드할 필요 없음 → 앱 실행이 빠름

<img src="./WWDC23 - Meet mergeable libraries/스크린샷_2025-04-06_오후_7.41.49.png">

- Xcode에서 프레임워크 타겟에 사용되는 바이너리 파일 형식
- 코드는 실행 파일로 복사되지 않고 해당 라이브러리의 설치 경로만 앱 바이너리에 기록해둠
    - 코드를 복사할 필요가 없음 → 빌드 속도 빠름
    - 런타임 시, dyld가 프레임워크 의존성을 찾아 로드해야 함
        - 사용하는 라이브러리가 많아질수록, 메모리 사용량과 앱 실행 시간이 늘어남

정적 라이브러리와 동적 라이브러리는 서로 장단점이 존재함

동적 라이브러리는 빌드 시간에는 거의 영향을 주지 않지만 **앱 실행 시간이 느려**지고,
정적 라이브러리는 앱 실행 시간에는 영향이 없지만 **빌드 시간이 길어**짐

## Mergeable Library?

<img src="./WWDC23 - Meet mergeable libraries/스크린샷_2025-04-06_오후_7.47.46.png">

Mergeable 라이브러리는 정적 링크와 동적 링크 방식의 장점을 모두 가져옴

앱을 개발할 때 **기능별로 모듈화**해서 개발해야 코드가 명확하게 나뉘고, 유지보수도 쉬워짐
이렇게 나누다 보면 프레임워크가 너무 많아져서 앱 성능이 떨어질 수도 있음

→ 프레임워크를 병합해서 최적화하는 것도 필요

### 동작방식

```swift
ForestApp (앱 타겟)
├─ ForestBuilder.framework
├─ ForestUI.framework
├─ Forest.framework
└─ SwiftUI.framework (Apple SDK)
```

라이브러리는 동적(.framework, .dylib) 형식이지만

Xcode에서 앱을 빌드할 때, 해당 라이브러리의 **코드를 앱 바이너리에 병합(merge)**

병합된 후에는 원래 라이브러리 파일이 앱 번들에서 **제거될 수 있음**

<img src="./WWDC23 - Meet mergeable libraries/스크린샷_2025-04-06_오후_9.51.16.png">

```swift
ForestApp
├─ ForestKit.framework (병합된 결과)
│  ├─ Forest.framework         → 병합됨
│  ├─ ForestBuilder.framework  → 병합됨
│  └─ ForestUI.framework       → 병합됨
├─ SwiftUI.framework           → 병합 X
└─ ForestTestSupport.framework → 병합 X
```

- **ForestBuilder, ForestUI, Forest → ForestKit에 병합됨**
- 앱은 이제 **ForestKit 하나만 링크**
    - 앱이 여러 프레임워크를 직접 링크하면 빌드 설정이 복잡해지고
        
        런타임에는 dyld가 각각을 로드해야 해서 **실행 성능, 메모리 부담이 커짐**
        
- Apple SDK 프레임워크는 앱 번들에 포함되는 게 아니라, 시스템에 이미 있기 때문에 **병합 대상이 아님**
- 테스트 타겟 등 다른 타겟이 사용하는 경우, 또는 병합하면 안 되는 상황이면 → 병합 대상에서 제외

### Merge 과정 (Release)

- Mergeable 라이브러리는 `make_mergeable` 옵션을 사용해 빌드
    - 옵션은 라이브러리 내부에 **병합을 위한 메타데이터**를 추가
        - Meta Data?
            - 링커가 이 라이브러리가 Mergable 한 애라는 걸 알려줌
            - 병합 시 중복된 코드(문자열, 심볼 등)을 제거할 수 있게 함
            - 심볼 정보 및 소스 위치 정보를 포함하여 디버깅도 지원
- 링커는 라이브러리 코드를 앱 실행 파일에 병합 + 중복 제거
- 병합된 원래의 dylib 파일은 디스크에서 제거
- 병합 결과물은 기존과 동일한 실행 파일 형식 유지 (.app, .framework 등)

Mergeable 라이브러리의 병합 과정이 정적 라이브러리랑 같다면, 빌드 속도도 다시 느려지는 것 아닌가?

**→ ✅ Debug 모드에서는 `merge`가 아니라 `reexport` 방식으로 처리**

- 병합된 프레임워크 하나만 링크
    - 예를 들어 앱이 `ForestKit` 하나만 링크했는데,
        
        `ForestBuilder`, `ForestUI`, `Forest` 코드는?
        
        → `ForestKit`이 이 프레임워크들을 **reexport**로 연결해줌
        
- 링커는 해당 프레임워크가 내부적으로 어떤 라이브러리를 reexport하고 있는지 기록
- 병합할 라이브러리는 실제로는 디스크에 그대로 존재
- 런타임에서 dyld는 이 기록을 보고 해당 의존성들을 함께 로드

즉, 개발·디버깅·테스트 단계에서는 속도 저하 없음

| Release | 코드를 **실제로 복사해서 병합** |
| --- | --- |
| Debug | **참조만(reexport)** 하고, 디스크에서 동적으로 로드됨 |

**동적 라이브러리처럼 쓰면 되는거 아닌가?**

- 그냥 동적 라이브러리처럼 각 프레임워크를 다 따로 링크하면 될 것 같지만,
    
    → 그렇게 하면 Debug와 Release의 구조가 달라지고 (앱은 Debug/Release 모두 `ForestKit`만 링크하면 됨)
    
    → 릴리즈 빌드 시 병합된 라이브러리 파일이 없어지면 빌드 오류가 발생할 수 있음
    
- 그래서 reexport를 쓰면 **Debug/Release 모드 모두 하나의 병합 대상만 참조하는 구조로 유지** 가능함

## 적용 방법

### 자동 병합 (Automatic Merging)

- `MERGED_BINARY_TYPE = Automatic`
- 빌드 시스템에 직접 연결된 모든 프레임워크 타겟을 자동으로 병합
    - 링커가 이들을 **앱 실행 파일에 병합**하고, 런타임에는 디스크에서 제거
    - dyld가 로드하지 않아도 됨 → 앱 실행 시간, 메모리 사용량 줄어듬

### 수동 병합 (Manual Merging)

```swift
ForestApp
├─ ForestKit.framework (병합된 결과)
│  ├─ Forest.framework         → 병합됨
│  ├─ ForestBuilder.framework  → 병합됨
│  └─ ForestUI.framework       → 병합됨
├─ SwiftUI.framework           → 병합 X
└─ ForestTestSupport.framework → 병합 X
```

- 타겟에서 `MERGED_BINARY_TYPE = manual`
- 개별 프레임워크에서 `MERGEABLE_LIBRARY = YES`
- 원하는 프레임워크만 선택적으로 병합 가능
- 테스트 타겟은 **Debug 모드**로만 빌드
    - 병합하려 해도 어차피 병합은 안 일어남 → 병합 설정 자체가 의미 없음
    - 테스트는 배포 대상이 아니고, 성능보다 정확한 검증이 우선

### 병합된 프레임워크에서 export 심볼 관리하기

프레임워크를 병합한 경우엔, **이미 앱 실행 파일 안에 코드가 복사되었기 때문에 export할 필요가 없음.**

- 병합된 라이브러리는 **기본적으로 export 심볼을 앱 안에 그대로 남김**
- 그런데 대부분의 앱은 외부에 심볼을 내보낼 필요가 없고,
    
    심볼이 남아 있으면 **앱 크기가 커지고, 빌드 시간이 느려질 수 있음**
    

→ 이럴 땐 다음 옵션을 써서 export 심볼 남기지 않게 처리

```swift
Other Linker Flags: -Wl,-no_exported_symbols
```

### 특정 symbol만 export해야 할 때 — Exported Symbols File

<img src="./WWDC23 - Meet mergeable libraries/스크린샷_2025-04-06_오후_9.55.34.png">

앱 익스텐션이나 외부에서 호출되는 진입점이 필요한 경우, 일부 심볼만 예외적으로 export해야 하는 경우도 있음

예를 들어 App Extension에서 메인 앱의 특정 함수를 호출해야 한다면, 진입점이 export되어 있어야 dyld가 찾을 수 있잖아요?

이럴 땐 전부 export하거나 다 막을 필요 없이, 필요한 symbol만 **리스트로 명시합니다**

```swift
General > Linking > Exported Symbols File
```

## 적용 시 주의할 점

### 기존 실행 파일/모듈이 mergeable 라이브러리에 의존하는 경우

<img src="./WWDC23 - Meet mergeable libraries/스크린샷_2025-04-07_오전_1.02.37.png">

: 병합된 라이브러리는 디스크에서 제거되므로, **기존에 해당 프레임워크를 직접 참조하던 실행 파일**은

병합된 프레임워크를 참조하도록 의존성 변경 필요

### Autolinking 사용 중일 때

<img src="./WWDC23 - Meet mergeable libraries/스크린샷_2025-04-07_오전_1.03.31.png">

- Swift 컴파일러는 import 문만으로도 자동으로 해당 프레임워크를 링크 (Autolinking)
- 병합된 프레임워크를 쓰는 경우엔, **명시적으로 병합된 프레임워크만 링크하도록 설정**해야 문제가 발생하지 않음

### dlopen, NSBundle 등 런타임 로딩 API 사용 시

<img src="./WWDC23 - Meet mergeable libraries/스크린샷_2025-04-07_오전_1.08.29.png">

- `dlopen`, `Bundle.module`, `NSBundle.bundle(for:)` 같은 API는
    
    런타임에 **프레임워크의 물리적 위치**를 참조하는 구조
    
- 병합된 경우 디스크에 바이너리가 없기 때문에 → **이 API들이 실패할 수 있음**

### 💡 해결 방법

- iOS 12 이상부터는 병합된 라이브러리에서도 번들 탐색 가능하도록 **hook**이 제공됨
- 최소 지원 버전이 iOS 12 이상이면 문제 없음
- 만약 안 쓴다면 → `no_merged_libraries_hook` 옵션으로 해당 기능 비활성화해도 됨

### 사용 중인 정적 링커 버전

- Mergeable Libraries는 **Xcode 15의 새 정적 링커**가 필요함
- 기존 링커는 armv7k (watchOS 8 이하) 같은 이전 아키텍처는 지원하지만,
    
    새 링커는 지원하지 않음
    
    → **watchOS 9 이상으로 최소 버전 올려야 새 링커 사용 가능**
    

### 병합된 프레임워크를 외부에 배포할 때

<img src="./WWDC23 - Meet mergeable libraries/스크린샷_2025-04-07_오전_1.10.39.png">

- `-make_mergeable` 옵션으로 빌드된 프레임워크에는 **병합용 메타데이터가 포함됨**
    - 메타데이터가 포함되므로 **dylib 크기는 커지지만**,
        
        앱에 병합한 뒤에는 메타데이터가 제거되어 앱 크기에는 영향 없음
        

### 기존 정적 라이브러리도 가능하면 동적으로 바꾸고 병합하도록 권장

<img src="./WWDC23 - Meet mergeable libraries/스크린샷_2025-04-07_오전_1.12.17.png">

- 정적 라이브러리는 병합 자체가 안 됨 (이미 앱 바이너리에 복사되기 때문)
- 동적 + 병합 으로 만들면:
    - **빌드 유연성**
    - **성능 최적화**
    - **관리 편의성** 모두 얻을 수 있음

 기존에 정적 라이브러리로 관리하던 모듈도, mergeable 라이브러리로 바꾸는 ㄴ걸 검토해봐라~

## 정리

<img src="./WWDC23 - Meet mergeable libraries/스크린샷_2025-04-07_오전_1.13.25.png">
