# Database Schema

This document describes the shared Supabase PostgreSQL database used by all SRP Platform applications.

## Overview

All apps connect to the same Supabase instance at `api.energyfreedom.team`. The database contains ~664k voter records and supports role-based access via Row-Level Security (RLS) policies.

## Tables

### Core Voter Data

#### `voters`
Primary table containing voter/property owner information. Each record represents a unique property-voter combination.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `unique_id` | varchar(16) | Unique identifier (format: `SRP_xxxxxxxxxxxx`) |
| `voter_id` | text | Arizona voter ID (if matched) |
| `apn` | text | Assessor's Parcel Number |
| `cso_parcel` | text | SRP CSO parcel identifier |
| `district` | integer | SRP district (1-10) |
| **Owner Info** |||
| `owner_name` | text | Full name as on property deed |
| `owner_code` | text | `A` = Owner, `T` = Trustee |
| `owner_key` | text | Generated: lowercase normalized `first|middle|last` for grouping |
| `first_name` | text | Parsed first name |
| `middle_name` | text | Parsed middle name |
| `last_name` | text | Parsed last name |
| **Contact Info** |||
| `phone` | text | Landline phone |
| `cell_phone` | text | Mobile phone |
| `phone_hash` | text | SHA256 hash for contact matching |
| `cell_phone_hash` | text | SHA256 hash for contact matching |
| **Property Address** |||
| `street_num` | text | Street number |
| `street_dir` | text | Street direction (N/S/E/W) |
| `street_name` | text | Street name |
| `city` | text | City |
| `zip` | text | ZIP code |
| `latitude` | numeric | Geocoded latitude |
| `longitude` | numeric | Geocoded longitude |
| **Mailing Address** |||
| `mail_address` | text | Mailing street address |
| `mail_city` | text | Mailing city |
| `mail_state` | text | Mailing state |
| `mail_zip` | text | Mailing ZIP |
| `lives_elsewhere` | boolean | True if mailing differs from property |
| **Voting Info** |||
| `votes` | numeric | Individual's voting acres |
| `total_property_votes` | numeric | Total property voting acres |
| `total_owner_votes` | numeric(10,2) | Aggregated votes across all owner properties |
| `owner_property_count` | integer | Number of properties owned by this owner |
| `assn_votes` | numeric | Association votes (individual) |
| `assn_total_votes` | numeric | Association votes (total) |
| `party` | text | Political party affiliation |
| `voter_age` | integer | Age in years |
| `gender` | text | `M`, `F`, or null |
| `registration_date` | text | Voter registration date |
| `vote_frequency` | text | Voting frequency score |
| `voter_score` | integer | Propensity score |
| `is_pevl` | boolean | On Permanent Early Voter List |
| `is_srp_registered` | boolean | Registered to vote in SRP elections |
| `user_reported_registered` | boolean | User confirmed SRP registration |
| `user_reported_registered_at` | timestamp | When user confirmed |
| **Canvass Data** |||
| `canvass_result` | text | Latest canvass outcome (see values below) |
| `canvass_notes` | text | Canvasser notes |
| `canvass_date` | timestamp | When canvassed |
| `contact_attempts` | integer | Number of contact attempts |
| `contact_attempt_count` | integer | Computed contact count |
| `last_contact_attempt` | timestamp | Last attempt timestamp |
| `last_contact_method` | text | `call`, `text`, or `door` |
| `last_contact_date` | timestamp | Last successful contact |
| `voicemail_left` | boolean | Voicemail left on last call |
| `contact_history` | jsonb | Legacy embedded history |
| **SMS Response** |||
| `last_sms_response` | text | Last SMS reply received |
| `last_sms_response_date` | timestamp | When reply received |
| **Solar Detection** |||
| `has_solar` | boolean | ML-detected rooftop solar |
| `solar_verified` | boolean | Human-verified solar status |
| `solar_verified_at` | timestamp | Verification timestamp |
| `solar_verified_by` | uuid | User who verified |
| **Search** |||
| `search_vector` | tsvector | Full-text search index |
| **Metadata** |||
| `residence_address` | text | Full formatted residence |
| `enriched_at` | timestamp | Last data enrichment |
| `cet_lookup_at` | timestamp | Last Clean Energy Team lookup |
| `created_at` | timestamp | Record creation |
| `updated_at` | timestamp | Last update |

**Canvass Result Values:**
- **Positive:** `Supportive`, `Strong Support`, `Leaning`, `Willing to Volunteer`, `Requested Sign`
- **Negative:** `Opposed`, `Strongly Opposed`, `Do Not Contact`, `Refused`
- **Neutral:** `Undecided`, `Needs Info`, `Callback Requested`
- **Contact outcomes:** `No Answer`, `Wrong Number`, `Left Voicemail`, `Not Home`

---

### User Management

#### `user_profiles`
User accounts linked to Supabase Auth.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key (matches auth.users.id) |
| `email` | text | User email |
| `full_name` | text | Display name |
| `role` | text | `pending`, `canvasser`, `team_lead`, `admin`, `super_admin` |
| `role_id` | uuid | Foreign key to roles table (for custom roles) |
| `districts` | integer[] | Assigned SRP districts |
| `approved_at` | timestamp | When approved for canvassing |
| `approved_by` | uuid | Admin who approved |
| `created_at` | timestamp | Account creation |
| `last_seen_events_at` | timestamp | Last viewed events page |
| `show_on_leaderboard` | boolean | Opt-in for public leaderboard |
| `leaderboard_display_name` | text | Custom leaderboard name |

#### `user_devices`
FCM tokens for push notifications.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `user_id` | uuid | User reference |
| `fcm_token` | text | Firebase Cloud Messaging token |
| `device_type` | text | `android` or `ios` |
| `created_at` | timestamp | Token registration |
| `updated_at` | timestamp | Last token update |

---

### Role-Based Access Control (RBAC)

#### `roles`
Custom role definitions with platform scoping.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `name` | text | Role identifier |
| `display_name` | text | Human-readable name |
| `description` | text | Role description |
| `priority` | integer | Role precedence (higher = more access) |
| `platform` | text | `admin`, `mobile`, or `both` |
| `is_system` | boolean | System-defined (cannot be deleted) |
| `is_active` | boolean | Whether role is usable |
| `created_at` | timestamp | Creation |
| `updated_at` | timestamp | Last update |

#### `permissions`
Granular permission definitions.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `resource` | text | Resource name (e.g., `voters`, `cut_lists`) |
| `action` | text | Action type (`read`, `create`, `update`, `delete`) |
| `display_name` | text | Human-readable name |
| `description` | text | Permission description |
| `category` | text | Grouping category |
| `created_at` | timestamp | Creation |

#### `role_permissions`
Links roles to permissions.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `role_id` | uuid | Role reference |
| `permission_id` | uuid | Permission reference |
| `created_at` | timestamp | Assignment time |

---

### Contact Tracking

#### `contact_history`
Individual contact attempts with voters.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `unique_id` | text | Voter unique_id (preferred FK) |
| `visitor_id` | text | Legacy voter ID |
| `contact_method` | text | `Call`, `Phone Call`, `Text`, `Text Message`, `Door Knock` |
| `result` | text | Contact outcome |
| `notes` | text | Canvasser notes |
| `left_lit` | boolean | Whether literature was left at door (door knocks only) |
| `contacted_at` | timestamp | When contact occurred |
| `contacted_by` | uuid | User who made contact |
| `outreach_user_id` | uuid | Clean Energy Team user (if applicable) |
| `created_at` | timestamp | Record creation |

**Notes:**
- 2026-02-05: Bulk update set `left_lit = true` for all 1,456 District 6 door knock records

#### `voice_notes`
Audio recordings attached to voter contacts.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `voter_unique_id` | text | Voter unique_id |
| `contact_history_id` | uuid | Related contact record |
| `audio_url` | text | Supabase storage path |
| `duration_seconds` | integer | Recording length |
| `transcription` | text | AI-generated transcript |
| `recorded_by` | uuid | User who recorded |
| `recorded_at` | timestamp | Recording timestamp |

#### `callback_reminders`
Scheduled callback notifications for canvassers.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `user_id` | uuid | Canvasser to notify |
| `voter_unique_id` | text | Voter to call back |
| `reminder_at` | timestamp | When to send reminder |
| `sent` | boolean | Whether reminder was sent |
| `created_at` | timestamp | When scheduled |

---

### Geographic Segmentation

#### `cut_lists`
Geographic voter segments defined by polygon boundaries or demographic filters.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `name` | text | Cut list name |
| `description` | text | Optional description |
| `boundary_polygon` | jsonb | Array of {lat, lng} coordinates |
| `demographics_filters` | jsonb | Demographic filter criteria |
| `map_filters` | jsonb | Map-based filter criteria |
| `districts` | integer[] | District restrictions |
| `visibility` | text | `personal`, `public`, or `team` |
| `voter_count` | integer | Cached count of voters |
| `created_by` | uuid | Creator user ID |
| `created_at` | timestamp | Creation timestamp |
| `updated_at` | timestamp | Last update |

#### `cut_list_assignments`
Assigns canvassers to cut lists.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `cut_list_id` | uuid | Cut list reference |
| `user_id` | uuid | Assigned canvasser |
| `assigned_at` | timestamp | Assignment timestamp |
| `assigned_by` | uuid | Admin who assigned |

#### `cut_list_voters`
Explicit voter membership in cut lists (for non-polygon cut lists).

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `cut_list_id` | uuid | Cut list reference |
| `voter_unique_id` | text | Voter unique_id |
| `added_at` | timestamp | When voter was added |

---

### Invite System

#### `invites`
Invite codes for self-service canvasser signup.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `code` | varchar(12) | Unique invite code |
| `created_by` | uuid | Admin who created |
| `districts` | integer[] | Districts to assign on signup |
| `cut_list_id` | uuid | Optional cut list to assign |
| `max_uses` | integer | Maximum signups allowed |
| `uses_count` | integer | Current signup count |
| `expires_at` | timestamp | Optional expiration |
| `is_active` | boolean | Whether code is usable |
| `app_type` | text | `canvass` or `cet` (Clean Energy Team) |
| `source` | text | `admin`, `team_lead`, or `cef_referral` |
| `outreach_user_id` | uuid | CET user who shared (for referrals) |
| `created_at` | timestamp | Creation timestamp |

#### `invite_uses`
Tracks which users signed up with which invite code.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `invite_id` | uuid | Invite code used |
| `user_id` | uuid | User who signed up |
| `used_at` | timestamp | Signup timestamp |

---

### Communication Templates

#### `text_templates`
SMS message templates with variable placeholders.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `name` | text | Template name |
| `category` | text | `intro` or `follow_up` |
| `message` | text | Message body with `{firstName}`, `{lastName}` placeholders |
| `district` | text | `all` or specific district number |
| `candidate_id` | uuid | Optional candidate filter |
| `position` | text | Optional position filter |
| `icon_name` | text | Flutter icon identifier |
| `display_order` | integer | Sort order |
| `is_active` | boolean | Whether template is available |
| `is_default` | boolean | System default template |
| `created_by` | uuid | Creator |
| `created_at` | timestamp | Creation |
| `updated_at` | timestamp | Last update |

#### `text_template_user_assignments`
Assigns templates to specific users.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `template_id` | uuid | Template reference |
| `user_id` | uuid | User reference |
| `assigned_by` | uuid | Admin who assigned |
| `assigned_at` | timestamp | Assignment time |

#### `text_template_cut_list_assignments`
Assigns templates to cut lists.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `template_id` | uuid | Template reference |
| `cut_list_id` | uuid | Cut list reference |
| `assigned_by` | uuid | Admin who assigned |
| `assigned_at` | timestamp | Assignment time |

#### `call_scripts`
Phone call script headers.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `name` | text | Script name |
| `category` | text | Script category |
| `district` | text | District filter |
| `candidate_id` | uuid | Optional candidate filter |
| `position` | text | Optional position filter |
| `display_order` | integer | Sort order |
| `is_active` | boolean | Whether script is available |
| `is_default` | boolean | System default script |
| `created_by` | uuid | Creator |
| `created_at` | timestamp | Creation |
| `updated_at` | timestamp | Last update |

#### `call_script_sections`
Individual sections within a call script.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `script_id` | uuid | Parent script |
| `title` | text | Section title |
| `content` | text | Script content with placeholders |
| `tips` | text[] | Helper tips for canvassers |
| `icon_name` | text | Flutter icon identifier |
| `color` | text | Hex color code |
| `display_order` | integer | Sort order within script |
| `created_at` | timestamp | Creation |

#### `call_script_user_assignments`
Assigns scripts to specific users.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `script_id` | uuid | Script reference |
| `user_id` | uuid | User reference |
| `assigned_by` | uuid | Admin who assigned |
| `assigned_at` | timestamp | Assignment time |

#### `call_script_cut_list_assignments`
Assigns scripts to cut lists.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `script_id` | uuid | Script reference |
| `cut_list_id` | uuid | Cut list reference |
| `assigned_by` | uuid | Admin who assigned |
| `assigned_at` | timestamp | Assignment time |

#### `cef_templates`
Text templates for Clean Energy Friends app.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `name` | text | Template name |
| `content` | text | Message content |
| `category` | text | Template category |
| `districts` | integer[] | District restrictions |
| `is_active` | boolean | Whether available |
| `display_order` | integer | Sort order |
| `is_default` | boolean | System default |
| `created_at` | timestamp | Creation |
| `updated_at` | timestamp | Last update |

---

### Candidates

#### `candidates`
Campaign candidate information.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `name` | text | Candidate name |
| `district` | text | District running in |
| `position` | text | Position sought |
| `organization` | text | Affiliated organization |
| `website` | text | Campaign website |
| `is_active` | boolean | Currently active |
| `created_at` | timestamp | Creation |

---

### Events

#### `event_types`
Configurable event type definitions.

| Column | Type | Description |
|--------|------|-------------|
| `id` | text | Primary key (e.g., `canvass`, `phone_bank`) |
| `display_name` | text | Human-readable name |
| `description` | text | Type description |
| `icon` | text | Icon identifier |
| `color` | text | Color for UI display |
| `default_virtual` | boolean | Default to virtual |
| `sort_order` | integer | Display order |
| `is_active` | boolean | Whether type is available |
| `created_at` | timestamp | Creation |

#### `events`
Calendar events for canvassing, training, etc.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `name` | text | Event title |
| `description` | text | Event description |
| `event_type` | text | Reference to event_types.id |
| `start_time` | timestamp | Event start |
| `end_time` | timestamp | Event end |
| `is_virtual` | boolean | Online event |
| `location_name` | text | Venue name |
| `location_address` | text | Physical address |
| `virtual_link` | text | Video call URL |
| `max_volunteers` | integer | Capacity limit |
| `district` | integer | District filter |
| `cut_list_id` | uuid | Associated cut list |
| `visibility` | text | `internal`, `public`, or `both` |
| `is_active` | boolean | Event is live |
| `created_by` | uuid | Creator |
| `created_at` | timestamp | Creation |
| `updated_at` | timestamp | Last update |

#### `event_signups`
Event registrations and attendance.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `event_id` | uuid | Event reference |
| `user_id` | uuid | Registered user (null for guests) |
| `guest_name` | text | Guest name (public signups) |
| `guest_email` | text | Guest email |
| `guest_phone` | text | Guest phone |
| `status` | text | `registered`, `attended`, `no_show`, `cancelled` |
| `registered_at` | timestamp | Registration time |
| `checked_in_at` | timestamp | Check-in time |
| `reminder_24h_sent` | boolean | 24-hour reminder sent |
| `reminder_1h_sent` | boolean | 1-hour reminder sent |

#### `event_results`
Per-volunteer performance tracking for events.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `event_id` | uuid | Event reference |
| `user_id` | uuid | Volunteer user |
| `contacts_made` | integer | Number of contacts |
| `positive_responses` | integer | Positive outcomes |
| `acres_contacted` | numeric(10,2) | Total acre-votes contacted |
| `notes` | text | Session notes |
| `created_at` | timestamp | Creation |
| `updated_at` | timestamp | Last update |

---

### Volunteer Matching

#### `volunteer_contacts_consent`
Tracks user consent for contact sync feature.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `user_id` | uuid | User who consented |
| `consented_at` | timestamp | When consent given |
| `revoked_at` | timestamp | When revoked (if applicable) |
| `last_sync_at` | timestamp | Last contact sync |
| `contact_count` | integer | Total contacts synced |
| `match_count` | integer | Voters matched |
| `device_name` | text | Device that synced |
| `notification_preference` | text | `immediate`, `digest`, or `none` |
| `created_at` | timestamp | Creation |

#### `volunteer_contact_hashes`
Stores phone hashes from synced contacts for matching.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `user_id` | uuid | User who synced |
| `phone_hash` | text | SHA256 hash of phone number |
| `name_hint` | text | Contact name for matching confidence |
| `synced_at` | timestamp | When hash was synced |

#### `voter_connections`
Links canvassers to voters they know personally.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `voter_id` | uuid | Voter record ID |
| `user_id` | uuid | Canvasser who knows this voter |
| `confidence` | match_confidence | `high`, `medium`, `low` |
| `matched_at` | timestamp | When match was found |
| `assigned` | boolean | Officially assigned to canvass |
| `assigned_at` | timestamp | Assignment timestamp |
| `assigned_by` | uuid | Admin who assigned |
| `locked` | boolean | Prevent reassignment |
| `last_contact_attempt` | timestamp | Last outreach |

---

### Clean Energy Team (Outreach App)

#### `outreach_users`
Device-based users for the simplified Clean Energy Friends app.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `device_id` | text | Unique device identifier |
| `phone_number` | text | User's phone (if provided) |
| `phone_hash` | text | SHA256 hash for matching |
| `first_name` | text | User's first name |
| `contact_count` | integer | Number of contacts reached |
| `matched_voter_id` | text | If user is also a voter |
| `has_contact_consent` | boolean | Consented to contact sync |
| `has_synced_contacts` | boolean | Has completed contact sync |
| `synced_phone_hashes` | text[] | Hashes of synced contacts (legacy) |
| `is_verified_voter` | boolean | Confirmed as SRP voter |
| `has_seen_upgrade_banner` | boolean | Shown full app promotion |
| `created_at` | timestamp | First app open |
| `last_active_at` | timestamp | Last activity |

#### `outreach_contact_hashes`
Stores phone hashes from synced contacts for each CEF device.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `device_id` | text | Device that synced this contact |
| `phone_hash` | text | SHA256 hash of contact's phone |
| `synced_at` | timestamp | When hash was synced |

**Unique constraint:** (`device_id`, `phone_hash`)

#### `cet_users`
Legacy CET device users (may be deprecated in favor of outreach_users).

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `device_id` | text | Device identifier |
| `invite_id` | uuid | Invite code used |
| `invite_code` | text | Code string |
| `contact_count` | integer | Contacts made |
| `created_at` | timestamp | First use |
| `last_active_at` | timestamp | Last activity |

#### `cet_invite_uses`
Tracks CET invite code usage.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `invite_id` | uuid | Invite used |
| `device_id` | text | Device that used it |
| `cet_user_id` | uuid | CET user record |
| `used_at` | timestamp | When used |

#### `registration_lookups`
Tracks self-registration check attempts from the CEF app.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `device_id` | text | Device that performed lookup |
| `email` | text | Email entered for lookup |
| `first_name` | text | First name entered |
| `last_name` | text | Last name entered |
| `street_address` | text | Address entered |
| `dob_year` | integer | Year of birth |
| `lookup_result` | text | `found`, `not_found`, `multiple`, `error` |
| `voter_unique_id` | text | Matched voter if found |
| `ip_address` | text | Request IP |
| `user_agent` | text | Browser/device info |
| `created_at` | timestamp | When lookup was performed |

#### `cef_requests`
Tracks invite code access requests from CEF app users who don't have an invite code.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `device_id` | text | Device requesting access |
| `name` | text | User-provided name |
| `email` | text | User-provided email for follow-up |
| `phone` | text | Optional phone number |
| `reason` | text | Optional reason for wanting access |
| `status` | text | `pending`, `approved`, `rejected` |
| `approved_by` | uuid | Admin who approved/rejected |
| `approved_at` | timestamp | When approved |
| `rejected_at` | timestamp | When rejected |
| `rejection_reason` | text | Optional rejection reason |
| `invite_code` | text | Invite code assigned when approved (FK to invites.code) |
| `notes` | text | Admin notes |
| `created_at` | timestamp | When request was submitted |
| `updated_at` | timestamp | Last update |

**RPC Functions:**
- `submit_cef_invite_request(device_id, name, email, phone?, reason?)` - Submit access request
- `check_cef_request_status(device_id)` - Check pending/approved/rejected status

---

### Ballot Requests

#### `ballot_requests`
Online ballot request form submissions.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `voter_unique_id` | text | Matched voter (if found) |
| **Personal Info** |||
| `first_name` | text | Legal first name |
| `middle_name` | text | Middle name |
| `last_name` | text | Legal last name |
| `suffix` | text | Jr., Sr., etc. |
| `date_of_birth` | date | DOB |
| `birth_state` | text | State of birth |
| `birth_country` | text | Country if not US |
| `az_registered_voter` | boolean | AZ voter registration status |
| `add_to_pevl` | boolean | Request PEVL enrollment |
| **Contact** |||
| `phone` | text | Phone number |
| `email` | text | Email address |
| **Property Address** |||
| `primary_apn` | text | Property APN |
| `primary_address` | text | Property street |
| `primary_city` | text | Property city |
| `primary_state` | text | Property state |
| `primary_zip` | text | Property ZIP |
| `property_is_residence` | boolean | Lives at property |
| **Residence (if different)** |||
| `residence_address` | text | Residence street |
| `residence_city` | text | Residence city |
| `residence_state` | text | Residence state |
| `residence_zip` | text | Residence ZIP |
| **Mailing Address** |||
| `mailing_same_as_property` | boolean | Mail to property |
| `mailing_address` | text | Mailing street |
| `mailing_city` | text | Mailing city |
| `mailing_state` | text | Mailing state |
| `mailing_zip` | text | Mailing ZIP |
| **Trust Info** |||
| `is_trust_property` | boolean | Property in trust |
| `trust_name` | text | Name of trust |
| `trust_for_estate_plan` | boolean | Estate planning trust |
| `trust_estate_person_name` | text | Person trust is for |
| `trustee_type` | text | `self`, `beneficiary`, `appointed` |
| **Submission** |||
| `signature_name` | text | Typed signature |
| `status` | text | `pending`, `processing`, `submitted`, `failed`, `manual_required` |
| `attempts` | integer | Submission attempts |
| `last_attempt_at` | timestamp | Last attempt |
| `last_error` | text | Error message |
| `srp_confirmation` | text | SRP confirmation number |
| `submitted_at` | timestamp | Successful submission |
| **Metadata** |||
| `created_by` | uuid | User who initiated |
| `created_at` | timestamp | Form submission |
| `updated_at` | timestamp | Last update |

---

### Solar Detection

#### `solar_installations`
ML-detected solar panel installations from aerial imagery.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `latitude` | double precision | Panel location latitude |
| `longitude` | double precision | Panel location longitude |
| `confidence_score` | real | Detection confidence (0-1) |
| `panel_area_sqm` | real | Estimated panel area |
| `detection_source` | text | Detection model/source |
| `imagery_date` | date | When imagery was captured |
| `voter_unique_id` | text | Matched voter record |
| `match_distance_m` | real | Distance to matched property |
| `import_batch_id` | text | Import batch identifier |
| `created_at` | timestamp | Detection timestamp |
| `updated_at` | timestamp | Last update |

---

### Solar Dashboard (Energy Monitoring)

These tables power the solar.energyfreedom.team dashboard.

#### `solar_current_state`
Single-row table with current system state.

| Column | Type | Description |
|--------|------|-------------|
| `id` | integer | Always 1 |
| `updated_at` | timestamp | Last update |
| `grid_w` | integer | Current grid power (watts) |
| `solar_w` | integer | Current solar production |
| `battery_w` | integer | Battery charge/discharge |
| `load_w` | integer | Current home load |
| `battery_soc` | smallint | Battery state of charge (%) |
| `is_on_peak` | boolean | Currently on-peak period |
| `current_30min_avg_w` | integer | Rolling 30-minute average |
| `cycle_peak_demand_kw` | numeric(6,2) | Billing cycle peak demand |
| `cycle_e27_running` | numeric(10,2) | Running E-27 cost |
| `cycle_e23_running` | numeric(10,2) | Hypothetical E-23 cost |
| `today_kwh_solar` | numeric(10,3) | Today's solar production |
| `today_kwh_export` | numeric(10,3) | Today's grid export |
| `today_savings` | numeric(10,2) | Today's calculated savings |

#### `power_readings`
Historical power readings (high-frequency).

| Column | Type | Description |
|--------|------|-------------|
| `id` | bigint | Primary key |
| `recorded_at` | timestamp | Reading timestamp |
| `grid_w` | integer | Grid power |
| `solar_w` | integer | Solar production |
| `battery_w` | integer | Battery power |
| `load_w` | integer | Home load |
| `battery_soc` | smallint | Battery SOC |
| `is_on_peak` | boolean | On-peak period |

#### `hourly_stats`
Aggregated hourly statistics.

| Column | Type | Description |
|--------|------|-------------|
| `id` | integer | Primary key |
| `hour_start` | timestamp | Hour beginning |
| `kwh_grid_import` | numeric(10,3) | kWh imported |
| `kwh_grid_export` | numeric(10,3) | kWh exported |
| `kwh_solar` | numeric(10,3) | kWh solar produced |
| `kwh_battery_discharge` | numeric(10,3) | kWh from battery |
| `kwh_battery_charge` | numeric(10,3) | kWh to battery |
| `kwh_load` | numeric(10,3) | kWh consumed |
| `peak_demand_w` | integer | Peak demand |
| `avg_demand_w` | integer | Average demand |
| `tou_period` | text | Time-of-use period |
| `season` | text | Summer/Winter |

#### `billing_cycles`
Monthly billing cycle summaries.

| Column | Type | Description |
|--------|------|-------------|
| `id` | integer | Primary key |
| `cycle_start` | date | Billing cycle start |
| `cycle_end` | date | Billing cycle end |
| `peak_demand_kw` | numeric(6,2) | Peak demand with battery |
| `peak_demand_without_battery_kw` | numeric(6,2) | Hypothetical peak |
| `kwh_grid_import` | numeric(10,3) | Total imported |
| `kwh_grid_export` | numeric(10,3) | Total exported |
| `kwh_load` | numeric(10,3) | Total consumed |
| `kwh_on_peak_import` | numeric(10,3) | On-peak imports |
| `kwh_off_peak_import` | numeric(10,3) | Off-peak imports |
| `e27_energy_cost` | numeric(10,2) | E-27 energy charges |
| `e27_demand_cost` | numeric(10,2) | E-27 demand charges |
| `e27_total_cost` | numeric(10,2) | Total E-27 bill |
| `e23_hypothetical_cost` | numeric(10,2) | What E-23 would cost |
| `total_savings` | numeric(10,2) | Total savings |
| `demand_savings` | numeric(10,2) | Demand savings |
| `is_final` | boolean | Cycle is complete |

#### `demand_windows`
30-minute demand window tracking.

| Column | Type | Description |
|--------|------|-------------|
| `id` | integer | Primary key |
| `window_start` | timestamp | Window start |
| `window_end` | timestamp | Window end |
| `avg_demand_w` | integer | Average demand |
| `max_load_w` | integer | Maximum load |
| `is_on_peak` | boolean | On-peak window |
| `billing_cycle_start` | date | Billing cycle |

#### `solar_system_config`
System configuration (single row).

| Column | Type | Description |
|--------|------|-------------|
| `id` | integer | Always 1 |
| `battery_capacity_kwh` | numeric(6,2) | Battery size |
| `battery_min_soc` | smallint | Minimum SOC |
| `essentials_load_w` | integer | Essential loads |
| `minimum_load_w` | integer | Minimum load |
| `billing_cycle_start_day` | smallint | Billing cycle day |
| `monthly_service_charge` | numeric(6,2) | Service charge |

---

### Notifications & Audit

#### `notifications`
In-app notifications for users.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `user_id` | uuid | Recipient user |
| `type` | text | Notification type (`info`, `signup`, `assignment`, etc.) |
| `title` | text | Notification title |
| `message` | text | Notification body |
| `read` | boolean | Has been read |
| `data` | jsonb | Additional payload |
| `created_at` | timestamp | Creation |

#### `audit_logs`
Activity tracking for admin actions.

| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `user_id` | uuid | Acting user |
| `user_email` | text | User email (denormalized) |
| `action` | text | `login`, `create`, `update`, `delete`, `export`, etc. |
| `entity_type` | text | `user`, `voter`, `cut_list`, etc. |
| `entity_id` | text | Affected entity ID |
| `entity_name` | text | Human-readable entity name |
| `old_values` | jsonb | Previous state |
| `new_values` | jsonb | New state |
| `details` | jsonb | Additional context |
| `created_at` | timestamp | When action occurred |

---

### Configuration

#### `_config`
Simple key-value configuration store.

| Column | Type | Description |
|--------|------|-------------|
| `key` | text | Configuration key |
| `value` | text | Configuration value |

---

## Custom Types

#### `match_confidence`
Enum type for voter connection confidence levels.

```sql
CREATE TYPE match_confidence AS ENUM ('high', 'medium', 'low');
```

---

## Key Functions

The database includes several security-definer functions for complex operations:

| Function | Description |
|----------|-------------|
| `check_permission(resource, action)` | Checks if current user has permission |
| `can_access_admin_dashboard()` | Returns true if user can access admin |
| `assign_voter_connection(voter_id, user_id)` | Assigns a voter connection |
| `bulk_assign_connections(assignments)` | Batch assign connections |
| `bulk_enrich_voters(updates)` | Batch update voter data |
| `create_cef_referral_code(device_id)` | Generate CEF referral code |
| `get_all_outreach_contact_matches()` | Get all CEF contact matches |
| `estimate_voter_count()` | Fast voter count estimate |

---

## Relationships

```
voters
  └── contact_history (via unique_id)
  └── voice_notes (via voter_unique_id)
  └── voter_connections (via voter_id)
  └── callback_reminders (via voter_unique_id)
  └── ballot_requests (via voter_unique_id)
  └── cut_list_voters (via voter_unique_id)
  └── solar_installations (via voter_unique_id)

user_profiles
  └── contact_history (via contacted_by)
  └── cut_list_assignments (via user_id)
  └── invite_uses (via user_id)
  └── event_signups (via user_id)
  └── event_results (via user_id)
  └── voter_connections (via user_id)
  └── volunteer_contact_hashes (via user_id)
  └── volunteer_contacts_consent (via user_id)
  └── notifications (via user_id)
  └── audit_logs (via user_id)
  └── user_devices (via user_id)
  └── roles (via role_id)

roles
  └── role_permissions (via role_id)
  └── user_profiles (via role_id)

permissions
  └── role_permissions (via permission_id)

cut_lists
  └── cut_list_assignments (via cut_list_id)
  └── cut_list_voters (via cut_list_id)
  └── invites (via cut_list_id)
  └── events (via cut_list_id)
  └── text_template_cut_list_assignments (via cut_list_id)
  └── call_script_cut_list_assignments (via cut_list_id)

invites
  └── invite_uses (via invite_id)
  └── cet_invite_uses (via invite_id)

events
  └── event_signups (via event_id)
  └── event_results (via event_id)
  └── event_types (via event_type)

candidates
  └── text_templates (via candidate_id)
  └── call_scripts (via candidate_id)

call_scripts
  └── call_script_sections (via script_id)
  └── call_script_user_assignments (via script_id)
  └── call_script_cut_list_assignments (via script_id)

text_templates
  └── text_template_user_assignments (via template_id)
  └── text_template_cut_list_assignments (via template_id)

outreach_users
  └── contact_history (via outreach_user_id)
  └── outreach_contact_hashes (via device_id)
  └── registration_lookups (via device_id)
  └── cef_requests (via device_id)
  └── invites (via outreach_user_id)
```

---

## Row-Level Security

RLS policies enforce access control:

- **Canvassers** can only see voters in their assigned districts and cut lists
- **Team leads** can see all voters in their districts
- **Admins** have full access to all data
- **Pending users** cannot access voter data until approved

---

## Indexes

Key indexes for performance:

- `voters(unique_id)` - Primary lookup
- `voters(district)` - District filtering
- `voters(latitude, longitude)` - Geo queries
- `voters(phone_hash)`, `voters(cell_phone_hash)` - Contact matching
- `voters(owner_key)` - Owner grouping
- `voters(search_vector)` - Full-text search (GIN)
- `contact_history(unique_id)` - Contact lookups
- `cut_list_assignments(user_id)` - User assignments
- `outreach_contact_hashes(device_id, phone_hash)` - Contact matching
