# SRP Platform

Parent repository for all Energy Freedom Team projects for the SRP Council District 6 campaign.

## 🏗️ Repository Structure

This is a **monorepo** containing all platform components as Git submodules:

### Main Applications

- **[energyfreedom-website](https://github.com/energy-freedom-team/energyfreedom-website)** - Marketing website + blog (Astro 5.0)
  - Live: https://energyfreedom.team
  - Also: energyfreedomteam.com, energyfreedomteam.org

- **[srp-canvass-admin](https://github.com/energy-freedom-team/srp-canvass-admin)** - Admin dashboard (Next.js 15)
  - Live: https://admin.energyfreedom.team

- **[srp_canvass_flutter](https://github.com/energy-freedom-team/srp-canvass-flutter)** - Full-featured canvassing app (Flutter)
  - iOS App Store, Android APK, Web, macOS

- **[clean_energy_team](https://github.com/energy-freedom-team/clean-energy-team)** - Simplified canvass app (Flutter)
  - iOS App Store: https://apps.apple.com/us/app/clean-energy-friends/id6757442071

- **[solar-savings-dashboard](https://github.com/energy-freedom-team/energy-freedom-solar-dashboard)** - Real-time solar calculator (Next.js 15)
  - Live: https://solar.energyfreedom.team

### Shared Infrastructure

- **CLAUDE.md** - AI assistant context and deployment instructions
- **DATABASE_SCHEMA.md** - Supabase database schema documentation
- **SRP_VOTING_SYSTEM.md** - SRP's acre-based voting system explained
- **supabase/** - Self-hosted Supabase configuration

## 🚀 Getting Started

### Clone with Submodules

```bash
# Clone the parent repo with all submodules
git clone --recurse-submodules https://github.com/energy-freedom-team/srp-platform.git

# Or if already cloned without submodules
git submodule update --init --recursive
```

### Update All Submodules

```bash
# Pull latest changes for all submodules
git submodule update --remote --merge

# Or update specific submodule
cd energyfreedom-website
git pull origin main
```

### Working with Submodules

Each submodule is a full Git repository. To make changes:

```bash
# Navigate to submodule
cd energyfreedom-website

# Make changes, commit, push
git add .
git commit -m "Your changes"
git push

# Back in parent repo, update submodule reference
cd ..
git add energyfreedom-website
git commit -m "Update energyfreedom-website submodule"
git push
```

## 📋 Individual Project Documentation

Each submodule has its own `CLAUDE.md` with:
- Build commands
- Deployment instructions
- Architecture details
- Environment variables

See the individual repositories for project-specific documentation.

## 🗄️ Shared Backend

All projects share a **self-hosted Supabase instance**:

- **API**: https://api.energyfreedom.team
- **Studio**: https://supabase.energyfreedom.team
- **Database**: PostgreSQL 17 with ~664k voter records
- **Auth**: Email, Google OAuth, Apple Sign-In

See `DATABASE_SCHEMA.md` for full schema documentation.

## 🏛️ Campaign Context

This platform supports John Travise and Sara Travise's campaigns for SRP Council District 6.

**Key dates:**
- March 9, 2026 — Voter registration deadline
- March 11, 2026 — Early ballots mailed
- April 7, 2026 — Election Day

**Learn more:** https://energyfreedom.team

---

**Paid for by Committee to Elect John Travise and Paid for by Committee to Elect Sara Travise**
**Authorized by John Travise and Sara Travise**
