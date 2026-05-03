# HealthEngine — CLAUDE.md

> **RULE: Update this file whenever you add, change, or remove any component, API route, DB model, hook, or architectural pattern. This is a living document — keep it current.**

## What this is
A full-stack fitness tracker built with Next.js 15 (App Router), TypeScript, TailwindCSS, Prisma ORM, and iron-session. Users log workouts, calories, and weight. Auth is passwordless (email OTP).

## Stack
- **Framework**: Next.js 15 App Router
- **Language**: TypeScript (strict)
- **Styling**: TailwindCSS + shadcn/ui (v4) — black/white/purple theme with dark mode (`darkMode: 'class'`)
- **UI components**: shadcn/ui — all primitive UI is from shadcn, never hand-rolled
- **Database**: SQLite in dev (`prisma/dev.db`), swap to PostgreSQL for production
- **ORM**: Prisma
- **Sessions**: iron-session (encrypted cookie, 1-week expiry)
- **Email**: Resend (prod) — console mock in dev

## Commands
```bash
npm run dev          # start dev server at localhost:3000
npm run build        # production build
npm run db:migrate   # apply Prisma migrations (creates dev.db on first run)
npm run db:seed      # seed test data (test@example.com)
npm run db:studio    # open Prisma Studio (visual DB browser)
npm run db:generate  # regenerate Prisma client after schema changes
npm run format       # Prettier
npm run lint         # ESLint
```

> **After any `prisma migrate dev`, always stop the dev server and run `npm run db:generate` before restarting.** The Next.js process holds the Prisma query engine binary, which prevents the client from being regenerated while it's running.

## Folder structure
```
src/
  middleware.ts              # Route guard — runs on Edge, checks iron-session cookie
  app/
    page.tsx                 # Root: redirects to /dashboard or /login
    layout.tsx               # Root HTML shell
    globals.css              # Tailwind base + scrollbar + animation keyframes/utilities
    (auth)/login/page.tsx    # 2-step OTP login (client component)
    (dashboard)/
      layout.tsx             # Wraps all dashboard pages with PreferencesProvider + ToastProvider + TopNav
      dashboard/page.tsx     # Server component — fetches raw data, renders DashboardStats + quick actions + GoalSetupCard
      workouts/page.tsx      # Client component — workout calendar + exercise history
      calories/page.tsx      # Client component — log + list calories + CalendarView date selector + per-day totals
      weight/page.tsx        # Client component — log + list weight + trend
      settings/page.tsx      # Client — theme + units; mobile-only Account card with Sign out (desktop sign out remains in TopNav)
    api/
      auth/send-code/        # POST — generate OTP, store in DB, mock-email it
      auth/verify-code/      # POST — validate OTP, create session cookie
      auth/logout/           # POST — destroy session
      goals/                 # GET + PATCH — fetch/upsert per-user goals (UserGoal model)
      workouts-v2/           # GET (?date=YYYY-MM-DD | ?month=YYYY-MM) + POST
      workouts-v2/[id]/      # DELETE + PATCH (update sets on a WorkoutExercise)
      exercises/             # GET (list user's exercises) + POST (create exercise)
      exercise-history/      # GET (?exerciseId=) — per-exercise set history
      calories/              # GET (?date=YYYY-MM-DD | ?month=YYYY-MM | all) + POST (mealName required)
      calories/[id]/         # DELETE
      calories/summary/      # GET (?days=N, default 30, max 90) — per-day aggregated totals for CalorieBarChart
      weight/                # GET + POST
      weight/[id]/           # DELETE
  components/
    nav/TopNav.tsx           # Client — desktop: nav links + sign out; mobile: logo only (sign out is on Settings); primary nav is MobileFloatingNav
    nav/MobileFloatingNav.tsx # Client — sm:hidden; floating bottom tab bar (icons + labels) + safe-area padding
    nav/dashboard-routes.ts  # Shared hrefs/labels for TopNav + mobile tabs
    ui/StatCard.tsx          # Big number stat display — supports trend chip + progress bar
    ui/ToastProvider.tsx     # Context-based toast system (success/error/info) — wraps dashboard layout
    dashboard/
      GoalSetupCard.tsx      # Full-width goals display + inline edit form (client) — unit-aware
      DashboardStats.tsx     # Client component — renders stat cards + suggestions with unit-aware display; receives raw numbers from Server Component dashboard/page.tsx
    charts/
      WeightChart.tsx        # Client — recharts AreaChart; weight trend over last 30 entries; unit-aware
      CalorieBarChart.tsx    # Client — recharts BarChart; 30-day daily calorie history; fetches /api/calories/summary
    forms/CalorieForm.tsx    # Controlled form → POST /api/calories
    forms/WeightForm.tsx     # Controlled form → POST /api/weight
    lists/CalorieList.tsx    # Calorie entry list with ghost delete button + empty state
    lists/WeightList.tsx
    workouts/
      CalendarView.tsx           # Month calendar — highlights workout dates, handles day selection
      WorkoutDayView.tsx         # Shows exercises + sets + volume for a selected date
      AddWorkoutModal.tsx        # Dialog — log a full workout; validates weight/reps/RPE ranges
      AddExerciseModal.tsx       # Dialog — create a new named exercise
      EditWorkoutExerciseModal.tsx # Dialog — edit sets; validates ranges; toast on save/delete
      ExerciseHistoryView.tsx    # Shows historical sets for a chosen exercise
  hooks/
    useToast.ts              # Hook — returns { toast(message, variant) } from ToastContext
    useChartColors.ts        # Hook — reads CSS variables at runtime; re-reads on dark/light class toggle; used by all chart components
  contexts/
    PreferencesContext.tsx   # React context — theme + unit preferences; persisted to localStorage; exposes usePreferences()
  lib/
    prisma.ts                # Singleton PrismaClient (safe for hot reload in dev)
    session.ts               # SessionData type, sessionOptions, getSession() helper
    email.ts                 # sendVerificationEmail() — Resend in prod, console mock in dev
    units.ts                 # Pure unit conversion + formatting utilities (kg↔lbs, g↔oz, kcal/Cal)
  types/index.ts             # Shared TS interfaces (CalorieEntry, Workout, UserGoal, StatTrend…)
prisma/
  schema.prisma              # DB models (see below)
  seed.ts                    # Test data seeder
```

## Auth flow
1. `/login` → user enters email → `POST /api/auth/send-code`
   - Deletes any existing code for that email
   - Generates a random 6-digit code, stores it in `VerificationCode` table with 10-min expiry
   - Calls `sendVerificationEmail()` — in dev this logs the code to the terminal
2. User enters the code → `POST /api/auth/verify-code`
   - Finds code in DB, checks it hasn't expired
   - Deletes the code (one-time use)
   - Upserts the `User` row (creates on first login)
   - Writes an encrypted `iron-session` cookie
3. Subsequent requests → middleware reads the cookie, redirects unauthenticated users to `/login`
4. `POST /api/auth/logout` → destroys the session cookie

## Session architecture
- `src/lib/session.ts` exports `sessionOptions` (plain object, Edge-safe) and `getSession()` (uses `next/headers`, Node.js only — call from API routes and Server Components)
- `src/middleware.ts` uses `getIronSession(request.cookies, sessionOptions)` directly — no `next/headers`, works in Edge runtime
- All API routes call `getSession()` and check `session.isLoggedIn` before touching the DB

## Database models
| Model | Key fields |
|---|---|
| User | id, email, createdAt |
| VerificationCode | email, code, expiresAt |
| CalorieEntry | userId, mealName?, calories, protein?, carbs?, fat? |
| WeightEntry | userId, weight (kg), bodyFat? |
| Exercise | userId, name — a named movement owned by the user |
| Workout | userId, date — a training session on a specific date |
| WorkoutExercise | workoutId, exerciseId — junction: one exercise within a workout |
| WorkoutSet | workoutExerciseId, weight?, reps?, effort? (RPE 1–10) |
| UserGoal | userId (unique), dailyCalories?, weeklyWorkouts?, targetWeight? |

**Workout hierarchy**: `Workout` → `WorkoutExercise[]` → `WorkoutSet[]`

One `Workout` per date per save. A workout contains multiple `WorkoutExercise` entries (one per movement), each with one or more `WorkoutSet` rows tracking weight, reps, and RPE.

**UserGoal**: one row per user (upserted via PATCH `/api/goals`). All fields are optional — only set fields are used for progress bars and suggestions on the dashboard.

## Goals API (`/api/goals`)
- `GET` — returns the current user's `UserGoal` row (or `null`)
- `PATCH` — body: `{ dailyCalories?, weeklyWorkouts?, targetWeight? }` — upserts the row; pass `null` to clear a field

## Workout API (`/api/workouts-v2`)
- `GET ?month=YYYY-MM` — returns array of ISO date strings that have workouts (for calendar highlighting)
- `GET ?date=YYYY-MM-DD` — returns full `Workout[]` with nested `workoutExercises → exercise + sets`
- `POST` — body: `{ date: string, exercises: [{ exerciseId, sets: [{ weight?, reps?, effort? }] }] }` — creates a Workout with all exercises and sets in one transaction
- `DELETE /api/workouts-v2/[id]` — deletes a Workout (cascades to WorkoutExercises and WorkoutSets)
- `PATCH /api/workouts-v2/[id]` — updates sets on a WorkoutExercise (replaces all sets)

## Exercise API
- `GET /api/exercises` — list all exercises for the current user (alphabetical)
- `POST /api/exercises` — body: `{ name: string }` — create a new exercise
- `GET /api/exercise-history?exerciseId=` — returns historical WorkoutSets for an exercise, grouped by workout date

## Toast system
`ToastProvider` wraps the dashboard layout and exposes `{ toast }` via React context. Use `useToast()` in any client component within the dashboard to show notifications.

```ts
import { useToast } from '@/hooks/useToast';

const { toast } = useToast();
toast('Workout saved!', 'success');  // variants: 'success' | 'error' | 'info'
```

- Toasts auto-dismiss after 3.5 s; click to dismiss early
- Rendered in a fixed bottom-right stack with `animate-scale-in`
- No external library — implemented in `src/components/ui/ToastProvider.tsx`

## StatCard component
`StatCard` accepts optional `trend` and `progress` props in addition to the base `title / value / unit / subtitle`.

```ts
// Trend chip (↑/↓ with color coding)
trend?: {
  delta: number;          // positive or negative change
  label: string;          // e.g. 'vs yesterday'
  positiveDirection: 'up' | 'down' | 'neutral';
  // 'up'  → green when delta > 0
  // 'down' → green when delta < 0 (lower is better, e.g. weight)
  // 'neutral' → always gray
}

// Progress bar toward a goal
progress?: {
  current: number;
  goal: number;
  label?: string;   // e.g. 'Daily goal'
}
```

## Preferences system

`PreferencesProvider` (in `src/contexts/PreferencesContext.tsx`) wraps the dashboard layout and exposes preferences + setters via `usePreferences()`.

```ts
import { usePreferences } from '@/contexts/PreferencesContext';

const { preferences, setTheme, setUnits } = usePreferences();
// preferences.theme: 'light' | 'dark' | 'system'
// preferences.units.bodyWeight: 'kg' | 'lbs'
// preferences.units.liftingWeight: 'kg' | 'lbs'
// preferences.units.macros: 'g' | 'oz'
// preferences.units.calories: 'kcal' | 'Cal'

setTheme('system');                            // immediately applies DOM class + writes localStorage
setUnits({ bodyWeight: 'lbs' });              // partial update, persisted to localStorage
```

- Stored in `localStorage` key `healthengine_prefs` (JSON)
- `localStorage.theme` key is also kept in sync for the root layout inline script (FOUC prevention)
- Theme 'system' attaches a `matchMedia` listener so it updates live when OS preference changes
- **All DB values stay in base units** (kg, g, kcal) — only display and form entry convert
- Unit conversion helpers live in `src/lib/units.ts`:
  - `formatWeight(kg, unit)` / `formatWeightWithUnit(kg, unit)` — display
  - `formatMacro(g, unit)` / `formatMacroWithUnit(g, unit)` — display
  - `calorieUnitLabel(unit)` — `'kcal'` or `'Cal'` (no numeric conversion)
  - `weightInputToKg(input, unit)` / `macroInputToG(input, unit)` — form submit
  - `weightMaxForUnit(unit)` / `weightRangeLabel(unit)` — validation
  - `convertWeight(kg, unit)` / `convertMacro(g, unit)` — raw numeric conversion

## Charts system

recharts is installed for all data visualizations. Chart components live in `src/components/charts/`.

**Theme integration**: use `useChartColors()` hook — reads CSS custom properties from `document.documentElement` via `getComputedStyle`, and observes `class` attribute changes so charts update when dark/light mode is toggled.

**SSR guard**: all chart components use a `mounted` state (`useEffect(() => setMounted(true), [])`) so recharts only renders client-side. While unmounted, they render a skeleton (`bg-muted animate-pulse`).

| Component | Chart type | Data source | Page |
|---|---|---|---|
| `WeightChart` | AreaChart | `entries` prop (from weight page state) | Weight page |
| `CalorieBarChart` | BarChart | `GET /api/calories/summary?days=30` | Calories page |

**`/api/calories/summary`**: new route that returns `{ date, calories, protein, carbs, fat }[]` per day for the last N days. Used exclusively by `CalorieBarChart`.

## Dashboard page architecture
`dashboard/page.tsx` is a **Server Component** that fetches all data in a single `Promise.all`, then renders:
1. Header + email
2. `DashboardStats` (Client Component) — receives raw numbers; reads unit preferences; renders three stat cards + smart suggestions
3. Quick Actions — three full-width cards linking to /workouts, /calories, /weight
4. `GoalSetupCard` — full-width, shows goal chips when set; inline 3-col edit form when editing

**Pattern for Server → Client with units:** Server components fetch numbers and pass them as plain props to a `'use client'` component, which calls `usePreferences()` to format for display. Never pass raw Prisma objects — extract and serialize fields explicitly.

## Workout input validation
All set fields are validated before submission in both `AddWorkoutModal` and `EditWorkoutExerciseModal`. Validation ranges adapt to the user's unit preference:
- Weight: 0–500 kg / 0–1102 lbs (uses `weightMaxForUnit` from `src/lib/units.ts`)
- Reps: 1–50
- RPE (effort): 1–10

Inputs also have HTML `min`/`max` attributes for browser-level hints.

## Workout input validation
All set fields are validated before submission in both `AddWorkoutModal` and `EditWorkoutExerciseModal`:
- Weight: 0–500 kg
- Reps: 1–50
- RPE (effort): 1–10

Inputs also have HTML `min`/`max` attributes for browser-level hints.

## shadcn/ui component system

shadcn/ui (v4) is fully installed. **Never hand-roll UI primitives** — always use the installed shadcn components.

### Available components and import paths
```ts
import { Button } from '@/components/ui/button'
// variants: default | destructive | outline | secondary | ghost | link
// sizes: default | sm | lg | icon

import { Card, CardContent, CardHeader, CardTitle, CardDescription, CardFooter } from '@/components/ui/card'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import { Textarea } from '@/components/ui/textarea'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { Badge } from '@/components/ui/badge'
import { Tabs, TabsList, TabsTrigger, TabsContent } from '@/components/ui/tabs'
import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogDescription, DialogFooter } from '@/components/ui/dialog'
import { Sheet, SheetContent, SheetHeader, SheetTitle, SheetTrigger } from '@/components/ui/sheet'
import { Separator } from '@/components/ui/separator'
import { cn } from '@/lib/utils'  // always use cn() for conditional class merging
```

Custom (non-shadcn) components keep PascalCase: `StatCard.tsx`, `ToastProvider.tsx`, `GoalSetupCard.tsx`.

### Styling rules
- **All colors via CSS variables** — use semantic tokens (`bg-background`, `text-foreground`, `text-muted-foreground`, `border-border`, `text-primary`, `bg-muted`, `text-destructive`) not raw Tailwind color values on UI primitives.
- **Never use raw zinc/gray/white/black** on new components — these bypass the theme and break dark mode. Prefer `bg-background`, `bg-muted`, `text-foreground`, etc.
- **Variant mapping** (old → new): `primary` → `default`, `secondary` → `outline`, `danger` → `destructive`, size `md` → omit (default).
- **`cn()` for conditional classes**: always import and use `cn()` from `@/lib/utils` when merging class strings conditionally.
- **Interactive states**: rely on shadcn's built-in `hover:`, `focus-visible:`, `disabled:` styles — don't override them unless strictly necessary.
- **Custom interactive elements** (e.g. plain `<button>` used for small actions): style with `text-muted-foreground hover:text-foreground transition-colors` to stay consistent with the theme.
- **Forms**: always pair `<Label>` with `<Input>` / `<Textarea>` / `<Select>`. Never use unstyled inputs.
- **Dialogs**: use `Dialog` + `DialogContent` for modals. Structure as header / scrollable body / footer — see `AddWorkoutModal.tsx` as the reference pattern.
- **Delete actions**: use `variant="ghost"` with `hover:text-destructive hover:bg-destructive/10` — not `variant="destructive"` for inline list items. Reserve `variant="destructive"` for confirmation dialogs.

## Color system

Colors are defined as CSS variables in `globals.css` and consumed via shadcn semantic tokens. Prefer tokens over raw values.

| Token | Light | Dark | Use |
|---|---|---|---|
| `--primary` / `text-primary` | violet-600 | violet-500 | Buttons, active nav, brand accent |
| `--background` / `bg-background` | white | zinc-950 | Page background |
| `--card` / `bg-card` | white | zinc-900 | Card surfaces |
| `--muted` / `bg-muted` | zinc-100 | zinc-800 | Subtle backgrounds |
| `--muted-foreground` / `text-muted-foreground` | zinc-500 | zinc-400 | Secondary text |
| `--border` / `border-border` | zinc-200 | zinc-800 | All borders |
| `--destructive` / `text-destructive` | red-600 | red-400 | Errors, delete actions |

- **Dark mode**: toggled via `class` strategy. Inline script in `layout.tsx` reads `localStorage.theme` before first paint to prevent flash. Toggle button in `TopNav`.
- **Stat cards**: left accent bar (`w-1.5 bg-primary`), `text-primary` value text at `text-4xl sm:text-6xl font-black`.
- **Raw Tailwind color values** (`violet-600`, `zinc-950`, etc.) are only acceptable in `globals.css` variable definitions and in one-off decorative elements (e.g. a gradient). Everywhere else use semantic tokens.

## Animation system

Three utility classes are defined in `globals.css` and automatically disabled for users with `prefers-reduced-motion: reduce`.

| Class | Keyframe | Duration | Easing | Use |
|---|---|---|---|---|
| `animate-fade-in-up` | opacity 0→1, translateY 10px→0 | 450ms | ease-out-quint | Page headings, stat cards, content entrances |
| `animate-fade-in` | opacity 0→1 | 250ms | ease-out | Context switches (date change, month change) |
| `animate-scale-in` | opacity 0→1, scale 0.97→1 | 300ms | ease-out-quint | Modals, new exercise/set rows appearing |

**Stagger pattern**: use inline `style={{ animationDelay: '120ms' }}` on sibling elements. Dashboard stat cards use 80ms / 180ms / 280ms. Exercise cards in WorkoutDayView use 0 / 60ms / 120ms per card.

**Key-based re-animation**: wrap content in a div with `key={someChangingValue}` to force React to remount and replay the animation when data changes (e.g. `key={date}` in WorkoutDayView, `key={year-month}` in CalendarView).

**Never animate**: layout properties (width, height, top, left) — always use `transform` and `opacity` for GPU acceleration.

## Typography conventions
- **Page headings**: `text-3xl sm:text-5xl font-black tracking-tight leading-none`
- **Section labels**: `text-[10px] font-bold text-muted-foreground uppercase tracking-widest`
- **Stat values**: `text-4xl sm:text-6xl font-black text-primary leading-none tracking-tight`
- **Exercise names in logs**: `text-base font-bold`
- **Data columns headers**: `text-[10px] font-bold text-muted-foreground uppercase tracking-wider`
- **Nav brand**: `font-black text-xl`

## Component rules
- **Mobile-first always**: Every component must work on both desktop and mobile. Use Tailwind responsive prefixes (`sm:`, `md:`) mobile-first — default styles target small screens, larger breakpoints override upward.
- **No fixed-width grids on mobile**: Multi-column grids must stack on mobile. Use `grid-cols-1 sm:grid-cols-N`, never `grid-cols-N` alone for forms or content layouts.
- **Responsive text**: Large display text must scale down. Use `text-2xl sm:text-3xl` patterns — never `text-3xl` or above without a smaller mobile base.
- **Responsive padding**: Cards and containers use `p-4 sm:p-6`, not fixed `p-6` or `p-8` alone.
- **Navigation**: The TopNav hides links on mobile and shows a hamburger menu. Do not add inline nav items that would overflow on small screens.
- **Touch targets**: Interactive elements (buttons, links) must be at least 44px tall on mobile. Use `py-2` or `py-3` on touch controls.
- **`max-w-*` containers**: Never use `max-w-xs` alone as a width constraint on content — wrap with `sm:max-w-xs` so it goes full-width on mobile.
- **Dark mode on all new components**: Use semantic tokens (`bg-background`, `text-foreground`, `border-border`, etc.) so dark mode is automatic. If you must use a raw Tailwind color, always pair it with a `dark:` counterpart. Never use `border-black`, `text-black`, `bg-white`, or `text-gray-*` alone.
- **Server → Client prop serialization**: When a Server Component passes Prisma model data to a Client Component, extract and pass only the needed fields as a plain object — never pass the raw Prisma result (it contains `Date` objects that conflict with TS string types in shared interfaces).

## PWA (Progressive Web App)

The app is configured as a PWA for iOS Safari "Add to Home Screen" and Android Chrome.

**Files:**
- `public/manifest.json` — Web App Manifest (name, icons, theme color, display mode)
- `src/components/ui/SplashScreen.tsx` — Logo splash overlay shown on PWA launch only (detects `display-mode: standalone` / `navigator.standalone`)
- `src/app/layout.tsx` — exports `viewport` (with `viewportFit: 'cover'`) + `metadata.appleWebApp` + `metadata.manifest`

**Key settings:**
- `display: "standalone"` in manifest — removes browser chrome when launched from home screen
- `appleWebApp.statusBarStyle: "black-translucent"` — status bar overlays the app (full-bleed)
- `viewportFit: "cover"` — content extends behind notch / Dynamic Island
- `themeColor: "#7c3aed"` — violet-600, tints browser chrome on Android
- Icons: uses `public/logo.png` — for best results the icon should be 512×512 PNG; iOS crops to a rounded square automatically

**To improve icon quality:** replace `public/logo.png` with a 512×512 version. iOS also benefits from a separate `public/apple-touch-icon.png` at 180×180.

## Switching to PostgreSQL (production)
1. Change `prisma/schema.prisma` datasource provider to `"postgresql"`
2. Set `DATABASE_URL="postgresql://..."` in env
3. Run `npm run db:migrate`

## Enabling Resend (real email)
`resend` is already installed. Set these env vars:
```
RESEND_API_KEY=re_...
RESEND_FROM_EMAIL=noreply@yourdomain.com
```
The `from` address must match a domain verified in the Resend dashboard.

**Domain verification** (add DNS records shown in Resend dashboard → Domains):
- TXT `resend._domainkey` — DKIM signing key
- TXT `@` — SPF record
- CNAME `em.yourdomain.com` — bounce tracking (optional)

`sendVerificationEmail()` auto-detects `RESEND_API_KEY` and switches from the console mock to real sending.

## Environment variables
| Variable | Required | Description |
|---|---|---|
| DATABASE_URL | yes | SQLite file path or Postgres connection string |
| SESSION_SECRET | yes | Min 32 chars, used to encrypt session cookies |
| NEXT_PUBLIC_APP_URL | no | Used for absolute URLs |
| RESEND_API_KEY | prod only | Resend API key (`re_...`) |
| RESEND_FROM_EMAIL | prod only | From address — must match verified Resend domain |
