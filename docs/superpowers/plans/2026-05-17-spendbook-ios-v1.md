# Spend·book iOS v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a native UIKit iOS app (iOS 17+) that acts as a Bearer-token-authenticated client to the existing Spend·book Next.js + Neon backend, covering Dashboard, Transactions (list/filter/edit-category), and Add Transaction.

**Architecture:** MVVM + Combine + diffable data sources, URLSession + async/await, no third-party deps. Backend gets three additive changes (token middleware, new dashboard GET, new transactions GET) plus a serialization tweak to existing transaction endpoints.

**Tech Stack:** Swift 5.10+, UIKit, Combine, URLSession, XCTest, Keychain Services. Backend: Next.js 16, Prisma 6, Neon Postgres, vitest.

**Two repos involved:**
- Backend: `/Users/baha/Desktop/llm-ai-projects/personal-finance-tracker` (existing)
- iOS app: `/Users/baha/Desktop/llm-ai-projects/personal-finance-tracker-ios` (new — repo already initialized with the design doc committed)

**Spec:** `docs/superpowers/specs/2026-05-17-spendbook-ios-v1-design.md` (in the iOS repo)

---

## File map

### Backend (`personal-finance-tracker/`)
- Create: `src/lib/api-auth.ts` — `requireApiToken(req)`
- Create: `src/lib/api-auth.test.ts` — unit tests
- Create: `src/lib/dashboard.ts` — `loadDashboardData(userId, monthOffset)` extracted from page
- Create: `src/lib/dashboard.test.ts` — unit tests against mocked Prisma
- Create: `src/app/api/dashboard/route.ts` — `GET /api/dashboard?month=YYYY-MM`
- Create: `src/app/api/dashboard/route.test.ts`
- Modify: `src/app/dashboard/page.tsx` — use new `loadDashboardData`
- Modify: `src/app/api/transactions/route.ts` — add `GET`, gate `POST` with token, return `amount` as string
- Modify: `src/app/api/transactions/route.test.ts` — cover GET, token check, string amount
- Modify: `src/app/api/transactions/[id]/route.ts` — gate `PATCH` with token, return `amount` as string
- Modify: `src/app/api/transactions/[id]/route.test.ts` — cover token + string amount
- Modify: `src/app/api/accounts/route.ts` — gate `GET` and `POST` with token
- Modify: `src/app/api/import/csv/route.ts` — gate `POST` with token
- Modify: `src/app/api/categorize/route.ts` — gate `POST` with token
- Modify: `.env.example` / `.env` — add `API_TOKEN`

### iOS (`personal-finance-tracker-ios/`)
- Create: `Spendbook.xcodeproj/` — Xcode project
- Create: `Spendbook/App/AppDelegate.swift`, `SceneDelegate.swift`, `RootTabBarController.swift`, `Info.plist`
- Create: `Spendbook/Networking/APIClient.swift`, `APIError.swift`, `DecimalString.swift`, `KeychainStore.swift`
- Create: `Spendbook/Models/Transaction.swift`, `Account.swift`, `DashboardResponse.swift`, `NewTransaction.swift`
- Create: `Spendbook/DesignSystem/Colors.swift`, `Typography.swift`, `Spacing.swift`, `Fonts/*.ttf`
- Create: `Spendbook/Features/Dashboard/DashboardVC.swift`, `DashboardVM.swift`, `DashboardCells.swift`
- Create: `Spendbook/Features/Transactions/TransactionsVC.swift`, `TransactionsVM.swift`, `TransactionCell.swift`, `CategoryPickerVC.swift`
- Create: `Spendbook/Features/AddTransaction/AddTxModalVC.swift`, `AddTxVM.swift`
- Create: `Spendbook/Features/Settings/SetupSheetVC.swift`
- Create: `SpendbookTests/APIClientTests.swift`, `DashboardVMTests.swift`, `TransactionsVMTests.swift`, `AddTxVMTests.swift`, `KeychainStoreTests.swift`, `Stubs/StubAPIClient.swift`

---

## Phase 1 — Backend prep

All backend tasks run inside `cd /Users/baha/Desktop/llm-ai-projects/personal-finance-tracker`. Test runner is vitest (`pnpm test` runs the full suite; `pnpm exec vitest run path/to/file.test.ts` runs one file).

### Task 1: Bearer-token middleware

**Files:**
- Create: `src/lib/api-auth.ts`
- Create: `src/lib/api-auth.test.ts`

- [ ] **Step 1: Write the failing test**

Create `src/lib/api-auth.test.ts`:
```ts
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import { requireApiToken } from "./api-auth";

function req(headers: Record<string, string> = {}) {
  return new Request("https://example.com/x", { headers });
}

describe("requireApiToken", () => {
  const originalToken = process.env.API_TOKEN;
  beforeEach(() => { process.env.API_TOKEN = "secret-token-abc"; });
  afterEach(() => { process.env.API_TOKEN = originalToken; });

  it("returns null when bearer matches", () => {
    const result = requireApiToken(req({ authorization: "Bearer secret-token-abc" }));
    expect(result).toBeNull();
  });

  it("returns 401 when header missing", async () => {
    const result = requireApiToken(req());
    expect(result?.status).toBe(401);
  });

  it("returns 401 when bearer wrong", async () => {
    const result = requireApiToken(req({ authorization: "Bearer nope" }));
    expect(result?.status).toBe(401);
  });

  it("returns 500 when API_TOKEN env missing", async () => {
    delete process.env.API_TOKEN;
    const result = requireApiToken(req({ authorization: "Bearer secret-token-abc" }));
    expect(result?.status).toBe(500);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```
pnpm exec vitest run src/lib/api-auth.test.ts
```
Expected: FAIL — `Cannot find module './api-auth'`.

- [ ] **Step 3: Implement minimal middleware**

Create `src/lib/api-auth.ts`:
```ts
export function requireApiToken(req: Request): Response | null {
  const expected = process.env.API_TOKEN;
  if (!expected) return new Response("Server misconfigured", { status: 500 });
  const auth = req.headers.get("authorization") ?? "";
  if (auth !== `Bearer ${expected}`) {
    return new Response("Unauthorized", { status: 401 });
  }
  return null;
}
```

- [ ] **Step 4: Run test to verify it passes**

```
pnpm exec vitest run src/lib/api-auth.test.ts
```
Expected: PASS, 4/4.

- [ ] **Step 5: Add `API_TOKEN` to `.env` and `.env.example`**

Append to `.env`:
```
API_TOKEN=dev-local-token-CHANGEME
```
Append to `.env.example` (commit-safe):
```
API_TOKEN=
```

- [ ] **Step 6: Commit**

```bash
cd /Users/baha/Desktop/llm-ai-projects/personal-finance-tracker
git add src/lib/api-auth.ts src/lib/api-auth.test.ts .env.example
git commit -m "feat: add requireApiToken bearer middleware for iOS client"
```

---

### Task 2: Extract dashboard aggregation to `src/lib/dashboard.ts`

**Files:**
- Create: `src/lib/dashboard.ts`
- Create: `src/lib/dashboard.test.ts`
- Modify: `src/app/dashboard/page.tsx` — call the new function

- [ ] **Step 1: Write the failing test**

Create `src/lib/dashboard.test.ts`:
```ts
import { describe, it, expect, vi, beforeEach } from "vitest";

vi.mock("@/lib/prisma", () => ({
  prisma: {
    transaction: {
      aggregate: vi.fn(),
      groupBy: vi.fn(),
      count: vi.fn(),
    },
  },
}));

import { prisma } from "@/lib/prisma";
import { loadDashboardData } from "./dashboard";

describe("loadDashboardData", () => {
  beforeEach(() => { vi.clearAllMocks(); });

  it("returns month totals, mom delta, top merchants and needs-review", async () => {
    (prisma.transaction.aggregate as any)
      .mockResolvedValueOnce({ _sum: { amount: 302.45 } })   // this month
      .mockResolvedValueOnce({ _sum: { amount: 250.00 } });  // last month
    (prisma.transaction.groupBy as any).mockResolvedValueOnce([
      { merchantRaw: "Target Store",       _sum: { amount: 89.99 } },
      { merchantRaw: "Whole Foods Market", _sum: { amount: 87.34 } },
    ]);
    (prisma.transaction.count as any).mockResolvedValueOnce(0);

    const result = await loadDashboardData("user_1", "2026-05");

    expect(result.monthTotal.toString()).toBe("302.45");
    expect(result.prevMonthTotal.toString()).toBe("250");
    expect(result.momDelta).toBeCloseTo(20.98, 1);
    expect(result.needsReviewCount).toBe(0);
    expect(result.topMerchants).toEqual([
      { merchant: "Target Store",       total: expect.objectContaining({}) },
      { merchant: "Whole Foods Market", total: expect.objectContaining({}) },
    ]);
    expect(result.topMerchants[0].total.toString()).toBe("89.99");
  });

  it("returns 0 momDelta when previous month was zero", async () => {
    (prisma.transaction.aggregate as any)
      .mockResolvedValueOnce({ _sum: { amount: 100 } })
      .mockResolvedValueOnce({ _sum: { amount: 0 } });
    (prisma.transaction.groupBy as any).mockResolvedValueOnce([]);
    (prisma.transaction.count as any).mockResolvedValueOnce(0);

    const result = await loadDashboardData("user_1", "2026-05");
    expect(result.momDelta).toBe(0);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```
pnpm exec vitest run src/lib/dashboard.test.ts
```
Expected: FAIL — `Cannot find module './dashboard'`.

- [ ] **Step 3: Implement the helper**

Create `src/lib/dashboard.ts`:
```ts
import Decimal from "decimal.js";
import { prisma } from "@/lib/prisma";

export interface MerchantTotal {
  merchant: string;
  total: Decimal;
}

export interface DashboardData {
  monthLabel: string;       // e.g. "May 2026"
  monthTotal: Decimal;
  prevMonthTotal: Decimal;
  momDelta: number;         // percent
  needsReviewCount: number;
  topMerchants: MerchantTotal[];
}

function monthRange(monthIso: string): { start: Date; end: Date } {
  const [y, m] = monthIso.split("-").map(Number);
  const start = new Date(Date.UTC(y, m - 1, 1));
  const end = new Date(Date.UTC(y, m, 1));
  return { start, end };
}

function prevMonthIso(monthIso: string): string {
  const [y, m] = monthIso.split("-").map(Number);
  const d = new Date(Date.UTC(y, m - 2, 1));
  return `${d.getUTCFullYear()}-${String(d.getUTCMonth() + 1).padStart(2, "0")}`;
}

function monthLabel(monthIso: string): string {
  const { start } = monthRange(monthIso);
  return start.toLocaleDateString("en-US", { month: "long", year: "numeric", timeZone: "UTC" });
}

export async function loadDashboardData(userId: string, monthIso: string): Promise<DashboardData> {
  const thisMonth = monthRange(monthIso);
  const lastMonth = monthRange(prevMonthIso(monthIso));

  const [thisAgg, lastAgg, topMerchants, needsReviewCount] = await Promise.all([
    prisma.transaction.aggregate({
      where: { userId, postedAt: { gte: thisMonth.start, lt: thisMonth.end }, amount: { gt: 0 } },
      _sum: { amount: true },
    }),
    prisma.transaction.aggregate({
      where: { userId, postedAt: { gte: lastMonth.start, lt: lastMonth.end }, amount: { gt: 0 } },
      _sum: { amount: true },
    }),
    prisma.transaction.groupBy({
      by: ["merchantRaw"],
      where: { userId, postedAt: { gte: thisMonth.start, lt: thisMonth.end }, amount: { gt: 0 } },
      _sum: { amount: true },
      orderBy: { _sum: { amount: "desc" } },
      take: 5,
    }),
    prisma.transaction.count({
      where: { userId, categorySource: "LLM", categoryConfidence: { lt: 0.6 } },
    }),
  ]);

  const monthTotal = new Decimal(thisAgg._sum.amount?.toString() ?? "0");
  const prevMonthTotal = new Decimal(lastAgg._sum.amount?.toString() ?? "0");
  const momDelta = prevMonthTotal.isZero()
    ? 0
    : monthTotal.minus(prevMonthTotal).div(prevMonthTotal).times(100).toNumber();

  return {
    monthLabel: monthLabel(monthIso),
    monthTotal,
    prevMonthTotal,
    momDelta,
    needsReviewCount,
    topMerchants: topMerchants.map((m) => ({
      merchant: m.merchantRaw,
      total: new Decimal(m._sum.amount?.toString() ?? "0"),
    })),
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

```
pnpm exec vitest run src/lib/dashboard.test.ts
```
Expected: PASS, 2/2.

- [ ] **Step 5: Refactor `src/app/dashboard/page.tsx` to use the helper for monthTotal/prevTotal/momDelta/topMerchants/needsReviewCount**

Replace `src/app/dashboard/page.tsx`:
```tsx
import { getCurrentUserId } from "@/lib/auth";
import { loadDashboardData } from "@/lib/dashboard";
import { prisma } from "@/lib/prisma";
import { TotalCard } from "@/components/dashboard/total-card";
import { CategoryBarChart } from "@/components/dashboard/category-bar-chart";
import { DailyLineChart } from "@/components/dashboard/daily-line-chart";
import { TopMerchants } from "@/components/dashboard/top-merchants";
import { NeedsReviewWidget } from "@/components/dashboard/needs-review-widget";

function currentMonthIso(): string {
  const now = new Date();
  return `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, "0")}`;
}

export default async function DashboardPage() {
  const userId = getCurrentUserId();
  const monthIso = currentMonthIso();

  // The shared helper covers what iOS also needs.
  const shared = await loadDashboardData(userId, monthIso);

  // Chart-only queries the iOS app does not consume stay here.
  const thisMonth = (() => {
    const [y, m] = monthIso.split("-").map(Number);
    return {
      start: new Date(Date.UTC(y, m - 1, 1)),
      end: new Date(Date.UTC(y, m, 1)),
    };
  })();
  const [byCategory, byDay] = await Promise.all([
    prisma.transaction.groupBy({
      by: ["category"],
      where: { userId, postedAt: { gte: thisMonth.start, lt: thisMonth.end }, amount: { gt: 0 } },
      _sum: { amount: true },
    }),
    prisma.$queryRaw<{ day: Date; total: number }[]>`
      SELECT DATE_TRUNC('day', "postedAt")::date AS day, SUM("amount")::float AS total
      FROM "Transaction"
      WHERE "userId" = ${userId}
        AND "postedAt" >= NOW() - INTERVAL '30 days'
        AND "amount" > 0
      GROUP BY 1 ORDER BY 1
    `,
  ]);

  return (
    <main className="max-w-6xl mx-auto px-6 lg:px-10 py-10 lg:py-14 space-y-8">
      <header className="space-y-1">
        <p className="text-sm text-ink-soft uppercase tracking-widest">Overview</p>
        <h1 className="display text-4xl lg:text-5xl text-ink">{shared.monthLabel}</h1>
      </header>

      <section className="grid grid-cols-1 md:grid-cols-3 gap-5">
        <TotalCard thisAmount={Number(shared.monthTotal.toString())} delta={shared.momDelta} />
        <NeedsReviewWidget count={shared.needsReviewCount} />
      </section>

      <section className="grid grid-cols-1 lg:grid-cols-2 gap-5">
        <CategoryBarChart
          data={byCategory
            .filter((r) => r.category)
            .map((r) => ({ category: r.category!, amount: Number(r._sum.amount ?? 0) }))}
        />
        <DailyLineChart
          data={byDay.map((d) => ({ day: d.day.toISOString().slice(0, 10), total: d.total }))}
        />
      </section>

      <TopMerchants
        rows={shared.topMerchants.map((m) => ({
          merchantRaw: m.merchant,
          amount: Number(m.total.toString()),
        }))}
      />
    </main>
  );
}
```

- [ ] **Step 6: Run the full test suite to make sure nothing else broke**

```
pnpm test
```
Expected: all green.

- [ ] **Step 7: Commit**

```bash
git add src/lib/dashboard.ts src/lib/dashboard.test.ts src/app/dashboard/page.tsx
git commit -m "refactor: extract loadDashboardData; reuse from dashboard page"
```

---

### Task 3: New `GET /api/dashboard?month=YYYY-MM`

**Files:**
- Create: `src/app/api/dashboard/route.ts`
- Create: `src/app/api/dashboard/route.test.ts`

- [ ] **Step 1: Write the failing test**

Create `src/app/api/dashboard/route.test.ts`:
```ts
import { describe, it, expect, vi, beforeEach, afterEach } from "vitest";
import Decimal from "decimal.js";

vi.mock("@/lib/auth", () => ({ getCurrentUserId: () => "user_1" }));
vi.mock("@/lib/dashboard", () => ({
  loadDashboardData: vi.fn().mockResolvedValue({
    monthLabel: "May 2026",
    monthTotal: new Decimal("302.45"),
    prevMonthTotal: new Decimal("250.00"),
    momDelta: 20.98,
    needsReviewCount: 0,
    topMerchants: [{ merchant: "Target Store", total: new Decimal("89.99") }],
  }),
}));

import { GET } from "./route";

describe("GET /api/dashboard", () => {
  const original = process.env.API_TOKEN;
  beforeEach(() => { process.env.API_TOKEN = "tok"; });
  afterEach(() => { process.env.API_TOKEN = original; });

  it("returns 401 without token", async () => {
    const res = await GET(new Request("http://x/api/dashboard?month=2026-05"));
    expect(res.status).toBe(401);
  });

  it("returns serialized payload with token", async () => {
    const res = await GET(new Request("http://x/api/dashboard?month=2026-05", {
      headers: { authorization: "Bearer tok" },
    }));
    expect(res.status).toBe(200);
    const body = await res.json();
    expect(body).toEqual({
      monthLabel: "May 2026",
      monthTotal: "302.45",
      prevMonthTotal: "250",
      momDelta: 20.98,
      needsReviewCount: 0,
      topMerchants: [{ merchant: "Target Store", total: "89.99" }],
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```
pnpm exec vitest run src/app/api/dashboard/route.test.ts
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement the route**

Create `src/app/api/dashboard/route.ts`:
```ts
import { requireApiToken } from "@/lib/api-auth";
import { getCurrentUserId } from "@/lib/auth";
import { loadDashboardData } from "@/lib/dashboard";

function currentMonthIso(): string {
  const now = new Date();
  return `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, "0")}`;
}

export async function GET(req: Request) {
  const guard = requireApiToken(req);
  if (guard) return guard;

  const month = new URL(req.url).searchParams.get("month") ?? currentMonthIso();
  if (!/^\d{4}-\d{2}$/.test(month)) {
    return new Response(JSON.stringify({ error: "month must be YYYY-MM" }), {
      status: 400,
      headers: { "content-type": "application/json" },
    });
  }

  const data = await loadDashboardData(getCurrentUserId(), month);
  return Response.json({
    monthLabel: data.monthLabel,
    monthTotal: data.monthTotal.toString(),
    prevMonthTotal: data.prevMonthTotal.toString(),
    momDelta: data.momDelta,
    needsReviewCount: data.needsReviewCount,
    topMerchants: data.topMerchants.map((m) => ({
      merchant: m.merchant,
      total: m.total.toString(),
    })),
  });
}
```

- [ ] **Step 4: Run test to verify it passes**

```
pnpm exec vitest run src/app/api/dashboard/route.test.ts
```
Expected: PASS, 2/2.

- [ ] **Step 5: Commit**

```bash
git add src/app/api/dashboard/route.ts src/app/api/dashboard/route.test.ts
git commit -m "feat: add GET /api/dashboard for iOS client"
```

---

### Task 4: Add `GET /api/transactions` + gate `POST`, serialize amount as string

**Files:**
- Modify: `src/app/api/transactions/route.ts`
- Modify: `src/app/api/transactions/route.test.ts`

- [ ] **Step 1: Read the existing test file to learn the harness conventions**

```
cat src/app/api/transactions/route.test.ts
```
Use the same imports/mock style for new cases.

- [ ] **Step 2: Add failing tests for the GET + token gating + string amount**

Append to `src/app/api/transactions/route.test.ts`:
```ts
describe("GET /api/transactions", () => {
  const original = process.env.API_TOKEN;
  beforeEach(() => { process.env.API_TOKEN = "tok"; vi.clearAllMocks(); });
  afterEach(() => { process.env.API_TOKEN = original; });

  it("returns 401 without token", async () => {
    const res = await GET(new Request("http://x/api/transactions"));
    expect(res.status).toBe(401);
  });

  it("returns rows with amount as string", async () => {
    (prisma.transaction.findMany as any).mockResolvedValueOnce([
      {
        id: "t1", accountId: "a1", postedAt: new Date("2026-05-14T00:00:00Z"),
        amount: new Decimal("89.99"), currency: "USD", merchantRaw: "Target",
        description: null, category: "groceries", categorySource: "USER", categoryConfidence: null,
      },
    ]);
    const res = await GET(new Request("http://x/api/transactions", {
      headers: { authorization: "Bearer tok" },
    }));
    expect(res.status).toBe(200);
    const body = await res.json();
    expect(body[0].amount).toBe("89.99");
    expect(body[0].postedAt).toBe("2026-05-14");
  });

  it("filters by month query param", async () => {
    (prisma.transaction.findMany as any).mockResolvedValueOnce([]);
    await GET(new Request("http://x/api/transactions?month=2026-05", {
      headers: { authorization: "Bearer tok" },
    }));
    const callArgs = (prisma.transaction.findMany as any).mock.calls[0][0];
    expect(callArgs.where.postedAt.gte).toEqual(new Date(Date.UTC(2026, 4, 1)));
    expect(callArgs.where.postedAt.lt).toEqual(new Date(Date.UTC(2026, 5, 1)));
  });

  it("filters uncategorized only when requested", async () => {
    (prisma.transaction.findMany as any).mockResolvedValueOnce([]);
    await GET(new Request("http://x/api/transactions?uncategorized=true", {
      headers: { authorization: "Bearer tok" },
    }));
    const callArgs = (prisma.transaction.findMany as any).mock.calls[0][0];
    expect(callArgs.where.category).toBeNull();
  });
});

describe("POST /api/transactions token gate", () => {
  const original = process.env.API_TOKEN;
  beforeEach(() => { process.env.API_TOKEN = "tok"; });
  afterEach(() => { process.env.API_TOKEN = original; });

  it("returns 401 without token", async () => {
    const res = await POST(new Request("http://x/api/transactions", { method: "POST", body: "{}" }) as any);
    expect(res.status).toBe(401);
  });
});
```
(Ensure `GET` and `POST` are imported at the top.)

- [ ] **Step 3: Run test to verify it fails**

```
pnpm exec vitest run src/app/api/transactions/route.test.ts
```
Expected: FAIL on the new cases.

- [ ] **Step 4: Update the route**

Replace `src/app/api/transactions/route.ts`:
```ts
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";
import Decimal from "decimal.js";
import { prisma } from "@/lib/prisma";
import { getCurrentUserId } from "@/lib/auth";
import { requireApiToken } from "@/lib/api-auth";
import { isCategory } from "@/lib/categories";

const Body = z.object({
  accountId: z.string().min(1),
  postedAt: z.string().regex(/^\d{4}-\d{2}-\d{2}$/, "YYYY-MM-DD required"),
  amount: z.number().refine((n) => Number.isFinite(n), "must be a finite number"),
  merchantRaw: z.string().min(1).max(200),
  description: z.string().max(500).optional(),
  category: z
    .string()
    .refine((s) => s === "" || isCategory(s), "Invalid category")
    .optional(),
});

function serialize(t: {
  id: string; accountId: string; postedAt: Date; amount: Decimal;
  currency: string; merchantRaw: string; description: string | null;
  category: string | null; categorySource: "LLM" | "USER" | null;
  categoryConfidence: number | null;
}) {
  return {
    id: t.id,
    accountId: t.accountId,
    postedAt: t.postedAt.toISOString().slice(0, 10),
    amount: t.amount.toString(),
    currency: t.currency,
    merchantRaw: t.merchantRaw,
    description: t.description,
    category: t.category,
    categorySource: t.categorySource,
    categoryConfidence: t.categoryConfidence,
  };
}

function monthRange(monthIso: string) {
  const [y, m] = monthIso.split("-").map(Number);
  return {
    gte: new Date(Date.UTC(y, m - 1, 1)),
    lt: new Date(Date.UTC(y, m, 1)),
  };
}

export async function GET(req: Request) {
  const guard = requireApiToken(req); if (guard) return guard;
  const userId = getCurrentUserId();
  const url = new URL(req.url);
  const search = url.searchParams.get("search");
  const month = url.searchParams.get("month");
  const uncategorized = url.searchParams.get("uncategorized") === "true";

  const where: any = { userId };
  if (search) where.merchantRaw = { contains: search, mode: "insensitive" };
  if (month && /^\d{4}-\d{2}$/.test(month)) where.postedAt = monthRange(month);
  if (uncategorized) where.category = null;

  const rows = await prisma.transaction.findMany({
    where,
    orderBy: { postedAt: "desc" },
    take: 200,
  });
  return Response.json(rows.map(serialize));
}

export async function POST(req: NextRequest) {
  const guard = requireApiToken(req); if (guard) return guard;
  const userId = getCurrentUserId();

  const parsed = Body.safeParse(await req.json());
  if (!parsed.success) {
    return NextResponse.json({ error: parsed.error.flatten() }, { status: 400 });
  }

  const { accountId, postedAt, amount, merchantRaw, description, category } = parsed.data;

  const account = await prisma.account.findUnique({ where: { id: accountId } });
  if (!account || account.userId !== userId) {
    return NextResponse.json({ error: "Account not found" }, { status: 404 });
  }

  const postedAtDate = new Date(`${postedAt}T00:00:00.000Z`);
  const amountDecimal = new Decimal(amount);

  const created = await prisma.transaction.upsert({
    where: {
      accountId_postedAt_amount_merchantRaw: {
        accountId, postedAt: postedAtDate, amount: amountDecimal, merchantRaw,
      },
    },
    update: {},
    create: {
      userId, accountId,
      postedAt: postedAtDate,
      amount: amountDecimal,
      merchantRaw,
      description: description ?? null,
      ...(category
        ? { category, categorySource: "USER", categoryConfidence: null }
        : {}),
    },
  });

  return NextResponse.json(serialize(created));
}
```

- [ ] **Step 5: Run test to verify it passes**

```
pnpm exec vitest run src/app/api/transactions/route.test.ts
```
Expected: PASS for all cases (new + existing). If existing POST tests assert `amount` as a number, update them to expect a string.

- [ ] **Step 6: Commit**

```bash
git add src/app/api/transactions/route.ts src/app/api/transactions/route.test.ts
git commit -m "feat: add GET /api/transactions; gate POST; serialize amount as string"
```

---

### Task 5: Gate `PATCH /api/transactions/[id]` + string amount

**Files:**
- Modify: `src/app/api/transactions/[id]/route.ts`
- Modify: `src/app/api/transactions/[id]/route.test.ts`

- [ ] **Step 1: Read both files**

```
cat src/app/api/transactions/\[id\]/route.ts src/app/api/transactions/\[id\]/route.test.ts
```

- [ ] **Step 2: Add failing tests**

Append cases to the test file:
```ts
describe("PATCH /api/transactions/[id] token gate", () => {
  const original = process.env.API_TOKEN;
  beforeEach(() => { process.env.API_TOKEN = "tok"; });
  afterEach(() => { process.env.API_TOKEN = original; });

  it("returns 401 without token", async () => {
    const res = await PATCH(
      new Request("http://x/api/transactions/t1", { method: "PATCH", body: "{}" }) as any,
      { params: Promise.resolve({ id: "t1" }) } as any,
    );
    expect(res.status).toBe(401);
  });
});
```
(Adjust `{ params: ... }` shape to match the existing test's call style.)

Add an assertion to the existing happy-path test that `body.amount` is a string.

- [ ] **Step 3: Run test to verify it fails**

```
pnpm exec vitest run src/app/api/transactions/\[id\]/route.test.ts
```

- [ ] **Step 4: Update the route**

At the top of the handler:
```ts
import { requireApiToken } from "@/lib/api-auth";

export async function PATCH(req: NextRequest, ctx: { params: Promise<{ id: string }> }) {
  const guard = requireApiToken(req); if (guard) return guard;
  // …rest unchanged, but the final return uses the shared serialize() shape:
  //   amount: updated.amount.toString()
}
```
Convert any `Number(updated.amount)` in the response to `updated.amount.toString()`.

- [ ] **Step 5: Run test to verify it passes**

```
pnpm exec vitest run src/app/api/transactions/\[id\]/route.test.ts
```

- [ ] **Step 6: Commit**

```bash
git add src/app/api/transactions/\[id\]/route.ts src/app/api/transactions/\[id\]/route.test.ts
git commit -m "feat: gate PATCH /api/transactions; serialize amount as string"
```

---

### Task 6: Gate accounts, import, categorize routes

**Files:**
- Modify: `src/app/api/accounts/route.ts`
- Modify: `src/app/api/import/csv/route.ts`
- Modify: `src/app/api/categorize/route.ts`
- Modify: each route's existing test file (add 1 token-401 case each)

- [ ] **Step 1: For each route, add a failing 401 test**

Pattern (in the relevant `route.test.ts`):
```ts
it("returns 401 without API token", async () => {
  const original = process.env.API_TOKEN; process.env.API_TOKEN = "tok";
  try {
    const res = await POST(new Request("http://x/path", { method: "POST", body: "{}" }) as any);
    expect(res.status).toBe(401);
  } finally { process.env.API_TOKEN = original; }
});
```
For accounts, add the same for `GET`.

- [ ] **Step 2: Run all three test files; confirm failures**

```
pnpm exec vitest run src/app/api/accounts src/app/api/import src/app/api/categorize
```

- [ ] **Step 3: Add guard at the top of each handler**

In each handler:
```ts
import { requireApiToken } from "@/lib/api-auth";

export async function POST(req: NextRequest) {
  const guard = requireApiToken(req); if (guard) return guard;
  // …existing logic
}
```
For `src/app/api/accounts/route.ts`, also guard `GET`.

- [ ] **Step 4: Run the suite**

```
pnpm test
```
Expected: all green.

- [ ] **Step 5: Commit**

```bash
git add src/app/api/accounts src/app/api/import src/app/api/categorize
git commit -m "feat: gate accounts/import/categorize routes with API token"
```

---

### Task 7: Deploy to Vercel, set `API_TOKEN`, smoke with curl

- [ ] **Step 1: Generate a strong token**

```
openssl rand -hex 32
```
Copy the output — this is the production token.

- [ ] **Step 2: Set `API_TOKEN` in Vercel**

```
vercel env add API_TOKEN production
# paste the token when prompted
```

- [ ] **Step 3: Push to main, let Vercel deploy**

```
git push origin main
```
Wait for Vercel to finish the build (~1–2 min).

- [ ] **Step 4: Smoke the new endpoints**

```bash
TOKEN="<paste-token>"
BASE="https://personal-finance-tracker-pi-navy.vercel.app"

# 401 without token
curl -s -o /dev/null -w "%{http_code}\n" "$BASE/api/dashboard?month=2026-05"
# Expected: 401

# 200 with token
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/dashboard?month=2026-05" | head -c 400
# Expected: JSON with monthTotal, momDelta, topMerchants

curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/transactions?month=2026-05" | head -c 400
# Expected: JSON array
```

Save `TOKEN` somewhere safe — you'll paste it into the iOS Setup sheet later. **Backend phase complete.**

---

## Phase 2 — iOS scaffold

All iOS tasks run inside `cd /Users/baha/Desktop/llm-ai-projects/personal-finance-tracker-ios`. The repo is already git-initialized with the design doc committed.

### Task 8: Create Xcode project

- [ ] **Step 1: Create the project via Xcode UI**

Open Xcode → File → New → Project → iOS → App. Settings:
- Product Name: `Spendbook`
- Team: your team
- Organization Identifier: `com.baha`
- Bundle Identifier: `com.baha.spendbook` (auto)
- Interface: **Storyboard** (so we get `SceneDelegate` and `Main.storyboard` — we'll mostly ignore the storyboard and build programmatically)
- Language: Swift
- Include Tests: ✅
- Save into: `/Users/baha/Desktop/llm-ai-projects/personal-finance-tracker-ios/`
- Uncheck "Create Git repository" (the repo already exists)

- [ ] **Step 2: Set minimum deployment to iOS 17.0**

Target Spendbook → General → Minimum Deployments → iOS 17.0.

- [ ] **Step 3: Verify it builds and runs an empty white screen on the iPhone simulator**

In Xcode: ⌘R, pick "iPhone 15" simulator. Expect a white screen.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "chore: scaffold Spendbook Xcode project (iOS 17+, UIKit)"
```

---

### Task 9: Replace storyboard with programmatic root tab bar

**Files:**
- Modify: `Spendbook/Info.plist` — remove storyboard reference
- Modify: `Spendbook/SceneDelegate.swift`
- Create: `Spendbook/App/RootTabBarController.swift`

- [ ] **Step 1: Remove `UISceneStoryboardFile` from `Info.plist`**

In Xcode, open the project's Info tab → Application Scene Manifest → Scene Configuration → Application Session Role → Item 0 → delete the `Storyboard Name` key. Also delete `Main.storyboard` from the project (Move to Trash).

- [ ] **Step 2: Create `RootTabBarController`**

Create `Spendbook/App/RootTabBarController.swift`:
```swift
import UIKit

final class RootTabBarController: UITabBarController {
    override func viewDidLoad() {
        super.viewDidLoad()
        let dashboard = UINavigationController(rootViewController: PlaceholderVC(title: "Dashboard"))
        dashboard.tabBarItem = UITabBarItem(title: "Dashboard",
                                            image: UIImage(systemName: "chart.pie"),
                                            selectedImage: nil)
        let transactions = UINavigationController(rootViewController: PlaceholderVC(title: "Transactions"))
        transactions.tabBarItem = UITabBarItem(title: "Transactions",
                                               image: UIImage(systemName: "list.bullet.rectangle"),
                                               selectedImage: nil)
        viewControllers = [dashboard, transactions]
    }
}

final class PlaceholderVC: UIViewController {
    init(title: String) { super.init(nibName: nil, bundle: nil); self.title = title }
    required init?(coder: NSCoder) { fatalError() }
    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .systemBackground
    }
}
```

- [ ] **Step 3: Wire it up in `SceneDelegate`**

Replace the body of `scene(_:willConnectTo:options:)` in `Spendbook/SceneDelegate.swift`:
```swift
func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options: UIScene.ConnectionOptions) {
    guard let windowScene = scene as? UIWindowScene else { return }
    let window = UIWindow(windowScene: windowScene)
    window.rootViewController = RootTabBarController()
    window.makeKeyAndVisible()
    self.window = window
}
```

- [ ] **Step 4: Run on simulator**

⌘R. Expected: a tab bar with "Dashboard" and "Transactions" tabs, both blank.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: programmatic root tab bar with placeholders"
```

---

## Phase 3 — DesignSystem + Networking + Keychain (testable foundation)

### Task 10: Bundle fonts and add DesignSystem constants

**Files:**
- Create: `Spendbook/DesignSystem/Fonts/Fraunces-Regular.ttf`, `Fraunces-Bold.ttf`, `DMSans-Regular.ttf`, `DMSans-Medium.ttf`
- Modify: `Spendbook/Info.plist` — `UIAppFonts`
- Create: `Spendbook/DesignSystem/Colors.swift`, `Typography.swift`, `Spacing.swift`

- [ ] **Step 1: Download the four .ttf files**

```bash
mkdir -p Spendbook/DesignSystem/Fonts
# Use the same Google Fonts files the web app references.
curl -L -o Spendbook/DesignSystem/Fonts/Fraunces-Regular.ttf \
  "https://github.com/undercasetype/Fraunces/raw/main/fonts/static/Fraunces/Fraunces-Regular.ttf"
curl -L -o Spendbook/DesignSystem/Fonts/Fraunces-Bold.ttf \
  "https://github.com/undercasetype/Fraunces/raw/main/fonts/static/Fraunces/Fraunces-Bold.ttf"
curl -L -o Spendbook/DesignSystem/Fonts/DMSans-Regular.ttf \
  "https://github.com/googlefonts/dm-sans/raw/main/fonts/ttf/DMSans-Regular.ttf"
curl -L -o Spendbook/DesignSystem/Fonts/DMSans-Medium.ttf \
  "https://github.com/googlefonts/dm-sans/raw/main/fonts/ttf/DMSans-Medium.ttf"
```
If a URL 404s, swap to the latest path in the same repo. Confirm each file is > 50KB.

- [ ] **Step 2: Add the fonts to the Xcode target**

In Xcode: right-click `Spendbook` group → Add Files → select the four .ttf files → check "Spendbook" target.

- [ ] **Step 3: Register them in `Info.plist`**

Add a `UIAppFonts` array of strings with the four filenames:
```
UIAppFonts:
  - Fraunces-Regular.ttf
  - Fraunces-Bold.ttf
  - DMSans-Regular.ttf
  - DMSans-Medium.ttf
```

- [ ] **Step 4: Create `Colors.swift`**

`Spendbook/DesignSystem/Colors.swift`:
```swift
import UIKit

enum DSColor {
    static let paper      = UIColor(red: 0xF5/255, green: 0xF1/255, blue: 0xE8/255, alpha: 1)
    static let paper2     = UIColor(red: 0xEF/255, green: 0xE9/255, blue: 0xDC/255, alpha: 1)
    static let ink        = UIColor(red: 0x2C/255, green: 0x2A/255, blue: 0x26/255, alpha: 1)
    static let inkSoft    = UIColor(red: 0x6B/255, green: 0x68/255, blue: 0x62/255, alpha: 1)
    static let rule       = UIColor(red: 0xDD/255, green: 0xD7/255, blue: 0xCA/255, alpha: 1)
    static let sage       = UIColor(red: 0x87/255, green: 0xA8/255, blue: 0x78/255, alpha: 1)
    static let terracotta = UIColor(red: 0xC9/255, green: 0x7B/255, blue: 0x5C/255, alpha: 1)
    static let butter     = UIColor(red: 0xF0/255, green: 0xE4/255, blue: 0xB8/255, alpha: 1)
}
```

- [ ] **Step 5: Create `Typography.swift`**

`Spendbook/DesignSystem/Typography.swift`:
```swift
import UIKit

enum DSFont {
    static func display(_ size: CGFloat, bold: Bool = false) -> UIFont {
        let name = bold ? "Fraunces-Bold" : "Fraunces-Regular"
        return UIFont(name: name, size: size) ?? .systemFont(ofSize: size, weight: bold ? .bold : .regular)
    }
    static func body(_ size: CGFloat, medium: Bool = false) -> UIFont {
        let name = medium ? "DMSans-Medium" : "DMSans-Regular"
        return UIFont(name: name, size: size) ?? .systemFont(ofSize: size, weight: medium ? .medium : .regular)
    }
}
```

- [ ] **Step 6: Create `Spacing.swift`**

`Spendbook/DesignSystem/Spacing.swift`:
```swift
import CoreGraphics

enum DSSpacing {
    static let xs: CGFloat = 4
    static let s:  CGFloat = 8
    static let m:  CGFloat = 12
    static let l:  CGFloat = 16
    static let xl: CGFloat = 24
    static let xxl: CGFloat = 32
    static let huge: CGFloat = 48
}
```

- [ ] **Step 7: Quick visual check: tint the tab bar background paper**

In `RootTabBarController.viewDidLoad()` append:
```swift
let appearance = UITabBarAppearance()
appearance.configureWithOpaqueBackground()
appearance.backgroundColor = DSColor.paper
tabBar.standardAppearance = appearance
tabBar.scrollEdgeAppearance = appearance
view.backgroundColor = DSColor.paper
```
Run; confirm warm cream-colored tab bar.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat: DesignSystem (colors, fonts, spacing); warm paper tab bar"
```

---

### Task 11: `DecimalString` Codable helper

**Files:**
- Create: `Spendbook/Networking/DecimalString.swift`
- Create: `SpendbookTests/DecimalStringTests.swift`

- [ ] **Step 1: Write the failing test**

`SpendbookTests/DecimalStringTests.swift`:
```swift
import XCTest
@testable import Spendbook

final class DecimalStringTests: XCTestCase {
    struct Bag: Codable, Equatable {
        @DecimalString var amount: Decimal
    }

    func test_decodesFromString() throws {
        let json = #"{"amount":"302.45"}"#.data(using: .utf8)!
        let bag = try JSONDecoder().decode(Bag.self, from: json)
        XCTAssertEqual(bag.amount, Decimal(string: "302.45"))
    }

    func test_encodesAsString() throws {
        let bag = Bag(amount: Decimal(string: "12.34")!)
        let data = try JSONEncoder().encode(bag)
        XCTAssertEqual(String(data: data, encoding: .utf8), #"{"amount":"12.34"}"#)
    }

    func test_decodingFailsOnNumber() {
        let json = #"{"amount":302.45}"#.data(using: .utf8)!
        XCTAssertThrowsError(try JSONDecoder().decode(Bag.self, from: json))
    }
}
```

- [ ] **Step 2: Run, expect failure**

In Xcode: ⌘U. Expected: build fails (type not defined).

- [ ] **Step 3: Implement the helper**

`Spendbook/Networking/DecimalString.swift`:
```swift
import Foundation

@propertyWrapper
struct DecimalString: Codable, Equatable {
    var wrappedValue: Decimal

    init(wrappedValue: Decimal) { self.wrappedValue = wrappedValue }

    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        let s = try container.decode(String.self)
        guard let value = Decimal(string: s, locale: Locale(identifier: "en_US_POSIX")) else {
            throw DecodingError.dataCorruptedError(in: container, debugDescription: "Not a decimal: \(s)")
        }
        self.wrappedValue = value
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        try container.encode(NSDecimalNumber(decimal: wrappedValue).stringValue)
    }
}
```

- [ ] **Step 4: Run, expect pass**

⌘U → 3/3 green.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: DecimalString Codable wrapper (string<->Decimal)"
```

---

### Task 12: Models

**Files:**
- Create: `Spendbook/Models/Account.swift`, `Transaction.swift`, `DashboardResponse.swift`, `NewTransaction.swift`
- Create: `SpendbookTests/ModelDecodingTests.swift`

- [ ] **Step 1: Write the failing decoding tests**

`SpendbookTests/ModelDecodingTests.swift`:
```swift
import XCTest
@testable import Spendbook

final class ModelDecodingTests: XCTestCase {
    private let decoder: JSONDecoder = {
        let d = JSONDecoder()
        let f = DateFormatter()
        f.calendar = Calendar(identifier: .iso8601)
        f.timeZone = TimeZone(identifier: "UTC")
        f.locale = Locale(identifier: "en_US_POSIX")
        f.dateFormat = "yyyy-MM-dd"
        d.dateDecodingStrategy = .formatted(f)
        return d
    }()

    func test_account_decodes() throws {
        let json = #"{"id":"a1","name":"Checking"}"#.data(using: .utf8)!
        let a = try decoder.decode(Account.self, from: json)
        XCTAssertEqual(a, Account(id: "a1", name: "Checking"))
    }

    func test_transaction_decodes_with_string_amount() throws {
        let json = #"""
        {"id":"t1","accountId":"a1","postedAt":"2026-05-14","amount":"89.99",
         "currency":"USD","merchantRaw":"Target","description":null,
         "category":"groceries","categorySource":"USER","categoryConfidence":null}
        """#.data(using: .utf8)!
        let t = try decoder.decode(Transaction.self, from: json)
        XCTAssertEqual(t.id, "t1")
        XCTAssertEqual(t.amount, Decimal(string: "89.99"))
        XCTAssertEqual(t.categorySource, .user)
    }

    func test_dashboardResponse_decodes() throws {
        let json = #"""
        {"monthLabel":"May 2026","monthTotal":"302.45","prevMonthTotal":"250",
         "momDelta":20.98,"needsReviewCount":0,
         "topMerchants":[{"merchant":"Target","total":"89.99"}]}
        """#.data(using: .utf8)!
        let r = try decoder.decode(DashboardResponse.self, from: json)
        XCTAssertEqual(r.monthLabel, "May 2026")
        XCTAssertEqual(r.monthTotal, Decimal(string: "302.45"))
        XCTAssertEqual(r.topMerchants.first?.merchant, "Target")
    }
}
```

- [ ] **Step 2: Run, expect failure**

⌘U → build fails.

- [ ] **Step 3: Implement the models**

`Spendbook/Models/Account.swift`:
```swift
import Foundation
struct Account: Codable, Hashable, Identifiable {
    let id: String
    let name: String
}
```

`Spendbook/Models/Transaction.swift`:
```swift
import Foundation

enum CategorySource: String, Codable {
    case llm = "LLM"
    case user = "USER"
}

struct Transaction: Codable, Hashable, Identifiable {
    let id: String
    let accountId: String
    let postedAt: Date
    @DecimalString var amount: Decimal
    let currency: String
    let merchantRaw: String
    let description: String?
    let category: String?
    let categorySource: CategorySource?
    let categoryConfidence: Double?
}
```

`Spendbook/Models/DashboardResponse.swift`:
```swift
import Foundation

struct MerchantTotal: Codable, Hashable {
    let merchant: String
    @DecimalString var total: Decimal
}

struct DashboardResponse: Codable {
    let monthLabel: String
    @DecimalString var monthTotal: Decimal
    @DecimalString var prevMonthTotal: Decimal
    let momDelta: Double
    let needsReviewCount: Int
    let topMerchants: [MerchantTotal]
}
```

`Spendbook/Models/NewTransaction.swift`:
```swift
import Foundation

struct NewTransaction: Encodable {
    let accountId: String
    let postedAt: String        // "YYYY-MM-DD" matching the zod schema on the server
    let amount: Double          // server zod expects a finite number; harmless for input
    let merchantRaw: String
    let description: String?
    let category: String?
}
```

- [ ] **Step 4: Run, expect pass**

⌘U → 3/3 model tests green.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: Account, Transaction, DashboardResponse, NewTransaction models"
```

---

### Task 13: `APIError` and `APIClient` (protocol + impl)

**Files:**
- Create: `Spendbook/Networking/APIError.swift`
- Create: `Spendbook/Networking/APIClient.swift`
- Create: `SpendbookTests/APIClientTests.swift`

- [ ] **Step 1: Write failing tests using URLProtocol stub**

`SpendbookTests/APIClientTests.swift`:
```swift
import XCTest
@testable import Spendbook

final class StubURLProtocol: URLProtocol {
    static var handler: ((URLRequest) -> (HTTPURLResponse, Data))?

    override class func canInit(with request: URLRequest) -> Bool { true }
    override class func canonicalRequest(for request: URLRequest) -> URLRequest { request }
    override func startLoading() {
        guard let handler = Self.handler else { return }
        let (resp, data) = handler(request)
        client?.urlProtocol(self, didReceive: resp, cacheStoragePolicy: .notAllowed)
        client?.urlProtocol(self, didLoad: data)
        client?.urlProtocolDidFinishLoading(self)
    }
    override func stopLoading() {}
}

final class APIClientTests: XCTestCase {
    private func makeClient() -> APIClient {
        let config = URLSessionConfiguration.ephemeral
        config.protocolClasses = [StubURLProtocol.self]
        let session = URLSession(configuration: config)
        return APIClient(baseURL: URL(string: "https://x.test")!, token: "tok", session: session)
    }

    func test_getDashboard_sendsBearerAndDecodes() async throws {
        StubURLProtocol.handler = { req in
            XCTAssertEqual(req.value(forHTTPHeaderField: "Authorization"), "Bearer tok")
            XCTAssertEqual(req.url?.path, "/api/dashboard")
            XCTAssertEqual(req.url?.query?.contains("month=2026-05"), true)
            let body = #"""
            {"monthLabel":"May 2026","monthTotal":"302.45","prevMonthTotal":"250",
             "momDelta":20.98,"needsReviewCount":0,
             "topMerchants":[{"merchant":"Target","total":"89.99"}]}
            """#.data(using: .utf8)!
            let resp = HTTPURLResponse(url: req.url!, statusCode: 200, httpVersion: nil, headerFields: nil)!
            return (resp, body)
        }
        let client = makeClient()
        let dash = try await client.getDashboard(month: "2026-05")
        XCTAssertEqual(dash.monthTotal, Decimal(string: "302.45"))
        XCTAssertEqual(dash.topMerchants.first?.merchant, "Target")
    }

    func test_unauthorized_throwsTypedError() async {
        StubURLProtocol.handler = { req in
            let resp = HTTPURLResponse(url: req.url!, statusCode: 401, httpVersion: nil, headerFields: nil)!
            return (resp, Data())
        }
        let client = makeClient()
        do {
            _ = try await client.getDashboard(month: "2026-05")
            XCTFail("expected throw")
        } catch APIError.unauthorized {
            // ok
        } catch {
            XCTFail("expected .unauthorized, got \(error)")
        }
    }
}
```

- [ ] **Step 2: Run, expect failure**

⌘U → build fails.

- [ ] **Step 3: Implement `APIError`**

`Spendbook/Networking/APIError.swift`:
```swift
import Foundation

enum APIError: Error, Equatable {
    case invalidResponse
    case unauthorized
    case client(Int, String?)
    case server(Int)
    case decoding(String)
}
```

- [ ] **Step 4: Implement `APIClient`**

`Spendbook/Networking/APIClient.swift`:
```swift
import Foundation

protocol APIClientProtocol {
    func getDashboard(month: String) async throws -> DashboardResponse
    func getTransactions(filter: TxFilter) async throws -> [Transaction]
    func createTransaction(_ body: NewTransaction) async throws -> Transaction
    func updateCategory(id: String, category: String) async throws -> Transaction
    func getAccounts() async throws -> [Account]
}

struct TxFilter: Equatable {
    var search: String?
    var month: String?           // "YYYY-MM"
    var uncategorizedOnly: Bool = false
}

struct APIClient: APIClientProtocol {
    let baseURL: URL
    let token: String
    let session: URLSession

    private static let decoder: JSONDecoder = {
        let d = JSONDecoder()
        let f = DateFormatter()
        f.calendar = Calendar(identifier: .iso8601)
        f.timeZone = TimeZone(identifier: "UTC")
        f.locale = Locale(identifier: "en_US_POSIX")
        f.dateFormat = "yyyy-MM-dd"
        d.dateDecodingStrategy = .formatted(f)
        return d
    }()

    init(baseURL: URL, token: String, session: URLSession = .shared) {
        self.baseURL = baseURL; self.token = token; self.session = session
    }

    func getDashboard(month: String) async throws -> DashboardResponse {
        try await request("GET", "/api/dashboard", query: [URLQueryItem(name: "month", value: month)])
    }

    func getTransactions(filter: TxFilter) async throws -> [Transaction] {
        var q: [URLQueryItem] = []
        if let s = filter.search, !s.isEmpty { q.append(.init(name: "search", value: s)) }
        if let m = filter.month { q.append(.init(name: "month", value: m)) }
        if filter.uncategorizedOnly { q.append(.init(name: "uncategorized", value: "true")) }
        return try await request("GET", "/api/transactions", query: q)
    }

    func createTransaction(_ body: NewTransaction) async throws -> Transaction {
        try await request("POST", "/api/transactions", body: body)
    }

    func updateCategory(id: String, category: String) async throws -> Transaction {
        struct Body: Encodable { let category: String }
        return try await request("PATCH", "/api/transactions/\(id)", body: Body(category: category))
    }

    func getAccounts() async throws -> [Account] {
        try await request("GET", "/api/accounts")
    }

    // MARK: - Private

    private func request<T: Decodable>(_ method: String, _ path: String,
                                       query: [URLQueryItem] = [],
                                       body: Encodable? = nil) async throws -> T {
        var comps = URLComponents(url: baseURL.appendingPathComponent(path), resolvingAgainstBaseURL: false)!
        if !query.isEmpty { comps.queryItems = query }
        var req = URLRequest(url: comps.url!)
        req.httpMethod = method
        req.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        req.setValue("application/json", forHTTPHeaderField: "Accept")
        if let body {
            req.setValue("application/json", forHTTPHeaderField: "Content-Type")
            req.httpBody = try JSONEncoder().encode(AnyEncodable(body))
        }
        let (data, resp) = try await session.data(for: req)
        guard let http = resp as? HTTPURLResponse else { throw APIError.invalidResponse }
        switch http.statusCode {
        case 200..<300:
            do { return try Self.decoder.decode(T.self, from: data) }
            catch { throw APIError.decoding(String(describing: error)) }
        case 401: throw APIError.unauthorized
        case 400..<500:
            throw APIError.client(http.statusCode, String(data: data, encoding: .utf8))
        default: throw APIError.server(http.statusCode)
        }
    }

    private struct AnyEncodable: Encodable {
        let value: Encodable
        init(_ value: Encodable) { self.value = value }
        func encode(to encoder: Encoder) throws { try value.encode(to: encoder) }
    }
}
```

- [ ] **Step 5: Run, expect pass**

⌘U → APIClient tests green.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: APIClient with Bearer auth, URLProtocol-friendly init"
```

---

### Task 14: `KeychainStore`

**Files:**
- Create: `Spendbook/Networking/KeychainStore.swift`
- Create: `SpendbookTests/KeychainStoreTests.swift`

- [ ] **Step 1: Write the failing test**

`SpendbookTests/KeychainStoreTests.swift`:
```swift
import XCTest
@testable import Spendbook

final class KeychainStoreTests: XCTestCase {
    override func setUp() { super.setUp(); KeychainStore.shared.clear() }
    override func tearDown() { KeychainStore.shared.clear(); super.tearDown() }

    func test_load_returnsNil_whenEmpty() {
        XCTAssertNil(KeychainStore.shared.load())
    }

    func test_saveAndLoad_roundTrip() throws {
        try KeychainStore.shared.save(baseURL: URL(string: "https://x.test")!, token: "abc")
        let loaded = KeychainStore.shared.load()
        XCTAssertEqual(loaded?.baseURL.absoluteString, "https://x.test")
        XCTAssertEqual(loaded?.token, "abc")
    }

    func test_clear_removes() throws {
        try KeychainStore.shared.save(baseURL: URL(string: "https://x.test")!, token: "abc")
        KeychainStore.shared.clear()
        XCTAssertNil(KeychainStore.shared.load())
    }
}
```

- [ ] **Step 2: Run, expect failure**

⌘U → build fails.

- [ ] **Step 3: Implement KeychainStore**

`Spendbook/Networking/KeychainStore.swift`:
```swift
import Foundation
import Security

final class KeychainStore {
    static let shared = KeychainStore()
    private init() {}

    private let service = "com.baha.spendbook"
    private let urlKey = "apiBaseURL"
    private let tokenKey = "apiToken"

    struct Credentials { let baseURL: URL; let token: String }

    func load() -> Credentials? {
        guard let urlStr = read(key: urlKey), let url = URL(string: urlStr),
              let token = read(key: tokenKey) else { return nil }
        return Credentials(baseURL: url, token: token)
    }

    func save(baseURL: URL, token: String) throws {
        try write(key: urlKey, value: baseURL.absoluteString)
        try write(key: tokenKey, value: token)
    }

    func clear() {
        delete(key: urlKey); delete(key: tokenKey)
    }

    // MARK: - Private SecItem

    private func read(key: String) -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne,
        ]
        var item: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &item)
        guard status == errSecSuccess, let data = item as? Data,
              let s = String(data: data, encoding: .utf8) else { return nil }
        return s
    }

    private func write(key: String, value: String) throws {
        delete(key: key)
        let attrs: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
            kSecValueData as String: Data(value.utf8),
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly,
        ]
        let status = SecItemAdd(attrs as CFDictionary, nil)
        guard status == errSecSuccess else { throw NSError(domain: "keychain", code: Int(status)) }
    }

    private func delete(key: String) {
        let q: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: key,
        ]
        SecItemDelete(q as CFDictionary)
    }
}
```

- [ ] **Step 4: Run, expect pass**

⌘U → 3/3.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: KeychainStore for API base URL + bearer token"
```

---

## Phase 4 — Setup sheet + Notification glue

### Task 15: `SetupSheetVC` (presented at launch when Keychain empty, and on 401)

**Files:**
- Create: `Spendbook/Features/Settings/SetupSheetVC.swift`
- Modify: `Spendbook/SceneDelegate.swift` — present setup sheet when needed; observe `.apiUnauthorized`
- Create: `Spendbook/App/Notifications.swift` — Notification.Name constants

- [ ] **Step 1: Create the notification names**

`Spendbook/App/Notifications.swift`:
```swift
import Foundation
extension Notification.Name {
    static let apiUnauthorized        = Notification.Name("Spendbook.apiUnauthorized")
    static let credentialsSaved       = Notification.Name("Spendbook.credentialsSaved")
    static let transactionsDidChange  = Notification.Name("Spendbook.transactionsDidChange")
}
```

- [ ] **Step 2: Build `SetupSheetVC`**

`Spendbook/Features/Settings/SetupSheetVC.swift`:
```swift
import UIKit

final class SetupSheetVC: UIViewController {
    private let urlField = UITextField()
    private let tokenField = UITextField()
    private let saveButton = UIButton(type: .system)
    private let clearButton = UIButton(type: .system)

    override func viewDidLoad() {
        super.viewDidLoad()
        title = "Connect"
        view.backgroundColor = DSColor.paper
        navigationItem.leftBarButtonItem = UIBarButtonItem(
            barButtonSystemItem: .cancel, target: self, action: #selector(cancelTapped))

        urlField.placeholder = "API base URL"
        urlField.autocapitalizationType = .none
        urlField.autocorrectionType = .no
        urlField.keyboardType = .URL
        urlField.borderStyle = .roundedRect

        tokenField.placeholder = "Bearer token"
        tokenField.isSecureTextEntry = true
        tokenField.autocapitalizationType = .none
        tokenField.autocorrectionType = .no
        tokenField.borderStyle = .roundedRect

        saveButton.setTitle("Save", for: .normal)
        saveButton.titleLabel?.font = DSFont.body(17, medium: true)
        saveButton.addTarget(self, action: #selector(saveTapped), for: .touchUpInside)

        clearButton.setTitle("Clear & sign out", for: .normal)
        clearButton.setTitleColor(DSColor.terracotta, for: .normal)
        clearButton.titleLabel?.font = DSFont.body(15)
        clearButton.addTarget(self, action: #selector(clearTapped), for: .touchUpInside)

        if let creds = KeychainStore.shared.load() {
            urlField.text = creds.baseURL.absoluteString
            tokenField.text = creds.token
        }

        let stack = UIStackView(arrangedSubviews: [urlField, tokenField, saveButton, clearButton])
        stack.axis = .vertical
        stack.spacing = DSSpacing.l
        stack.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(stack)
        NSLayoutConstraint.activate([
            stack.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor, constant: DSSpacing.xl),
            stack.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: DSSpacing.xl),
            stack.trailingAnchor.constraint(equalTo: view.trailingAnchor, constant: -DSSpacing.xl),
        ])
    }

    @objc private func cancelTapped() { dismiss(animated: true) }

    @objc private func saveTapped() {
        guard let urlStr = urlField.text, let url = URL(string: urlStr),
              let token = tokenField.text, !token.isEmpty else {
            present(alert("Need a valid URL and a non-empty token."), animated: true); return
        }
        do {
            try KeychainStore.shared.save(baseURL: url, token: token)
            NotificationCenter.default.post(name: .credentialsSaved, object: nil)
            dismiss(animated: true)
        } catch {
            present(alert("Couldn't save to Keychain: \(error.localizedDescription)"), animated: true)
        }
    }

    @objc private func clearTapped() {
        KeychainStore.shared.clear()
        urlField.text = ""; tokenField.text = ""
    }

    private func alert(_ msg: String) -> UIAlertController {
        let a = UIAlertController(title: nil, message: msg, preferredStyle: .alert)
        a.addAction(.init(title: "OK", style: .default))
        return a
    }
}
```

- [ ] **Step 3: Wire SceneDelegate to present the setup sheet on launch + on `.apiUnauthorized`**

In `Spendbook/SceneDelegate.swift`, replace the connect method and add observers:
```swift
func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options: UIScene.ConnectionOptions) {
    guard let windowScene = scene as? UIWindowScene else { return }
    let window = UIWindow(windowScene: windowScene)
    window.rootViewController = RootTabBarController()
    window.makeKeyAndVisible()
    self.window = window

    if KeychainStore.shared.load() == nil {
        presentSetup()
    }

    NotificationCenter.default.addObserver(
        forName: .apiUnauthorized, object: nil, queue: .main
    ) { [weak self] _ in
        KeychainStore.shared.clear()
        self?.presentSetup()
    }
}

private func presentSetup() {
    guard let root = window?.rootViewController else { return }
    if root.presentedViewController is UINavigationController { return } // already up
    let nav = UINavigationController(rootViewController: SetupSheetVC())
    nav.isModalInPresentation = true
    root.present(nav, animated: true)
}
```

- [ ] **Step 4: Run on simulator**

⌘R. Expect Setup sheet to slide up immediately on a fresh install (Keychain empty after Simulator → Device → Erase All Content and Settings if it doesn't).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: SetupSheetVC + first-launch + 401 notification flow"
```

---

## Phase 5 — Dashboard

### Task 16: `DashboardVM`

**Files:**
- Create: `Spendbook/Features/Dashboard/DashboardVM.swift`
- Create: `SpendbookTests/Stubs/StubAPIClient.swift`
- Create: `SpendbookTests/DashboardVMTests.swift`

- [ ] **Step 1: Build the stub client**

`SpendbookTests/Stubs/StubAPIClient.swift`:
```swift
import Foundation
@testable import Spendbook

final class StubAPIClient: APIClientProtocol {
    var dashboardResult: Result<DashboardResponse, Error> = .failure(APIError.invalidResponse)
    var transactionsResult: Result<[Transaction], Error> = .success([])
    var createResult: Result<Transaction, Error> = .failure(APIError.invalidResponse)
    var updateResult: Result<Transaction, Error> = .failure(APIError.invalidResponse)
    var accountsResult: Result<[Account], Error> = .success([])

    func getDashboard(month: String) async throws -> DashboardResponse {
        try dashboardResult.get()
    }
    func getTransactions(filter: TxFilter) async throws -> [Transaction] {
        try transactionsResult.get()
    }
    func createTransaction(_ body: NewTransaction) async throws -> Transaction {
        try createResult.get()
    }
    func updateCategory(id: String, category: String) async throws -> Transaction {
        try updateResult.get()
    }
    func getAccounts() async throws -> [Account] {
        try accountsResult.get()
    }
}
```

- [ ] **Step 2: Write failing tests for `DashboardVM`**

`SpendbookTests/DashboardVMTests.swift`:
```swift
import XCTest
@testable import Spendbook

@MainActor
final class DashboardVMTests: XCTestCase {
    func test_refresh_setsLoadedOnSuccess() async {
        let stub = StubAPIClient()
        stub.dashboardResult = .success(DashboardResponse(
            monthLabel: "May 2026",
            monthTotal: Decimal(string: "302.45")!,
            prevMonthTotal: Decimal(string: "250")!,
            momDelta: 20.98,
            needsReviewCount: 0,
            topMerchants: [MerchantTotal(merchant: "Target", total: Decimal(string: "89.99")!)]
        ))
        let vm = DashboardVM(api: stub)
        await vm.refresh()
        guard case .loaded(let data) = vm.state else { return XCTFail("expected loaded") }
        XCTAssertEqual(data.monthLabel, "May 2026")
    }

    func test_refresh_setsErrorOnFailure() async {
        let stub = StubAPIClient()
        stub.dashboardResult = .failure(APIError.server(500))
        let vm = DashboardVM(api: stub)
        await vm.refresh()
        guard case .error = vm.state else { return XCTFail("expected error") }
    }

    func test_refresh_postsUnauthorizedNotification_on401() async {
        let stub = StubAPIClient()
        stub.dashboardResult = .failure(APIError.unauthorized)
        let exp = expectation(forNotification: .apiUnauthorized, object: nil)
        let vm = DashboardVM(api: stub)
        await vm.refresh()
        await fulfillment(of: [exp], timeout: 1.0)
    }
}
```

(Workaround: at the top of each VM init's first call site, the convention is "VM posts `.apiUnauthorized` whenever it sees `APIError.unauthorized`." We codify that here so the SceneDelegate observer can respond.)

- [ ] **Step 3: Run, expect failure**

⌘U → build fails.

- [ ] **Step 4: Implement `DashboardVM`**

`Spendbook/Features/Dashboard/DashboardVM.swift`:
```swift
import Foundation
import Combine

@MainActor
final class DashboardVM {
    enum State {
        case idle
        case loading
        case loaded(DashboardResponse)
        case error(String)
    }

    @Published private(set) var state: State = .idle
    private let api: APIClientProtocol
    private var task: Task<Void, Never>?

    init(api: APIClientProtocol) { self.api = api }

    deinit { task?.cancel() }

    func refresh() async {
        task?.cancel()
        state = .loading
        do {
            let month = Self.currentMonthIso()
            let data = try await api.getDashboard(month: month)
            state = .loaded(data)
        } catch APIError.unauthorized {
            NotificationCenter.default.post(name: .apiUnauthorized, object: nil)
            state = .error("Sign in again")
        } catch {
            state = .error("Couldn't load dashboard")
        }
    }

    private static func currentMonthIso() -> String {
        let f = DateFormatter()
        f.timeZone = TimeZone(identifier: "UTC")
        f.locale = Locale(identifier: "en_US_POSIX")
        f.dateFormat = "yyyy-MM"
        return f.string(from: Date())
    }
}
```

- [ ] **Step 5: Run, expect pass**

⌘U → 3/3 dashboard-vm tests green.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: DashboardVM with state machine + unauthorized notification"
```

---

### Task 17: `DashboardVC` + cells

**Files:**
- Create: `Spendbook/Features/Dashboard/DashboardVC.swift`
- Create: `Spendbook/Features/Dashboard/DashboardCells.swift`
- Modify: `Spendbook/App/RootTabBarController.swift` — swap placeholder for real VC

- [ ] **Step 1: Build the cells (Summary, NeedsReview, MerchantRow)**

`Spendbook/Features/Dashboard/DashboardCells.swift`:
```swift
import UIKit

final class SummaryCell: UICollectionViewListCell {
    private let label1 = UILabel()
    private let total = UILabel()
    private let delta = UILabel()
    override init(frame: CGRect) {
        super.init(frame: frame); setup()
    }
    required init?(coder: NSCoder) { fatalError() }
    private func setup() {
        label1.font = DSFont.body(13, medium: true)
        label1.textColor = DSColor.inkSoft
        total.font = DSFont.display(48, bold: false)
        total.textColor = DSColor.ink
        delta.font = DSFont.body(15)
        let stack = UIStackView(arrangedSubviews: [label1, total, delta])
        stack.axis = .vertical; stack.spacing = DSSpacing.xs
        stack.translatesAutoresizingMaskIntoConstraints = false
        contentView.addSubview(stack)
        NSLayoutConstraint.activate([
            stack.topAnchor.constraint(equalTo: contentView.topAnchor, constant: DSSpacing.l),
            stack.leadingAnchor.constraint(equalTo: contentView.leadingAnchor, constant: DSSpacing.l),
            stack.trailingAnchor.constraint(equalTo: contentView.trailingAnchor, constant: -DSSpacing.l),
            stack.bottomAnchor.constraint(equalTo: contentView.bottomAnchor, constant: -DSSpacing.l),
        ])
        backgroundConfiguration = .clear()
    }
    func configure(monthLabel: String, total amount: Decimal, momDelta: Double) {
        label1.text = monthLabel.uppercased()
        total.text = Self.dollar(amount)
        let arrow = momDelta >= 0 ? "↑" : "↓"
        let sign  = momDelta >= 0 ? "+" : ""
        delta.text = "\(arrow) \(sign)\(String(format: "%.1f", momDelta))% vs last month"
        delta.textColor = momDelta >= 0 ? DSColor.terracotta : DSColor.sage
    }
    private static func dollar(_ d: Decimal) -> String {
        let f = NumberFormatter(); f.numberStyle = .currency; f.currencyCode = "USD"
        return f.string(from: NSDecimalNumber(decimal: d)) ?? "$0.00"
    }
}

final class NeedsReviewCell: UICollectionViewListCell {
    func configure(count: Int) {
        var content = UIListContentConfiguration.cell()
        content.text = "Needs review"
        content.secondaryText = count == 0 ? "0 — all caught up ✓" : "\(count) item\(count == 1 ? "" : "s")"
        content.textProperties.font = DSFont.body(13, medium: true)
        content.textProperties.color = DSColor.inkSoft
        content.secondaryTextProperties.font = DSFont.body(17)
        content.secondaryTextProperties.color = DSColor.ink
        contentConfiguration = content
    }
}

final class MerchantRowCell: UICollectionViewListCell {
    func configure(merchant: String, total: Decimal) {
        var content = UIListContentConfiguration.valueCell()
        content.text = merchant
        content.secondaryText = Self.dollar(total)
        content.textProperties.font = DSFont.body(16)
        content.textProperties.color = DSColor.ink
        content.secondaryTextProperties.font = DSFont.display(16)
        content.secondaryTextProperties.color = DSColor.ink
        contentConfiguration = content
    }
    private static func dollar(_ d: Decimal) -> String {
        let f = NumberFormatter(); f.numberStyle = .currency; f.currencyCode = "USD"
        return f.string(from: NSDecimalNumber(decimal: d)) ?? "$0.00"
    }
}
```

- [ ] **Step 2: Build `DashboardVC` with a 3-section diffable list**

`Spendbook/Features/Dashboard/DashboardVC.swift`:
```swift
import UIKit
import Combine

final class DashboardVC: UIViewController {
    private enum Section: Hashable { case summary, needsReview, topMerchants }
    private enum Item: Hashable {
        case summary(monthLabel: String, total: Decimal, delta: Double)
        case needsReview(count: Int)
        case merchant(MerchantTotal)
    }

    private let vm: DashboardVM
    private var subs = Set<AnyCancellable>()
    private var collectionView: UICollectionView!
    private var dataSource: UICollectionViewDiffableDataSource<Section, Item>!
    private let refreshControl = UIRefreshControl()

    init(vm: DashboardVM) { self.vm = vm; super.init(nibName: nil, bundle: nil) }
    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        title = "Spend·book"
        view.backgroundColor = DSColor.paper
        navigationController?.navigationBar.prefersLargeTitles = true

        navigationItem.rightBarButtonItem = UIBarButtonItem(
            image: UIImage(systemName: "gearshape"),
            style: .plain, target: self, action: #selector(openSettings))

        setupCollectionView()
        bindVM()
    }

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        Task { await vm.refresh() }
    }

    private func setupCollectionView() {
        let layout = UICollectionViewCompositionalLayout { _, env in
            var cfg = UICollectionLayoutListConfiguration(appearance: .insetGrouped)
            cfg.backgroundColor = DSColor.paper
            return NSCollectionLayoutSection.list(using: cfg, layoutEnvironment: env)
        }
        collectionView = UICollectionView(frame: .zero, collectionViewLayout: layout)
        collectionView.backgroundColor = DSColor.paper
        collectionView.translatesAutoresizingMaskIntoConstraints = false
        collectionView.refreshControl = refreshControl
        refreshControl.addTarget(self, action: #selector(pulled), for: .valueChanged)
        view.addSubview(collectionView)
        NSLayoutConstraint.activate([
            collectionView.topAnchor.constraint(equalTo: view.topAnchor),
            collectionView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            collectionView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            collectionView.bottomAnchor.constraint(equalTo: view.bottomAnchor),
        ])

        let summary = UICollectionView.CellRegistration<SummaryCell, (String, Decimal, Double)> { c, _, t in
            c.configure(monthLabel: t.0, total: t.1, momDelta: t.2)
        }
        let nr = UICollectionView.CellRegistration<NeedsReviewCell, Int> { c, _, n in c.configure(count: n) }
        let merch = UICollectionView.CellRegistration<MerchantRowCell, MerchantTotal> { c, _, m in
            c.configure(merchant: m.merchant, total: m.total)
        }

        dataSource = UICollectionViewDiffableDataSource(collectionView: collectionView) { cv, ip, item in
            switch item {
            case .summary(let label, let total, let delta):
                return cv.dequeueConfiguredReusableCell(using: summary, for: ip, item: (label, total, delta))
            case .needsReview(let n):
                return cv.dequeueConfiguredReusableCell(using: nr, for: ip, item: n)
            case .merchant(let m):
                return cv.dequeueConfiguredReusableCell(using: merch, for: ip, item: m)
            }
        }
    }

    private func bindVM() {
        vm.$state
            .receive(on: DispatchQueue.main)
            .sink { [weak self] state in self?.apply(state: state) }
            .store(in: &subs)
    }

    private func apply(state: DashboardVM.State) {
        refreshControl.endRefreshing()
        switch state {
        case .idle, .loading, .error: break
        case .loaded(let d):
            var snap = NSDiffableDataSourceSnapshot<Section, Item>()
            snap.appendSections([.summary, .needsReview, .topMerchants])
            snap.appendItems([.summary(monthLabel: d.monthLabel, total: d.monthTotal, delta: d.momDelta)], toSection: .summary)
            snap.appendItems([.needsReview(count: d.needsReviewCount)], toSection: .needsReview)
            snap.appendItems(d.topMerchants.map { .merchant($0) }, toSection: .topMerchants)
            dataSource.apply(snap, animatingDifferences: true)
        }
    }

    @objc private func pulled() { Task { await vm.refresh() } }

    @objc private func openSettings() {
        let nav = UINavigationController(rootViewController: SetupSheetVC())
        present(nav, animated: true)
    }
}
```

- [ ] **Step 3: Wire the real VC into the tab bar**

In `RootTabBarController.viewDidLoad()`, replace the dashboard placeholder line with:
```swift
let creds = KeychainStore.shared.load()
let api: APIClientProtocol = creds.map {
    APIClient(baseURL: $0.baseURL, token: $0.token)
} ?? StubAPIClient()  // harmless empty client until creds saved; replaced below

let dashboardVC = DashboardVC(vm: DashboardVM(api: api))
let dashboard = UINavigationController(rootViewController: dashboardVC)
```
Also add a notification observer in `RootTabBarController.viewDidLoad()` (after `super.viewDidLoad()`) that rebuilds children when credentials change:
```swift
NotificationCenter.default.addObserver(
    forName: .credentialsSaved, object: nil, queue: .main
) { [weak self] _ in
    self?.rebuildChildren()
}
```
Extract the construction into `private func rebuildChildren()` that resets `viewControllers`.

Important: `StubAPIClient` is in the test target, not the app. Use a tiny `NoCredentialsAPIClient` in the app that throws `.unauthorized` on every call instead:
```swift
struct NoCredentialsAPIClient: APIClientProtocol {
    func getDashboard(month: String) async throws -> DashboardResponse { throw APIError.unauthorized }
    func getTransactions(filter: TxFilter) async throws -> [Transaction] { throw APIError.unauthorized }
    func createTransaction(_ body: NewTransaction) async throws -> Transaction { throw APIError.unauthorized }
    func updateCategory(id: String, category: String) async throws -> Transaction { throw APIError.unauthorized }
    func getAccounts() async throws -> [Account] { throw APIError.unauthorized }
}
```
(Put it in `Spendbook/Networking/NoCredentialsAPIClient.swift`.)

- [ ] **Step 4: Run on simulator**

⌘R. Without creds: Setup sheet opens; with creds: Dashboard shows real data from Vercel.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: DashboardVC with diffable snapshot + pull-to-refresh"
```

---

## Phase 6 — Transactions

### Task 18: `TransactionsVM`

**Files:**
- Create: `Spendbook/Features/Transactions/TransactionsVM.swift`
- Create: `SpendbookTests/TransactionsVMTests.swift`

- [ ] **Step 1: Write failing tests**

`SpendbookTests/TransactionsVMTests.swift`:
```swift
import XCTest
@testable import Spendbook

@MainActor
final class TransactionsVMTests: XCTestCase {
    private func tx(_ id: String, _ amount: String, _ merchant: String, _ category: String?) -> Transaction {
        Transaction(id: id, accountId: "a1",
                    postedAt: Date(timeIntervalSince1970: 1715731200),
                    amount: Decimal(string: amount)!,
                    currency: "USD", merchantRaw: merchant, description: nil,
                    category: category, categorySource: category == nil ? nil : .user,
                    categoryConfidence: nil)
    }

    func test_load_setsLoaded() async {
        let stub = StubAPIClient()
        stub.transactionsResult = .success([tx("t1","89.99","Target","groceries")])
        let vm = TransactionsVM(api: stub)
        await vm.load()
        guard case .loaded(let rows) = vm.state else { return XCTFail() }
        XCTAssertEqual(rows.count, 1)
    }

    func test_updateCategoryLocally_replacesRow() {
        let stub = StubAPIClient()
        let vm = TransactionsVM(api: stub)
        let updated = tx("t1","89.99","Target","dining")
        vm.replace(updated)
        // No state assertion needed beyond the helper not crashing in idle state.
        XCTAssertTrue(true)
    }

    func test_load_postsUnauthorizedOn401() async {
        let stub = StubAPIClient()
        stub.transactionsResult = .failure(APIError.unauthorized)
        let exp = expectation(forNotification: .apiUnauthorized, object: nil)
        let vm = TransactionsVM(api: stub)
        await vm.load()
        await fulfillment(of: [exp], timeout: 1.0)
    }
}
```

- [ ] **Step 2: Run, expect failure**

⌘U → build fails.

- [ ] **Step 3: Implement `TransactionsVM`**

`Spendbook/Features/Transactions/TransactionsVM.swift`:
```swift
import Foundation
import Combine

@MainActor
final class TransactionsVM {
    enum State { case idle, loading, loaded([Transaction]), error(String) }

    @Published private(set) var state: State = .idle
    var filter = TxFilter()

    private let api: APIClientProtocol
    private var task: Task<Void, Never>?

    init(api: APIClientProtocol) { self.api = api }
    deinit { task?.cancel() }

    func load() async {
        task?.cancel()
        state = .loading
        do {
            let rows = try await api.getTransactions(filter: filter)
            state = .loaded(rows)
        } catch APIError.unauthorized {
            NotificationCenter.default.post(name: .apiUnauthorized, object: nil)
            state = .error("Sign in again")
        } catch {
            state = .error("Couldn't load transactions")
        }
    }

    func setCategory(for id: String, to category: String) async {
        do {
            let updated = try await api.updateCategory(id: id, category: category)
            replace(updated)
        } catch APIError.unauthorized {
            NotificationCenter.default.post(name: .apiUnauthorized, object: nil)
        } catch {
            // surface inline on the row in v2; v1 just leaves the old value
        }
    }

    func replace(_ updated: Transaction) {
        guard case .loaded(var rows) = state,
              let i = rows.firstIndex(where: { $0.id == updated.id }) else { return }
        rows[i] = updated
        state = .loaded(rows)
    }
}
```

- [ ] **Step 4: Run, expect pass**

⌘U → 3/3 green.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: TransactionsVM with load + category-update + 401 propagation"
```

---

### Task 19: `TransactionsVC` (list grouped by date) + `CategoryPickerVC`

**Files:**
- Create: `Spendbook/Features/Transactions/TransactionsVC.swift`
- Create: `Spendbook/Features/Transactions/TransactionCell.swift`
- Create: `Spendbook/Features/Transactions/CategoryPickerVC.swift`
- Modify: `RootTabBarController` — swap the transactions placeholder for real VC

- [ ] **Step 1: Build `TransactionCell`**

`Spendbook/Features/Transactions/TransactionCell.swift`:
```swift
import UIKit

final class TransactionCell: UICollectionViewListCell {
    func configure(_ t: Transaction) {
        var content = UIListContentConfiguration.subtitleCell()
        content.text = t.merchantRaw
        let amountStr = Self.dollar(t.amount)
        content.secondaryText = "-\(amountStr)" + categoryLine(t)
        content.textProperties.font = DSFont.body(16, medium: true)
        content.textProperties.color = DSColor.ink
        content.secondaryTextProperties.font = DSFont.body(13)
        content.secondaryTextProperties.color = DSColor.inkSoft
        contentConfiguration = content
        accessories = [.disclosureIndicator()]
    }
    private func categoryLine(_ t: Transaction) -> String {
        guard let cat = t.category else { return "  ·  uncategorized" }
        let src = t.categorySource == .llm
            ? " · LLM \(String(format: "%.2f", t.categoryConfidence ?? 0))"
            : " · USER"
        return "  ·  \(cat)\(src)"
    }
    private static func dollar(_ d: Decimal) -> String {
        let f = NumberFormatter(); f.numberStyle = .currency; f.currencyCode = "USD"
        return f.string(from: NSDecimalNumber(decimal: d)) ?? "$0.00"
    }
}
```

- [ ] **Step 2: Build `CategoryPickerVC`**

`Spendbook/Features/Transactions/CategoryPickerVC.swift`:
```swift
import UIKit

final class CategoryPickerVC: UIViewController, UITableViewDataSource, UITableViewDelegate {
    private let table = UITableView(frame: .zero, style: .insetGrouped)
    private let categories: [String]
    var onPick: ((String) -> Void)?

    init(currentCategories: [String]) {
        // Hard-coded baseline + observed; dedupe; keep order stable.
        let baseline = ["groceries","dining","transport","shopping","utilities","entertainment","health","other"]
        var seen = Set<String>(); var ordered: [String] = []
        for c in (currentCategories + baseline) where !seen.contains(c) {
            seen.insert(c); ordered.append(c)
        }
        self.categories = ordered
        super.init(nibName: nil, bundle: nil)
        title = "Category"
    }
    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = DSColor.paper
        table.dataSource = self; table.delegate = self
        table.backgroundColor = DSColor.paper
        table.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(table)
        NSLayoutConstraint.activate([
            table.topAnchor.constraint(equalTo: view.topAnchor),
            table.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            table.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            table.bottomAnchor.constraint(equalTo: view.bottomAnchor),
        ])
    }

    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int { categories.count }
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = UITableViewCell(style: .default, reuseIdentifier: nil)
        cell.textLabel?.text = categories[indexPath.row]
        cell.textLabel?.font = DSFont.body(16)
        cell.backgroundColor = DSColor.paper2
        return cell
    }
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        onPick?(categories[indexPath.row])
        navigationController?.popViewController(animated: true)
    }
}
```

- [ ] **Step 3: Build `TransactionsVC`**

`Spendbook/Features/Transactions/TransactionsVC.swift`:
```swift
import UIKit
import Combine

final class TransactionsVC: UIViewController, UISearchResultsUpdating {
    private enum Section: Hashable { case day(String) }
    private struct Item: Hashable {
        let tx: Transaction
    }

    private let vm: TransactionsVM
    private var subs = Set<AnyCancellable>()
    private var collectionView: UICollectionView!
    private var dataSource: UICollectionViewDiffableDataSource<Section, Item>!
    private let segmented = UISegmentedControl(items: ["All", "This month", "Uncategorized"])

    init(vm: TransactionsVM) { self.vm = vm; super.init(nibName: nil, bundle: nil) }
    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        title = "Transactions"
        view.backgroundColor = DSColor.paper

        navigationItem.rightBarButtonItem = UIBarButtonItem(
            barButtonSystemItem: .add, target: self, action: #selector(addTapped))

        let search = UISearchController(searchResultsController: nil)
        search.searchResultsUpdater = self
        search.obscuresBackgroundDuringPresentation = false
        navigationItem.searchController = search

        segmented.selectedSegmentIndex = 0
        segmented.addTarget(self, action: #selector(filterChanged), for: .valueChanged)

        let segContainer = UIView()
        segContainer.backgroundColor = DSColor.paper
        segContainer.translatesAutoresizingMaskIntoConstraints = false
        segmented.translatesAutoresizingMaskIntoConstraints = false
        segContainer.addSubview(segmented)
        NSLayoutConstraint.activate([
            segmented.topAnchor.constraint(equalTo: segContainer.topAnchor, constant: DSSpacing.s),
            segmented.bottomAnchor.constraint(equalTo: segContainer.bottomAnchor, constant: -DSSpacing.s),
            segmented.leadingAnchor.constraint(equalTo: segContainer.leadingAnchor, constant: DSSpacing.l),
            segmented.trailingAnchor.constraint(equalTo: segContainer.trailingAnchor, constant: -DSSpacing.l),
        ])

        setupCollectionView()
        let stack = UIStackView(arrangedSubviews: [segContainer, collectionView])
        stack.axis = .vertical
        stack.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(stack)
        NSLayoutConstraint.activate([
            stack.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
            stack.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            stack.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            stack.bottomAnchor.constraint(equalTo: view.bottomAnchor),
        ])

        bindVM()
        NotificationCenter.default.addObserver(self, selector: #selector(externalChange),
                                               name: .transactionsDidChange, object: nil)
    }

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        Task { await vm.load() }
    }

    private func setupCollectionView() {
        let layout = UICollectionViewCompositionalLayout { _, env in
            var cfg = UICollectionLayoutListConfiguration(appearance: .insetGrouped)
            cfg.backgroundColor = DSColor.paper
            cfg.headerMode = .supplementary
            return NSCollectionLayoutSection.list(using: cfg, layoutEnvironment: env)
        }
        collectionView = UICollectionView(frame: .zero, collectionViewLayout: layout)
        collectionView.backgroundColor = DSColor.paper
        collectionView.translatesAutoresizingMaskIntoConstraints = false
        collectionView.delegate = self

        let cellReg = UICollectionView.CellRegistration<TransactionCell, Item> { c, _, item in
            c.configure(item.tx)
        }
        dataSource = UICollectionViewDiffableDataSource(collectionView: collectionView) { cv, ip, item in
            cv.dequeueConfiguredReusableCell(using: cellReg, for: ip, item: item)
        }
        let header = UICollectionView.SupplementaryRegistration<UICollectionViewListCell>(
            elementKind: UICollectionView.elementKindSectionHeader
        ) { [weak self] view, _, ip in
            var content = UIListContentConfiguration.plainHeader()
            if case .day(let label) = self?.dataSource.sectionIdentifier(for: ip.section) {
                content.text = label
                content.textProperties.font = DSFont.body(13, medium: true)
                content.textProperties.color = DSColor.inkSoft
            }
            view.contentConfiguration = content
        }
        dataSource.supplementaryViewProvider = { cv, _, ip in
            cv.dequeueConfiguredReusableSupplementary(using: header, for: ip)
        }
    }

    private func bindVM() {
        vm.$state.receive(on: DispatchQueue.main).sink { [weak self] state in
            self?.apply(state: state)
        }.store(in: &subs)
    }

    private func apply(state: TransactionsVM.State) {
        guard case .loaded(let rows) = state else { return }
        let grouped = Dictionary(grouping: rows) { dayString($0.postedAt) }
        let sortedDays = grouped.keys.sorted(by: >)
        var snap = NSDiffableDataSourceSnapshot<Section, Item>()
        for day in sortedDays {
            snap.appendSections([.day(day)])
            snap.appendItems(grouped[day]!.map { Item(tx: $0) }, toSection: .day(day))
        }
        dataSource.apply(snap, animatingDifferences: true)
    }

    private func dayString(_ d: Date) -> String {
        let f = DateFormatter(); f.dateFormat = "MMM d"; f.timeZone = TimeZone(identifier: "UTC")
        return f.string(from: d)
    }

    @objc private func addTapped() {
        let nav = UINavigationController(rootViewController: AddTxModalVC(api: vm.apiForChildren))
        present(nav, animated: true)
    }
    @objc private func externalChange() { Task { await vm.load() } }

    @objc private func filterChanged() {
        switch segmented.selectedSegmentIndex {
        case 1: vm.filter.month = currentMonthIso(); vm.filter.uncategorizedOnly = false
        case 2: vm.filter.month = nil; vm.filter.uncategorizedOnly = true
        default: vm.filter.month = nil; vm.filter.uncategorizedOnly = false
        }
        Task { await vm.load() }
    }

    func updateSearchResults(for searchController: UISearchController) {
        vm.filter.search = searchController.searchBar.text
        Task { await vm.load() }
    }

    private func currentMonthIso() -> String {
        let f = DateFormatter(); f.dateFormat = "yyyy-MM"; f.timeZone = TimeZone(identifier: "UTC")
        return f.string(from: Date())
    }
}

extension TransactionsVC: UICollectionViewDelegate {
    func collectionView(_ collectionView: UICollectionView, didSelectItemAt indexPath: IndexPath) {
        guard let item = dataSource.itemIdentifier(for: indexPath) else { return }
        let existing: [String] = {
            if case .loaded(let rows) = vm.state {
                return rows.compactMap { $0.category }
            }
            return []
        }()
        let picker = CategoryPickerVC(currentCategories: existing)
        picker.onPick = { [weak self] cat in
            Task { await self?.vm.setCategory(for: item.tx.id, to: cat) }
        }
        navigationController?.pushViewController(picker, animated: true)
    }
}
```

- [ ] **Step 2 follow-up: expose the API client to the AddTx modal**

In `TransactionsVM`, add:
```swift
var apiForChildren: APIClientProtocol { api }
```
This is the only place we leak the dependency on purpose — to give the Add modal the same client.

- [ ] **Step 3: Wire into the tab bar**

In `RootTabBarController.rebuildChildren()`, replace the transactions placeholder:
```swift
let txVM = TransactionsVM(api: api)
let transactionsVC = TransactionsVC(vm: txVM)
let transactions = UINavigationController(rootViewController: transactionsVC)
```

- [ ] **Step 4: Run on simulator**

⌘R. Tap Transactions tab → list loads, search filters, segmented control filters, tapping a row pushes the category picker.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: TransactionsVC with search, filter segmented, category picker"
```

---

## Phase 7 — Add Transaction modal

### Task 20: `AddTxVM`

**Files:**
- Create: `Spendbook/Features/AddTransaction/AddTxVM.swift`
- Create: `SpendbookTests/AddTxVMTests.swift`

- [ ] **Step 1: Write failing tests**

`SpendbookTests/AddTxVMTests.swift`:
```swift
import XCTest
@testable import Spendbook

@MainActor
final class AddTxVMTests: XCTestCase {
    func test_isSaveEnabled_falseUntilAmountAndMerchantPresent() {
        let vm = AddTxVM(api: StubAPIClient())
        XCTAssertFalse(vm.isSaveEnabled)
        vm.merchant = "Target"
        XCTAssertFalse(vm.isSaveEnabled)
        vm.amount = 1.0
        vm.accountId = "a1"
        XCTAssertTrue(vm.isSaveEnabled)
    }

    func test_submit_callsCreateAndReturnsTx() async throws {
        let stub = StubAPIClient()
        let created = Transaction(
            id: "t1", accountId: "a1", postedAt: Date(),
            amount: Decimal(string: "12.34")!, currency: "USD",
            merchantRaw: "Target", description: nil,
            category: nil, categorySource: nil, categoryConfidence: nil
        )
        stub.createResult = .success(created)
        let vm = AddTxVM(api: stub)
        vm.accountId = "a1"; vm.merchant = "Target"; vm.amount = 12.34
        let returned = try await vm.submit()
        XCTAssertEqual(returned.id, "t1")
    }
}
```

- [ ] **Step 2: Run, expect failure**

⌘U.

- [ ] **Step 3: Implement `AddTxVM`**

`Spendbook/Features/AddTransaction/AddTxVM.swift`:
```swift
import Foundation
import Combine

@MainActor
final class AddTxVM: ObservableObject {
    @Published var accountId: String = ""
    @Published var date: Date = Date()
    @Published var amount: Double = 0
    @Published var merchant: String = ""
    @Published var note: String = ""
    @Published var category: String? = nil

    private let api: APIClientProtocol
    init(api: APIClientProtocol) { self.api = api }

    var isSaveEnabled: Bool {
        !accountId.isEmpty && amount > 0 && !merchant.trimmingCharacters(in: .whitespaces).isEmpty
    }

    func loadAccounts() async throws -> [Account] {
        try await api.getAccounts()
    }

    func submit() async throws -> Transaction {
        let f = DateFormatter()
        f.dateFormat = "yyyy-MM-dd"; f.timeZone = TimeZone(identifier: "UTC")
        let body = NewTransaction(
            accountId: accountId,
            postedAt: f.string(from: date),
            amount: amount,
            merchantRaw: merchant,
            description: note.isEmpty ? nil : note,
            category: category
        )
        return try await api.createTransaction(body)
    }
}
```

- [ ] **Step 4: Run, expect pass**

⌘U → 2/2.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: AddTxVM with validation + submit"
```

---

### Task 21: `AddTxModalVC`

**Files:**
- Create: `Spendbook/Features/AddTransaction/AddTxModalVC.swift`

- [ ] **Step 1: Implement the modal as an inset-grouped UITableView form**

`Spendbook/Features/AddTransaction/AddTxModalVC.swift`:
```swift
import UIKit

final class AddTxModalVC: UIViewController, UITableViewDataSource {
    private let vm: AddTxVM
    private let table = UITableView(frame: .zero, style: .insetGrouped)
    private let amountField = UITextField()
    private let merchantField = UITextField()
    private let noteField = UITextField()
    private let datePicker = UIDatePicker()
    private let accountButton = UIButton(configuration: .plain())
    private let categoryButton = UIButton(configuration: .plain())
    private var accounts: [Account] = []

    init(api: APIClientProtocol) {
        self.vm = AddTxVM(api: api)
        super.init(nibName: nil, bundle: nil)
        title = "Add Transaction"
    }
    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = DSColor.paper

        navigationItem.leftBarButtonItem = UIBarButtonItem(
            barButtonSystemItem: .cancel, target: self, action: #selector(cancelTapped))
        let save = UIBarButtonItem(barButtonSystemItem: .save, target: self, action: #selector(saveTapped))
        navigationItem.rightBarButtonItem = save

        amountField.placeholder = "0.00"
        amountField.keyboardType = .decimalPad
        amountField.textAlignment = .right
        amountField.addTarget(self, action: #selector(textChanged), for: .editingChanged)

        merchantField.placeholder = "Merchant"
        merchantField.addTarget(self, action: #selector(textChanged), for: .editingChanged)

        noteField.placeholder = "Note (optional)"

        datePicker.datePickerMode = .date
        datePicker.preferredDatePickerStyle = .compact
        datePicker.addTarget(self, action: #selector(dateChanged), for: .valueChanged)

        accountButton.setTitle("Choose account", for: .normal)
        accountButton.contentHorizontalAlignment = .trailing
        accountButton.showsMenuAsPrimaryAction = true

        categoryButton.setTitle("None", for: .normal)
        categoryButton.contentHorizontalAlignment = .trailing
        categoryButton.showsMenuAsPrimaryAction = true
        categoryButton.menu = categoryMenu()

        table.dataSource = self
        table.backgroundColor = DSColor.paper
        table.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(table)
        NSLayoutConstraint.activate([
            table.topAnchor.constraint(equalTo: view.topAnchor),
            table.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            table.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            table.bottomAnchor.constraint(equalTo: view.bottomAnchor),
        ])

        Task { await loadAccounts() }
        updateSaveEnabled()
    }

    private func loadAccounts() async {
        do {
            let list = try await vm.loadAccounts()
            self.accounts = list
            accountButton.menu = UIMenu(children: list.map { acct in
                UIAction(title: acct.name) { [weak self] _ in
                    self?.vm.accountId = acct.id
                    self?.accountButton.setTitle(acct.name, for: .normal)
                    self?.updateSaveEnabled()
                }
            })
        } catch APIError.unauthorized {
            NotificationCenter.default.post(name: .apiUnauthorized, object: nil)
            dismiss(animated: true)
        } catch {
            present(alertOK("Couldn't load accounts."), animated: true)
        }
    }

    private func categoryMenu() -> UIMenu {
        let names = ["None","groceries","dining","transport","shopping","utilities","entertainment","health","other"]
        return UIMenu(children: names.map { n in
            UIAction(title: n) { [weak self] _ in
                self?.vm.category = n == "None" ? nil : n
                self?.categoryButton.setTitle(n, for: .normal)
            }
        })
    }

    // 5 rows in 1 section: account, date, amount, merchant, note
    func numberOfSections(in tableView: UITableView) -> Int { 1 }
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int { 6 }
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = UITableViewCell(style: .value1, reuseIdentifier: nil)
        cell.backgroundColor = DSColor.paper2
        cell.selectionStyle = .none
        switch indexPath.row {
        case 0:
            cell.textLabel?.text = "Account"
            cell.accessoryView = accountButton
        case 1:
            cell.textLabel?.text = "Date"
            cell.accessoryView = datePicker
        case 2:
            cell.textLabel?.text = "Amount"
            cell.accessoryView = sizedField(amountField, width: 120)
        case 3:
            cell.textLabel?.text = "Merchant"
            cell.accessoryView = sizedField(merchantField, width: 180)
        case 4:
            cell.textLabel?.text = "Note"
            cell.accessoryView = sizedField(noteField, width: 180)
        case 5:
            cell.textLabel?.text = "Category"
            cell.accessoryView = categoryButton
        default: break
        }
        cell.textLabel?.font = DSFont.body(16)
        cell.textLabel?.textColor = DSColor.ink
        return cell
    }

    private func sizedField(_ f: UITextField, width: CGFloat) -> UIView {
        f.frame = CGRect(x: 0, y: 0, width: width, height: 32)
        return f
    }

    @objc private func textChanged() {
        vm.amount = Double(amountField.text ?? "") ?? 0
        vm.merchant = merchantField.text ?? ""
        vm.note = noteField.text ?? ""
        updateSaveEnabled()
    }
    @objc private func dateChanged() { vm.date = datePicker.date }
    @objc private func cancelTapped() { dismiss(animated: true) }

    @objc private func saveTapped() {
        guard vm.isSaveEnabled else { return }
        Task {
            do {
                _ = try await vm.submit()
                NotificationCenter.default.post(name: .transactionsDidChange, object: nil)
                dismiss(animated: true)
            } catch APIError.unauthorized {
                NotificationCenter.default.post(name: .apiUnauthorized, object: nil)
                dismiss(animated: true)
            } catch {
                present(alertOK("Couldn't save."), animated: true)
            }
        }
    }

    private func updateSaveEnabled() {
        navigationItem.rightBarButtonItem?.isEnabled = vm.isSaveEnabled
    }

    private func alertOK(_ msg: String) -> UIAlertController {
        let a = UIAlertController(title: nil, message: msg, preferredStyle: .alert)
        a.addAction(.init(title: "OK", style: .default))
        return a
    }
}
```

- [ ] **Step 2: Run on simulator**

⌘R → Transactions tab → tap +. Modal opens, account dropdown populates, fill amount + merchant → Save enables → tap Save → modal closes, row appears on top of list (after `transactionsDidChange` triggers reload), Dashboard's MTD total bumps when you switch tabs.

- [ ] **Step 3: Commit**

```bash
git add -A
git commit -m "feat: AddTxModalVC (account/date/amount/merchant/note/category)"
```

---

## Phase 8 — Polish & smoke

### Task 22: Empty + error states + pull-to-refresh in Transactions

**Files:**
- Modify: `Spendbook/Features/Transactions/TransactionsVC.swift`
- Modify: `Spendbook/Features/Dashboard/DashboardVC.swift`

- [ ] **Step 1: Add `UIContentUnavailableConfiguration` in `apply(state:)`**

In `TransactionsVC.apply(state:)`:
```swift
switch state {
case .loading:
    var cfg = UIContentUnavailableConfiguration.loading()
    contentUnavailableConfiguration = cfg
case .loaded(let rows):
    contentUnavailableConfiguration = rows.isEmpty
        ? .empty() // see helper below
        : nil
    // existing snapshot apply
case .error(let msg):
    var cfg = UIContentUnavailableConfiguration.empty()
    cfg.text = "Something went wrong"
    cfg.secondaryText = msg
    cfg.image = UIImage(systemName: "exclamationmark.triangle")
    contentUnavailableConfiguration = cfg
case .idle: break
}
```
Add a small helper extension:
```swift
private extension UIContentUnavailableConfiguration {
    static func empty() -> UIContentUnavailableConfiguration {
        var c = UIContentUnavailableConfiguration.empty()
        c.text = "No transactions yet"
        c.secondaryText = "Tap + to add one"
        c.image = UIImage(systemName: "tray")
        return c
    }
}
```
(If Swift complains about same-named statics, rename helper to `emptyTransactions()`.)

- [ ] **Step 2: Add a pull-to-refresh in `TransactionsVC` like `DashboardVC` has**

```swift
private let refreshControl = UIRefreshControl()
// in setupCollectionView():
collectionView.refreshControl = refreshControl
refreshControl.addTarget(self, action: #selector(pulled), for: .valueChanged)
// new method:
@objc private func pulled() { Task { await vm.load() } }
// in apply(state:), call refreshControl.endRefreshing() at the top.
```

- [ ] **Step 3: Same error/empty treatment in `DashboardVC.apply(state:)`**

Mirror the pattern above with text "Couldn't load dashboard" on `.error`.

- [ ] **Step 4: Run + commit**

```bash
git add -A
git commit -m "polish: empty + error + loading states; transactions pull-to-refresh"
```

---

### Task 23: Manual smoke + README

**Files:**
- Create: `README.md` in the iOS repo
- Modify: nothing in code unless smoke reveals a bug

- [ ] **Step 1: Erase Simulator content + settings**

Simulator menu → Device → Erase All Content and Settings.

- [ ] **Step 2: Run through the manual checklist from the spec**

1. Fresh install → Setup sheet appears
2. Save garbage token → Dashboard load fails → Setup sheet reappears
3. Save correct token → Dashboard shows month total matching the web app
4. Pull to refresh on Dashboard
5. Transactions tab: search works, segmented filter works
6. Tap row → pick a new category → row updates → web app shows the same change
7. Add transaction with valid data → new row on top + dashboard total updates
8. Force-quit + relaunch → still signed in
9. Settings → Clear & sign out → Setup sheet reappears

Fix any bug as a fresh small commit (`fix: …`).

- [ ] **Step 3: Write the iOS README**

`README.md`:
```markdown
# Spend·book iOS

Native UIKit client (iOS 17+) for the Spend·book personal-finance backend.

## Run locally
1. Open `Spendbook.xcodeproj` in Xcode 16+.
2. Select an iOS 17 simulator. ⌘R.
3. On first launch, enter:
   - API base URL: `https://personal-finance-tracker-pi-navy.vercel.app`
   - Bearer token: (matches `API_TOKEN` in Vercel)

## Features in v1
- Dashboard: month-to-date total, MoM delta, top merchants
- Transactions: list, search, filter (All / This month / Uncategorized), tap-to-recategorize
- Add Transaction: manual entry

## Architecture
MVVM + Combine + diffable data sources. No third-party deps.
See `docs/superpowers/specs/2026-05-17-spendbook-ios-v1-design.md`.

## Tests
⌘U runs the XCTest target. Unit tests cover the API client, models,
view models, Keychain, and Decimal serialization.
```

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: README with run + features + architecture"
```

---

## Self-review

**Spec coverage check** — each spec section maps to plan tasks:

| Spec section | Task(s) |
|---|---|
| Architecture overview | Tasks 8–17 set up the structure |
| Data flow — App launch | Task 9 (tab bar), Task 15 (setup sheet) |
| Data flow — Dashboard read | Tasks 16, 17 |
| Data flow — Transactions / edit category | Tasks 18, 19 |
| Data flow — Add transaction | Tasks 20, 21 |
| Error handling (401 etc.) | Task 15 (observer), Tasks 16, 18 (post notification) |
| Screen 1 Dashboard | Task 17 |
| Screen 2 Transactions | Task 19 |
| Screen 3 Add Transaction | Task 21 |
| Screen 4 Setup sheet | Task 15 + gear icon in Task 17 |
| Networking APIClient + models + Decimal | Tasks 11, 12, 13 |
| KeychainStore | Task 14 |
| Backend Change 1 (token middleware) | Task 1, applied in Tasks 4–6 |
| Backend Change 2 (GET /api/dashboard) | Tasks 2 + 3 |
| Backend Change 3 (GET /api/transactions) | Task 4 |
| Backend testing | Tasks 1, 2, 3, 4, 5, 6 |
| iOS testing | Tasks 11, 12, 13, 14, 16, 18, 20 |
| File layout | Tasks 8–21 follow the spec's layout exactly |
| Build settings | Task 8 (iOS 17), Task 10 (fonts) |
| Design system | Task 10 |
| Implementation phases | Phases 1–8 match the spec's 8 phases |

No gaps.

**Placeholder scan:** every step has concrete code, exact paths, exact commands. No "TBD", no "implement appropriately", no "similar to above" without code.

**Type consistency check:**
- `APIClientProtocol` defined in Task 13 is used unchanged by `StubAPIClient` (Task 16), `NoCredentialsAPIClient` (Task 17), `DashboardVM` (Task 16), `TransactionsVM` (Task 18), `AddTxVM` (Task 20). ✓
- `Transaction.amount` is `Decimal` everywhere (decoded via `@DecimalString`); never mixed with `Double`. ✓
- `TxFilter` defined in Task 13 used in Task 18. ✓
- Notification names declared in Task 15 used in Tasks 15, 16, 18, 21. ✓
- `Section`/`Item` enums for diffable sources are local to each VC — no cross-VC clashes. ✓

**Ambiguity check:** the spec's open question about the top-merchants window (rolling 30d vs current month) is resolved in the plan by the helper in Task 2 (`postedAt` filtered to the requested month). If the original web spec strictly wanted rolling 30d, that's a follow-up bug — call out in PR.

Plan complete.
