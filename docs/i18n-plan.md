# Flutter 앱 다국어 지원(i18n) 계획서

## 1. 현재 상태 분석

### 문제 현황

약 30개 Dart 파일에 한국어 문자열이 하드코딩되어 있음.

| 위치 | 내용 |
|------|------|
| shell 파일 (네비게이션) | 하단 탭 라벨, 앱바 타이틀 |
| screen 파일 | 버튼 텍스트, 섹션 제목, 안내 메시지 |
| provider / repository | 에러 메시지, 예외 처리 문자열 |

### 영향 범위

- `client/lib/screens/` — 화면별 UI 텍스트 전반
- `client/lib/shells/` — 네비게이션 라벨
- `client/lib/providers/` — 에러·상태 메시지

---

## 2. 추천 방식: Flutter 공식 방식

**Flutter 공식 i18n (flutter_localizations + gen-l10n + .arb)** 채택 권장.

> 참고: https://docs.flutter.dev/ui/internationalization

### 선택 이유

- Flutter SDK 내장 — 추가 패키지 불필요
- `.arb` 파일 기반 타입 안전 코드 자동 생성 (`AppLocalizations.of(context).someKey`)
- IDE 자동완성 지원
- Google 공식 표준이므로 장기적으로 안정적

---

## 3. 방식 비교

| 항목 | flutter_localizations (공식) | easy_localization | slang |
|------|------------------------------|-------------------|-------|
| 추가 패키지 | 불필요 | 필요 | 필요 |
| 타입 안전성 | ✅ 코드 생성 | ❌ 런타임 키 조회 | ✅ 코드 생성 |
| ARB 표준 지원 | ✅ | ✅ | ❌ (자체 포맷) |
| 복수형/성별 지원 | ✅ ICU 메시지 포맷 | ✅ | ✅ |
| 동적 언어 전환 | 별도 구현 필요 | ✅ 내장 | 별도 구현 필요 |
| 학습 곡선 | 낮음 | 낮음 | 낮음 |
| 권장 대상 | 일반 프로젝트 | 빠른 프로토타입 | 대규모 프로젝트 |

**결론:** 공식 방식이 가장 안정적. 동적 전환은 `Locale` 상태를 `Riverpod`으로 관리하면 간단히 해결.

---

## 4. 파일 구조

```
client/
├── lib/
│   └── l10n/
│       ├── app_ko.arb      # 한국어 (기본)
│       └── app_en.arb      # 영어
└── l10n.yaml               # gen-l10n 설정
```

### l10n.yaml

```yaml
arb-dir: lib/l10n
template-arb-file: app_ko.arb
output-localization-file: app_localizations.dart
nullable-getter: false
```

### app_ko.arb (예시)

```json
{
  "@@locale": "ko",
  "navHome": "홈",
  "navLeague": "리그",
  "navClub": "클럽",
  "navProfile": "프로필",
  "buttonConfirm": "확인",
  "buttonCancel": "취소",
  "errorNetworkFailed": "네트워크 연결에 실패했습니다.",
  "matchStatusLive": "진행중",
  "matchStatusFinished": "종료",
  "playerCountLabel": "{count}명",
  "@playerCountLabel": {
    "placeholders": {
      "count": { "type": "int" }
    }
  }
}
```

### app_en.arb (예시)

```json
{
  "@@locale": "en",
  "navHome": "Home",
  "navLeague": "League",
  "navClub": "Club",
  "navProfile": "Profile",
  "buttonConfirm": "Confirm",
  "buttonCancel": "Cancel",
  "errorNetworkFailed": "Network connection failed.",
  "matchStatusLive": "Live",
  "matchStatusFinished": "Finished",
  "playerCountLabel": "{count} players"
}
```

---

## 5. 구현 단계

### Phase 1 — 인프라 설정

1. `pubspec.yaml`에 의존성 추가:
   ```yaml
   dependencies:
     flutter_localizations:
       sdk: flutter
     intl: ^0.19.0

   flutter:
     generate: true
   ```

2. `l10n.yaml` 생성 (위 내용 참조)

3. `MaterialApp` 설정:
   ```dart
   MaterialApp(
     localizationsDelegates: AppLocalizations.localizationsDelegates,
     supportedLocales: AppLocalizations.supportedLocales,
     locale: ref.watch(localeProvider), // Riverpod 상태
   )
   ```

4. `flutter gen-l10n` 실행 → `.dart_tool/flutter_gen/` 코드 자동 생성

### Phase 2 — 공통 문자열 추출

우선순위 높음. 전 화면에서 반복 사용되는 요소부터 처리.

- 네비게이션 라벨 (shell 파일)
- 공통 버튼 텍스트 (확인, 취소, 저장, 삭제 등)
- 공통 에러 메시지 (네트워크 오류, 권한 오류 등)
- 공통 상태 텍스트 (로딩중, 데이터 없음 등)

### Phase 3 — 화면별 문자열 이전

각 screen 파일의 하드코딩 문자열을 `.arb` 키로 교체.

예상 작업 파일:
- `match_screen.dart`, `league_screen.dart`, `club_screen.dart`
- `profile_screen.dart`, `auction_screen.dart`, `bracket_screen.dart`
- `transfer_screen.dart`, `competition_screen.dart`

### Phase 4 — 동적 언어 전환 UI

설정 화면에 언어 선택 옵션 추가.

```dart
// locale_provider.dart
final localeProvider = StateProvider<Locale>((ref) => const Locale('ko'));
```

```dart
// 설정 화면
DropdownButton<Locale>(
  value: ref.watch(localeProvider),
  items: [
    DropdownMenuItem(value: Locale('ko'), child: Text('한국어')),
    DropdownMenuItem(value: Locale('en'), child: Text('English')),
  ],
  onChanged: (locale) => ref.read(localeProvider.notifier).state = locale!,
)
```

선택된 언어는 `SharedPreferences`에 저장하여 앱 재시작 후에도 유지.

### Phase 5 — 영어 번역 완성

Phase 2~3에서 추가된 모든 한국어 문자열의 영어 번역을 `app_en.arb`에 완성.

---

## 6. 코드 예시 (Before / After)

### Before

```dart
// shells/main_shell.dart
NavigationDestination(
  icon: Icon(Icons.home),
  label: '홈',
),
```

### After

```dart
// shells/main_shell.dart
import 'package:flutter_gen/gen_l10n/app_localizations.dart';

final l10n = AppLocalizations.of(context);

NavigationDestination(
  icon: Icon(Icons.home),
  label: l10n.navHome,
),
```

### Provider 에러 메시지

```dart
// Before
throw Exception('네트워크 연결에 실패했습니다.');

// After — 에러 코드 사용, UI 레이어에서 번역
throw NetworkException('network_failed');

// 또는 BuildContext 접근 가능한 레이어에서
Text(l10n.errorNetworkFailed)
```

> **주의:** Provider/Repository 레이어는 `BuildContext`에 접근 불가하므로 에러 코드(enum 또는 상수)를 던지고, UI 레이어에서 번역 처리.

---

## 7. 주의사항

### 복수형 처리

```json
// app_ko.arb
"memberCount": "{count, plural, =0{멤버 없음} =1{멤버 1명} other{{count}명}}",
"@memberCount": {
  "placeholders": { "count": { "type": "int" } }
}
```

### 날짜/시간 포맷

`intl` 패키지의 `DateFormat` 활용. 로케일별 포맷 자동 적용.

```dart
DateFormat.yMd(Localizations.localeOf(context).toString()).format(date)
```

### 숫자 포맷

```dart
NumberFormat.decimalPattern(locale.toString()).format(number)
```

### 서버 메시지 처리

서버에서 내려오는 에러/상태 메시지는 다국어 처리 어려움. 두 가지 접근:

1. **에러 코드 기반:** 서버가 코드(`"PLAYER_NOT_FOUND"`)를 반환하면 클라이언트에서 번역
2. **서버 다국어 지원:** `Accept-Language` 헤더로 서버에 로케일 전달 (서버 작업 필요)

---

## 예상 작업량

| Phase | 작업 내용 | 난이도 |
|-------|-----------|--------|
| Phase 1 | 인프라 설정 | 낮음 |
| Phase 2 | 공통 문자열 (~50개) | 낮음 |
| Phase 3 | 화면별 이전 (~200개 예상) | 중간 |
| Phase 4 | 언어 전환 UI | 낮음 |
| Phase 5 | 영어 번역 | 중간 |
