# Voice Moderator Onboarding System — Technical Specification

> **Version:** 1.1
> **Date:** February 26, 2026
> **Status:** Design Complete — Validated
> **Framework:** [Carbon](https://github.com/buape/carbon) (Discord bot framework by Buape)
> **Database:** SQLite (file-based, simple deployment)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Roles & Permissions](#3-roles--permissions)
4. [Workflow Stages](#4-workflow-stages)
5. [Bot Commands](#5-bot-commands)
6. [Time Slot System](#6-time-slot-system)
7. [Promotion Pipeline](#7-promotion-pipeline)
8. [Notification System](#8-notification-system)
9. [Data Model & Retention](#9-data-model--retention)
10. [Edge Cases & Failure Modes](#10-edge-cases--failure-modes)
11. [Guidelines Channel Content](#11-guidelines-channel-content)
12. [Clawtributor Role System](#12-clawtributor-role-system)

---

## 1. Overview

A semi-automated voice moderator onboarding pipeline for a Discord community of 50,000+ members. The system manages the full lifecycle from application → trial → promotion (or permanent decline) with bot-driven logistics and human decision-making at key checkpoints.

### Design Principles

- **Semi-automated:** Bot handles logistics, tracking, and notifications. All voicemods vote on promotions, with final sign-off from lead moderator (Andy).
- **Zero tolerance during trial:** Any missed claimed time slot results in immediate trial failure. Applicants may reapply after a 7-day cooldown period.
- **Second chance:** Declined applicants can reapply after 7 days. This ensures fairness while maintaining accountability.
- **Full audit trail:** Every action, decision, and data point is permanently retained.
- **Shadow-first:** Trial mods observe and report — they take no direct moderation actions.

### Expected Volume

- **5–20 applicants per month** (steady flow)
- **Pipeline duration:** ~1–2 weeks per applicant (review + 1-week trial)

---

## 2. Architecture

### Framework

**The bot MUST be implemented using the [Carbon framework](https://github.com/buape/carbon)** by Buape. Carbon provides a structured approach to building Discord bots with built-in support for commands, components, and interactions.

### High-Level Components

```
┌──────────────────────────────────────────────────────────────┐
│                      Carbon Bot                              │
├──────────────┬──────────────┬──────────────┬─────────────────┤
│  Application │  Onboarding  │   Tracking   │   Promotion     │
│   Module     │   Module     │   Module     │   Module        │
├──────────────┴──────────────┴──────────────┴─────────────────┤
│                     Data Layer (Database)                     │
│              Full audit trail, permanent retention            │
└──────────────────────────────────────────────────────────────┘
```

### Required Infrastructure

- **Database:** SQLite for persistent storage of applications, slot claims, reports, votes, nominations, and audit logs
- **Scheduler/Cron:** For slot monitoring, deadline checks, and automated stage transitions
- **Discord Resources:** Roles, private channels, DM capabilities, voice channel permission overrides

---

## 3. Roles & Permissions

### Discord Roles

| Role | Purpose | Permissions |
|------|---------|-------------|
| **Trial Voicemod** | Assigned during the 1-week trial period | View private guidelines channel. No voice moderation permissions. Can join voice channels as a regular user. |
| **Voicemod** (Full) | Assigned upon successful promotion | Mute members, deafen members, move members, disconnect members in voice channels. Can vote on promotions and nominate Clawtributors. |
| **Clawtributor** | Community contributors with speaking access to protected voice channels | Can speak in designated protected voice channels. Can nominate other users for Clawtributor role. |
| **Lead Voicemod (Andy)** | Designated lead moderator | All voicemod permissions + final approval authority on promotions and ability to manage the onboarding pipeline. |

### Permission Boundaries

- **Trial voicemods** have NO moderation powers. They are observers only.
- **Full voicemods** can mute, deafen, move, and disconnect users in voice channels. No ban/kick/timeout powers. Can participate in promotion votes and nominate Clawtributors.
- **Clawtributors** have speaking permissions in protected voice channels. No moderation powers.
- **Lead voicemod (Andy)** provides final approval on all promotions after voicemod team votes.

---

## 4. Workflow Stages

### Stage 1: Application

```
User submits application form
        │
        ▼
Bot stores application in database
        │
        ▼
Bot posts application to senior mod review channel
        │
        ▼
Bot checks auto-reject criteria:
  - Account must be in server for ≥14 days
  - No previous bans or warnings on record
        │
        ├── AUTO-REJECTED → Bot DMs applicant (can reapply after 7 days)
        └── ELIGIBLE → Posted to senior mod review channel
                │
                ▼
        Senior mods review and approve/deny
                │
                ├── APPROVED → Stage 2
                └── DENIED → Bot DMs applicant (can reapply after 7 days)
```

**Application Form Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| Timezone | Select/Text | ✅ | Applicant's timezone (for slot scheduling context) |
| Availability | Text | ✅ | General availability (days/times they're typically free) |
| Motivation | Text (long) | ✅ | Why they want to become a voice moderator |

The form should be triggered via a Discord modal (slash command or button interaction).

**Auto-Reject Criteria:**
- **Minimum server tenure:** User must have been in the OpenClaw Discord for at least 14 days
- **Clean record:** No previous bans, kicks, or warnings on record
- **One application at a time:** Cannot have another application pending or in trial
- **Cooldown period:** If previously declined or failed trial, must wait 7 days before reapplying

**Review Process:**
- Applications that pass auto-checks are posted as embeds in a private senior mod channel
- Senior mods use approve/deny buttons on the embed
- Any single senior mod can approve or deny
- Decision is logged with the reviewer's identity and timestamp

---

### Stage 2: Onboarding

```
Application approved
        │
        ▼
Bot grants "Trial Voicemod" role
        │
        ▼
Bot unlocks private guidelines channel (read-only access)
        │
        ▼
Bot DMs applicant:
  - Confirmation of approval
  - Link to guidelines channel
  - Instructions on claiming time slots
  - Explanation of trial expectations
  - Clear warning: missed slots = immediate trial failure, 7-day cooldown before reapplication
        │
        ▼
Trial mod claims ≥6 hours of time slots (6-8 hour range recommended) → Stage 3
```

---

### Stage 3: Trial Week (Shadow/Observe)

```
Trial mod covers claimed time slots
        │
        ├── During each slot:
        │     - Present in voice channels
        │     - Observe moderation situations
        │     - Use /report to log incidents
        │
        ├── Missed a slot? → IMMEDIATE TRIAL FAILURE
        │     - Bot removes Trial Voicemod role
        │     - Bot DMs: trial failed, can reapply after 7 days
        │     - Application marked as TRIAL_FAILED in database
        │
        └── All slots completed → Stage 4
```

**What "covering a slot" means:**
- The trial mod must be present in a voice channel for at least 45 minutes of their claimed 1-hour slot (75% threshold)
- Presence is tracked by the bot (voice state events)
- The bot also tracks which specific voice channels were covered for promotion review
- A reasonable grace period (e.g., 5 minutes late) may be configured before presence tracking begins

**Reporting during slots:**
- Trial mods use `/report <user|message> <reason>` to log anything noteworthy
- Reports are stored in the database and visible to senior mods
- There is no minimum report requirement — some slots may be uneventful
- Quality/thoughtfulness of reports factors into the subjective mod vote

---

### Stage 4: Promotion Decision

```
All claimed slots completed (bot-verified)
        │
        ▼
Bot confirms metrics in voicemod channel
Bot opens promotion vote
        │
        ▼
All voicemods vote (approve/deny)
        │
        ▼
Simple majority (>50%) reached?
        │
        ├── YES → Vote sent to Andy (lead mod) for final sign-off
        │         │
        │         ├── APPROVED → Bot promotes to full Voicemod
        │         │               Bot removes Trial Voicemod role
        │         │               Bot DMs + public announcement: congratulations
        │         │
        │         └── DENIED → Bot declines
        │                       Bot removes Trial Voicemod role
        │                       Bot DMs: declined, can reapply after 7 days
        │
        └── NO → Bot declines
                  Bot removes Trial Voicemod role
                  Bot DMs: declined, can reapply after 7 days
```

**Vote Mechanics:**
- Vote only opens AFTER the bot confirms all quantitative metrics are met (hours completed, no missed slots)
- Presented as a message with approve/deny buttons in the voicemod channel
- Each voicemod (full role) gets one vote
- Vote has a defined window (48 hours) — if no majority by deadline, the application is auto-declined
- If simple majority (>50%) approves, the vote is escalated to Andy (lead mod) for final sign-off
- Andy can approve or deny at any time (no time limit)
- All votes are logged with voter identity and timestamp

---

## 5. Bot Commands

### Applicant-Facing Commands

| Command | Description |
|---------|-------------|
| `/apply` | Opens the voice moderator application form (Discord modal) |
| `/slots` | Shows available time slots for the current/upcoming week |
| `/claim <slot_id>` | Claims a specific 1-hour time slot |
| `/unclaim <slot_id>` | Unclaims a slot (only allowed before the slot starts) |
| `/my-slots` | Shows all slots the trial mod has claimed |
| `/report <user\|message> <reason>` | Logs an incident report during a slot |
| `/status` | Shows the applicant's current stage in the pipeline |

### Voicemod Commands

| Command | Description |
|---------|-------------|
| `/voc-nominate @user` | Nominate a user for Clawtributor role (speaking access to protected VCs) |
| `/voc-remove-nomination @user` | Withdraw a nomination you created (only if not yet seconded) |
| `/voc-remove @user` | Vote to remove Clawtributor role from a user (requires 2 voicemods) |
| `/voc-nominations` | List all pending Clawtributor nominations |

### Senior Mod / Lead Commands

| Command | Description |
|---------|-------------|
| `/applications` | Lists pending applications awaiting review |
| `/review <application_id>` | Pulls up a specific application for review |
| `/trial-status <user>` | Shows a trial mod's progress (slots claimed, completed, reports filed, channels covered) |
| `/reports <user>` | Shows all reports filed by a specific trial mod |
| `/onboarding-stats` | Dashboard: active trials, pending votes, recent outcomes |
| `/voc-promote <user>` | (Andy only) Final approval for a promotion after voicemod team vote passes |
| `/voc-deny <user>` | (Andy only) Deny a promotion after voicemod team vote passes |

### Admin Commands

| Command | Description |
|---------|-------------|
| `/onboarding-config` | Configure system settings (grace period, vote window, minimum hours, etc.) |

---

## 6. Time Slot System

### Slot Generation

- **Slots are auto-generated 24/7** in 1-hour blocks (00:00–01:00, 01:00–02:00, ..., 23:00–00:00 UTC)
- Slots are generated on a rolling basis (e.g., 2 weeks ahead)
- Each slot is independently claimable

### Claiming Rules

- A trial mod must claim a **minimum of 6 hours** (6 slots), with 6-8 hours recommended, across their trial week
- There is no maximum — they can claim more if desired
- Slots cannot overlap for the same trial mod
- A slot can only be unclaimed if it hasn't started yet
- Once a slot's start time passes, it is locked — the trial mod is committed

### Presence Tracking

- Bot monitors voice state events (`voiceStateUpdate`)
- For each claimed slot, the bot tracks:
  - Whether the trial mod joined any voice channel during the slot window
  - Total time spent in voice channels during the slot
  - **Which specific voice channels** were visited (for promotion review diversity assessment)
  - A slot is considered "covered" if the trial mod was present for ≥45 minutes of the 60-minute window (75% threshold, configurable)
- If a slot ends and the trial mod did not meet the presence threshold → **immediate trial failure**

### Slot Monitoring Cron

A scheduled job runs every minute (or on voice state changes) to:
1. Check if any active trial mod has a slot that just ended
2. Verify presence threshold was met
3. If not met → trigger immediate failure flow
4. If met → mark slot as completed, update progress

---

## 7. Promotion Pipeline

### Quantitative Gate (Automated)

The bot automatically checks:
- [ ] All claimed slots have been completed
- [ ] Minimum 6 hours of total slot coverage (6-8 hours recommended)
- [ ] No missed slots (zero tolerance during trial)

If all checks pass, the bot:
1. Posts a summary in the voicemod channel with:
   - Total hours completed
   - Number of reports filed
   - Links to all reports
   - Slot coverage heatmap (what hours/days they covered)
   - **Channel coverage breakdown** (which voice channels were observed)
2. Opens the promotion vote

### Qualitative Gate (Mod Team Vote)

- Vote is presented as an embed with ✅ Approve / ❌ Deny buttons
- Each voicemod votes once (can change their vote before the window closes)
- Vote window: **48 hours** (configurable)
- Threshold: **Simple majority (>50%)** of votes cast
- If the vote window expires with no votes → **auto-decline** (applicant can reapply after 7 days)
- If majority approves → escalated to Andy for final sign-off via `/voc-promote` or `/voc-deny` commands
- Results are logged and announced in the voicemod channel, mod log channel, and via DM to applicant

---

## 8. Notification System

Notifications are delivered through multiple channels:
- **Bot DMs** to the applicant/trial mod for all process updates
- **Public announcements channel** for promotions and Clawtributor grants
- **Mod log channel** for all events (audit trail)

| Event | DM Content |
|-------|------------|
| Application submitted | "Your application has been received. You'll be notified when it's reviewed." |
| Application auto-rejected | "Your application could not be processed. [Reason: server tenure/previous warnings]. You may reapply after 7 days." |
| Application approved | "Congratulations! You've been approved for a voice mod trial. [Guidelines link] [How to claim slots] [⚠️ Zero tolerance: missed slots = trial failure, 7-day cooldown]" |
| Application denied | "Your application has been reviewed and was not approved. You may reapply after 7 days." |
| Slot claimed | "You've claimed [slot time]. Remember: missing this slot will result in trial failure." |
| Slot completed | "Slot [time] completed ✅ Progress: [X/6+] hours done." |
| Slot missed | "You missed your claimed slot [time]. Your trial has ended. You may reapply after 7 days." |
| Trial week complete | "You've completed all your claimed slots! Your promotion is now under review by the voicemod team." |
| Vote passed, awaiting Andy | "The voicemod team has approved your promotion! Awaiting final sign-off from lead moderator." |
| Promoted | "🎉 Congratulations! You've been promoted to Voice Moderator. [New permissions overview]" (Also posted in public announcements channel) |
| Declined (vote failed) | "After review, the voicemod team has decided not to proceed with your promotion. You may reapply after 7 days." |
| Declined (Andy denied) | "After final review, your promotion was not approved. You may reapply after 7 days." |

### Voicemod / Mod Log Notifications

**In Voicemod Channel:**
- New application received → ping in review channel
- Trial mod missed a slot → alert posted
- Trial mod completed all slots → summary posted + vote opened
- Vote concluded → results announced
- Awaiting Andy's final approval → notification with `/voc-promote` and `/voc-deny` instructions

**In Public Announcements Channel:**
- New voicemod promoted → celebration announcement with @mention
- New Clawtributor granted → announcement with @mention

**In Mod Log Channel (audit trail):**
- All application status changes
- All slot claims, completions, and failures
- All votes cast
- All Clawtributor nominations and grants
- All role additions and removals

---

## 9. Data Model & Retention

### Core Entities

```
Application {
  id: UUID
  user_id: Discord snowflake
  guild_id: Discord snowflake
  timezone: string
  availability: string
  motivation: string
  status: PENDING | APPROVED | DENIED | TRIAL_ACTIVE | TRIAL_FAILED | PROMOTED | VOTE_FAILED
  reviewer_id: Discord snowflake (nullable)
  reviewed_at: timestamp (nullable)
  created_at: timestamp
  updated_at: timestamp
}

TimeSlot {
  id: UUID
  application_id: FK → Application
  user_id: Discord snowflake
  start_time: timestamp
  end_time: timestamp
  status: CLAIMED | COMPLETED | MISSED | UNCLAIMED
  presence_minutes: integer (nullable)
  channels_visited: JSON array of channel IDs (tracks which VCs were observed)
  created_at: timestamp
  updated_at: timestamp
}

Report {
  id: UUID
  application_id: FK → Application
  reporter_id: Discord snowflake
  target_type: USER | MESSAGE
  target_id: Discord snowflake
  reason: string
  slot_id: FK → TimeSlot (nullable, links to which slot they were covering)
  created_at: timestamp
}

PromotionVote {
  id: UUID
  application_id: FK → Application
  voter_id: Discord snowflake
  vote: APPROVE | DENY
  created_at: timestamp
  updated_at: timestamp
}

ClawtributorNomination {
  id: UUID
  nominee_id: Discord snowflake
  nominator_id: Discord snowflake
  seconder_id: Discord snowflake (nullable, set when seconded)
  status: PENDING | GRANTED | WITHDRAWN
  message_id: Discord snowflake (the nomination message for reactions)
  created_at: timestamp
  seconded_at: timestamp (nullable)
  updated_at: timestamp
}

ClawtributorRemoval {
  id: UUID
  user_id: Discord snowflake
  initiator_id: Discord snowflake
  seconder_id: Discord snowflake (nullable)
  status: PENDING | COMPLETED | WITHDRAWN
  created_at: timestamp
  completed_at: timestamp (nullable)
  updated_at: timestamp
}

AuditLog {
  id: UUID
  application_id: FK → Application (nullable)
  nomination_id: FK → ClawtributorNomination (nullable)
  actor_id: Discord snowflake
  action: string (e.g., "APPLICATION_SUBMITTED", "SLOT_CLAIMED", "SLOT_MISSED", "VOTE_CAST", "PROMOTED", "DECLINED", "CLAWTRIBUTOR_NOMINATED", "CLAWTRIBUTOR_GRANTED", "CLAWTRIBUTOR_REMOVED")
  details: JSON
  created_at: timestamp
}
```

### Retention Policy

- **All data is retained permanently** (full audit trail)
- No automatic deletion or archival
- Data should be queryable for historical analysis and accountability

---

## 10. Edge Cases & Failure Modes

| Scenario | Behavior |
|----------|----------|
| Trial mod misses a claimed slot | Immediate failure. Role removed. DM sent. Status → TRIAL_FAILED. Can reapply after 7 days. |
| Trial mod goes offline mid-slot | If presence drops below threshold (45 min of 60), slot is marked MISSED → failure. |
| Trial mod tries to unclaim a slot that already started | Denied. Slot is locked once start time passes. |
| Trial mod claims fewer than 6 hours | They cannot proceed. Bot reminds them of the minimum. Trial week clock doesn't start until ≥6 hours are claimed. |
| Voicemod votes after the vote window closes | Vote is rejected. Window is enforced. |
| No voicemods vote within the window | Auto-decline. Applicant can reapply after 7 days. |
| User who was previously declined tries to reapply within 7 days | Bot checks database. Application is rejected with message: "You must wait [X days] before reapplying." |
| User tries to reapply after 7-day cooldown | Allowed. Previous application history is logged but does not block new application. |
| Voicemod team approves but Andy never responds | Application stays in "awaiting final approval" state indefinitely. No timeout. |
| Trial mod is banned/leaves server during trial | Auto-fail. Application marked as TRIAL_FAILED. |
| Bot goes offline during a slot | Presence data may be incomplete. **Recommendation:** build in a recovery mechanism that checks voice state on reconnect, but err on the side of the trial mod (don't fail them for bot downtime). |
| Multiple trial mods claim the same slot | Allowed. Slots are not exclusive — multiple people can observe the same time window. |
| User nominates someone who is already a Clawtributor | Bot blocks with message: "User is already a Clawtributor!" |
| User tries to second their own nomination | Denied. Bot responds: "You cannot second your own nomination." |
| User tries to nominate themselves | Denied. Bot responds: "You cannot nominate yourself." |
| Nomination is seconded (✅ reaction) | Role is immediately granted. Notification sent via DM + public announcement + mod log. |
| Clawtributor is removed (2 voicemod votes) | Role is removed. User is notified via DM. Logged in mod log channel. |
| Non-voicemod tries to use /voc-nominate | Permission denied. Command only available to voicemods and Clawtributors. |
| Clawtributor nominates a user | Allowed. Same rules apply (must be seconded by different person). |

---

## 11. Guidelines Channel Content

The private guidelines channel (read-only, unlocked for trial voicemods) should contain a comprehensive handbook covering:

### Section 1: Voice Channel Rules
- What behavior is acceptable/unacceptable
- Noise, music, harassment, hate speech policies
- Age-restricted content policies
- Channel-specific rules (if different VCs have different norms)

### Section 2: Moderation Tools & Commands
- How to use `/report <user|message> <reason>`
- What constitutes a reportable incident
- Examples of good vs. poor reports

### Section 3: Escalation Paths
- When to report vs. when to act (for full mods)
- How to escalate to senior mods
- Emergency procedures (raids, doxxing, threats)

### Section 4: Real Scenario Examples
- Example scenarios with correct responses
- Common mistakes new moderators make
- "What would you do?" practice scenarios

### Section 5: Trial Period Expectations
- Reiteration of zero-tolerance slot policy
- How presence is tracked
- What the promotion vote considers
- Timeline and next steps

**Note:** OpenClaw already has existing voice moderation guidelines that will be referenced in this channel.

---

## 12. Clawtributor Role System

### Overview

The Clawtributor role grants speaking permissions in designated protected voice channels. This system allows the voicemod team to recognize and empower trusted community contributors who actively participate in specific voice channels (e.g., Developers, Design, Music Production, etc.).

### Purpose

- **Protected voice channels** are voice channels where only voicemods and Clawtributors can speak
- Regular members can listen but cannot speak without the Clawtributor role
- This creates focused, high-signal voice spaces for collaboration

### Nomination & Grant Process

**IMPORTANT:** Only voicemods (full role) and existing Clawtributors can nominate users. Regular members cannot nominate.

```
Voicemod or Clawtributor runs /voc-nominate @user
        │
        ▼
Bot checks permissions (must be voicemod or Clawtributor)
        │
        ├── NOT voicemod/Clawtributor → Denied: "You don't have permission to nominate."
        │
        └── IS voicemod/Clawtributor → Continue
                │
                ▼
        Bot posts nomination message in configured channel
        Bot reacts with ✅ emoji
                │
                ▼
        Another voicemod or Clawtributor reacts with ✅ to second
                │
                ├── Same person tries to second → Denied
                ├── Regular user reacts → Ignored (only voicemod/Clawtributor reactions count)
                │
                └── Different voicemod/Clawtributor seconds → Role granted immediately
                        │
                        ▼
                Bot grants Clawtributor role
                Bot sends DM to recipient
                Bot posts public announcement
                Bot logs to mod log channel
```

### Commands

| Command | Who Can Use | Description |
|---------|-------------|-------------|
| `/voc-nominate @user` | Voicemods & Clawtributors | Nominate a user for Clawtributor role |
| `/voc-remove-nomination @user` | Original nominator only | Withdraw a nomination (only if not yet seconded) |
| `/voc-remove @user` | Voicemods only | Vote to remove Clawtributor role (requires 2 voicemods) |
| `/voc-nominations` | Voicemods & Clawtributors | List all pending nominations |

### Role Grant Rules

- **Who can nominate:** ONLY voicemods (full role) or existing Clawtributors. Regular members cannot nominate.
- **Who can second:** ONLY voicemods or Clawtributors (except the nominator). Regular member reactions are ignored.
- **Seconding method:** React with ✅ to the nomination message
- **Permission check:** Bot enforces permissions - command will fail if used by non-voicemod/non-Clawtributor
- **Grant timing:** Immediate upon valid second reaction from voicemod/Clawtributor
- **Expiration:** Nominations never expire (stay open until seconded or withdrawn)
- **Self-nomination:** Not allowed
- **Duplicate nomination:** Blocked if user already has the role

### Role Removal Process

```
Voicemod runs /voc-remove @user
        │
        ▼
Bot posts removal vote in voicemod channel
        │
        ▼
Second voicemod runs /voc-remove @user (or reacts to confirm)
        │
        ▼
Role removed immediately
Bot sends DM to user
Bot logs to mod log channel
```

- **Requires 2 voicemods** to remove (same 2-endorsement pattern)
- Only voicemods can initiate and second removals (Clawtributors cannot remove)
- Removals are logged permanently in audit trail

### Protected Voice Channels

The Clawtributor role grants speaking permissions to a configurable set of protected voice channels:
- Channels are defined in bot configuration
- Can be updated without code changes
- Examples: "Developers", "Design Studio", "Music Production", etc.

**Permission Structure:**
- @everyone: Can join and listen, cannot speak
- Voicemod role: Can speak
- Clawtributor role: Can speak

### Notifications

| Event | DM | Public Announcement | Mod Log |
|-------|-----|---------------------|---------|
| Nominated | ❌ | ❌ | ✅ |
| Granted Clawtributor | ✅ | ✅ | ✅ |
| Removed from Clawtributor | ✅ | ❌ | ✅ |

**DM Templates:**
- **Granted:** "🎉 You've been granted the Clawtributor role! You can now speak in protected voice channels: [channel list]"
- **Removed:** "Your Clawtributor role has been removed by the voicemod team."

**Public Announcement Template:**
- "🎉 Welcome @user as a new Clawtributor! They can now contribute in [protected channels]."

### Data Model

See Section 9 for `ClawtributorNomination` and `ClawtributorRemoval` entities.

### Edge Cases

| Scenario | Behavior |
|----------|----------|
| User already has Clawtributor role | Bot blocks nomination: "User is already a Clawtributor!" |
| User tries to second own nomination | Denied: "You cannot second your own nomination." |
| User tries to nominate themselves | Denied: "You cannot nominate yourself." |
| Nominator withdraws before second | Nomination removed, reaction tracking stopped |
| User leaves server after nomination | Nomination stays open (in case they return) |
| User leaves server after granted role | Role is removed automatically by Discord |
| Clawtributor nominates another user | Allowed, follows same rules as voicemod nominations |
| Regular member tries /voc-nominate | Permission denied: "You don't have permission to nominate." Command not executed. |
| Regular member reacts ✅ to nomination | Reaction is ignored. Only voicemod/Clawtributor reactions count toward seconding. |

---

## Appendix: Configuration Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `MIN_TRIAL_HOURS` | 6 | Minimum hours a trial mod must claim (6-8 recommended) |
| `SLOT_DURATION_MINUTES` | 60 | Duration of each time slot |
| `PRESENCE_THRESHOLD_MINUTES` | 45 | Minutes present required to "pass" a slot (75% threshold) |
| `VOTE_WINDOW_HOURS` | 48 | How long the promotion vote stays open |
| `VOTE_THRESHOLD` | 0.5 | Fraction of votes needed to approve (>50%) |
| `SLOT_GENERATION_DAYS_AHEAD` | 14 | How far in advance to generate slots |
| `GRACE_PERIOD_MINUTES` | 5 | Late arrival grace period before presence clock starts |
| `DEFAULT_VOTE_OUTCOME` | DECLINE | What happens if no votes are cast in the window |
| `REAPPLY_COOLDOWN_DAYS` | 7 | Days user must wait after decline/failure before reapplying |
| `MIN_SERVER_TENURE_DAYS` | 14 | Minimum days in server before user can apply |
| `ANDY_USER_ID` | (configured) | Discord user ID of lead mod with final promotion authority |
| `PROTECTED_VOICE_CHANNELS` | (configured) | Array of voice channel IDs where Clawtributor role grants speak permissions |
| `NOMINATION_CHANNEL_ID` | (configured) | Channel where Clawtributor nominations are posted |
| `PUBLIC_ANNOUNCEMENT_CHANNEL_ID` | (configured) | Channel for promotion/Clawtributor announcements |
| `MOD_LOG_CHANNEL_ID` | (configured) | Channel for audit trail logging |
