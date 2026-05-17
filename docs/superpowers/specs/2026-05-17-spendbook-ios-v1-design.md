# Spend·book iOS — v1 design

**Date:** 2026-05-17
**Status:** Approved for implementation planning
**Companion repo:** `../personal-finance-tracker` (existing Next.js + Neon web app)

## Summary

Native iOS app (UIKit) that acts as a client to the existing Spend·book backend. v1 covers three screens: Dashboard, Transactions (list + filter + edit category), and Add Transaction (manual entry). CSV import, auto-categorize trigger, and narrative history are deferred to v2+.

## Decisions

| Area | Decision |
|---|---|
| Backend | Reuse existing Next.js API + Neon Postgres on Vercel |
| Auth | Static Bearer token in iOS Keychain, set at first launch |
| Min iOS | 17.0 |
| Language | Swift 5.10+ |
| UI framework | UIKit (no SwiftUI in v1) |
| Architecture | MVVM + Combine + diffable data sources |
| Dependencies | None. Stdlib only. |
| Visual design | Port Spend·book aesthetic (Fraunces + DM Sans, warm paper palette, sage/terracotta accents) |
| Build system | Single Xcode project, single app target. No SPM packages in v1. |

## v1 scope

**In scope:**
- Dashboard screen: month-to-date total, MoM delta, needs-review count, top merchants (30d)
- Transactions screen: list grouped by date, search, segmented filter, tap-to-recategorize
- Add Transaction modal: account, date, amount, merchant, description, category
- Setup sheet: API base URL + bearer token, persisted in Keychain
- Settings reopen (gear icon in Dashboard nav)

**Out of scope (defer to v2+):**
- CSV import
- Triggering auto-categorize from the device
- Reading monthly narrative reports
- Charts/graphs
- Swipe-to-delete on transactions
- Multi-user / proper accounts
- Push notifications
- iPad / Mac Catalyst optimizations beyond what compiles for free

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  iOS app (UIKit, MVVM + Combine, iOS 17+)                   │
│                                                             │
│  DashboardVC ◀▶ DashboardVM                                 │
│  TransactionsVC ◀▶ TransactionsVM ─▶ CategoryPickerVC       │
│  AddTxModalVC ◀▶ AddTxVM                                    │
│  SetupSheetVC                                               │
│                                                             │
│         ▼ all VMs depend on                                 │
│  APIClient (protocol) ──▶ URLSession + async/await          │
│  KeychainStore (URL + token)                                │
│  DesignSystem (colors, fonts, spacing constants)            │
└────────────────────────┼────────────────────────────────────┘
                         │ HTTPS (Bearer <token>)
              ┌──────────▼──────────┐
              │  Spend·book API     │
              │  Next.js on Vercel  │  + token middleware (new)
              │  Neon Postgres      │
              └─────────────────────┘
```

**Key principles:**
- Each screen has a thin `UIViewController` that binds to a `@Published`-driven `ViewModel` via Combine
- Lists use `UICollectionViewDiffableDataSource` for safe, animated updates
- `APIClient` is a single struct using `URLSession` + async/await; ViewModels call it from `Task {}` blocks
- `KeychainStore` is the only persistence layer; no Core Data, no SwiftData
- No third-party dependencies

## Data flow

### App launch
```
AppDelegate ─► SceneDelegate ─► RootTabBarController
                                  ├─ NavController → DashboardVC
                                  └─ NavController → TransactionsVC
                                                       └─ "+ Add" → AddTxModalVC
```
On launch, if `KeychainStore.load() == nil`, present `SetupSheetVC` modally over root.

### Dashboard read
```
DashboardVC.viewWillAppear
  → DashboardVM.refresh()
    → Task { APIClient.getDashboard(month:) }
      → GET /api/dashboard?month=2026-05  (new endpoint)
      → DashboardResponse decoded
    → @Published state = .loaded(response)
  → Combine sink → diffable snapshot applied
    (sections: summary, needsReview, topMerchants)
```

### Transactions list + edit category
```
TransactionsVC.viewWillAppear
  → TransactionsVM.load(filter)
    → GET /api/transactions?…  (new GET on existing route)
    → [Transaction] decoded
  → diffable snapshot applied (sectioned by postedAt date)

Tap row → push CategoryPickerVC
  → user picks category
    → PATCH /api/transactions/:id
    → on success, VM updates that single item; snapshot reapplied
```

### Add transaction
```
"+ Add" → present AddTxModalVC (.formSheet)
  → AddTxVM validates locally (amount > 0, merchant non-empty)
    → POST /api/transactions
    → 201 Created
  → dismiss modal
  → NotificationCenter.default.post(.transactionsDidChange)
    → TransactionsVC + DashboardVC reload (immediately if visible, else on next viewWillAppear)
```

### Error handling
- Network/decoding errors → VM state `.error(message)` → VC shows `UIContentUnavailableView` with "Try again"
- 401 → APIClient throws `.unauthorized` → SceneDelegate observes via Notification → clears Keychain → presents `SetupSheetVC`
- Each VM stores its in-flight `Task` and cancels on deinit / before new request

## Screens

### 1. Dashboard (Tab 1)
```
┌─────────────────────────────────┐
│ Spend·book          [⚙ Settings]│  large title, Fraunces
├─────────────────────────────────┤
│ MAY 2026                        │  DM Sans caps, ink-soft
│ $302.45                         │  Fraunces 48pt, ink
│ ↑ +0.0% vs April                │  terracotta if up, sage if down
│ ───────────────────────────     │
│ NEEDS REVIEW                    │
│ 0  — all caught up ✓            │
│ ───────────────────────────     │
│ TOP MERCHANTS · 30 DAYS         │
│ Target Store          $89.99    │
│ Whole Foods Market    $87.34    │
│ Trader Joe's          $47.22    │
│ Shell Oil             $42.10    │
│ Uber Eats             $23.45    │
└─────────────────────────────────┘
```
- `UICollectionView` with compositional layout, list appearance for merchants section
- Three diffable sections: `summary`, `needsReview`, `topMerchants`
- Pull-to-refresh (`UIRefreshControl`)
- No charts in v1

### 2. Transactions (Tab 2)
```
┌─────────────────────────────────┐
│ Transactions             [+]    │  + opens AddTxModal
├─────────────────────────────────┤
│ 🔍 Search                       │  UISearchController in nav
│ [All] [This month] [Uncat'd]    │  UISegmentedControl
├─────────────────────────────────┤
│ May 14                          │  section header (grouped by date)
│   Target Store     -$89.99      │
│   groceries · LLM 0.92          │
│ May 13                          │
│   Whole Foods Market -$87.34    │
│   groceries · USER              │
└─────────────────────────────────┘
```
- `UICollectionView` with list appearance, sectioned by `postedAt` date
- Row tap → push `CategoryPickerVC` (list of distinct categories already used + free-text "Other…")
- Empty state: `UIContentUnavailableView` with "Add your first transaction"
- Swipe actions deferred to v2

### 3. Add Transaction (modal, .formSheet)
```
┌─────────────────────────────────┐
│ Cancel    Add Transaction  Save │
├─────────────────────────────────┤
│ Account     [Checking      ▾]   │  UIMenu picker; loaded from /api/accounts
│ Date        [May 17, 2026  📅]  │  UIDatePicker .compact
│ Amount      [$  0.00          ] │  decimal keypad
│ Merchant    [                  ]│
│ Description [                  ]│  optional, multi-line
│ Category    [None         ▾]    │  optional; empty allowed
└─────────────────────────────────┘
```
- `UITableView` `.insetGrouped`
- Save disabled until amount > 0 AND merchant non-empty
- On 201: dismiss + post `.transactionsDidChange`

### 4. Setup sheet (first-launch + manual reopen)
```
API base URL:  [https://personal-finance…]
Bearer token:  [••••••••••••]
                                              [Save]
                                              [Clear & sign out]
```
- Reachable via gear icon in Dashboard nav bar
- Token field `isSecureTextEntry = true`

## Networking

### APIClient (protocol-backed for testability)
```swift
protocol APIClientProtocol {
    func getDashboard(month: String) async throws -> DashboardResponse
    func getTransactions(filter: TxFilter) async throws -> [Transaction]
    func createTransaction(_ body: NewTransaction) async throws -> Transaction
    func updateCategory(id: String, category: String) async throws -> Transaction
    func getAccounts() async throws -> [Account]
}

struct APIClient: APIClientProtocol {
    let baseURL: URL
    let token: String
    private let session = URLSession.shared
    private let decoder: JSONDecoder = { … iso8601 dates … }()
}
```

### Request builder
- Sets `Authorization: Bearer <token>` on every request
- JSON encode body if present
- Status code switch: 2xx → decode; 401 → `.unauthorized`; 4xx → `.client`; 5xx → `.server`

### Models (mirror Prisma shapes)
```swift
struct Transaction: Decodable, Hashable, Identifiable {
    let id: String
    let accountId: String
    let postedAt: Date
    let amount: Decimal           // decoded from string
    let currency: String
    let merchantRaw: String
    let description: String?
    let category: String?
    let categorySource: CategorySource?
    let categoryConfidence: Double?
}
struct Account: Decodable, Hashable, Identifiable { let id, name: String }
struct DashboardResponse: Decodable {
    let monthTotal: Decimal
    let prevMonthTotal: Decimal
    let momDelta: Double
    let needsReviewCount: Int
    let topMerchants: [MerchantTotal]
}
struct MerchantTotal: Decodable, Hashable { let merchant: String; let total: Decimal }
struct NewTransaction: Encodable {
    let accountId: String
    let postedAt: Date
    let amount: Decimal
    let merchantRaw: String
    let description: String?
    let category: String?
}
enum CategorySource: String, Decodable { case llm = "LLM", user = "USER" }
enum APIError: Error {
    case invalidResponse, unauthorized
    case client(Int, String?), server(Int)
}
```

### Decimal handling
Prisma serializes `Decimal` as a JSON string (`"302.45"`). Swift's `Decimal` doesn't natively decode from a string, so we add a small `KeyedDecodingContainer` extension (`decimalFromString(forKey:)`) that reads the string and constructs `Decimal(string:)`. Prevents floating-point drift on money values. Encoding goes the same way: `Decimal` → `String` in `NewTransaction`.

### KeychainStore
- Wraps Keychain Services C API (no SPM dep)
- Two items: `apiBaseURL` (string), `apiToken` (string)
- `KeychainStore.shared.load() -> (URL, String)?` returns nil if either missing → triggers Setup sheet
- `save(url:, token:)`, `clear()`

### Cancellation
- Each VM stores its current `Task` handle; calls `cancel()` in `deinit` and before starting a new request

## Backend changes

Three additive changes to the existing `personal-finance-tracker` repo. None break the web UI (which reads via server components, bypassing the HTTP API).

### Change 1 — Bearer token middleware
New file `src/lib/api-auth.ts` (separate from the existing `src/lib/auth.ts`, which holds `getCurrentUserId()`):
```ts
export function requireApiToken(req: Request): Response | null {
  const expected = process.env.API_TOKEN;
  if (!expected) return new Response("Server misconfigured", { status: 500 });
  const auth = req.headers.get("authorization") ?? "";
  if (auth !== `Bearer ${expected}`) return new Response("Unauthorized", { status: 401 });
  return null;
}
```

Also: existing `POST /api/transactions` and `PATCH /api/transactions/[id]` currently return `amount` as a JS number. Change them (and their tests) to return `amount` as a string so iOS has one consistent `Decimal`-decoding rule across every endpoint. Web UI is unaffected (it reads via server components, not these routes).

Note: amounts are stored **positive** in the DB (spending = positive value, filtered by `amount > 0` in the dashboard query). The iOS UI prepends a `-` for display only; wire format stays positive.

Applied to:
- `POST /api/transactions`
- `PATCH /api/transactions/[id]`
- `POST /api/accounts`
- `POST /api/import/csv`
- `POST /api/categorize`
- `GET /api/accounts`
- `GET /api/dashboard` (new — see Change 2)
- `GET /api/transactions` (new — see Change 3)

New env var: `API_TOKEN` (long random string, set in Vercel + iOS Keychain).

### Change 2 — `GET /api/dashboard?month=YYYY-MM`
Extract aggregation logic from the existing `/dashboard` server component into `src/lib/dashboard.ts` (`loadDashboardData(userId, month)`), so both the server component and the new API route call it. Targeted refactor that improves code we're already touching.

```ts
export async function GET(req: Request) {
  const guard = requireApiToken(req); if (guard) return guard;
  const month = new URL(req.url).searchParams.get("month") ?? currentMonth();
  const data = await loadDashboardData(USER_ID, month);
  return Response.json({
    monthTotal: data.monthTotal.toString(),
    prevMonthTotal: data.prevMonthTotal.toString(),
    momDelta: data.momDelta,
    needsReviewCount: data.needsReviewCount,
    topMerchants: data.topMerchants.map(m => ({ merchant: m.merchant, total: m.total.toString() })),
  });
}
```

### Change 3 — `GET /api/transactions?…`
Add a `GET` handler to the existing `src/app/api/transactions/route.ts`. Filters: `search` (merchant substring), `month` (YYYY-MM), `uncategorized` (bool). Returns top 200 ordered by `postedAt desc`. `Decimal` fields serialized as strings via a `serializeTx` helper.

### Backend testing additions
- `requireApiToken` unit tests (missing header, wrong token, right token)
- `loadDashboardData` unit tests (same Prisma boundary the existing tests use)
- `serializeTx` round-trip test (Decimal → string)

## iOS testing

### Targets
- `SpendbookTests` (XCTest, fast, no host app)
- `SpendbookUITests` — skipped in v1 (three screens don't justify XCUITest flake)

### Coverage matrix
| Layer | Test? | Why |
|---|---|---|
| `APIClient` request builder | ✅ | URL/method/headers/body construction |
| `APIClient` response decoding | ✅ | Decimal-as-string, ISO8601, enum mapping |
| `APIError` mapping by status code | ✅ | Common regression site |
| ViewModels (state machine) | ✅ | idle → loading → loaded/error, with stub APIClient |
| `KeychainStore` | ✅ | save → load → clear round-trip |
| Form validation in `AddTxVM` | ✅ | Pure VM logic |
| Formatters | ✅ | Locale-dependent |
| ViewControllers / layout | ❌ | Brittle in XCTest; verify by running the app |
| Live `URLSession` network calls | ❌ | We test our code, not Apple's |

### Manual smoke checklist (run before tagging a build)
1. Fresh install → Setup sheet appears
2. Save garbage token → 401 → Setup sheet reappears
3. Save correct token → Dashboard total matches web app
4. Pull to refresh on Dashboard
5. Transactions tab: search works, segmented filter works
6. Tap row → pick category → row updates → web app shows same change
7. Add transaction → row appears on top → dashboard total updates
8. Force-quit + relaunch → still signed in
9. Settings → Clear & sign out → Setup sheet reappears

## File layout

```
personal-finance-tracker-ios/
├── README.md
├── .gitignore
├── docs/superpowers/specs/2026-05-17-spendbook-ios-v1-design.md
├── Spendbook.xcodeproj/
└── Spendbook/
    ├── App/
    │   ├── AppDelegate.swift
    │   ├── SceneDelegate.swift
    │   ├── RootTabBarController.swift
    │   └── Info.plist
    ├── Networking/
    │   ├── APIClient.swift              ← protocol + concrete impl
    │   ├── APIError.swift
    │   ├── DecimalString.swift          ← Codable helper
    │   └── KeychainStore.swift
    ├── Models/
    │   ├── Transaction.swift
    │   ├── Account.swift
    │   ├── DashboardResponse.swift
    │   └── NewTransaction.swift
    ├── DesignSystem/
    │   ├── Colors.swift                 ← paper, ink, sage, terracotta, butter
    │   ├── Typography.swift             ← Fraunces + DM Sans
    │   ├── Spacing.swift
    │   └── Fonts/
    │       ├── Fraunces-Regular.ttf
    │       ├── Fraunces-Bold.ttf
    │       ├── DMSans-Regular.ttf
    │       └── DMSans-Medium.ttf
    ├── Features/
    │   ├── Dashboard/
    │   │   ├── DashboardVC.swift
    │   │   ├── DashboardVM.swift
    │   │   └── DashboardCells.swift
    │   ├── Transactions/
    │   │   ├── TransactionsVC.swift
    │   │   ├── TransactionsVM.swift
    │   │   ├── TransactionCell.swift
    │   │   └── CategoryPickerVC.swift
    │   ├── AddTransaction/
    │   │   ├── AddTxModalVC.swift
    │   │   └── AddTxVM.swift
    │   └── Settings/
    │       └── SetupSheetVC.swift
    └── Resources/
        └── Assets.xcassets
SpendbookTests/
├── APIClientTests.swift
├── DashboardVMTests.swift
├── TransactionsVMTests.swift
├── AddTxVMTests.swift
├── KeychainStoreTests.swift
└── Stubs/
    └── StubAPIClient.swift
```

## Build settings
- Min deployment: iOS 17.0
- Swift 5.10+ (Xcode 16+)
- Bundle ID: `com.baha.spendbook`
- Custom fonts registered via `UIAppFonts` in `Info.plist`
- No third-party deps, no SPM packages, no CocoaPods

## Design system (port from web)

| Token | Value | Use |
|---|---|---|
| `paper` | `#f5f1e8` | Window background |
| `paper2` | `#efe9dc` | Card / row surface |
| `ink` | `#2c2a26` | Primary text |
| `inkSoft` | `#6b6862` | Secondary text |
| `rule` | `#ddd7ca` | Hairline separators |
| `sage` | `#87a878` | Primary accent (down deltas, income, focus) |
| `terracotta` | `#c97b5c` | Secondary accent (up deltas) |
| `butter` | `#f0e4b8` | Highlights |

Typography:
- Display: Fraunces (Regular, Bold) — large titles, dollar amounts
- Body: DM Sans (Regular, Medium) — everything else
- Bundle .ttf files in `DesignSystem/Fonts/`, register via `UIAppFonts`

Spacing scale: 4 / 8 / 12 / 16 / 24 / 32 / 48 pt.

Light mode only in v1. Dark mode deferred (the web app is light-only too).

## Implementation plan phases

The detailed plan is generated by the `writing-plans` skill. High-level order:

1. **Backend prep** — `requireApiToken`, refactor `lib/dashboard.ts`, add `GET /api/dashboard`, add `GET /api/transactions`, deploy, set `API_TOKEN` in Vercel, verify with `curl`
2. **Xcode scaffold** — create project, folder structure, font registration, empty tab bar shell runs on simulator
3. **DesignSystem + Networking + Keychain** — constants, `APIClient`, models, `KeychainStore`, unit tests
4. **Setup sheet** — first-launch + 401 flow
5. **Dashboard screen** — VM + VC + cells + diffable snapshot + wire to real API
6. **Transactions screen** — list + search + segmented filter + `CategoryPickerVC`
7. **Add Transaction modal** — form + validation + POST + notification refresh
8. **Polish + manual smoke** — empty states, error states, pull-to-refresh, Settings reopen

## Open questions / known unknowns

- **Top-merchants window:** the web dashboard says "last 30 days" — confirm the SQL in `loadDashboardData` matches that (rolling 30d, not current month). The new API must return the same window the web UI shows.
- **Currency:** v1 assumes USD everywhere (matches the web seed). Multi-currency display deferred.
- **iPad:** the app will run on iPad in compatibility mode. No bespoke iPad layout in v1.
- **Telemetry / crash reporting:** none in v1. Add in v2 if usage warrants.
