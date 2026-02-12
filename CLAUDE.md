# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a multi-platform voter canvassing application for SRP (Salt River Project) elections. The platform uses SRP's unique **acre-based voting system** where voting power is tied to land ownership (see `SRP_VOTING_SYSTEM.md` for details).

### Main Projects (shared Supabase backend)
1. **srp_canvass_flutter/** - Cross-platform Flutter app (iOS/Android/Web/macOS)
2. **srp-canvass-admin/** - Admin dashboard (Next.js 15 + TypeScript)

### Related Projects
3. **clean_energy_team/** - "Clean Energy Friends" simplified canvass app (Flutter) - see its CLAUDE.md
4. **energyfreedom-website/** - Static marketing website (HTML/CSS) - see docs/CLAUDE.md
5. **srp-events-public/** - Public events listing page (Next.js)
6. **srp-ballot-form/** - Ballot request form (Next.js, port 3002)
7. **solar-savings-dashboard/** - Solar calculator (Next.js, port 3001)
8. **solar-detection/** - ML pipeline for rooftop solar detection (Python/Jupyter, runs in Google Colab)

## Build Commands

### Flutter App (srp_canvass_flutter/)
```bash
flutter pub get
cd ios && pod install && cd ..
flutter run                          # Development
flutter build apk --release          # Android APK
flutter build ios --release          # iOS
flutter build web --release --base-href=/canvass/  # Web (note: base-href required)
dart run build_runner build --delete-conflicting-outputs  # Code gen (after Drift changes)
flutter analyze                      # Linting
flutter test                         # Run all tests
flutter test test/specific_test.dart # Run single test file
```

### Admin Dashboard (srp-canvass-admin/)
```bash
npm install
npm run dev      # Development (http://localhost:3000/admin)
npm run build    # Production build
npm run lint     # ESLint
./deploy.sh      # Full deploy (build, rsync, restart PM2)
```

### Solar Savings Dashboard (solar-savings-dashboard/)
```bash
npm install
npm run dev      # Development (http://localhost:3001) - MUST use port 3001
npm run build    # Production build
```

### Static Website (energyfreedom-website/)
No build process - deploy static files directly via scp.

## Architecture

### ⚠️ CRITICAL: Database Schema Verification

**BEFORE writing ANY database queries, ALWAYS verify the actual schema in DATABASE_SCHEMA.md:**

1. **Check DATABASE_SCHEMA.md** - This is the SINGLE SOURCE OF TRUTH for all table schemas
2. **Verify column names** - Do NOT assume, do NOT guess based on similar tables
3. **Test queries before committing** - Use the psql command or Supabase client to verify

**Common mistakes that WILL break production:**
- ❌ Using `created_by` instead of ✅ `contacted_by` in `contact_history` table
- ❌ Assuming column names without checking the schema
- ❌ Using wrong table or column names will cause silent query failures

**Quick schema verification:**
```bash
# Direct PostgreSQL connection
psql "postgresql://postgres:Kj9mP2vL5nQ8xR3wF6yT4@supabase.energyfreedom.team:5432/postgres" \
  -c "\d+ table_name"

# Or check DATABASE_SCHEMA.md (ALWAYS DO THIS FIRST)
```

**DATABASE_SCHEMA.md is authoritative. Always check it before writing queries.**

### Shared Backend (Self-Hosted Supabase)
All apps connect to a **self-hosted Supabase instance** on Hetzner VPS:

**Server:** `46.225.19.215` (Hetzner CX43 - 16GB RAM, 4 vCPU, Nuremberg)

| Service | URL | Notes |
|---------|-----|-------|
| **API (Kong)** | https://api.energyfreedom.team | Main API endpoint |
| **Studio** | https://supabase.energyfreedom.team | Admin UI (login: supabase / SrpAdmin2026Secure) |
| **Dev API** | https://dev-api.energyfreedom.team | Testing endpoint |

**Database:**
- PostgreSQL 17 with ~664k voter records
- 158 auth users (email, Google, Apple SSO)
- 3 storage buckets (voice-notes, import, imports)

**Auth Providers:**
- Email/Password
- Google OAuth
- Apple Sign-In

**Supabase Keys (self-hosted):**
```
ANON_KEY: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoiYW5vbiIsImlzcyI6InN1cGFiYXNlIiwiaWF0IjoxNzA0MDY3MjAwLCJleHAiOjE4OTM0NTYwMDB9.ijz5qNCy2uf8u9K0Bg8piaaI0MeH6CKBJTmN22dFoZU
SERVICE_ROLE_KEY: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoic2VydmljZV9yb2xlIiwiaXNzIjoic3VwYWJhc2UiLCJpYXQiOjE3MDQwNjcyMDAsImV4cCI6MTg5MzQ1NjAwMH0.xLebcbRkxlZ6-9pIkeiid1wX34Bexr8xFfGP0XK3234
```

**Server Management:**
```bash
# SSH access
ssh root@46.225.19.215

# Supabase location
cd /root/supabase/docker

# View logs
docker compose logs -f auth    # Auth service
docker compose logs -f rest    # PostgREST
docker compose logs -f kong    # API Gateway

# Restart services
docker compose restart auth
docker compose up -d --force-recreate auth

# Database access
docker exec -it supabase-db psql -U postgres -d postgres
```

**Caddy Reverse Proxy:**
- Config: `/etc/caddy/Caddyfile`
- SSL via Let's Encrypt with Cloudflare DNS challenge
- Systemd service: `systemctl status caddy`

**Features:**
- Row-level security policies
- User roles: pending, canvasser, team_lead, admin
- Districts: Users can be assigned to multiple SRP districts (1-10)

### Database Schema
See **[DATABASE_SCHEMA.md](DATABASE_SCHEMA.md)** for complete table documentation.

**Core Tables:**
- `voters` - ~664k voter/property records with canvass results, contact info, solar detection
- `user_profiles` - User accounts with roles (`pending`, `canvasser`, `team_lead`, `admin`)
- `contact_history` - Call/text/door tracking (links via `unique_id`)
- `cut_lists` / `cut_list_assignments` - Geographic voter segments and assignments
- `events` / `event_signups` - Calendar events and registrations
- `invites` / `invite_uses` - Self-service signup codes
- `text_templates` / `call_scripts` / `call_script_sections` - Communication templates
- `outreach_users` - Clean Energy Team device-based users
- `audit_logs` - Admin activity tracking

### State Management Patterns
- **Flutter**: Riverpod with NotifierProvider pattern
- **Admin**: TanStack Query (server state) + Zustand (client state)

### Offline-First (Flutter)
The Flutter app uses Drift (SQLite) for local caching:
- `CacheService` with platform-specific implementations
- `SyncService` for background sync (5-minute intervals)
- `PendingVoterUpdates` queue for offline changes

## Business Logic

### Votes Display Rules
- **Owners (ownerCode='A')**: Show individual `votes` AND `totalPropertyVotes` when different (e.g., "3.33 of 10.00 acres")
- **Trustees (ownerCode='T')**: Show only `votes` (trustee gets all property votes)

### Canvass Result Categories
- **Positive (isSupportive)**: Supportive, Strong Support, Leaning, Willing to Volunteer, Requested Sign
- **Negative (isNegative)**: Opposed, Strongly Opposed, Do Not Contact, Refused
- **Neutral (isNeutral)**: Undecided, Needs Info, Callback Requested

### Contact Attempts
Contact attempts are calculated from `contact_history` table, not stored separately on voters.

### Invite Code System
Admins and team leads can create invite codes that allow new canvassers to self-register without manual approval.

**How it works:**
1. Admin/team lead creates invite code at `/admin/invites`
2. Configure: districts, cut list (optional), max uses, expiration date (optional)
3. Share link: `https://energyfreedom.team/join/CODE`
4. New user clicks link → completes signup form → auto-approved as canvasser
5. User redirected to `/app` to download the mobile app

**Invite Code Features:**
- **Reusable**: Set max uses (1-1000) per code
- **Pre-assign Districts**: Users get assigned to specified districts on signup
- **Pre-assign Cut List**: Optionally assign users to a cut list automatically
- **Expiration**: Optional expiration date
- **Enable/Disable**: Toggle codes on/off without deleting
- **Usage Tracking**: See who signed up with each code

**URLs:**
- Admin: `https://energyfreedom.team/admin/invites`
- Public signup: `https://energyfreedom.team/join/{CODE}`

## Deployment

### Hosting Architecture

The platform uses a multi-host deployment strategy for reliability and performance:

| Site | Host | URL | Deploy Method |
|------|------|-----|---------------|
| **Marketing Site** | Cloudflare Pages | energyfreedom.team | `wrangler pages deploy` |
| **Flutter Web** | Cloudflare Pages | canvass.energyfreedom.team | `wrangler pages deploy` |
| **Docs** | Cloudflare Pages | docs.energyfreedom.team | `wrangler pages deploy` |
| **Admin Dashboard** | Vercel | admin.energyfreedom.team | `vercel --prod` |
| **Join/Invites** | Vercel | join.energyfreedom.team | (same as admin) |
| **Solar Dashboard** | VPS (port 3001) | solar.energyfreedom.team | rsync + PM2 |
| **Events Public** | VPS (port 3002) | events.energyfreedom.team | rsync + PM2 |
| **Auth Service** | VPS (port 1411) | auth.energyfreedom.team | PM2 |
| **OpenShortLink** | Cloudflare Workers | qr.energyfreedom.team | Cloudflare Dashboard |

**Cloudflare Pages Projects:**
- `energyfreedom-team` → energyfreedom.team
- `srpcanvass-web` → canvass.energyfreedom.team
- `docs-energyfreedom` → docs.energyfreedom.team

**Vercel Project:**
- `srp-canvass-admin` → admin.energyfreedom.team, join.energyfreedom.team

### VPS Configuration (for remaining services)

**CRITICAL: SSH user is `root` (VPS IP: 100.85.1.68)**

The VPS still runs some Next.js apps. Each MUST use its assigned port:

| App | Port | URL | PM2 Name | Server Path |
|-----|------|-----|----------|-------------|
| solar-dashboard | 3001 | solar.energyfreedom.team | solar-dashboard | /var/www/solar-dashboard |
| srp-events-public | 3002 | events.energyfreedom.team | srp-events-public | /var/www/srp-events-public |
| auth-service | 1411 | auth.energyfreedom.team | - | - |

**If you get EADDRINUSE errors during deployment:**
```bash
# 1. Check what's on each port
ssh johnt@energyfreedom.team "sudo ss -tlnp | grep 3000"

# 2. Check if root PM2 service is running (common conflict!)
ssh johnt@energyfreedom.team "sudo systemctl status pm2-root"

# 3. If pm2-root service is active, stop it first
ssh johnt@energyfreedom.team "sudo systemctl stop pm2-root; sudo pm2 kill"

# 4. Kill any remaining processes on the port
ssh johnt@energyfreedom.team "sudo pkill -9 -f 'next-server'; sleep 2"

# 5. Start PM2 as johnt user (NOT root)
ssh johnt@energyfreedom.team "cd /var/www/srp-canvass-admin && pm2 start server.js --name srp-canvass-admin && pm2 save"
```

**PM2 Root Service Conflict (IMPORTANT!):**
The server has a `pm2-root.service` systemd service that can conflict with the johnt user's PM2. Symptoms:
- PM2 shows many restarts (↺ 50+)
- EADDRINUSE errors in logs
- `pm2 restart` doesn't fix the issue

Fix:
```bash
# Stop the root PM2 service
ssh johnt@energyfreedom.team "sudo systemctl stop pm2-root; sudo pm2 kill; sleep 2"

# Verify port is free
ssh johnt@energyfreedom.team "sudo ss -tlnp | grep 3000 || echo 'Port free'"

# Start as johnt user
ssh johnt@energyfreedom.team "cd /var/www/srp-canvass-admin && pm2 start server.js --name srp-canvass-admin && pm2 save"
```

### Pre-Deployment Checklist (ALWAYS DO THIS FIRST)
```bash
# 1. Check what's running on the server BEFORE deploying
ssh johnt@energyfreedom.team "pm2 list && netstat -tlnp | grep -E '300[01]'"

# 2. Check existing files on server
ssh johnt@energyfreedom.team "ls -la /var/www/srp-canvass-admin/"

# 3. DO NOT touch .env files - rsync with --exclude='.env*' handles this
```

### Flutter Android APK
```bash
flutter build apk --release
# APK hosted on Cloudflare Pages with marketing site
# Upload to app/ folder in energyfreedom-website, then deploy
# Download URL: https://energyfreedom.team/app/SRPCanvass.apk
```

### Flutter Web (Cloudflare Pages)
```bash
cd /path/to/srp_canvass_flutter
flutter build web --release --base-href=/

# Deploy to Cloudflare Pages
wrangler pages deploy build/web --project-name srpcanvass-web --branch main --commit-dirty=true

# URL: https://canvass.energyfreedom.team
```

### Admin Dashboard (Vercel)
```bash
cd /path/to/srp-canvass-admin

# Deploy to Vercel (no GitHub needed)
vercel --prod

# Environment variables are configured in Vercel dashboard
# URL: https://admin.energyfreedom.team
```

**Note:** Environment variables (SUPABASE, MAPBOX, TELNYX) are set in Vercel project settings.

### Solar Savings Dashboard
```bash
cd /path/to/solar-savings-dashboard
npm run build
cp -r public .next/standalone/public 2>/dev/null || true
cp -r .next/static .next/standalone/.next/static
rsync -avz --exclude='.env*' .next/standalone/ root@100.85.1.68:/var/www/solar-dashboard/
rsync -avz .next/static/ root@100.85.1.68:/var/www/solar-dashboard/.next/static/
ssh root@100.85.1.68 "cd /var/www/solar-dashboard && PORT=3001 nohup node server.js > /dev/null 2>&1 &"
# URL: https://solar.energyfreedom.team
```

### Events Public
```bash
cd /path/to/srp-events-public
npm run build
cp -r public .next/standalone/public 2>/dev/null || true
cp -r .next/static .next/standalone/.next/static
rsync -avz --exclude='.env*' .next/standalone/ johnt@energyfreedom.team:/var/www/srp-events-public/
rsync -avz .next/static/ johnt@energyfreedom.team:/var/www/srp-events-public/.next/static/
ssh johnt@energyfreedom.team "pm2 restart srp-events-public || (cd /var/www/srp-events-public && PORT=3003 pm2 start server.js --name srp-events-public && pm2 save)"
# URL: https://events.energyfreedom.team
```

### Ballot Request Form
```bash
cd /path/to/srp-ballot-form
npm run build
cp -r public .next/standalone/public 2>/dev/null || true
cp -r .next/static .next/standalone/.next/static
rsync -avz --exclude='.env*' .next/standalone/ johnt@energyfreedom.team:/var/www/srp-ballot-form/
rsync -avz .next/static/ johnt@energyfreedom.team:/var/www/srp-ballot-form/.next/static/
ssh johnt@energyfreedom.team "pm2 restart srp-ballot-form || (cd /var/www/srp-ballot-form && PORT=3002 pm2 start server.js --name srp-ballot-form && pm2 save)"
# URL: https://energyfreedom.team/register
```

### Marketing Website (Cloudflare Pages)
```bash
cd /path/to/energyfreedom-website

# Build with Astro (outputs to dist/)
npm run build

# Deploy to Cloudflare Pages (exclude APK files >25MB)
mkdir -p /tmp/ef-deploy
rsync -av --exclude='*.apk' dist/ /tmp/ef-deploy/
wrangler pages deploy /tmp/ef-deploy --project-name energyfreedom-team --branch main --commit-dirty=true

# URLs: https://energyfreedom.team, https://energyfreedomteam.com, https://energyfreedomteam.org
```

### Docs Site (Cloudflare Pages)
```bash
# Deploy docs to Cloudflare Pages
wrangler pages deploy /path/to/docs --project-name docs-energyfreedom --branch main --commit-dirty=true

# URL: https://docs.energyfreedom.team
```

### Deployment Safety Rules
1. **NEVER use `rsync --delete`** without `--exclude='.env*'` - this will destroy server config
2. **ALWAYS investigate server state first** - check running processes before deploying
3. **ALWAYS backup .env files** before any deployment
4. **Check process ownership** - if a process is owned by root, you may need sudo to stop it
5. **Verify port availability** - check `netstat -tlnp | grep 3000` before starting new processes

## Demo Account
For App Store/Play Store review:
- Email: `demo@apple-review.com`
- Password: `AppleReview2024!`

Demo mode bypasses authentication, creates a fake canvasser profile, and disables sync operations.

## Project-Specific Notes

### Flutter (srp_canvass_flutter/)
- Has its own CLAUDE.md with detailed Flutter-specific guidance
- Version bumping: Update `version: X.Y.Z+BUILD` in pubspec.yaml
- Uses flutter_map (not google_maps_flutter) for maps

### Admin Dashboard (srp-canvass-admin/)
- Uses shadcn/ui components (in src/components/ui/)
- Maps use Leaflet with Mapbox tiles (dynamic import required - no SSR)
- Route groups: (dashboard) for protected routes, login for auth
- Requires .env.local with SUPABASE and MAPBOX tokens

### Solar Detection (solar-detection/)
- Python/Jupyter notebooks for ML-based solar panel detection
- Runs in Google Colab with GPU
- Uses YOLOv8 model (finloop/yolov8s-seg-solar-panels)
- Updates `voters.has_solar` column in shared Supabase database
- See solar-detection/CLAUDE.md for detailed pipeline info

### Clean Energy Friends (clean_energy_team/)
- Simplified canvass app - "canvassing for people who hate canvassing"
- No login required (device ID based)
- Contact sync finds voters user already knows
- Funnels users to full SRP Canvass app after 5 contacts
- See clean_energy_team/CLAUDE.md for detailed guidance

### Events Public (srp-events-public/)
- Public-facing event listing page at events.energyfreedom.team
- Shows public/both visibility events from shared events table
- Guest signup with email notifications via Resend

### OpenShortLink (qr.energyfreedom.team)
- URL shortener with QR code generation
- Hosted on Cloudflare Workers
- Admin dashboard: https://qr.energyfreedom.team/dashboard/login
- Use for campaign short links and QR codes on printed materials

## Design Context

### Brand Identity

**Brand Personality:** Bold, Grassroots, Warm

The Energy Freedom brand is a campaign voice—confident and action-oriented without being corporate. It speaks to Arizona neighbors fighting for energy independence, not a faceless utility.

### Color Palette

| Token | Hex | Use |
|-------|-----|-----|
| **Copper** | `#de6d48` | Primary CTA, accents, energy |
| **Copper Dark** | `#c9512d` | Hover states, emphasis |
| **Sage** | `#587758` | Secondary accent, success states |
| **Sage Dark** | `#3d5a3d` | Hover states |
| **Desert 50** | `#f9f7f4` | Light backgrounds |
| **Desert 200** | `#e2d9cb` | Cards, borders |
| **Midnight** | `#1e2328` | Text, dark sections |
| **Midnight 600** | `#4e5d69` | Secondary text |

### Typography

| Role | Font | Weight | Notes |
|------|------|--------|-------|
| **Display** | Archivo Black | 400 | Headlines, uppercase, -0.02em tracking |
| **Serif** | Source Serif 4 | 400/600/700 | Body copy, editorial warmth |
| **Body** | DM Sans | 400/500/600/700 | UI text, labels, buttons |

### Design Patterns

- **Corners:** 12px radius for cards/buttons (Material 3 style)
- **Spacing:** 16px base, 24px for sections, 8px for tight groupings
- **Cards:** Subtle elevation (2dp), rounded corners, full-bleed color for emphasis
- **Buttons:** Solid copper fill, slight lift on hover (-2px), press down on active (+1px)
- **Accents:** Copper left-border for pull quotes, diagonal stripe backgrounds
- **Dark sections:** Midnight (#1e2328) for contrast blocks and footers

### Emotional Design by Surface

| Surface | Audience | Emotional Job | Tone | Key Feeling |
|---------|----------|---------------|------|-------------|
| **Campaign Site** | Voters (strangers) | Interrupt apathy, create stakes | Urgent, righteous | "Wait, this matters" |
| **Admin Portal** | Campaign managers | Enable oversight, reduce anxiety | Clear, dense, calm | "I see everything" |
| **SRP Canvass App** | Committed volunteers | Equip for action | Empowering, supportive | "I've got what I need" |
| **Clean Energy Friends** | Casual volunteers | Remove barriers | Reassuring, low-pressure | "That was easy" |
| **Solar Dashboard** | Homeowners | Show value, build trust | Informative, optimistic | "This makes sense" |

### Design Principles

1. **Bold over timid** — Use confident typography, strong color blocks, and clear hierarchy. Don't hedge with pastels or whisper with thin fonts.

2. **Warm over corporate** — Desert tones and copper accents feel human and Arizonan. Avoid sterile blues and grays that read as generic tech.

3. **Action over decoration** — Every element should drive toward a goal. Animations should feel satisfying (checkbox bounce, button press) not gratuitous.

4. **Dense for operators, simple for volunteers** — Admin tools can pack information tightly; volunteer-facing apps should feel effortless with generous spacing.

5. **Grassroots authenticity** — This is neighbors helping neighbors, not a faceless corporation. Keep copy direct, personal, and action-oriented.

### Implementation Notes

**Flutter Apps:**
- Use the existing `ColorPalette` in `color_themes.dart` (already aligned with brand)
- Default theme: `energyFreedom` palette
- Support light/dark/system modes
- Material 3 with `useMaterial3: true`

**Next.js Apps:**
- Use Tailwind with custom `copper`, `sage`, `desert`, `midnight` color scales
- Import DM Sans and Source Serif 4 from Google Fonts
- shadcn/ui components with copper accent overrides

**Iconography:**
- Lightning bolt (⚡) as brand mark
- Material Icons for UI consistency
- Filled tonal icon buttons for primary actions
