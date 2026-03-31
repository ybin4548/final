# StockHome 앱 구상안

> 마지막 업데이트: 2026년 3월 (ViewModel / 알림 / 커스텀 필드 렌더링 설계 진행 중)

---

## 1. 프로젝트 개요

### 배경
자취 세대의 증가에 따라 본인이 무엇을 구매했는지, 어디에 보관하는지, 식재료라면 유통기한은 언제까지인지를 카테고리별로 관리할 수 있는 앱의 필요성에서 출발.

### 핵심 목표
- 카테고리 / 위치 단위로 재고 정보를 기록·관리
- 바코드 / 영수증 OCR / AI 카메라 / 텍스트 등 다양한 등록 방식 제공
- 자연어 기반 AI 챗봇으로 재고 검색 및 레시피 추천
- 유통기한 임박 / 재고 부족 알림
- CoreData 로컬 저장 + Supabase 서버 하이브리드 구조
- 실제 앱스토어 배포 목표

### 기술 스택

| 항목 | 선택 | 비고 |
|---|---|---|
| UI | UIKit + SnapKit | |
| 반응형 바인딩 | RxSwift | |
| 로컬 저장 | CoreData | 재고 데이터 오프라인 접근 |
| 서버 / 알림 | Supabase | Gemini 호출, 패턴 분석, 푸시 알림 |
| AI 카메라 | Gemini 2.5 Flash | 상품 사진 인식 → 등록 초안 생성 |
| 영수증 OCR | Vision Framework (VNRecognizeTextRequest) | iPhone 기본 OCR |
| 바코드 스캔 | AVFoundation | 카메라 바코드 인식 |
| 바코드 상품 정보 | 식품안전나라 바코드연계제품정보 API (C005) | 식품 상품 정보 조회 |
| 레시피 API | 식품안전나라 조리식품 레시피 DB (COOKRCP01) | 보유 재료 기반 레시피 조회 |
| 백엔드 구현 | ClaudeCode MCP | Supabase 백엔드 로직 자동화 |
| 최소 지원 버전 | iOS 18.0+ | AI 기능은 Foundation Model 분기 처리 |
| 추후 확장 | iCloud 동기화 (NSPersistentCloudKitContainer) | |

### 데이터 저장 전략 (하이브리드)

| 데이터 | 저장 위치 | 이유 |
|---|---|---|
| 재고 / 카테고리 / 위치 | CoreData (로컬) | 오프라인에서도 조회 가능 |
| 소비 이력 / 패턴 분석 | Supabase | 서버에서 통계 처리 |
| 챗봇 대화 히스토리 | CoreData (로컬) | 빠른 접근, 오프라인 지원 |
| Gemini API 호출 | Supabase 서버 경유 | API 키 보안, 서버 로직 처리 |
| 푸시 알림 발송 | Supabase | 소비 패턴 기반 서버 알림 |

---

## 2. 화면별 기능 정의

### 메인 화면
1. 전체 재고 요약 정보 확인
   - 식재료 개수 확인
   - 생활용품 개수 확인
   - 전체 등록 아이템 수 확인
2. 유통기한 임박 식재료 확인
   - 유통기한이 가까운 식재료 목록 조회
   - D-day 형태로 표시
   - 해당 아이템 상세 화면으로 이동
3. 재고 부족 물품 확인
   - 수량이 부족한 생활용품 목록 조회
   - 부족 상태 우선 노출
   - 해당 아이템 상세 화면으로 이동
4. 최근 등록 아이템 확인
   - 최근 등록된 식재료 / 생활용품 목록 조회
   - 썸네일, 이름, 카테고리, 등록일 표시
   - 상세 화면으로 이동
5. 전체 현황 확인
   - 현재 재고를 요약 카드 형태로 제공
   - 식재료 / 생활용품별 현황 구분 표시

### 등록 화면
1. 바코드 스캔 등록
   - 카메라를 통해 바코드 인식
   - 식품안전나라 C005 API로 인식된 상품 정보 조회
   - 등록 초안으로 생성 후 사용자가 정보 수정 후 저장
2. 영수증 기반 아이템 등록
   - 카메라로 영수증 촬영
   - Vision Framework OCR로 품목명 / 가격 / 날짜 추출
   - 추출된 항목을 등록 후보 리스트로 제공
   - 사용자가 필요한 항목만 선택 후 저장
3. AI 기반 아이템 등록
   - 카메라로 상품 촬영
   - Gemini 2.5 Flash로 사진 전송 → 상품 정보 인식
   - 처리된 데이터를 리스트로 제공
   - 사용자가 필요한 항목만 선택 후 저장
4. 텍스트 기반 아이템 등록
   - 이름 직접 입력
   - 카테고리 선택
   - 수량 입력
   - 구매 시간 (자동 현재 Date 기입)
   - 컴포넌트 모듈 방식으로 선택적 추가 입력
     - 위치 입력
     - 메모 입력
     - 사진 입력
     - 시간 입력 (예: 유통기한)
   - 저장
5. 사용자 정의 카테고리 선택
   - 기본 카테고리 선택
   - 사용자가 만든 카테고리 선택
   - 등록 중 새 카테고리 추가 가능
6. 위치 선택 기능
   - 기존 위치 선택
   - 신규 위치 추가 후 선택 가능
7. 등록 완료 후 후속 처리
   - 저장 후 재고 현황 반영
   - 저장 완료 안내 제공

### 재고 현황 화면
1. 상위 분류 전환
   - 기본 탭 조회
   - 생활용품 탭 조회
2. 하위 카테고리별 아이템 조회
   - 기본 카테고리별 조회
   - 사용자 정의 카테고리별 조회
   - 전체 카테고리 조회
3. 아이템 리스트 표시
   - 썸네일, 이름, 카테고리, 수량, 위치, 상태 표시
4. 식재료 전용 상태 표시
   - 유통기한 상태 / 만료 상태 표시
   - 보관 방식 표시
   - 소비 여부 표시
5. 생활용품 전용 상태 표시
   - 재고 부족 여부, 물건 위치, 대표 사진 표시
6. 아이템 상세 조회
   - 리스트 선택 시 상세 화면 이동
   - 상세 정보 확인, 컴포넌트 모듈화 표기
7. 아이템 검색 및 정렬
   - 카테고리 기준 / 최근 등록순 / 유통기한순 / 부족 재고순 정렬
8. 재고 수정 / 삭제
   - 상세 화면 진입 후 수정 / 삭제 기능 제공

### 챗봇 검색 화면
1. 채팅 기반 재고 검색
   - 자연어 문장 입력
   - 예: "욕실에 있는 세제 보여줘"
   - 예: "유통기한 얼마 안 남은 식재료 알려줘"
2. 재고 조건 해석
   - 식재료 / 생활용품 구분
   - 카테고리 / 위치 / 상태 조건 해석
3. 검색 결과 출력
   - 조건에 맞는 아이템 리스트 표시
   - 결과 없을 경우 안내 메시지 제공
   - 관련 재고 화면으로 이동 가능
4. 레시피 검색
   - 보유 식재료 기반 레시피 추천
   - 특정 재료를 포함한 레시피 검색
   - 부족한 재료가 있는 경우 함께 안내
   - 요리 질문 → Foundation Model → 1차 메뉴 제공 → 조리법 질문 → COOKRCP01 API 결과 출력
5. 대화형 후속 질문 지원 (v1.1)
   - 검색 결과 기반 추가 질문 가능
   - 예: "그중 유통기한 임박한 것만 보여줘"
6. 추천 문장 제공
   - 검색 의도에 맞는 결과 요약
   - 예: "욕실 카테고리에서 세제 2개를 찾았어요"
7. 재고 상세 연동
   - 챗봇 결과에서 특정 아이템 선택 시 상세 화면 이동

### 카메라 기능
1. 영수증 OCR → CoreData 저장
   - Vision OCR → Foundation Model 처리 → 저장
2. 바코드 스캔 → CoreData 저장
   - 데이터 미존재 시 수동 입력 유도
3. AI를 통한 상품 유추 → CoreData 저장
   - Gemini 2.5 Flash 추론 모델 사용
   - 상품 사진 → 추론 → 결과 도출 → 상품 등록 화면 바인드

### 위젯 기능 (+ 퀵액션) — v1.1
1. 아이폰 메인화면 위젯을 통한 등록화면 진입
2. 동작 순서
   - 위젯 / 퀵액션 / 빠른 실행 → 앱 → 카메라 연결
   - 촬영 후 작업 선택 조건 처리

### Notification (Supabase)
1. 재고 부족 알림
   - 단순 재고 현황 알림 🔔
     - 예: "몽쉘 재고가 2개 남았어요!"
   - 소비 패턴 기반 구매 타이밍 알림 🔔 (v1.1)
     - 예: "몽쉘을 보통 3일마다 구매하세요. 현재 재고로는 2일치 남았어요!"

---

## 3. 아키텍처

### 패턴
**MVVM + RxSwift**

Clean Architecture는 도입하지 않음. 현재 프로젝트 규모와 2인 개발 환경을 고려해 MVVM만으로 충분한 구조를 유지하는 것을 목표로 함.

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

> 팀에서 기존에 View, ViewModel, Model, Manager, Utils 단위로 분리해온 방식을 따름. Coordinator는 화면 전환 전담 역할이므로 별도 폴더로 분리.

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
│   ├── ChatCoordinator.swift
│   ├── CameraCoordinator.swift
│   └── CoordinatorProtocol.swift
│
├── View/                          # ViewController + 커스텀 View / Cell
│   ├── Main/
│   │   ├── MainViewController.swift
│   │   └── SummaryCardView.swift
│   ├── Register/
│   │   ├── RegisterViewController.swift
│   │   ├── BarcodeRegisterViewController.swift
│   │   ├── ReceiptRegisterViewController.swift
│   │   ├── AIRegisterViewController.swift
│   │   └── ComponentModuleView.swift
│   ├── Stock/
│   │   ├── StockListViewController.swift
│   │   ├── StockDetailViewController.swift
│   │   └── StockCell.swift
│   ├── Chat/
│   │   ├── ChatViewController.swift
│   │   └── ChatBubbleCell.swift
│   ├── Category/
│   │   ├── CategoryViewController.swift
│   │   └── CategoryCell.swift
│   └── Camera/
│       └── CameraViewController.swift
│
├── ViewModel/
│   ├── MainViewModel.swift
│   ├── RegisterViewModel.swift
│   ├── StockListViewModel.swift
│   ├── StockDetailViewModel.swift
│   ├── ChatViewModel.swift
│   └── CategoryViewModel.swift
│
├── Model/
│   ├── StockHome.xcdatamodeld
│   ├── CategoryEntity+CoreDataClass.swift
│   ├── LocationEntity+CoreDataClass.swift
│   ├── ItemEntity+CoreDataClass.swift
│   ├── CustomFieldEntity+CoreDataClass.swift
│   ├── ItemImageEntity+CoreDataClass.swift
│   └── ChatHistoryEntity+CoreDataClass.swift
│
├── Manager/
│   ├── CoreDataManager.swift
│   ├── SupabaseManager.swift          # Supabase 클라이언트
│   ├── GeminiManager.swift            # Gemini 2.5 Flash API
│   ├── NotificationManager.swift
│   ├── ImageManager.swift
│   ├── BarcodeManager.swift           # AVFoundation 바코드 스캔
│   ├── OCRManager.swift               # Vision Framework OCR
│   ├── RecipeAPIManager.swift         # 식품안전나라 COOKRCP01
│   ├── UserDefaultsManager.swift
│   └── PermissionManager.swift
│
├── Protocols/
│   ├── CoreDataManagerProtocol.swift
│   ├── SupabaseManagerProtocol.swift
│   ├── GeminiManagerProtocol.swift
│   ├── NotificationManagerProtocol.swift
│   ├── ImageManagerProtocol.swift
│   ├── BarcodeManagerProtocol.swift
│   ├── OCRManagerProtocol.swift
│   └── RecipeAPIManagerProtocol.swift
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

#### 비교 검토

| 방식 | 장점 | 단점 |
|---|---|---|
| 싱글톤 직접 참조 | 구현 단순 | 테스트 시 Mock 교체 불가, 전역 상태 공유 문제 |
| Protocol + DI | Mock 주입 가능, 테스트 용이, 결합도 낮음 | Protocol 파일 추가 필요 |

#### 선택: **Protocol + DI (기본값은 싱글톤 유지)**

> 싱글톤 자체는 유지하되 ViewModel이 `.shared`를 직접 참조하지 않고 Protocol을 통해 주입받는 구조. 추후 테스트 도입 시 Mock 교체 가능.

```swift
protocol CoreDataManagerProtocol {
    func fetchItems() -> [Item]
    func saveItem(_ item: Item)
}

class CoreDataManager: CoreDataManagerProtocol {
    static let shared = CoreDataManager()
    func fetchItems() -> [Item] { ... }
    func saveItem(_ item: Item) { ... }
}

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
ChatHistory (독립)
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
| barcodeValue | String? | 바코드 스캔 값 |
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
| fileName | String | 파일명 (FileManager 경로 참조용) |
| order | Int16 | 이미지 순서 |
| itemId | UUID (FK) | 소속 아이템 |

#### ChatHistory
| 필드 | 타입 | 설명 |
|---|---|---|
| id | UUID | 고유 식별자 |
| role | String | "user" / "assistant" |
| content | String | 대화 내용 |
| createdAt | Date | 생성일 |

### 이미지 저장 방식

#### 비교 검토

| 방식 | 장점 | 단점 |
|---|---|---|
| Binary 직접 저장 | 별도 파일 관리 불필요 | CoreData 용량 부담, 성능 저하 |
| External Storage 옵션 | 자동 용량 분리 | iCloud 동기화 버그 위험, 디버깅 어려움 |
| FileManager + 경로 저장 | 용량 제어 가능, iCloud 확장 안전 | 파일 삭제 로직 직접 관리 필요 |

#### 선택: **FileManager + 경로 저장**

> 이미지 실체는 `Documents/Images/`에 저장하고 CoreData에는 파일명만 보관. Item 삭제 시 `CoreDataManager`에서 `ImageManager`를 호출해 파일도 함께 삭제 처리.

### 커스텀 필드 저장 방식

#### 비교 검토

| 방식 | 장점 | 단점 |
|---|---|---|
| JSON 직렬화 | 스키마 단순, 구현 빠름 | 커스텀 필드 기준 검색 불가, 파싱 오류 시 전체 유실 위험 |
| EAV 패턴 | 필드 단위 CRUD, 검색 가능, 데이터 무결성 보장 | 스키마 복잡도 증가 |
| Transformable | Swift 객체 그대로 저장 가능 | NSCoding 필요, iCloud 궁합 나쁨 |

#### 선택: **EAV 패턴**

> 커스텀 필드 기준 검색 가능성을 열어두기 위해 채택. Transformable은 iCloud 확장 목표와 맞지 않아 제외. JSON은 검색 불가로 제외.

---

## 6. MVP 범위 정의

### 개발 조건
- 인원: 2인
- 기간: 3주 (주말 포함 21일 풀타임)
- 숙련도: 주니어~미들

### v1.0 MVP

| 기능 | 상세 |
|---|---|
| 카테고리 관리 | 생성 / 수정 / 삭제, 이름 / 아이콘 / 색상 설정 |
| 위치 관리 | 보관 위치 등록 / 수정 / 삭제 |
| 텍스트 기반 아이템 등록 | 컴포넌트 모듈 방식, 기본 필드 + 커스텀 필드 |
| 바코드 스캔 등록 | AVFoundation + 식품안전나라 C005 API |
| 영수증 OCR 등록 | Vision Framework OCR + Foundation Model 파싱 |
| AI 카메라 등록 | Gemini 2.5 Flash 상품 인식 |
| 재고 현황 조회 | 카테고리 / 위치별 조회, 정렬, 상세 보기 |
| 커스텀 필드 시스템 | TextField / Stepper / DatePicker / Toggle |
| 홈 요약 | 유통기한 임박 / 재고 부족 배너 |
| 유통기한 / 재고 부족 알림 | 단순 재고 현황 푸시 알림 (Supabase) |
| AI 챗봇 재고 검색 | 자연어 재고 조회 + 레시피 추천 (COOKRCP01 API) |

### v1.1 이후 추가 예정

| 기능 | 제외 이유 |
|---|---|
| 소비 패턴 기반 알림 | 소비 이력 데이터 누적 필요, 신규 사용자 온보딩 처리 필요 |
| 대화형 후속 질문 | 챗봇 히스토리 관리 복잡도 높음 |
| 위젯 / 퀵액션 | 편의 기능, MVP 핵심 가치와 무관 |
| 카테고리 순서 변경 (드래그) | 편의 기능, 우선순위 낮음 |
| 아이템 정렬 / 필터 고도화 | AI 챗봇으로 대체 가능 |
| iCloud 동기화 | 로컬 안정화 후 확장 |

### 예상 작업 일정

| 단계 | 예상 기간 |
|---|---|
| 프로젝트 세팅 / CoreData / Supabase 초기 설정 | 1~2일 |
| 카테고리 / 위치 CRUD + UI | 2~3일 |
| 텍스트 기반 아이템 등록 + 커스텀 필드 | 3~4일 |
| 재고 현황 조회 + 상세 UI | 2~3일 |
| 바코드 스캔 + 식품안전나라 API 연동 | 1~2일 |
| 영수증 OCR + Foundation Model 파싱 | 2~3일 |
| AI 카메라 등록 (Gemini 연동) | 2~3일 |
| 홈 요약 + Supabase 알림 | 1~2일 |
| AI 챗봇 + 레시피 API 연동 | 2~3일 |
| QA / 버그 수정 / 배포 준비 | 3일 (최소 확보 권장) |
| **합계** | **약 19~28일** |

> 21일 내 완료를 목표로 하되, 일정이 밀릴 경우 AI 챗봇 레시피 → 영수증 OCR → AI 카메라 순으로 v1.1 이연하고 QA 시간 우선 확보.

---

## 7. 외부 API 목록

| API | 용도 | 제공처 |
|---|---|---|
| 바코드연계제품정보 (C005) | 바코드 스캔 후 식품 상품 정보 조회 | 식품안전나라 |
| 조리식품 레시피 DB (COOKRCP01) | 보유 재료 기반 레시피 조회 | 식품안전나라 |
| Gemini 2.5 Flash | AI 카메라 상품 인식, 영수증 파싱 보조 | Google |
| Supabase | 서버 로직, 푸시 알림, 소비 패턴 분석 | Supabase |

---

## 8. 향후 논의 예정 항목

| 순위 | 항목 | 상태 |
|---|---|---|
| 3 | ViewModel Input/Output 설계 | 예정 |
| 4 | 알림 시스템 구조 | 예정 |
| 5 | 커스텀 필드 렌더링 전략 | 예정 |
| - | UI / 디자인 시스템 | 디자이너와 협의 예정 |

---

## 9. 주요 의사결정 요약

| 항목 | 선택 | 핵심 이유 |
|---|---|---|
| 아키텍처 | MVVM + RxSwift | Clean Architecture 없이 충분한 규모 |
| 화면 전환 | Coordinator 패턴 | 전환 로직 집중화, RxSwift output 연동 |
| 폴더 구조 | Layer 중심 (팀 컨벤션) | 팀 익숙도, 단순한 MVVM 구조에 적합 |
| Manager 방식 | Protocol + DI | 테스트 시 Mock 교체 가능 |
| 데이터 저장 | CoreData + Supabase 하이브리드 | 오프라인 접근(CoreData) + 서버 로직(Supabase) |
| AI 기능 | Gemini 2.5 Flash (Supabase 경유) | API 키 보안, 물체 추론 인식 |
| 커스텀 필드 저장 | EAV 패턴 | 검색 가능성, 데이터 무결성 |
| 이미지 저장 | FileManager + 경로 | 용량 제어, iCloud 확장 안전 |
| 바코드 상품 정보 | 식품안전나라 C005 API | 무료 오픈소스, 한국 식품 DB |
| 레시피 API | 식품안전나라 COOKRCP01 | 무료 오픈소스, 한국 레시피 DB |
| 소비 패턴 알림 | v1.1 이연 | 초기 데이터 누적 필요, 온보딩 처리 필요 |
| 위젯 / 퀵액션 | v1.1 이연 | 편의 기능, MVP 핵심 가치와 무관 |
