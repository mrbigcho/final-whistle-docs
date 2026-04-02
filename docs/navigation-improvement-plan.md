# 네비게이션 구조 개선 계획서

> 작성일: 2026-04-02
> 버전: v1.0
> 대상: Final Whistle 앱 네비게이션 리팩토링

---

## 1. 현재 구조 분석

### 1.1 라우팅 방식

GoRouter의 `StatefulShellRoute.indexedStack`을 사용하여 탭 간 상태를 유지한다.

### 1.2 메인 BottomNav (4탭)

| 탭 | 아이콘 | 역할 |
|---|---|---|
| 홈 | 🏠 | 피드, 최근 경기, 공지 |
| 대회 | 🏆 | 전체 대회 목록 · 검색 |
| 클럽 | 🛡️ | 내 클럽 목록 |
| MY | 👤 | 프로필, 설정 |

### 1.3 하위 화면 탭 구조

**클럽 상세** (`/clubs/:id`)
- 내부 TabBar: `대회` / `리그` / `멤버`
- 클럽 홈(대시보드)은 AppBar 타이틀 아래 고정 영역으로 표시

**대회 상세** (`/competitions/:id`)
- 내부 TabBar: `대진표` / `순위표`
- 참가자 목록은 별도 화면(`/competitions/:id/participants`)으로 이동

### 1.4 현재 구조의 문제점

1. **클럽 내 컨텍스트 손실**: 클럽 상세 TabBar에서 `대회` 탭을 눌러 대회 상세로 이동하면 메인 BottomNav의 `대회` 탭과 혼용되어 뒤로가기 계층이 불명확해진다.
2. **대회 상세 확장 어려움**: 대진표/순위표 두 탭만 있지만 향후 `경기 일정`, `참가자`, `통계` 등 탭이 늘어날 경우 TabBar가 비좁아진다.
3. **클럽 관리 접근성**: 클럽 설정은 클럽 상세 > 더보기 메뉴 안에 숨어 있어 관리자가 자주 사용하는 기능 접근이 불편하다.
4. **역할 기반 UI 분기 부재**: 일반 멤버와 관리자(Admin)에게 동일한 탭 구조를 제공하고 있어 권한 없는 메뉴가 노출된다.

---

## 2. 개선안 — 클럽 독립 네비게이션

### 2.1 개념

클럽 상세 진입 시 메인 BottomNav를 **클럽 전용 BottomNav**로 교체한다. 이는 중첩 Shell Route를 이용하여 구현하며, 클럽 컨텍스트 안에서의 탐색을 명확하게 분리한다.

```
메인 앱
└── StatefulShellRoute (메인 BottomNav: 홈/대회/클럽/MY)
    └── 클럽 목록 → 클럽 선택
        └── StatefulShellRoute (클럽 BottomNav: 홈/대회/멤버/설정)
            ├── /clubs/:id/home       — 클럽 대시보드
            ├── /clubs/:id/competitions — 클럽 내 대회 목록
            ├── /clubs/:id/members    — 멤버 관리
            └── /clubs/:id/settings  — 클럽 설정
```

### 2.2 클럽 전용 BottomNav 구성

| 탭 | 아이콘 | 역할 | 권한 |
|---|---|---|---|
| 홈 | 🏠 | 클럽 대시보드 (최근 대회, 멤버 현황, 공지) | 전체 |
| 대회 | 🏆 | 클럽 내 대회 생성 · 목록 | 전체 (생성은 Admin) |
| 멤버 | 👥 | 멤버 목록 · 초대 · 역할 관리 | 전체 (관리는 Admin) |
| 설정 | ⚙️ | 클럽 정보 수정, 공개 설정, 삭제 | Admin 전용 |

> **설정 탭 표시 조건**: 로그인 사용자가 해당 클럽의 Admin인 경우에만 BottomNav에 `설정` 탭을 노출한다. 일반 멤버에게는 3탭(홈/대회/멤버)만 표시한다.

### 2.3 진입/이탈 UX

- 메인 BottomNav의 `클럽` 탭 → 내 클럽 목록
- 클럽 카드 탭 → 클럽 BottomNav로 전환 (슬라이드 애니메이션)
- 클럽 BottomNav AppBar 좌측 `←` 버튼 → 메인 BottomNav 복귀
- 시스템 Back 제스처 → 내 클럽 목록으로 이동 (클럽 BottomNav 루트에서)

---

## 3. 대회 독립 네비게이션 비교

대회 상세에 독립 BottomNav를 적용하는 방안과 TabBar를 유지하는 방안을 비교한다.

### 3.1 옵션 A — TabBar 유지 (현행 + 탭 확장)

```
대회 상세 AppBar
└── TabBar: 대진표 / 경기 / 참가자 / 순위표
    └── 각 탭별 스크롤 콘텐츠
```

**장점**
- 구현 변경 최소화
- 탭 수가 4개 이하일 때 충분한 공간

**단점**
- 탭이 5개 이상이면 레이블이 잘림
- 클럽 내 대회와 전체 대회 탭 구조가 달라질 경우 일관성 저하

### 3.2 옵션 B — 독립 BottomNav (권장)

```
메인 앱
└── 대회 상세 진입
    └── StatefulShellRoute (대회 BottomNav)
        ├── /competitions/:id/bracket  — 대진표
        ├── /competitions/:id/matches  — 경기 일정 · 결과
        ├── /competitions/:id/participants — 참가자
        └── /competitions/:id/standings   — 순위표
```

**장점**
- 탭 확장성 확보 (최대 5탭까지 자연스럽게 수용)
- 클럽 BottomNav와 패턴 통일 → UX 일관성
- 각 탭 상태 독립 유지 (StatefulShellRoute 특성)

**단점**
- 중첩 Shell Route 2단계 증가로 라우터 복잡도 상승
- 클럽 내 대회 진입 시 BottomNav가 3단 중첩될 수 있음 (클럽 → 대회 → 탭)

### 3.3 권장 결정

**단기(v1)**: 옵션 A (TabBar 유지, 탭 4개로 확장)
**중기(v2)**: 옵션 B (독립 BottomNav 적용, 클럽 독립 네비게이션 안정화 후)

클럽 독립 네비게이션을 먼저 검증한 후 대회에도 동일 패턴을 적용하는 순차적 접근을 권장한다.

---

## 4. GoRouter 구조 변경안

### 4.1 현재 구조 (개략)

```dart
GoRouter(
  routes: [
    StatefulShellRoute.indexedStack(  // 메인 BottomNav
      branches: [
        StatefulShellBranch(routes: [GoRoute(path: '/home')]),
        StatefulShellBranch(routes: [GoRoute(path: '/competitions')]),
        StatefulShellBranch(routes: [
          GoRoute(path: '/clubs', routes: [
            GoRoute(path: ':id'),  // 클럽 상세 (내부 TabBar)
          ]),
        ]),
        StatefulShellBranch(routes: [GoRoute(path: '/my')]),
      ],
    ),
  ],
)
```

### 4.2 변경 구조 (제안)

```dart
GoRouter(
  routes: [
    StatefulShellRoute.indexedStack(  // [L1] 메인 BottomNav
      branches: [
        StatefulShellBranch(routes: [GoRoute(path: '/home')]),
        StatefulShellBranch(routes: [
          GoRoute(path: '/competitions', routes: [
            // v1: TabBar 유지
            GoRoute(path: ':id', builder: (_,__) => CompetitionDetailPage()),
          ]),
        ]),
        StatefulShellBranch(routes: [
          GoRoute(path: '/clubs', routes: [
            // 클럽 상세 진입 시 L2 Shell 진입
            ShellRoute(
              builder: (_, __, child) => ClubShell(child: child),
              routes: [
                StatefulShellRoute.indexedStack(  // [L2] 클럽 BottomNav
                  branches: [
                    StatefulShellBranch(routes: [
                      GoRoute(path: ':id/home'),
                    ]),
                    StatefulShellBranch(routes: [
                      GoRoute(path: ':id/competitions', routes: [
                        GoRoute(path: ':cid'),
                      ]),
                    ]),
                    StatefulShellBranch(routes: [
                      GoRoute(path: ':id/members'),
                    ]),
                    StatefulShellBranch(routes: [
                      GoRoute(path: ':id/settings'),  // Admin만 노출
                    ]),
                  ],
                ),
              ],
            ),
          ]),
        ]),
        StatefulShellBranch(routes: [GoRoute(path: '/my')]),
      ],
    ),
  ],
)
```

### 4.3 ClubShell 위젯 역할

```dart
class ClubShell extends StatelessWidget {
  const ClubShell({required this.child});
  final Widget child;

  @override
  Widget build(BuildContext context) {
    final isAdmin = context.watch<ClubProvider>().isAdmin;
    return Scaffold(
      body: child,
      bottomNavigationBar: ClubBottomNav(showSettings: isAdmin),
    );
  }
}
```

---

## 5. 구현 단계별 계획

### Phase 1 — 기반 정비 (1주)

- [ ] 현재 라우터 구조 단위 테스트 작성 (회귀 방지)
- [ ] `ClubShell` 위젯 스켈레톤 작성
- [ ] 클럽 관련 라우트 경로 `/clubs/:id/home` 형식으로 정규화
- [ ] `ClubProvider` — `isAdmin` 상태 노출 확인

### Phase 2 — 클럽 독립 네비게이션 구현 (1~2주)

- [ ] `StatefulShellRoute` L2 클럽 브랜치 구성
- [ ] 클럽 BottomNav 컴포넌트 (`ClubBottomNav`) 구현
  - Admin 여부에 따른 탭 수 조건부 렌더링
- [ ] 메인 BottomNav ↔ 클럽 BottomNav 전환 애니메이션
- [ ] Back 제스처/버튼 처리 (클럽 루트에서 클럽 목록으로)
- [ ] 딥링크 검증 (`/clubs/:id/members` 직접 진입 시 BottomNav 상태)

### Phase 3 — 대회 탭 확장 (1주)

- [ ] 대회 상세 TabBar에 `경기` / `참가자` 탭 추가
- [ ] 클럽 내 대회 상세 → 클럽 컨텍스트 유지 확인
- [ ] 전체 대회 상세와 클럽 내 대회 상세 공통 위젯 추출

### Phase 4 — QA 및 정리 (1주)

- [ ] iOS / Android 전 화면 Back 네비게이션 검증
- [ ] 딥링크 전체 시나리오 테스트
- [ ] 권한별 탭 노출 E2E 테스트
- [ ] 스크린 리더(접근성) 레이블 업데이트

---

## 6. 리스크 및 대응

| 리스크 | 가능성 | 영향 | 대응 |
|---|---|---|---|
| L2 Shell Route 중첩 시 Go Back 동작 불안정 | 중 | 높음 | Phase 1에서 단위 테스트 선작성, GoRouter 이슈 트래커 확인 |
| 클럽 BottomNav 상태가 앱 재시작 후 초기화 | 낮 | 중 | `initialLocation` 저장 로직 추가 검토 |
| Admin 판별 API 지연으로 설정 탭 깜빡임 | 중 | 낮음 | 로딩 중 설정 탭 숨김 처리 (shimmer 없이 단순 조건부) |
| 기존 딥링크(`/clubs/:id`) 경로 변경 | 높음 | 중 | 리다이렉트 규칙 `/clubs/:id` → `/clubs/:id/home` 추가 |
