# Stocky: Calorie Tracker (Spec)

Status: **Draft for review**. Items marked **(assumption)** are my defaults. Please confirm or change them (see [Open Questions](#14-open-questions)).

**Decisions made:** search covers the user's own foods only (no public database); calories required, protein/carbs/fat optional; daily goal is a number the user types; **Google sign-in only**.

## 1. Overview

Stocky is a mobile-first web app (Vue 3 + Vite, in `StockyApp/`) for tracking daily calorie intake, in the spirit of MyFitnessPal. Users sign in, log food into four meals (Breakfast, Lunch, Dinner, Snacks) for any day, and keep a personal food library they can reuse on later days.

### Goals
- Fast daily logging: re-adding a food you've eaten before should take ~2 taps.
- Clear view of calories consumed vs. daily goal.
- All data private per user, stored in Firestore.

### Non-goals (v1)
- Barcode scanning, public/global food database, social features, wearable sync, native apps.
- Exercise tracking, water tracking, recipes (candidates for later, see §13).

## 2. Tech Stack

| Concern | Choice |
|---|---|
| Frontend | Vue 3 (`<script setup>`), Vite, plain JavaScript (matches existing app) |
| Routing | `vue-router` (needed; none currently) |
| State | Pinia **(assumption)**, or simple composables if you prefer fewer deps |
| Auth | Firebase Authentication |
| Database | Cloud Firestore |
| Hosting | Firebase Hosting **(assumption)** |
| PWA | Installable via manifest + service worker (`vite-plugin-pwa`) **(assumption, optional)** |

Firebase config lives in `.env` (`VITE_FIREBASE_*`), never committed.

## 3. Users and Auth

### 3.1 Sign-in method
- Google sign-in only (Firebase Auth `GoogleAuthProvider`). No passwords are stored or managed by the app.

### 3.2 Flows
1. **Log in / sign up**: one "Continue with Google" button on `/login`. First-time users get a `users/{uid}` profile created from their Google profile and go to onboarding (§4). Returning users go to Diary (today).
2. **Log out**: from Profile screen. Clears local state and returns to Login.
3. **Session persistence**: user stays signed in across reloads (Firebase default persistence).

### 3.3 Route guards
- Unauthenticated users can only reach `/login`. Anything else redirects to `/login`.
- Authenticated users visiting `/login` redirect to `/diary`.
- App shows a splash/loading state until Firebase auth state resolves, to avoid flashing the login page.

### 3.4 Errors
Friendly messages for: sign-in popup closed or blocked, network error, account exists with a different credential.

## 4. Onboarding and Profile

Minimal in v1: a one-screen setup after first sign-in. Display name, email and photo come from the Google profile.

- **Daily calorie goal** (number the user types, default 2000; no calculator in v1).
- Optional later: age, sex, height, weight, activity, goal → auto-calculated target (§13).

Profile screen: display name and email (read-only, from Google), daily calorie goal (editable), units (see Open Questions), log out, delete account (with confirmation, deletes all user data).

## 5. Screens and Navigation

Mobile-first, single column, designed at 360–430 px width, usable (centered, max-width ~480 px) on desktop.

**Bottom tab bar** (after login):

| Tab | Route | Purpose |
|---|---|---|
| Diary | `/diary` | Daily log by meal (home) |
| Foods | `/foods` | My food library (browse/add/edit/delete) |
| Profile | `/profile` | Goal, account, logout |

Plus a prominent "+" action to add food from the Diary.

Other routes: `/login`, `/onboarding`, `/diary/add?meal=lunch&date=YYYY-MM-DD` (search/add), `/foods/new`, `/foods/:id/edit`.

### 5.1 Diary (home)
- **Date header** with prev/next arrows and tap-to-open date picker. Defaults to today. Can view and log past and future dates.
- **Summary card**: Goal, Food (consumed), Remaining, with a progress bar/ring. Turns a warning color when over goal.
- **Macros row**: total Protein / Carbs / Fat for the day (if macros are enabled, see Open Questions).
- **Four meal sections**: Breakfast, Lunch, Dinner, Snacks. Each shows:
  - Section calorie total.
  - List of entries (food name, serving info, calories).
  - "Add food" button for that meal.
- **Entry actions**: tap to edit quantity; swipe or menu to delete. Delete shows undo toast.
- Empty state per meal: "Nothing logged yet" + Add food.

### 5.2 Add Food to Meal (search/select)
Opened from a meal's "Add food" button (meal and date preselected).

- **Search box** (top, autofocus) filters the user's food library by name (case-insensitive, substring/prefix).
- Tabs or sections: **Recent** (last ~20 foods logged), **All my foods**, search results.
- Tapping a food opens the **Log entry sheet**: choose servings/quantity (default 1), meal (preselected, changeable), live calorie preview, "Add to diary".
- If no match: prominent "Create '<query>' as new food" button, prefilling the name.
- After adding: return to Diary with a confirmation toast. Option "Add another" keeps user on the screen.

### 5.3 Create / Edit Food
Fields:
- **Name** (required)
- **Brand** (optional)
- **Serving size** (number) + **serving unit** (g, ml, piece, cup, tbsp, tsp, slice, custom label) (required)
- **Calories per serving** (required, ≥ 0)
- **Protein / Carbs / Fat (g)** per serving (optional)

Validation: required fields, numeric ≥ 0, name max 80 chars, calories max sane bound (e.g. 10,000).
Actions: Save, Save and add to diary (when opened from Add flow), Delete (edit mode).

Editing a food does **not** change past diary entries (they store a snapshot, see §6).

### 5.4 Foods (library)
- Searchable alphabetical list of the user's foods with calories per serving.
- Tap to edit; "+" to create; delete with confirmation.
- Optional "favorite" star to pin to top **(assumption: skip in v1)**.

### 5.5 Profile
See §4. Also shows app version.

## 6. Data Model (Firestore)

All data is scoped under the user's UID.

```
users/{uid}
  displayName: string                  // from Google profile
  email: string
  photoURL: string | null
  dailyCalorieGoal: number
  units: "metric" | "imperial"         // if kept
  createdAt: timestamp
  updatedAt: timestamp

users/{uid}/foods/{foodId}
  name: string
  nameLower: string                    // for case-insensitive search
  brand: string | null
  servingSize: number
  servingUnit: string
  calories: number                     // per serving
  protein: number | null
  carbs: number | null
  fat: number | null
  lastUsedAt: timestamp | null         // powers "Recent"
  createdAt: timestamp
  updatedAt: timestamp

users/{uid}/entries/{entryId}
  date: string                         // "YYYY-MM-DD", user's local date
  meal: "breakfast" | "lunch" | "dinner" | "snacks"
  foodId: string | null                // reference; may be deleted later
  foodSnapshot: {                      // copy at log time, history never changes
    name, brand, servingSize, servingUnit,
    calories, protein, carbs, fat
  }
  quantity: number                     // number of servings, e.g. 1.5
  calories: number                     // quantity * snapshot.calories (denormalized)
  protein, carbs, fat: number | null   // denormalized totals
  createdAt: timestamp
```

Design notes:
- **Snapshot on entries** means editing or deleting a library food never rewrites history.
- **Date as a string** (`YYYY-MM-DD`) avoids timezone bugs when querying a day.
- Day query: `entries where date == D orderBy createdAt`. Totals computed client-side. No aggregation docs needed at this scale.
- Search: v1 loads the user's foods once (cached, realtime listener) and filters client-side. Fine up to a few thousand items. No third-party search needed.
- Indexes: `entries(date, createdAt)` composite; add via `firestore.indexes.json`.

## 7. Security Rules

```
match /users/{uid} {
  allow read, write: if request.auth != null && request.auth.uid == uid;
  match /{document=**} {
    allow read, write: if request.auth != null && request.auth.uid == uid;
  }
}
```

Plus field validation in rules (types, `calories >= 0`, `meal` in the allowed set, string length limits). Everything else denied by default.

## 8. Key Behaviors and Rules

- **Calorie math**: entry calories = `quantity × food.calories`, rounded to nearest integer for display; stored unrounded or to 1 decimal.
- **Remaining** = goal − consumed (can go negative, shown as "over").
- **Day boundary** uses the device's local date.
- **Offline**: enable Firestore offline persistence so logging works with a poor connection and syncs later **(assumption)**. Show a subtle offline indicator.
- **Realtime**: diary and foods use snapshot listeners so multiple devices stay in sync.
- **Optimistic UI** for add/delete, with error toast and rollback on failure.
- **Duplicates**: allow duplicate food names (no uniqueness constraint); warn softly if the exact name already exists when creating.
- **Deleting a food** with prior diary entries is allowed; entries keep their snapshot.

## 9. UX and Design Requirements

- Mobile-first CSS; touch targets ≥ 44 px; bottom nav reachable with thumb; numeric inputs use `inputmode="decimal"`.
- Loading skeletons, empty states, and error states on every data screen.
- Light and dark mode via `prefers-color-scheme` **(assumption)**.
- Accessible: labels on inputs, focus states, sufficient contrast, ARIA for tab bar and dialogs.
- Toasts for success/undo/error.
- Visual style: clean, friendly, a single accent color; I'll propose a palette once you approve the spec.

## 10. Project Structure (planned)

```
StockyApp/src/
  main.js
  App.vue
  router/index.js
  firebase.js                 // init app, auth, firestore
  stores/ (auth, foods, diary)
  views/ (Login, Onboarding, Diary, AddFood, FoodForm, Foods, Profile)
  components/ (TabBar, DateNav, MealSection, EntryRow, SummaryCard, FoodListItem, QuantitySheet, Toast)
  utils/ (date.js, nutrition.js)
```

The current `create-vue` starter components (`HelloWorld`, `TheWelcome`, etc.) will be removed.

## 11. Milestones

1. **Setup**: Firebase project, env vars, router, layout shell, remove starter code.
2. **Auth**: Google sign-in, logout, route guards, onboarding.
3. **Food library**: CRUD for foods + rules.
4. **Diary**: day view, meal sections, add/edit/delete entries, totals, date nav.
5. **Search and recents**: add-food search flow, recents, create-from-search.
6. **Polish**: offline, PWA, empty/error states, dark mode, a11y.
7. **Deploy**: Firebase Hosting + rules + indexes.

## 12. Testing

No test runner exists today. Proposed: Vitest for utils (calorie math, date handling) and Firebase Emulator Suite for security-rule tests **(assumption)**. Manual QA checklist on iOS Safari and Android Chrome.

## 13. Future Ideas (out of scope for v1)

Weekly/monthly history and charts, macro goals, calorie goal calculator, copy meal/day to another day, water and weight tracking, barcode scanning, public food database import (e.g. Open Food Facts / USDA), saved meals/recipes, CSV export, reminders.

## 14. Open Questions

Already decided: Google-only auth, calories required with optional macros, typed daily goal, user's own foods only.

Still open (defaults I'll use if you don't specify are in bold):

1. **Units**: metric only (**yes**), or support imperial toggle?
2. **Food entry model**: per-serving with quantity multiplier (**yes**), or let users log by grams directly?
3. **Past/future dates**: allow logging on any date (**yes**)?
4. **Snacks**: one "Snacks" section (**yes**), or let users add custom meal names?
5. **Offline/PWA**: want installable and offline support (**yes**), or keep it a plain website?
6. **State library**: Pinia (**yes**) or composables only?
7. **Hosting**: Firebase Hosting (**yes**) or elsewhere (Vercel/Netlify)?
8. **Firebase project**: do you already have one, or should I walk you through creating it (needed for the env vars)?
9. **Branding**: any preferred name styling, colors, or logo? ("Stocky" as the app name?)
10. **Account deletion**: include in v1 (**yes**, good practice)?
