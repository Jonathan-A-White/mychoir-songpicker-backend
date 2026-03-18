# Song Picker — App Specification

> **Purpose:** A multi-user web app for choir directors to manage song repertoire, build service plans, and communicate with their choir. The app tracks what's been sung, how well the choir knows each piece, and uses that history — along with liturgical season and voice/difficulty metadata — to help directors make informed song selections.
>
> **Audience:** An agent team implementing this app using the PWA scaffold (`scaffold.md`) for the frontend and a Go backend for multi-user data and Gmail integration.
>
> **Scaffold reference:** This spec extends the PWA scaffold. Everything in `scaffold.md` applies unless explicitly overridden here.

---

## 1. Architecture Overview

Song Picker departs from the scaffold's offline-first, no-backend pattern. It is a **multi-user app with a Go backend** and a PWA frontend.

### 1.1 System Components

| Component | Technology | Hosting | Role |
|---|---|---|---|
| Frontend | React PWA (scaffold) | GitHub Pages | UI, offline shell, local cache |
| Backend API | Go (net/http or Chi) | Vultr VPS | Auth, data persistence, Gmail integration |
| Database | PostgreSQL | Vultr VPS | Canonical data store |
| Auth | Google OAuth 2.0 | Google Cloud | Identity + Gmail API scopes |

### 1.2 Data Flow

```
Director (PWA) ──► Go API (Vultr) ──► PostgreSQL
                       │
                       ├──► Google OAuth (sign-in)
                       └──► Gmail API (send emails)
```

The PWA still uses IndexedDB (Dexie) as a **local cache** for offline browsing of previously loaded data, but the Go backend is the source of truth. The app is usable offline in read-only mode for cached data; writes require connectivity.

### 1.3 Scaffold Overrides

| Scaffold pattern | Song Picker override |
|---|---|
| Dexie as sole data store | Dexie as local cache; Go API + PostgreSQL as source of truth |
| No authentication | Google OAuth 2.0 sign-in (directors only) |
| No backend | Go REST API on Vultr |
| Export/import JSON files | Export/import still supported for backup, but primary data lives server-side |

---

## 2. Domain Entities

### 2.1 Entity Relationship Overview

```
Choir ──┬──► Director (user, many-to-many via ChoirDirector)
        ├──► Hymnal ──► Song ──► SongSeasonTag
        ├──► ServiceOrderTemplate ──► ServiceSlot
        ├──► ServicePlan ──► ServicePlanEntry (ordered, references slots + songs)
        └──► ChoirMember (email recipients)
```

### 2.2 Entity Definitions

All IDs are UUIDs (`string`). All timestamps are epoch milliseconds (`number`).

#### Choir

A choir is the top-level organizational unit. Directors are scoped to one or more choirs.

```ts
interface Choir {
  id: string;
  name: string;                 // e.g., "NAC Bethel Choir"
  createdAt: number;
  createdBy: string;            // director ID
}
```

#### Director (User)

A director is an authenticated Google user who has been granted access to one or more choirs.

```ts
interface Director {
  id: string;
  googleId: string;             // from Google OAuth sub claim
  email: string;
  displayName: string;
  avatarUrl: string | null;
  createdAt: number;
  lastLoginAt: number;
}

interface ChoirDirector {
  choirId: string;
  directorId: string;
  role: "owner" | "director";   // owner can manage other directors
  joinedAt: number;
}
```

#### Hymnal

A named collection of songs. Each hymnal has a **prefix** — a short code that directors use when referencing songs (e.g., "H165" means Hymnal song 165). This is how directors think about and communicate song references.

```ts
interface Hymnal {
  id: string;
  choirId: string;
  name: string;                 // e.g., "New Apostolic Hymnal"
  prefix: string;               // e.g., "H", "B", "C" — used in display like "H165"
  description: string | null;
  createdAt: number;
}
```

**Pre-configured hymnals for NAC choirs** (created automatically during choir setup if the director selects "New Apostolic Church"):

| Prefix | Name | Description | Audience |
|---|---|---|---|
| H | New Apostolic Hymnal | Main congregational hymnal (588 songs) | Whole congregation |
| B | Choir Binder | Choir-only song collection | Choir members only |
| C | Celebration Hymnal | Supplemental celebration songs | Whole congregation |

Directors can add additional hymnals (e.g., for gospel songs, special collections) or use "Custom" for standalone songs not in any published hymnal.

#### Song

A song in the catalog. Belongs to a hymnal or is a standalone custom entry.

```ts
interface Song {
  id: string;
  choirId: string;
  hymnalId: string | null;      // null for custom/standalone songs
  title: string;
  hymnNumber: string | null;    // e.g., "165", "604" — string for flexible formats
  composer: string | null;
  arranger: string | null;
  voiceParts: VoicePart[];      // which parts the song is written for
  difficulty: Difficulty | null;
  totalVerses: number | null;   // how many verses the song has in total
  notes: string | null;         // director notes, performance tips
  createdAt: number;
  updatedAt: number;
}

type VoicePart = "soprano" | "alto" | "tenor" | "bass" | "unison" | "descant";
type Difficulty = "easy" | "moderate" | "challenging";
```

**Display format:** When showing a song reference, always use `{prefix}{hymnNumber}` (e.g., "H165", "B110", "C604"). For custom songs with no hymnal, show just the title.

#### Song–Season Tag

Links a song to one or more liturgical seasons it is appropriate for.

```ts
interface SongSeasonTag {
  id: string;
  songId: string;
  season: LiturgicalSeason;
}

type LiturgicalSeason =
  | "advent"
  | "christmas"
  | "epiphany"
  | "lent"
  | "holy_week"
  | "easter"
  | "pentecost"
  | "ordinary"
  | "reformation"
  | "thanksgiving"
  | "general";           // appropriate any time
```

#### Service Order Template

A choir's default service structure — the ordered list of slots that make up a typical divine service. This defines what "When" column values are available when building a service plan.

The template is mostly fixed but can be customized for special services (e.g., a holiday service might add extra slots or remove some).

```ts
interface ServiceOrderTemplate {
  id: string;
  choirId: string;
  name: string;                 // e.g., "Standard Divine Service", "Holiday Service"
  isDefault: boolean;           // one template is marked as default
  createdAt: number;
}

interface ServiceSlot {
  id: string;
  templateId: string;
  label: string;                // e.g., "Pre 1", "Open", "After Text", "Mid1"
  sortOrder: number;            // position in the service order
  defaultPerformer: Performer | null;  // typical performer for this slot
  required: boolean;            // must this slot be filled before finalizing?
}
```

**Default NAC service order template:**

| Sort | Label | Default Performer | Required |
|---|---|---|---|
| 1 | Pre 1 | Choir | no |
| 2 | Pre 2 | Choir | no |
| 3 | Pre 3 | Congregation | no |
| 4 | Pre 4 | — | no |
| 5 | Open | Congregation | yes |
| 6 | After Text | Choir | no |
| 7 | Mid1 | Congregation | no |
| 8 | Repentance | Choir | no |
| 9 | Communion | Congregation | no |
| 10 | After Communion | — | no |
| 11 | Closing | Choir & Congregation | yes |

Directors can add/remove/reorder slots in a template, and can create alternate templates for special services.

#### Service Plan

The core unit: a dated plan of what the choir will sing for a specific service.

```ts
interface ServicePlan {
  id: string;
  choirId: string;
  templateId: string;           // which service order template this plan uses
  serviceDate: number;          // epoch ms, date of the service
  title: string | null;         // optional label, e.g., "Easter Sunday Morning"
  status: PlanStatus;
  notes: string | null;         // general notes for the choir
  createdBy: string;            // director ID
  createdAt: number;
  updatedAt: number;
  emailSentAt: number | null;   // when the plan was emailed to the choir
}

type PlanStatus = "draft" | "finalized" | "completed";
```

#### Service Plan Entry

An entry in a service plan. Each entry corresponds to a slot in the service order and specifies what song (if any) is assigned, who performs it, and which verses.

A plan entry **can exist without a song** — this represents a TBD/placeholder slot that the director hasn't filled yet.

```ts
interface ServicePlanEntry {
  id: string;
  servicePlanId: string;
  slotId: string;               // references a ServiceSlot from the template
  songId: string | null;        // null = TBD / placeholder
  performer: Performer;         // who sings/plays this entry
  verses: string | null;        // e.g., "1,2", "1,2,3", "4", "1 or 3", null = all
  notes: string | null;         // e.g., "tenors only on verse 2", or "Solo - Addie"
  sortOrder: number;            // allows overriding template order if needed
}

type Performer =
  | "choir"
  | "congregation"
  | "choir_and_congregation"
  | "piano_organ"              // instrumental only
  | "instrumental"             // other instruments
  | "solo";                    // when solo, the notes field should name the soloist

// Note: When performer is "solo", the `notes` field carries the soloist name
// e.g., performer: "solo", notes: "Solo - Addie"
```

#### Choir Member

Email recipients — not app users, just contacts for sending service plan emails.

```ts
interface ChoirMember {
  id: string;
  choirId: string;
  name: string;
  email: string;
  voicePart: VoicePart | null;
  active: boolean;
  createdAt: number;
}
```

---

## 3. Computed / Derived Data

These are not stored entities but are computed by the backend or frontend from service plan history.

### 3.1 Song Familiarity Score

A numeric score (0–100) representing how well the choir knows a song, derived from:

- **Frequency**: How many times it has appeared in completed service plans (more = higher)
- **Recency**: When it was last sung (more recent = higher, with decay over time)
- **Recency window**: Songs not sung in 6+ months begin decaying; 12+ months decay significantly

The formula should be tunable, but a starting point:

```
familiarity = (frequency_score * 0.5) + (recency_score * 0.5)
```

Where `frequency_score` is normalized against the most-sung song in the catalog, and `recency_score` decays exponentially from the last service date.

### 3.2 Song Suggestion Ranking

When a director is building a service plan, the song picker ranks songs by:

1. **Liturgical season match** — songs tagged for the current season float to top
2. **Familiarity sweet spot** — prefer songs the choir knows moderately well (not too stale, not overplayed). Configurable: some directors may want to prioritize well-known songs, others may want to surface forgotten ones.
3. **Recency penalty** — songs sung very recently (last 2–3 weeks) are deprioritized
4. **Performer context** — when filling a slot with a default performer (e.g., "Choir"), prefer songs the choir typically performs; when filling a "Congregation" slot, prefer congregational hymns
5. **Voice part availability** — if the director knows which sections are light this week, filter accordingly
6. **Difficulty filter** — optional filter by difficulty level

The ranking is a suggestion, not a constraint. Directors always have full catalog access.

---

## 4. Pages & Navigation

### 4.1 Navigation Pattern

**Bottom tabs** (mobile-first, 4 tabs):

| Tab | Icon | Label | Route |
|---|---|---|---|
| 1 | calendar | Plans | `/plans` |
| 2 | music note | Songs | `/songs` |
| 3 | users | Choir | `/choir` |
| 4 | gear | Settings | `/settings` |

### 4.2 Page Inventory

#### Plans Tab

| Page | Route | Description |
|---|---|---|
| Plans List | `/plans` | Upcoming and recent service plans, sorted by date |
| Plan Detail | `/plans/:id` | View a finalized or completed plan (table format matching director email style) |
| Plan Editor | `/plans/:id/edit` | Build/edit a plan: fills service order slots with songs, performers, verses |
| New Plan | `/plans/new` | Create a new service plan (picks date + template, opens editor) |

#### Songs Tab

| Page | Route | Description |
|---|---|---|
| Song Catalog | `/songs` | Browse/search all songs, filter by hymnal, season, difficulty |
| Song Detail | `/songs/:id` | Song metadata, history (when it was sung, in which slot, with which performer), familiarity score |
| Song Editor | `/songs/:id/edit` | Edit song metadata, tags, notes |
| New Song | `/songs/new` | Add a song to the catalog |
| Hymnal Manager | `/songs/hymnals` | List/add/edit hymnals and their prefixes |

#### Choir Tab

| Page | Route | Description |
|---|---|---|
| Choir Members | `/choir` | List of choir members (email contacts) |
| Member Editor | `/choir/:id/edit` | Edit member details |
| New Member | `/choir/new` | Add a choir member |

#### Settings Tab

| Page | Route | Description |
|---|---|---|
| Settings | `/settings` | Standard scaffold settings + choir management |
| Service Order | `/settings/service-order` | Edit the default service order template; create alternate templates |
| Email Template | `/settings/email-template` | Customize the default email format/template |

#### Auth (No Tab)

| Page | Route | Description |
|---|---|---|
| Sign In | `/sign-in` | Google sign-in button, app branding |
| Choir Setup | `/setup` | First-run: create or join a choir (includes denomination/hymnal selection) |

---

## 5. Key User Flows

### 5.1 First-Time Setup

1. Director lands on `/sign-in`, taps "Sign in with Google"
2. Google OAuth flow completes, backend creates Director record
3. Director is redirected to `/setup`
4. Options: **Create a new choir** (enter name) or **Join existing** (enter invite code from another director)
5. If creating:
   a. Director becomes `owner`
   b. Prompted to select denomination/tradition — selecting "New Apostolic Church" pre-creates the three hymnals (H, B, C) and the default service order template
   c. Selecting "Other" or "Custom" starts with an empty hymnal list and a blank service order template
   d. Redirected to `/plans`
6. If joining: director becomes `director` role, redirected to `/plans`

### 5.2 Building a Service Plan

1. Director navigates to Plans tab, taps "New Plan"
2. Picks a service date (defaults to next Sunday)
3. Selects service order template (defaults to the choir's default template; can pick an alternate for special services)
4. Enters optional title (e.g., "Palm Sunday Morning")
5. The plan editor shows the **service order as a table** with columns: When | Who | Nbr | Vs | Title
6. Each row corresponds to a slot from the template, pre-filled with the slot label and default performer
7. To fill a slot, the director taps the row — this opens the **Song Picker**:
   a. Song Picker shows the full catalog, sorted by suggestion ranking (Section 3.2)
   b. Ranking is contextual — if the slot's default performer is "Choir", choir songs are prioritized
   c. Director can filter by: season, hymnal, difficulty, voice parts, text search
   d. Each song card shows: title, hymnal prefix + number, familiarity score, last sung date, season tags
   e. Director taps a song to assign it to the slot
8. Back in the plan editor, the director can for each row:
   - Change the performer (dropdown: Choir, Congregation, Choir & Congregation, Solo, Piano/Organ, Instrumental)
   - Specify verses (text field, e.g., "1,2" or "all" or "1 or 3")
   - Add per-entry notes
   - Clear the song (reverts to TBD)
   - Change the assigned song
9. The director can also:
   - Add extra slots not in the template (e.g., an additional prelude)
   - Remove optional slots they don't need for this service
   - Reorder slots if the service order differs from the template
10. Director taps "Save" (status = `draft`) or "Finalize"
11. On finalize, option to **Email Choir** appears

### 5.3 Emailing the Choir

1. Director taps "Email Choir" on a finalized plan
2. App generates an email from the plan data using the choir's **email template** (configured in Settings)
3. The email is shown in a preview/edit screen — the director can modify subject line, body text, add notes
4. The generated content renders the plan as a table or formatted list matching the When/Who/Nbr/Vs/Title structure the directors are accustomed to
5. Director taps "Send" — Go backend sends via Gmail API using the director's OAuth token
6. `emailSentAt` is recorded on the service plan
7. Recipients: all active choir members for that choir

**Default email template** (customizable per choir):

```
Subject: {{choir_name}} — Songs for {{service_date}}

{{#if title}}{{title}}{{/if}}

{{#each entries}}
{{slot_label}}: {{performer}} — {{song_ref}} {{#if verses}}(vs {{verses}}){{/if}} {{title}}
{{/each}}

{{#if notes}}
Notes: {{notes}}
{{/if}}

— {{director_name}}
```

The template uses simple Mustache-style placeholders. Directors can rearrange, add static text, or change formatting in the Settings email template editor.

### 5.4 Adding Songs to the Catalog

1. Director navigates to Songs tab, taps "Add Song"
2. Fills in: title (required), hymnal (dropdown including prefix preview), hymn number, composer, arranger, total verses, voice parts (multi-select), difficulty, liturgical seasons (multi-select), notes
3. The preview shows the formatted reference (e.g., "H165 — While the sun for us shines brightly")
4. Saves — song is available in the catalog for all directors in that choir

### 5.5 Inviting Another Director

1. Owner goes to Settings → "Manage Directors"
2. Taps "Generate Invite Code"
3. Code is a short alphanumeric string (e.g., `CHOIR-7X9K`), valid for 48 hours
4. Shares code with the new director out-of-band (text, email, in person)
5. New director signs in, enters code at `/setup`, joins the choir

### 5.6 Customizing the Service Order

1. Director goes to Settings → "Service Order"
2. Sees the current default template as an ordered list of slots
3. Can:
   - Rename slots (e.g., change "Mid1" to "Mid-Service")
   - Add new slots (specify label, default performer, required flag)
   - Remove slots
   - Reorder via drag or up/down arrows
   - Set default performer for each slot
4. Can create alternate templates for special services (e.g., "Holiday Service", "Confirmation Service")
5. Changes to the default template do not affect existing plans — only new plans

---

## 6. Backend API

### 6.1 Technology

- **Language:** Go
- **Router:** Chi or standard `net/http` mux (Go 1.22+)
- **Database:** PostgreSQL
- **Auth:** Google OAuth 2.0 (server-side flow)
- **Gmail:** Gmail API via Google OAuth token (requires `gmail.send` scope)
- **Deployment:** Vultr VPS, reverse proxy via Caddy or Nginx

### 6.2 API Routes

All routes prefixed with `/api/v1`. Auth required unless noted.

#### Auth

| Method | Route | Description |
|---|---|---|
| POST | `/auth/google` | Exchange Google auth code for session |
| POST | `/auth/refresh` | Refresh session token |
| DELETE | `/auth/session` | Sign out |

#### Choirs

| Method | Route | Description |
|---|---|---|
| POST | `/choirs` | Create a new choir (with denomination selection for auto-setup) |
| GET | `/choirs/:id` | Get choir details |
| POST | `/choirs/join` | Join a choir via invite code |
| POST | `/choirs/:id/invite` | Generate invite code (owner only) |
| GET | `/choirs/:id/directors` | List directors for a choir |
| DELETE | `/choirs/:id/directors/:dirId` | Remove a director (owner only) |

#### Songs

| Method | Route | Description |
|---|---|---|
| GET | `/choirs/:id/songs` | List songs (supports query params for filtering) |
| POST | `/choirs/:id/songs` | Create a song |
| GET | `/choirs/:id/songs/:songId` | Get song detail + computed familiarity |
| PUT | `/choirs/:id/songs/:songId` | Update a song |
| DELETE | `/choirs/:id/songs/:songId` | Delete a song |
| GET | `/choirs/:id/songs/:songId/history` | Service plan appearances for this song |

**Query params for song list:** `?hymnal=`, `?season=`, `?difficulty=`, `?search=`, `?sort=suggestion` (triggers ranking algorithm), `?performer=` (filter by typical performer context)

#### Hymnals

| Method | Route | Description |
|---|---|---|
| GET | `/choirs/:id/hymnals` | List hymnals |
| POST | `/choirs/:id/hymnals` | Create a hymnal (name + prefix) |
| PUT | `/choirs/:id/hymnals/:hymnalId` | Update a hymnal |
| DELETE | `/choirs/:id/hymnals/:hymnalId` | Delete a hymnal (fails if songs reference it) |

#### Service Order Templates

| Method | Route | Description |
|---|---|---|
| GET | `/choirs/:id/templates` | List service order templates |
| POST | `/choirs/:id/templates` | Create a template |
| GET | `/choirs/:id/templates/:tplId` | Get template with slots |
| PUT | `/choirs/:id/templates/:tplId` | Update template metadata |
| DELETE | `/choirs/:id/templates/:tplId` | Delete template (fails if plans reference it) |
| PUT | `/choirs/:id/templates/:tplId/slots` | Replace the full slot list (ordered array) |

#### Service Plans

| Method | Route | Description |
|---|---|---|
| GET | `/choirs/:id/plans` | List plans (supports `?status=`, `?from=`, `?to=`) |
| POST | `/choirs/:id/plans` | Create a plan (date + template ID) |
| GET | `/choirs/:id/plans/:planId` | Get plan with entries (joined with song + slot data) |
| PUT | `/choirs/:id/plans/:planId` | Update plan metadata |
| DELETE | `/choirs/:id/plans/:planId` | Delete a plan |
| PUT | `/choirs/:id/plans/:planId/entries` | Replace the full entry list |
| POST | `/choirs/:id/plans/:planId/finalize` | Set status to finalized |
| POST | `/choirs/:id/plans/:planId/complete` | Set status to completed |
| POST | `/choirs/:id/plans/:planId/email` | Send plan to choir via Gmail API |

#### Choir Members

| Method | Route | Description |
|---|---|---|
| GET | `/choirs/:id/members` | List choir members |
| POST | `/choirs/:id/members` | Add a member |
| PUT | `/choirs/:id/members/:memberId` | Update a member |
| DELETE | `/choirs/:id/members/:memberId` | Remove a member |

#### Email Templates

| Method | Route | Description |
|---|---|---|
| GET | `/choirs/:id/email-template` | Get the choir's email template |
| PUT | `/choirs/:id/email-template` | Update the email template |

### 6.3 Authentication Flow

1. Frontend initiates Google OAuth with scopes: `openid`, `email`, `profile`, `https://www.googleapis.com/auth/gmail.send`
2. Google redirects back with an auth code
3. Frontend sends the auth code to `POST /api/v1/auth/google`
4. Backend exchanges code for tokens (access + refresh), stores refresh token encrypted in DB
5. Backend creates/updates Director record, returns a session JWT
6. Frontend stores JWT, includes it as `Authorization: Bearer <token>` on all API calls
7. Backend uses stored refresh token to obtain fresh Gmail access tokens when sending email

### 6.4 Gmail Send

When a director triggers email send:

1. Backend loads the director's stored Google refresh token
2. Exchanges it for a fresh access token
3. Renders the email from the choir's email template + plan data
4. Constructs a MIME email (both plain text and HTML versions)
5. Sends via Gmail API `POST https://gmail.googleapis.com/gmail/v1/users/me/messages/send`
6. Records `emailSentAt` on the service plan

---

## 7. Frontend–Backend Sync

### 7.1 Local Cache Strategy

The PWA uses Dexie to cache data fetched from the API for offline read access:

- On each API response, update the local Dexie cache
- When offline, serve data from the local cache (read-only)
- When back online, re-fetch from API (API is always source of truth)
- Writes always go to the API first; on failure, show an error (no offline write queue in v1)

### 7.2 Sync Indicators

- Show a connectivity indicator in the app header (green dot = online, gray = offline)
- When offline, show a subtle banner: "You're offline — viewing cached data"
- Disable write actions (new plan, edit, send email) when offline

---

## 8. App-Specific Settings

In addition to the scaffold's standard Settings sections (Appearance, Storage, About), add:

| Section | Contents |
|---|---|
| **Choir** | Current choir name, switch choir (if member of multiple), leave choir |
| **Directors** | List directors, generate invite code (owner only), remove director (owner only) |
| **Service Order** | Edit the default service order template; create alternate templates (links to `/settings/service-order`) |
| **Email Template** | Customize the email format for sending plans to the choir (links to `/settings/email-template`) |
| **Account** | Google account info, sign out |
| **Familiarity Tuning** | Slider or preset for familiarity scoring: "Prefer well-known songs" ↔ "Surface forgotten songs" |

---

## 9. PWA Configuration (App-Specific)

| Field | Value |
|---|---|
| `name` | Song Picker |
| `short_name` | SongPick |
| `description` | Choir service planning and song selection |
| `theme_color` | `#4A5568` (slate gray — neutral, church-appropriate) |
| `background_color` | `#FFFFFF` |

### 9.1 Semantic Colors

```css
@theme {
  --color-primary: #4A5568;         /* slate gray */
  --color-primary-light: #718096;   /* lighter slate */
  --color-surface: #F7FAFC;         /* off-white */
  --color-danger: #E53E3E;          /* red */
  --color-season-advent: #6B46C1;   /* purple */
  --color-season-christmas: #D69E2E;/* gold */
  --color-season-lent: #805AD5;     /* violet */
  --color-season-easter: #F6E05E;   /* bright gold */
  --color-season-pentecost: #E53E3E;/* red */
  --color-season-ordinary: #48BB78; /* green */
}
```

### 9.2 Icon

The app icon should depict a music note combined with a small cross or church silhouette, rendered in the slate/purple palette. Generate as SVG, export to 192px and 512px PNG.

---

## 10. Feedback Pattern

**Toast notifications** (bottom of screen, auto-dismiss after 3s):

- Success: "Plan saved", "Email sent to 12 members", "Song added to catalog"
- Error: "Failed to save — check your connection", "Email send failed"
- Info: "You're offline — viewing cached data"

---

## 11. Mobile Interaction Patterns

Adopt from scaffold:

- **Swipe to reveal actions** on song list items (remove from plan) and choir member list (edit, remove)
- **Long press** on plan entries to reorder (or use up/down arrow buttons as fallback)

---

## 12. Testing Strategy (App-Specific)

### 12.1 BDD Feature Files

In addition to scaffold features, add:

```
tests/features/
├── auth.feature              # Google sign-in, session, sign-out
├── choir-setup.feature       # Create choir, denomination selection, join via invite code
├── song-catalog.feature      # CRUD songs, hymnals with prefixes, filtering, search
├── service-order.feature     # Template management, slot CRUD, alternate templates
├── service-plan.feature      # Create from template, fill slots, assign performers/verses, TBD slots
├── song-picker.feature       # Suggestion ranking, performer-context filtering, adding to plan
├── email-choir.feature       # Template rendering, preview, edit, send via Gmail
├── choir-members.feature     # CRUD members, voice parts
├── offline.feature           # Cached data, offline indicators, disabled writes
├── familiarity.feature       # Score calculation, tuning
└── multi-director.feature    # Invite, roles, shared catalog/plans
```

### 12.2 Backend Tests

- Go test files alongside source (`_test.go`)
- Use `testing` package + `testify` for assertions
- Integration tests against a test PostgreSQL database
- Mock Gmail API for email send tests

---

## 13. Deployment

### 13.1 Frontend

Standard scaffold GitHub Pages deployment (Section 12 of scaffold). The `base` URL and API base URL are configured via environment variables:

```ts
// src/config.ts
export const API_BASE = import.meta.env.VITE_API_BASE || "http://localhost:8080/api/v1";
```

### 13.2 Backend

- Go binary built with `go build`, deployed to Vultr VPS
- Managed via systemd service
- Caddy as reverse proxy (automatic HTTPS via Let's Encrypt)
- PostgreSQL on the same VPS (or managed DB if preferred)
- Environment variables for: `DATABASE_URL`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `JWT_SECRET`, `CORS_ORIGIN`

### 13.3 Google Cloud Setup

- Google Cloud project with Gmail API and Google Sign-In enabled
- OAuth 2.0 credentials (web application type)
- Authorized redirect URI pointing to the backend callback endpoint
- **Production mode** (not testing) to avoid 7-day refresh token expiry

---

## 14. Future Considerations (Out of Scope for v1)

These are not part of the initial build but are natural extensions:

- **Bulk song import** — CSV or hymnal database import to seed catalog (e.g., scraping the NAC hymnal app data)
- **Sheet music attachments** — upload/link PDFs per song
- **Rehearsal tracking** — mark which songs were actually rehearsed vs. just planned
- **Choir member self-service** — members sign in to view plans (read-only)
- **Calendar integration** — sync service plans to Google Calendar
- **Song lyrics display** — show lyrics in the app (copyright considerations apply)
- **Multiple choirs per director** — already modeled in the schema, but UI only supports one choir in v1
- **Divine service guide integration** — match songs to lectionary readings or minister's sermon plan
- **Historical import** — parse old director emails to backfill service plan history and bootstrap familiarity scores
- **Soloist management** — track named soloists as first-class entities with availability

---

## 15. Implementation Workstreams

Suggested parallel workstreams for agent implementation:

| # | Workstream | Dependencies | Description |
|---|---|---|---|
| 1 | **Scaffold + PWA shell** | None | Initialize project from scaffold, configure PWA, set up app shell with bottom tabs and routing |
| 2 | **Backend foundation** | None | Go project setup, PostgreSQL schema (all tables), Chi router, health check, CORS |
| 3 | **Auth** | WS-2 | Google OAuth flow (backend + frontend), JWT sessions, protected routes |
| 4 | **Hymnals + Song catalog** | WS-1, WS-2, WS-3 | Hymnal CRUD (with prefixes), Songs CRUD, season tags, display formatting — backend + frontend |
| 5 | **Service order templates** | WS-2, WS-3 | Template CRUD, slot management, NAC default template seeding — backend + frontend |
| 6 | **Service plans + plan editor** | WS-4, WS-5 | Plans CRUD, plan editor with slot-based table UI, performer/verse assignment, TBD slots, song picker with ranking — backend + frontend |
| 7 | **Choir members + email** | WS-3, WS-6 | Member management, email template system, Gmail API send, preview/edit — backend + frontend |
| 8 | **Offline cache** | WS-4, WS-6 | Dexie cache layer, sync indicators, offline read mode |
| 9 | **Settings + director management** | WS-3, WS-5 | Invite codes, multi-director, choir settings, familiarity tuning, service order editor, email template editor |
| 10 | **Polish + testing** | All | BDD features, backend tests, responsive polish, PWA verification |
