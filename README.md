# StockHome 앱 구상안

> 마지막 업데이트: 2024년 (ViewModel / 알림 / 커스텀 필드 렌더링 설계 진행 중)

---

## 1. 프로젝트 개요

### 배경
자취 세대의 증가에 따라 본인이 무엇을 구매했는지, 어디에 보관하는지, 식재료라면 유통기한은 언제까지인지를 카테고리별로 관리할 수 있는 앱의 필요성에서 출발.

### 핵심 목표
- 카테고리(폴더) 단위로 물건 정보를 기록·관리
- 유통기한 기반 푸시 알림
- 사용자가 직접 커스텀 필드를 정의할 수 있는 동적 입력 시스템
- 완전한 로컬 동작 (인터넷 연결 없이도 모든 기능 사용 가능)
- 실제 앱스토어 배포 목표

### 기술 스택
| 항목 | 선택 | 비고 |
|---|---|---|
| UI | UIKit + SnapKit | |
| 반응형 바인딩 | RxSwift | |
| 로컬 저장 | CoreData | |
| 최소 지원 버전 | iOS 18.0+ | AI 기능은 iOS 26.0+ 버전 분기 처리 |
| 바코드 스캔 | AVFoundation + Open Food Facts API | 오픈소스 상품 DB 연동 |
| 영수증 OCR | Vision Framework (VNRecognizeTextRequest) | iPhone 기본 OCR 기능 활용 |
| AI 검색 / 챗봇 | Swift Foundation Model | iOS 26.0+ 기기에서만 활성화 |
| 추후 확장 | iCloud 동기화 (NSPersistentCloudKitContainer) | |

---

## 2. 핵심 기능 정의

### 카테고리 관리
- 사용자가 폴더처럼 카테고리를 직접 생성
- 이름 / 아이콘(SF Symbols) / 색상 커스텀
- 카테고리 간 스위칭으로 물건 목록 전환

### 위치 관리
- 아이템이 보관된 위치를 별도로 관리 (예: 냉장고, 서랍장, 베란다)
- 위치 등록 / 수정 / 삭제
- 아이템 등록 시 위치 선택

### 아이템 관리
- 아이템 등록 / 수정 / 삭제
- 재고 조회 (카테고리 / 위치별)
- 기본 필드: 이름, 수량, 위치, 유통기한, 메모, 사진
- 커스텀 필드: 사용자가 직접 필드를 정의 (동적 필드 시스템)

### 사진 등록
- 아이템별 사진 등록 / 삭제
- FileManager 로컬 저장, CoreData에 파일명(경로)만 보관

### 동적 커스텀 필드 시스템
카테고리마다 고정된 필드가 아니라, 사용자가 직접 필드명과 입력 타입을 정의하는 구조.

| 필드 타입 | 설명 | 예시 |
|---|---|---|
| TextField | 텍스트 자유 입력 | "선물해준 사람" → 여자친구 |
| Stepper | 숫자 증감 입력 | "리필 횟수" → 3 |
| DatePicker | 날짜 선택 | "구매일" → 2024.01.01 |
| Toggle | 온/오프 선택 | "개봉 여부" → true |

### 홈 요약
- 메인 화면 상단에 유통기한 임박 아이템 / 재고 부족 아이템 요약 배너 표시
- 유통기한 임박: UNUserNotificationCenter 로컬 푸시 알림 + 홈 상단 배너
- 재고 부족: 수량이 기준치 이하일 때 홈 상단 배너 + 푸시 알림

### AI 검색 및 챗봇
- Swift Foundation Model을 이용해 자연어로 재고 조회
- CoreData 데이터를 기반으로 프롬프트를 구성해 응답 생성
- iOS 26.0+ 기기에서만 활성화, 미만 버전은 기능 비노출 처리

| 기능 | 예시 |
|---|---|
| 재고 조회 | "지금 집에 과자 몇 개 있어?", "몽쉘은 몇 개야?" |
| 식재료 조회 | "집에 식재료는 뭐가 있어?" |
| 레시피 추천 | "집에 있는 재료로 저녁 뭐 먹을까?", "김치로 할 수 있는 요리는?" |

### 바코드 스캔 등록
- AVFoundation으로 카메라에 접근해 바코드 스캔
- Open Food Facts API로 상품 정보(이름, 카테고리 등) 자동 조회
- 조회된 정보를 CoreData에 자동 등록
- 홈 화면 위젯 / 빠른 실행 기능으로 앱 실행 전 바탕화면에서 바로 카메라 접근 가능

### 영수증 스캔 등록
- 카메라로 영수증 촬영
- Vision Framework의 `VNRecognizeTextRequest`로 텍스트 OCR 추출
- 추출된 텍스트를 Swift Foundation Model에 전달해 아이템 파싱
- 파싱된 아이템 목록을 CoreData에 일괄 자동 등록

---

## 3. 아키텍처

### 패턴
**MVVM + RxSwift**

Clean Architecture는 도입하지 않음. 현재 프로젝트 규모와 단독 개발 환경을 고려해 MVVM만으로 충분한 구조를 유지하는 것을 목표로 함.

### 화면 전환

#### 비교 검토

| 방식 | 장점 | 단점 |
|---|---|---|
| AppDelegate/SceneDelegate 기반 | 구현 단순, 학습 비용 없음 | VC 간 결합도 높음, 화면 많아질수록 전환 로직 분산 |
| Coordinator 패턴 | 전환 로직 집중 관리, VC 재사용성 향상, RxSwift output과 자연스럽게 연결 | 초기 설계 비용 존재 |

#### 선택: **Coordinator 패턴**

> 배포 목표이며 화면 수가 5개 이상으로 예상되고, RxSwift output과의 연동이 자연스럽기 때문에 장기적인 유지보수와 확장성을 고려해 Coordinator 패턴을 채택.

```swift
// ViewController는 이벤트만 전달
viewModel.output.showItemList
    .bind(with: self) { owner, category in
        owner.coordinator?.showItemList(category: category)
    }

// Coordinator가 실제 전환 담당
func showItemList(category: Category) {
    let vc = ItemListViewController()
    navigationController.pushViewController(vc, animated: true)
}
```

---

## 4. 폴더 / 파일 구조

### 구조 방식

#### 비교 검토

| 방식 | 장점 | 단점 |
|---|---|---|
| Feature 중심 (화면별 묶기) | 화면 단위 응집도 높음, 기능 추가/삭제 용이 | 팀 내 익숙하지 않은 구조, 공통 코드 위치 혼란 가능 |
| Layer 중심 (View / ViewModel / Model 등) | 역할 기준이 명확, 팀 컨벤션과 일치, 탐색 직관적 | 화면 수 많아지면 같은 폴더 내 파일 증가 |

#### 선택: **Layer 중심 (팀 컨벤션 기반)**

> 팀에서 기존에 `View`, `ViewModel`, `Model`, `Manager`, `Utils` 단위로 분리해온 방식을 따름. Feature 중심은 Clean Architecture와 유사한 느낌을 주어 현재 프로젝트의 단순한 MVVM 구조와 맞지 않는다고 판단. Coordinator는 화면 전환 전담 역할이므로 별도 폴더로 분리.

### 디렉토리 구조

```
StockHome/
├── App/
│   ├── AppDelegate.swift
│   └── SceneDelegate.swift
│
├── Coordinator/
│   ├── AppCoordinator.swift
│   ├── CategoryCoordinator.swift
│   ├── ItemCoordinator.swift
│   └── CoordinatorProtocol.swift
│
├── View/                          # ViewController + 커스텀 View / Cell
│   ├── Home/
│   │   ├── HomeViewController.swift
│   │   └── CategoryCell.swift
│   ├── Category/
│   │   ├── CategoryListViewController.swift
│   │   └── CategoryCreateViewController.swift
│   ├── Item/
│   │   ├── ItemListViewController.swift
│   │   ├── ItemDetailViewController.swift
│   │   ├── ItemCreateViewController.swift
│   │   └── ItemCell.swift
│   └── CustomField/
│       ├── CustomFieldViewController.swift
│       └── CustomFieldCell.swift
│
├── ViewModel/
│   ├── HomeViewModel.swift
│   ├── CategoryViewModel.swift
│   ├── ItemListViewModel.swift
│   ├── ItemDetailViewModel.swift
│   └── CustomFieldViewModel.swift
│
├── Model/
│   ├── StockHome.xcdatamodeld
│   ├── CategoryEntity+CoreDataClass.swift
│   ├── ItemEntity+CoreDataClass.swift
│   ├── CustomFieldEntity+CoreDataClass.swift
│   └── ItemImageEntity+CoreDataClass.swift
│
├── Manager/
│   ├── CoreDataManager.swift
│   ├── NotificationManager.swift
│   ├── ImageManager.swift
│   ├── UserDefaultsManager.swift
│   ├── PermissionManager.swift
│   ├── BarcodeManager.swift        # AVFoundation 바코드 스캔
│   ├── OCRManager.swift            # Vision Framework 영수증 OCR
│   └── AIManager.swift             # Swift Foundation Model 연동
│
├── Protocols/
│   ├── CoreDataManagerProtocol.swift
│   ├── NotificationManagerProtocol.swift
│   ├── ImageManagerProtocol.swift
│   ├── UserDefaultsManagerProtocol.swift
│   ├── BarcodeManagerProtocol.swift
│   ├── OCRManagerProtocol.swift
│   └── AIManagerProtocol.swift
│
├── Utils/
│   ├── Extensions/
│   │   ├── UIColor+Hex.swift
│   │   ├── Date+Format.swift
│   │   └── UIImage+Resize.swift
│   └── Base/
│       └── BaseViewController.swift
│
└── Resources/
    ├── Assets.xcassets
    └── Info.plist
```

### Manager + Protocol 설계 (DI 기반)

싱글톤의 테스트 문제를 해결하기 위해 Protocol로 추상화하고 ViewModel에 주입하는 방식을 채택.

#### 비교 검토

| 방식 | 장점 | 단점 |
|---|---|---|
| 싱글톤 직접 참조 | 구현 단순 | 테스트 시 Mock 교체 불가, 전역 상태 공유 문제 |
| Protocol + DI | Mock 주입 가능, 테스트 용이, 결합도 낮음 | Protocol 파일 추가 필요 |

#### 선택: **Protocol + DI (기본값은 싱글톤 유지)**

> 배포 목표 앱으로 추후 테스트 도입을 고려해 Protocol 추상화를 적용. 싱글톤 자체는 유지하되 ViewModel이 `.shared`를 직접 참조하지 않고 Protocol을 통해 주입받는 구조.

```swift
// Protocol 정의
protocol CoreDataManagerProtocol {
    func fetchItems() -> [Item]
    func saveItem(_ item: Item)
}

// 실제 구현체
class CoreDataManager: CoreDataManagerProtocol {
    static let shared = CoreDataManager()
    func fetchItems() -> [Item] { ... }
    func saveItem(_ item: Item) { ... }
}

// Mock (테스트용)
class MockCoreDataManager: CoreDataManagerProtocol {
    var stubbedItems: [Item] = []
    func fetchItems() -> [Item] { return stubbedItems }
    func saveItem(_ item: Item) { stubbedItems.append(item) }
}

// ViewModel — 기본값은 실제 구현체, 테스트 시 Mock 주입
class ItemListViewModel {
    private let manager: CoreDataManagerProtocol

    init(manager: CoreDataManagerProtocol = CoreDataManager.shared) {
        self.manager = manager
    }
}
```

---

## 5. CoreData 스키마

### Entity 관계도

```
Category  1 ──< Item  1 ──< CustomField
                     1 ──< ItemImage
Location  1 ──< Item
```

### 삭제 규칙 (Delete Rule)
- Category 삭제 → Item 자동 삭제 (Cascade)
- Location 삭제 → Item의 location 관계 nil 처리 (Nullify)
- Item 삭제 → CustomField 자동 삭제 (Cascade)
- Item 삭제 → ItemImage 자동 삭제 (Cascade)

### Entity 상세

#### Category
| 필드 | 타입 | 설명 |
|---|---|---|
| id | UUID | 고유 식별자 |
| name | String | 카테고리 이름 |
| iconName | String | SF Symbols 이름 |
| colorHex | String | 색상 (#FF5733 형식) |
| order | Int16 | 정렬 순서 |
| createdAt | Date | 생성일 |

#### Location
| 필드 | 타입 | 설명 |
|---|---|---|
| id | UUID | 고유 식별자 |
| name | String | 위치 이름 (예: 냉장고, 서랍장) |
| order | Int16 | 정렬 순서 |
| createdAt | Date | 생성일 |

#### Item
| 필드 | 타입 | 설명 |
|---|---|---|
| id | UUID | 고유 식별자 |
| name | String | 아이템 이름 |
| quantity | Int32 | 수량 |
| lowStockThreshold | Int32 | 재고 부족 기준 수량 (홈 요약 알림용) |
| expirationDate | Date? | 유통기한 (없을 수 있음) |
| notificationEnabled | Boolean | 유통기한 알림 활성화 여부 |
| notificationDaysBefore | Int16 | 며칠 전 알림 |
| barcodeValue | String? | 바코드 스캔 값 (없을 수 있음) |
| memo | String? | 메모 |
| createdAt | Date | 생성일 |
| updatedAt | Date | 수정일 |
| categoryId | UUID (FK) | 소속 카테고리 |
| locationId | UUID? (FK) | 보관 위치 (없을 수 있음) |

#### CustomField (EAV 패턴)
| 필드 | 타입 | 설명 |
|---|---|---|
| id | UUID | 고유 식별자 |
| key | String | 필드명 ("선물해준 사람") |
| fieldType | String | "text" / "stepper" / "date" / "toggle" |
| value | String | 값 (모두 String으로 통일 후 타입별 변환) |
| order | Int16 | 필드 표시 순서 |
| itemId | UUID (FK) | 소속 아이템 |

#### ItemImage
| 필드 | 타입 | 설명 |
|---|---|---|
| id | UUID | 고유 식별자 |
| fileName | String | 파일명 (경로 참조용) |
| order | Int16 | 이미지 순서 |
| itemId | UUID (FK) | 소속 아이템 |

### 이미지 저장 방식

#### 비교 검토

| 방식 | 장점 | 단점 |
|---|---|---|
| Binary 직접 저장 | 별도 파일 관리 불필요 | CoreData 용량 부담, 성능 저하 |
| External Storage 옵션 | 자동 용량 분리 | iCloud 동기화 버그 위험, 디버깅 어려움 |
| FileManager + 경로 저장 | 용량 제어 가능, iCloud 확장 안전, 웹 경험과 구조 동일 | 파일 삭제 로직 직접 관리 필요 |

#### 선택: **FileManager + 경로 저장**

> 완전한 로컬 동작을 보장하면서 iCloud 확장 시 안전한 구조. 웹 개발에서의 디렉토리 경로 기반 저장 경험과 구조가 동일. 이미지 실체는 `Documents/Images/`에 저장하고, CoreData에는 파일명만 보관. Item 삭제 시 `CoreDataManager`에서 `ImageManager`를 호출해 파일도 함께 삭제 처리.

```swift
// 저장
let fileName = "\(UUID().uuidString).jpg"
let url = FileManager.default
    .urls(for: .documentDirectory, in: .userDomainMask)[0]
    .appendingPathComponent("Images/\(fileName)")
try imageData.write(to: url)

// 삭제 (CoreDataManager에서 함께 처리)
func deleteItem(_ item: Item) {
    item.images?.forEach { image in
        ImageManager.shared.deleteImage(fileName: image.fileName)
    }
    context.delete(item)
    saveContext()
}
```

### 커스텀 필드 저장 방식

#### 비교 검토

| 방식 | 장점 | 단점 |
|---|---|---|
| JSON 직렬화 | 스키마 단순, 구현 빠름 | 커스텀 필드 기준 검색 불가, 파싱 오류 시 전체 유실 위험 |
| EAV 패턴 | 필드 단위 CRUD, 검색 가능, 데이터 무결성 보장, iCloud 충돌 처리 유리 | 스키마 복잡도 증가 |
| Transformable | Swift 객체 그대로 저장 가능 | NSCoding 필요, iCloud 궁합 나쁨, 마이그레이션 까다로움 |

#### 선택: **EAV 패턴**

> 커스텀 필드 기준 검색 가능성을 열어두기 위해 EAV 패턴 채택. Transformable은 iCloud 확장 목표와 맞지 않아 제외. JSON은 검색이 불가능해 제외.

---

## 6. MVP 범위 정의

### 개발 조건
- 인원: 2인
- 기간: 3주 (주말 포함 21일 풀타임)
- 숙련도: 주니어~미들

### v1.0 MVP (3주 내 배포 목표)

| 기능 | 상세 |
|---|---|
| 카테고리 관리 | 생성 / 수정 / 삭제, 이름 / 아이콘(SF Symbols) / 색상 설정 |
| 위치 관리 | 보관 위치 등록 / 수정 / 삭제, 아이템 등록 시 위치 선택 |
| 아이템 관리 | 등록 / 수정 / 삭제, 재고 조회 (카테고리 / 위치별) |
| 아이템 기본 필드 | 이름, 수량, 위치, 유통기한, 메모, 사진 |
| 커스텀 필드 시스템 | TextField / Stepper / DatePicker / Toggle 타입 직접 정의 |
| 홈 요약 | 유통기한 임박 / 재고 부족 배너 + 푸시 알림 |
| AI 검색 / 챗봇 | 자연어 재고 조회 + 레시피 추천 (iOS 26.0+ 분기) |
| 바코드 스캔 등록 | AVFoundation + Open Food Facts API 자동 등록, 위젯/빠른 실행 |
| 영수증 스캔 등록 | Vision OCR + Foundation Model 파싱 → 일괄 자동 등록 |

### v1.1 이후 추가 예정

| 기능 | 제외 이유 |
|---|---|
| 카테고리 순서 변경 (드래그) | MVP 핵심 가치와 무관한 편의 기능 |
| 아이템 정렬 / 필터 | AI 검색으로 대체 가능, 추후 사용자 피드백 반영 후 추가 |
| iCloud 동기화 | 로컬 완결 우선, 안정화 후 확장 |

### 예상 작업 일정

| 단계 | 예상 기간 |
|---|---|
| 프로젝트 세팅 / 폴더 구조 / CoreData 설정 | 1~2일 |
| 카테고리 / 위치 CRUD + UI | 2~3일 |
| 아이템 CRUD + UI | 3~4일 |
| 홈 요약 + 유통기한 알림 시스템 | 1~2일 |
| 커스텀 필드 시스템 | 3~4일 |
| 바코드 스캔 + Open Food Facts 연동 | 1~2일 |
| 영수증 OCR + Foundation Model 파싱 | 2~3일 |
| AI 검색 / 챗봇 (Foundation Model) | 2~3일 |
| 위젯 / 빠른 실행 | 1일 |
| QA / 버그 수정 / 배포 준비 | 3일 (최소 확보 권장) |
| **합계** | **약 19~27일** |

> 21일 내 완료를 목표로 하되, 일정이 밀릴 경우 위젯/빠른 실행 → 레시피 추천 → 영수증 스캔 순으로 v1.1로 이연하고 QA 시간을 우선 확보할 것.

---

## 7. 향후 논의 예정 항목

| 순위 | 항목 | 상태 |
|---|---|---|
| 3 | ViewModel Input/Output 설계 | 예정 |
| 4 | 알림 시스템 구조 | 예정 |
| 5 | 커스텀 필드 렌더링 전략 | 예정 |

---

## 8. 주요 의사결정 요약

| 항목 | 선택 | 핵심 이유 |
|---|---|---|
| 커스텀 필드 MVP 포함 여부 | v1.0에 포함 | 주말 포함 21일 풀타임으로 일정 내 완료 가능 판단 |
| 최소 지원 버전 | iOS 18.0+ | AI 기능만 iOS 26.0+ 버전 분기 처리 |
| 바코드 상품 정보 | Open Food Facts API | 무료 오픈소스 API로 상품 정보 자동 조회 |
| 영수증 텍스트 추출 | Vision Framework OCR | iPhone 기본 기능 활용, 외부 의존 없음 |
| AI 기능 | Swift Foundation Model | 온디바이스 처리, 외부 API 불필요 |
| 위치 관리 | 별도 Location Entity | Nullify 삭제 규칙으로 아이템 데이터 보존 |
| 아키텍처 | MVVM + RxSwift | Clean Architecture 없이 충분한 규모 |
| 화면 전환 | Coordinator 패턴 | 전환 로직 집중화, RxSwift output 연동 |
| 폴더 구조 | Layer 중심 (팀 컨벤션) | 팀 익숙도, 단순한 MVVM 구조에 적합 |
| Manager 방식 | Protocol + DI | 테스트 시 Mock 교체 가능 |
| 커스텀 필드 저장 | EAV 패턴 | 검색 가능성, 데이터 무결성 |
| 이미지 저장 | FileManager + 경로 | 용량 제어, iCloud 확장 안전 |
| 데이터 저장 | CoreData (로컬) | 완전한 오프라인 동작 보장 |
| iCloud 확장 | NSPersistentCloudKitContainer | CoreDataStack 교체만으로 확장 가능 |
