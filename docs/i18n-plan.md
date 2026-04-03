# Flutter 다국어(i18n) 지원 구현 계획

> 작성일: 2026-04-03
> 대상 앱: Final Whistle Flutter 클라이언트 (`client/`)
> 참고: [Flutter 공식 국제화 문서](https://docs.flutter.dev/ui/internationalization)

---

## 1. 현황 분석

### 앱 규모

| 항목 | 수치 |
|------|------|
| 전체 Dart 파일 | 76개 |
| Feature 모듈 | 11개 (auth, home, club, competition, league, match, transfer, auction, profile 등) |
| 주요 화면 수 | 약 25개 |
| Shell / Navigation | 4개 (main, club, competition) |
| 하드코딩 한국어 문자열 (추정) | 150–200개 |

### 현재 i18n 설정 현황

현재 i18n 관련 설정이 **전혀 없으며**, 모든 UI 텍스트가 Dart 소스 내에 하드코딩되어 있다.

- `l10n.yaml` — 없음
- `.arb` 파일 — 없음
- `localizationsDelegates` — 없음
- `flutter_localizations`, `intl`, `easy_localization` 등 패키지 — 없음

### 하드코딩 한국어 문자열 분포

#### 카테고리별

| 카테고리 | 예시 | 예상 개수 |
|----------|------|----------|
| Navigation labels | '홈', '대회', '리그', '멤버', '설정' | ~15개 |
| Button / Dialog labels | '가입', '만들기', '닫기', '취소', '다시 시도' | ~30개 |
| Status badges | '진행중', '모집중', '대기중', '완료', '초안', '종료' | ~20개 |
| Error / Validation | '오류', '로드 실패: $e', '이름을 입력해주세요' | ~25개 |
| Screen content | '빠른 동작', '참가 중인 대회', '클럽 소식' 등 | ~60개 |
| Empty state | '등록된 대회가 없습니다', '아직 가입한 클럽이 없습니다' | ~20개 |

#### 주요 파일 목록

```
core/shell/club_shell.dart             — 네비게이션 라벨, 가입 Dialog
core/shell/competition_shell.dart      — 네비게이션 라벨
core/widgets/status_chip.dart          — Enum → 한국어 상태 문자열 매핑

features/home/screens/home_screen.dart
features/club/screens/club_list_screen.dart
features/competition/screens/competition_list_screen.dart
features/league/screens/league_detail_screen.dart
features/match/screens/match_detail_screen.dart
features/transfer/screens/transfer_screen.dart
features/auction/screens/auction_screen.dart
features/auth/screens/auth_screen.dart
features/profile/screens/my_screen.dart
```

---

## 2. i18n 방식 비교

### 2-1. Flutter 공식 방식 — `flutter_localizations` + `gen-l10n` + `.arb`

Flutter 팀이 공식 권장하는 표준 방식이다. `.arb` 파일에 번역 문자열을 정의하면 `flutter gen-l10n` 도구가 빌드 시 타입-세이프한 `AppLocalizations` 클래스를 자동 생성한다.

**동작 원리**

```
.arb 파일 (번역 원본)
    ↓
flutter gen-l10n (코드 생성 도구, flutter run 시 자동 실행)
    ↓
AppLocalizations.dart (자동 생성 클래스, .dart_tool/ 하위)
    ↓
AppLocalizations.of(context)!.someKey (위젯에서 접근)
```

`Localizations` 위젯이 InheritedWidget처럼 동작하여 디바이스 로케일 변경 시 위젯 트리를 자동으로 재빌드한다.

**설정 예시**

```yaml
# pubspec.yaml
dependencies:
  flutter_localizations:
    sdk: flutter
  intl: any

flutter:
  generate: true  # ← 코드 자동 생성 활성화 (핵심)
```

```yaml
# l10n.yaml (프로젝트 루트)
arb-dir: lib/l10n
template-arb-file: app_ko.arb        # 기본 언어 (한국어)
output-localization-file: app_localizations.dart
nullable-getter: false               # null-check 연산자(!") 생략 가능
```

**.arb 파일 형식**

`.arb`(Application Resource Bundle)는 Google이 정의한 업계 표준 번역 파일 형식이다. JSON 기반이며 외부 번역 서비스(Crowdin, Lokalise 등)와 호환된다.

```json
// lib/l10n/app_ko.arb
{
  "@@locale": "ko",

  "navHome": "홈",
  "@navHome": { "description": "홈 탭 라벨" },

  "memberCount": "{count}명",
  "@memberCount": {
    "description": "멤버 수 표시",
    "placeholders": {
      "count": { "type": "int", "example": "42" }
    }
  },

  "statusInProgress": "진행중",
  "@statusInProgress": { "description": "진행 중 상태" },

  "nWombats": "{count, plural, =0{멤버 없음} =1{1명} other{{count}명}}",
  "@nWombats": {
    "description": "복수형 예시",
    "placeholders": { "count": { "type": "num" } }
  }
}
```

**코드에서의 사용**

```dart
// MaterialApp 설정
MaterialApp.router(
  localizationsDelegates: AppLocalizations.localizationsDelegates,
  supportedLocales: AppLocalizations.supportedLocales,
  locale: ref.watch(localeNotifierProvider), // Riverpod으로 동적 전환
  routerConfig: router,
)

// 위젯에서 접근
final l10n = AppLocalizations.of(context);
Text(l10n.navHome)              // → "홈" / "Home"
Text(l10n.memberCount(42))      // → "42명" / "42 members"
```

**장점**
- Flutter SDK 내장 — 외부 패키지 의존성 없음
- 타입 안전 — IDE 자동완성 및 컴파일 타임 검증
- `.arb` 표준 — 외부 번역 도구와 바로 호환
- 복수형(plural), 성별(select), 날짜/숫자 포맷 기본 지원
- 디바이스 로케일 자동 감지 및 재빌드

**단점**
- `AppLocalizations.of(context)` 호출이 장황함
- 코드 생성 단계 추가 (`flutter gen-l10n`)
- 초기 설정에 보일러플레이트 존재

---

### 2-2. `easy_localization` 패키지

JSON/YAML 파일 기반의 서드파티 패키지.

```dart
Text('nav.home'.tr())
Text('member.count'.tr(args: ['42']))
```

**장점:** 설정 간단, 런타임 전환 내장
**단점:** 타입 안전성 없음 (오타 시 런타임 에러), 서드파티 의존성

---

### 2-3. `slang` 패키지

`.arb` 또는 JSON을 분석해 타입-세이프 클래스를 생성하는 최신 패키지.

```dart
Text(t.nav.home)
Text(t.member.count(n: 42))
```

**장점:** 간결한 문법, 타입 안전, 자동완성 우수
**단점:** 서드파티 의존성, 공식 지원 아님

---

### 2-4. 비교 요약

| 항목 | 공식 방식 (gen-l10n) | easy_localization | slang |
|------|---------------------|-------------------|-------|
| 타입 안전성 | ✅ 강함 | ❌ 없음 | ✅ 강함 |
| 설정 복잡도 | 중간 | 낮음 | 낮음 |
| 코드 간결성 | 중간 | 높음 | 매우 높음 |
| 공식 Flutter 지원 | ✅ SDK 포함 | ❌ 서드파티 | ❌ 서드파티 |
| `.arb` 호환 | ✅ | 부분 | ✅ |
| 복수형 / 날짜 포맷 | ✅ ICU 메시지 내장 | 부분 | ✅ |
| 런타임 언어 전환 | 별도 구현 필요 | ✅ 내장 | 별도 구현 필요 |
| 추천 대상 | 일반 프로젝트 | 빠른 프로토타입 | 대규모 프로젝트 |

---

## 3. 추천 방식

**Flutter 공식 방식(`flutter_localizations` + `gen-l10n` + `.arb`)을 채택한다.**

**선택 이유:**

1. **외부 의존성 없음** — SDK에 포함, 패키지 관리 부담 없음
2. **타입 안전성** — 컴파일 타임에 누락 번역 키 감지
3. **확장성** — `.arb` 표준으로 외부 번역 서비스 연동 용이
4. **앱 규모 적합** — 76개 파일, 150–200개 문자열의 중상 규모
5. **장기 유지보수** — Flutter 팀이 직접 관리하며 공식 문서 풍부
6. **동적 전환** — Riverpod과 조합하면 간단히 구현 가능

---

## 4. 파일 구조

```
client/
├── l10n.yaml                        # gen-l10n 설정 (프로젝트 루트)
└── lib/
    ├── l10n/
    │   ├── app_ko.arb               # 한국어 (템플릿, 기본 언어)
    │   └── app_en.arb               # 영어
    └── ...
```

> 자동 생성 파일(`.dart_tool/flutter_gen/gen_l10n/`)은 `.gitignore`에 추가한다.
> `.arb` 파일만 버전 관리한다.

---

## 5. 지원 언어

| 언어 | 로케일 코드 | 파일명 | 우선순위 |
|------|------------|--------|---------|
| 한국어 | `ko` | `app_ko.arb` | 1순위 (기본, 템플릿) |
| 영어 | `en` | `app_en.arb` | 2순위 |

초기에는 한국어와 영어만 지원한다. 추후 일본어(`ja`), 중국어(`zh`) 등 추가 가능.

---

## 6. 구현 단계

### Phase 1 — 기반 설정

**목표:** 패키지 추가, `.arb` 구조 생성, `MaterialApp` 연결

1. `pubspec.yaml`에 의존성 추가

   ```yaml
   dependencies:
     flutter_localizations:
       sdk: flutter
     intl: any

   flutter:
     generate: true
   ```

2. `l10n.yaml` 생성

   ```yaml
   arb-dir: lib/l10n
   template-arb-file: app_ko.arb
   output-localization-file: app_localizations.dart
   nullable-getter: false
   ```

3. `lib/l10n/app_ko.arb`, `lib/l10n/app_en.arb` 빈 파일 생성

4. `MaterialApp.router`에 `localizationsDelegates`, `supportedLocales` 추가

   ```dart
   MaterialApp.router(
     localizationsDelegates: AppLocalizations.localizationsDelegates,
     supportedLocales: AppLocalizations.supportedLocales,
     locale: ref.watch(localeNotifierProvider),
     routerConfig: router,
   )
   ```

5. `flutter gen-l10n` 실행 후 코드 생성 확인

6. 자동 생성 디렉터리를 `.gitignore`에 추가

**검증:** 앱이 정상 빌드되고 기본 로케일이 적용되는지 확인

---

### Phase 2 — 공통 문자열 이전

**목표:** 전 화면에서 반복 사용되는 공통 문자열을 `.arb`로 이전

우선순위 항목:

- Navigation labels: '홈', '대회', '리그', '멤버', '설정'
- 공통 버튼: '확인', '취소', '닫기', '다시 시도', '만들기', '수정', '삭제'
- 공통 에러: '오류', '로드 실패', '다시 시도해주세요'
- Status 문자열: `status_chip.dart`의 Enum 상태 매핑 전체

```json
// app_ko.arb (공통 문자열 예시)
{
  "@@locale": "ko",

  "navHome": "홈",
  "navCompetition": "대회",
  "navLeague": "리그",
  "navMember": "멤버",
  "navSettings": "설정",

  "btnConfirm": "확인",
  "btnCancel": "취소",
  "btnClose": "닫기",
  "btnRetry": "다시 시도",
  "btnCreate": "만들기",
  "btnEdit": "수정",
  "btnDelete": "삭제",

  "errorGeneral": "오류",
  "errorLoadFailed": "로드 실패",
  "errorNetworkFailed": "네트워크 연결에 실패했습니다",

  "statusDraft": "초안",
  "statusInProgress": "진행중",
  "statusCompleted": "완료",
  "statusCancelled": "취소",
  "statusRecruiting": "모집중",
  "statusPending": "대기중",
  "statusFinished": "종료"
}
```

---

### Phase 3 — Feature별 문자열 이전

**목표:** 각 Feature 화면의 문자열을 순서대로 `.arb`로 이전

작업 순서 (노출 빈도 기준):

1. `home/` — 홈 화면
2. `competition/` — 대회 목록/상세
3. `club/` — 클럽 목록/상세 (Dialog 포함)
4. `league/` — 리그 상세
5. `match/` — 매치 상세
6. `transfer/` — 이적 시장
7. `auction/` — 경매
8. `auth/` — 인증 (오류 메시지 중심)
9. `profile/` — 마이 페이지

**키 네이밍 컨벤션**

```
[feature][Screen][Element]

예시:
  clubListTitle         → "클럽"
  clubListSearchHint    → "클럽 이름으로 검색"
  clubListEmpty         → "아직 가입한 클럽이 없습니다"
  clubListCreateDialog  → "클럽 만들기"
  clubListJoinError     → "가입 중 오류가 발생했습니다"
```

**코드 변경 예시 (Before / After)**

```dart
// Before
NavigationDestination(
  icon: Icon(Icons.home),
  label: '홈',
)

// After
final l10n = AppLocalizations.of(context);
NavigationDestination(
  icon: Icon(Icons.home),
  label: l10n.navHome,
)
```

---

### Phase 4 — 영어 번역 추가 및 언어 전환 UI

**목표:** `app_en.arb` 완성, 설정 화면에 언어 전환 옵션 추가

1. `app_en.arb`에 모든 키의 영어 번역 추가

   ```json
   // app_en.arb
   {
     "@@locale": "en",
     "navHome": "Home",
     "navCompetition": "Competition",
     "statusDraft": "Draft",
     "statusInProgress": "In Progress",
     "clubListEmpty": "You haven't joined any clubs yet",
     "memberCount": "{count} members",
     "@memberCount": {
       "placeholders": { "count": { "type": "int" } }
     }
   }
   ```

2. `flutter gen-l10n` 재실행

3. Riverpod Provider로 로케일 상태 관리

   ```dart
   // locale_notifier.dart
   @riverpod
   class LocaleNotifier extends _$LocaleNotifier {
     @override
     Locale build() {
       // SharedPreferences에서 저장된 언어 로드
       return const Locale('ko');
     }

     Future<void> setLocale(Locale locale) async {
       await _prefs.setString('locale', locale.languageCode);
       state = locale;
     }
   }
   ```

4. 설정 화면에 언어 선택 UI 추가

   ```dart
   DropdownButton<Locale>(
     value: ref.watch(localeNotifierProvider),
     items: const [
       DropdownMenuItem(value: Locale('ko'), child: Text('한국어')),
       DropdownMenuItem(value: Locale('en'), child: Text('English')),
     ],
     onChanged: (locale) {
       if (locale != null) {
         ref.read(localeNotifierProvider.notifier).setLocale(locale);
       }
     },
   )
   ```

---

### Phase 5 — 검증 및 마무리

1. 한국어 / 영어 전환 후 전체 화면 시각적 확인
2. 누락 번역 키 확인 (빌드 경고)
3. 긴 영어 문자열로 인한 레이아웃 깨짐 확인 및 수정
4. 위젯 테스트에 로케일 설정 추가

---

## 7. 동적 언어 전환

디바이스 시스템 언어에 따른 자동 전환과 앱 내 수동 전환을 모두 지원한다.

```dart
// main.dart
MaterialApp.router(
  locale: ref.watch(localeNotifierProvider),
  localizationsDelegates: AppLocalizations.localizationsDelegates,
  supportedLocales: AppLocalizations.supportedLocales,
  routerConfig: router,
)
```

`locale`이 `null`이면 디바이스 시스템 언어를 자동 적용한다. Riverpod Provider가 값을 제공하면 앱 내 선택 언어가 우선된다. 선택한 언어는 `SharedPreferences`에 저장하여 앱 재시작 후에도 유지한다.

---

## 8. 하드코딩 문자열 이전 전략

### 단계별 접근

1. **Grep으로 전수 조사** — `Text('..한글..')` 패턴으로 전체 파일 스캔
2. **파일 단위 작업** — 한 파일씩 처리, 매 파일마다 빌드 검증
3. **Enum 상태 매핑 분리** — `CompetitionStatus.label`을 `BuildContext`를 받는 메서드로 변경
4. **SnackBar / Dialog 분리** — 비즈니스 로직의 메시지를 에러 코드로 반환, UI 레이어에서 번역

### Enum 처리 패턴

```dart
// Before
extension CompetitionStatusLabel on CompetitionStatus {
  String get label => switch (this) {
    CompetitionStatus.draft => '초안',
    CompetitionStatus.inProgress => '진행중',
  };
}

// After
extension CompetitionStatusLabel on CompetitionStatus {
  String label(BuildContext context) {
    final l10n = AppLocalizations.of(context);
    return switch (this) {
      CompetitionStatus.draft => l10n.statusDraft,
      CompetitionStatus.inProgress => l10n.statusInProgress,
    };
  }
}
```

### 서버 에러 메시지 처리

서버에서 내려오는 에러 메시지는 에러 코드 기반으로 처리한다.

```dart
// Before: Repository에서 한국어 throw
throw Exception('네트워크 연결에 실패했습니다.');

// After: 에러 코드 enum으로 throw, UI에서 번역
throw AppException(AppErrorCode.networkFailed);

// UI 레이어
Text(l10n.errorNetworkFailed)
```

---

## 9. 주의사항

### 레이아웃 대비

영어 문자열은 한국어보다 최대 30% 길어질 수 있다. 주요 대비 방법:

- 텍스트 오버플로우: `overflow: TextOverflow.ellipsis`
- 자동 축소: `FittedBox(fit: BoxFit.scaleDown)`
- Navigation 라벨: 영어 약어 고려 (예: 'Competition' 대신 'Compete')

### `BuildContext` 의존성

`AppLocalizations.of(context)`는 `BuildContext`가 필요하다. Riverpod `Notifier` 내부나 Repository 레이어에서 직접 사용 불가하다.

- 비즈니스 로직의 에러는 에러 코드/Enum으로 반환
- UI 레이어에서만 번역 처리

### 자동 생성 파일 관리

- `lib/core/l10n/` 하위 자동 생성 파일은 `.gitignore`에 추가
- CI 파이프라인에서 `flutter gen-l10n`을 빌드 전 단계로 추가

### ARB 키 관리

- 키 삭제 시 모든 `.arb` 파일에서 동시에 제거
- 사용하지 않는 키는 주기적으로 정리

### iOS 추가 설정

- Xcode `Info.plist`의 `CFBundleLocalizations`에 지원 언어 추가 필요
- App Store의 언어별 설명 표시에 영향

---

## 10. 예상 작업량

| Phase | 내용 | 예상 시간 |
|-------|------|----------|
| Phase 1 | 패키지 설정, ARB 구조, MaterialApp 연결 | 2–4시간 |
| Phase 2 | 공통 문자열 이전 (~65개) | 3–4시간 |
| Phase 3 | Feature별 문자열 이전 (9개 모듈) | 8–12시간 |
| Phase 4 | 영어 번역 + 언어 전환 UI | 4–6시간 |
| Phase 5 | 검증 및 마무리 | 2–3시간 |
| **합계** | | **19–29시간** |

---

## 참고

- [Flutter 공식 국제화 문서](https://docs.flutter.dev/ui/internationalization)
- [Application Resource Bundle (.arb) 명세](https://github.com/google/app-resource-bundle)
- [intl 패키지 (pub.dev)](https://pub.dev/packages/intl)
- [Crowdin — .arb 호환 번역 서비스](https://crowdin.com)
