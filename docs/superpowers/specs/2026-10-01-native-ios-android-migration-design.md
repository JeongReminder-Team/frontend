# Jeong Reminder 네이티브 iOS·Android 전환 설계

## 목적

기존 Flutter/Dart 프런트엔드의 기능과 사용자 경험을 유지하면서 다음 두 네이티브 앱으로 단계적으로 전환한다.

- iOS: Swift와 SwiftUI
- Android: Kotlin과 Jetpack Compose

기존 Flutter 앱은 네이티브 앱의 기능 동등성이 확인될 때까지 운영 및 비교 기준으로 보존한다. 이 전환에서는 기존 백엔드 API와 이미지 자산을 재사용하며, 백엔드 계약 변경은 범위에 포함하지 않는다.

## 프로젝트 구조

현재 저장소 루트 아래의 `frontend/` Flutter 프로젝트는 `flutter-legacy/`로 이동하고 다음 구조를 사용한다.

```text
저장소 루트/
├─ flutter-legacy/       기존 Flutter 앱
├─ ios-swift/            Swift + SwiftUI 앱
├─ android-kotlin/       Kotlin + Jetpack Compose 앱
└─ docs/                 공통 API 명세와 이관 체크리스트
```

기존 Flutter 프로젝트의 이동은 네이티브 프로젝트 기반을 만들 때 함께 수행한다. Git 이력을 보존할 수 있도록 파일 내용을 다시 생성하지 않고 경로 이동으로 처리한다.

## 플랫폼 기술

### iOS

- 언어 및 UI: Swift, SwiftUI
- 언어 모드 및 최소 버전: Swift 6 strict concurrency, iOS 17 이상
- 아키텍처: 기능별 MVVM과 Repository
- 상태 및 비동기 처리: Observation의 `@Observable`, Swift Concurrency의 `async/await`
- 네트워크 및 직렬화: URLSession의 async API, Codable
- 인증정보 저장: Keychain
- 알림: Firebase Messaging, Apple Push Notification service
- 테스트: Swift Testing 단위·통합 테스트, XCTest/XCUIAutomation UI 테스트

### Android

- 언어 및 UI: Kotlin, Jetpack Compose
- SDK: `minSdk 23`, `compileSdk 36`, `targetSdk 36`
- 아키텍처: 기능별 MVVM, Repository와 단방향 데이터 흐름(UDF)
- 상태 및 비동기 처리: Kotlin Coroutines, `StateFlow`, `collectAsStateWithLifecycle`
- 화면 이동: Navigation 3
- 네트워크 및 직렬화: Retrofit, OkHttp, Kotlin Serialization
- 인증정보 저장: DataStore와 Android Keystore 기반 암호화 저장소
- 알림: Firebase Messaging
- 테스트: JUnit, Compose UI Test

플랫폼 간 화면 코드는 공유하지 않는다. API 계약, 기능 정의, 화면 상태, 오류 문구와 수용 기준을 문서로 공유한다.

## 기능 범위

기존 Flutter 앱의 다음 기능을 두 플랫폼에 동일하게 이관한다.

- 로그인, 로그아웃, 토큰 갱신, 비밀번호 변경
- 사용자 권한에 따른 홈 화면과 이동 흐름
- 전체·학년·대회 게시판
- 게시글 조회·작성·수정·숨김
- 이미지 및 파일 업로드·다운로드
- 추천과 투표
- 사용자 프로필, 기술 스택, 프로젝트 경험
- 팀 모집, 지원, 승인과 팀원 관리
- 알림 목록과 푸시 알림 이동
- 관리자용 사용자·팀·모집 관리 화면
- 개인정보 처리방침과 외부 링크

웹, Windows와 macOS용 신규 네이티브 클라이언트는 이번 범위에 포함하지 않는다.

## 아키텍처와 데이터 흐름

두 앱 모두 다음 계층 흐름을 따른다.

```text
SwiftUI / Compose 화면
→ ViewModel
→ Repository
→ API Client
→ reminder.sungkyul.ac.kr 백엔드
```

- 화면은 네트워크나 저장소를 직접 호출하지 않는다.
- ViewModel은 로딩, 성공, 빈 결과와 오류 상태를 화면에 제공한다.
- Repository는 원격 데이터와 로컬 인증 상태의 사용 방식을 결정한다.
- API Client는 기본 URL, 공통 헤더, 직렬화, 인증 재시도를 담당한다.
- API DTO와 화면 표시 모델을 분리해 백엔드 응답 변경이 UI 전체로 전파되지 않게 한다.
- API 주소와 Firebase 환경값은 플랫폼별 환경 설정 한 곳에서 관리한다.

## 인증과 보안

- 기존 백엔드의 로그인 및 `/api/v1/reissue` 토큰 갱신 계약을 유지한다.
- iOS에서는 Keychain, Android에서는 Keystore 기반 저장소에 인증정보를 저장한다.
- Android에서는 폐기된 `EncryptedSharedPreferences`를 사용하지 않는다.
- API가 인증 만료를 반환하면 동시 갱신 요청을 하나로 합치고 토큰을 한 번 갱신한 뒤 원래 요청을 한 번만 재시도한다.
- 갱신 실패 시 저장된 인증정보를 정리하고 로그인 화면으로 이동한다.
- 민감한 토큰, Firebase 키 또는 사용자 데이터는 로그에 출력하지 않는다.

## 오류 및 사용자 상태 처리

- 네트워크 단절, 시간 초과, 서버 오류, 인증 실패, 권한 부족, 유효성 검사 실패를 구분한다.
- 사용자에게는 행동 가능한 한국어 메시지를 표시하고 내부 오류 세부정보는 개발 로그에만 남긴다.
- 게시글 작성, 팀 지원 등 중복 요청 위험이 있는 작업은 처리 중 재실행을 막는다.
- 목록 화면은 로딩, 데이터, 빈 결과, 오류와 재시도 상태를 명확히 제공한다.
- 업로드와 다운로드 실패 시 진행 상태를 종료하고 재시도 가능한 상태로 복귀한다.
- 푸시 알림의 데이터가 불완전하거나 대상이 삭제된 경우 홈 또는 알림 목록으로 안전하게 이동한다.

## 이관 단계

### 1. 프로젝트 기반

- SwiftUI 및 Compose 프로젝트 생성
- 공통 테마, 내비게이션과 환경 설정 구성
- API Client, 공통 응답과 오류 처리 구현
- 보안 토큰 저장소와 자동 갱신 구현
- Firebase 설정

### 2. 인증과 기본 화면

- 로그인, 로그아웃과 비밀번호 변경
- 홈 화면과 사용자 권한 분기
- 앱 시작 시 로그인 상태 복원

### 3. 핵심 콘텐츠

- 전체·학년·대회 게시판
- 게시글 조회·작성·수정·숨김
- 이미지 및 파일 처리
- 추천과 투표

### 4. 사용자와 팀 기능

- 프로필, 기술 스택과 프로젝트 경험
- 팀 모집, 지원과 승인
- 알림 목록과 푸시 알림

### 5. 관리자와 전환 준비

- 관리자 기능
- 개인정보 처리방침과 외부 링크
- 접근성, 로딩·빈 결과·오류 화면 점검
- Flutter 앱과 기능 동등성 비교
- 배포 준비 및 단계적 전환

각 단계에서는 Swift와 Kotlin 구현을 함께 완료한다. 한 플랫폼의 다음 단계 작업은 다른 플랫폼의 현재 단계가 검증되기 전에 시작하지 않는 것을 기본 원칙으로 한다.

## 테스트와 완료 기준

각 기능은 다음 검증을 통과해야 한다.

- 모델 직렬화와 Repository 단위 테스트
- API 성공·실패·인증 만료 테스트
- ViewModel의 로딩·성공·빈 결과·오류 상태 테스트
- 주요 화면 렌더링과 사용자 상호작용 테스트
- 로그인, 게시글 조회·작성, 팀 지원, 알림 이동의 핵심 흐름 테스트
- 플랫폼별 빌드 및 정적 분석 통과
- 실제 백엔드의 비파괴적인 조회 흐름을 사용한 통합 확인

기능 동등성 체크리스트의 모든 항목이 양쪽 플랫폼에서 통과하고 배포 전 검증이 끝날 때까지 `flutter-legacy/`를 삭제하지 않는다. 기존 Flutter 앱의 삭제는 별도 승인과 별도 작업으로 수행한다.

## 자산과 구성 이전

- 기존 `assets/images`, `assets/skilImages`, 앱 아이콘과 스플래시 자산을 플랫폼 규격에 맞게 복사·변환한다.
- Firebase 구성 파일은 각 플랫폼 프로젝트에 연결하되 저장소에 이미 추적 중인 기존 설정의 노출 범위를 확대하지 않는다.
- 앱 버전과 번들·패키지 식별자는 현재 배포 앱과의 업데이트 호환성을 확인한 뒤 확정한다.
- 운영 및 개발 API 환경을 분리할 수 있도록 빌드 구성을 마련한다.

## 전환 및 롤백

- 기능 단위로 기존 Flutter 동작과 네이티브 동작을 비교한다.
- 네이티브 앱의 배포가 안정화되기 전까지 Flutter 앱은 빌드 가능한 상태로 유지한다.
- 출시 중 중대한 문제가 발견되면 직전 Flutter 배포 버전으로 복귀할 수 있도록 릴리스 정보를 보존한다.
- 두 네이티브 앱의 안정화와 별도 삭제 승인이 끝난 후에만 Flutter 코드를 제거한다.

## 범위 밖 사항

- 백엔드 API 재설계
- 새로운 제품 기능 추가
- 웹·데스크톱 앱의 네이티브 재구현
- iOS와 Android 화면 코드를 공유하기 위한 Kotlin Multiplatform 도입
- Flutter 코드의 조기 삭제
