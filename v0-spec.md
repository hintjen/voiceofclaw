# Voice Moderator Onboarding System — Technical Specification

> **Version:** 1.2
> **Date:** March 1, 2026
> **Status:** Design Complete — Validated
> **Framework:** [Carbon](https://github.com/buape/carbon) (Discord bot framework by Buape)
> **Database:** SQLite (file-based, simple deployment)
> **Documentation:** [openclaw/community](https://github.com/openclaw/community) (primary docs repo)

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

### Organizational Context

This bot serves the **VC Moderator Team** within OpenClaw's four-team moderation structure:

- **Discord Moderators** (text channels, rules enforcement)
- **VC Moderators** (voice channels — this bot's domain)
- **Helpers** (community support, questions)
- **Configurators** (bot & server config management)

All team leads report to Shadow (Administrator). The VC Mod Lead coordinates voice channel events and management. All moderators receive the **Community Staff** umbrella role (visible in sidebar, no inherent permissions).

Entry into the VC moderation track requires emailing shadow@openclaw.ai. Shadow acts as the funnel, directing candidates to the VC Mod Lead if appropriate.

### Design Principles

- **Semi-automated:** Bot handles logistics, tracking, and notifications. All voicemods vote on promotions, with final sign-off from VC Mod Lead.
- **Zero tolerance during trial:** Any missed claimed time slot results in immediate trial failure. Applicants may reapply after a 7-day cooldown period.
- **Second chance:** Declined applicants can reapply after 7 days. This ensures fairness while maintaining accountability.
- **Full audit trail:** Every action, decision, and data point is permanently retained.
- **Shadow-first:** Trial mods observe and report — they take no direct moderation actions.
- **Privacy-first for trials:** Trial moderators remain private (no public visibility) until promotion is confirmed, to avoid embarrassment if they back out.

### Expected Volume

- **5–20 applicants per month** (steady flow)
- **Pipeline duration:** ~1–2 weeks per applicant (review + 1-week trial)

---

## 2. Architecture

### Framework

**The bot MUST be implemented using the [Carbon framework](https://github.com/buape/carbon)** by Buape. Carbon provides a structured approach to building Discord bots with built-in support for commands, components, and interactions.

### Bot Ecosystem Positioning

This is a **standalone Carbon bot** within OpenClaw's broader bot ecosystem:

- **Barnacle Bot** (2-in-1: Sapphire + Carbon) — multi-purpose server management
- **Audrey** — voice channel automation
- **Krill** — support ticket system
- **KI** — XP and leveling system
- **This bot** (VC Mod Onboarding) — dedicated VC moderator pipeline management

This bot operates as a separate bot account and codebase, focused exclusively on the VC moderator onboarding workflow.

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
| **Trial Moderator** (org-wide) | Provisional role for people exploring which team to join | Access to one social channel only. No server permissions. **Note:** This is distinct from Trial Voicemod. |
| **Trial Voicemod** (VC-specific) | Assigned during the 1-week VC trial period, after email funnel and Stage 1 approval | View private guidelines channel. No voice moderation permissions. Can join voice channels as a regular user. |
| **Voicemod** (Full) | Assigned upon successful promotion | Mute members, deafen members, move members, disconnect members in voice channels. Can vote on promotions and nominate Clawtributors. Also receives Community Staff umbrella role. |
| **Community Staff** (umbrella) | Visible sidebar role for all moderators across all 4 teams | No inherent permissions (permissions granted via team-specific roles). |
| **Clawtributor** | Community contributors with speaking access to protected voice channels | Can speak in designated protected voice channels. Can nominate other users for Clawtributor role. |
| **VC Mod Lead** | Designated lead moderator for the VC team | All voicemod permissions + final approval authority on promotions and ability to manage the onboarding pipeline. Reports to Shadow (Administrator). |

### Permission Boundaries

- **Trial voicemods** have NO moderation powers. They are observers only.
- **Full voicemods** can mute, deafen, move, and disconnect users in voice channels. No ban/kick/timeout powers. Can participate in promotion votes and nominate Clawtributors.
- **Clawtributors** have speaking permissions in protected voice channels. No moderation powers.
- **VC Mod Lead** provides final approval on all promotions after voicemod team votes. Coordinates voice channel events and management.

### Security Requirements

- **Discord 2FA:** All voicemods (trial and full) must have Discord 2FA enabled.
- **GitHub 2FA:** Required if accessing the openclaw/community documentation repository.

---

## 4. Workflow Stages

### Stage 1: Application

```
User emails shadow@openclaw.ai expressing interest in VC moderation
        │
        ▼
Shadow has initial conversation, determines if VC track is appropriate
        │
        ├── NOT APPROPRIATE → Shadow directs to different team or declines
        │
        └── APPROPRIATE → Shadow directs to VC Mod Lead
                │
                ▼
        VC Mod Lead or Shadow runs /onboard-start @user
                │
                ▼
        Bot checks if user is in Discord server
                │
                ├── NOT IN SERVER → Error: "User must be in the Discord server to onboard"
                └── IN SERVER → Continue
                        │
                        ▼
                Bot DMs user with application form modal
                        │
                        ▼
                User completes form (timezone, availability, motivation)
                        │
                        ▼
                Bot stores application in database
                        │
                        ▼
                Bot checks auto-reject criteria:
                  - Account must be in server for ≥14 days
                  - No previous bans or warnings on record
                  - No application already pending/in trial
                  - Not in cooldown period (7 days after decline/failure)
                        │
                        ├── AUTO-REJECTED → Bot DMs applicant (can reapply after 7 days)
                        │                   Bot notifies initiator in VC Mod channel
                        │
                        └── ELIGIBLE → Posted to VC Mod team channel for review
                                │
                                ▼
                        VC Mod team reviews and approves/denies
                                │
                                ├── APPROVED → Stage 2
                                └── DENIED → Bot DMs applicant (can reapply after 7 days)
```

**Application Entry Point:**

The **only** way to enter the VC moderator pipeline is by emailing shadow@openclaw.ai. Shadow acts as the initial funnel, has a conversation with the candidate, and directs them to the VC Mod Lead if appropriate. The VC Mod Lead (or Shadow) then initiates the bot-driven onboarding process.

**Application Form Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| Timezone | Select/Text | ✅ | Applicant's timezone (for slot scheduling context) |
| Availability | Text | ✅ | General availability (days/times they're typically free) |
| Motivation | Text (long) | ✅ | Why they want to become a voice moderator |

The form is delivered via Discord DM (modal) after `/onboard-start @user` is run, not via a user-facing slash command.

**Auto-Reject Criteria:**
- **Minimum server tenure:** User must have been in the OpenClaw Discord for at least 14 days
- **Clean record:** No previous bans, kicks, or warnings on record
- **One application at a time:** Cannot have another application pending or in trial
- **Cooldown period:** If previously declined or failed trial, must wait 7 days before reapplying

**Review Process:**
- Applications that pass auto-checks are posted as embeds in the VC Mod team channel
- VC Mod team members use approve/deny buttons on the embed
- Any single VC mod can approve or deny
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
Bot confirms metrics in VC Mod team channel
Bot opens promotion vote
        │
        ▼
All voicemods vote (approve/deny)
        │
        ▼
Simple majority (>50%) reached?
        │
        ├── YES → Vote sent to VC Mod Lead for final sign-off
        │         │
        │         ├── APPROVED → Bot promotes to full Voicemod
        │         │               Bot grants Community Staff umbrella role
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
- Presented as a message with approve/deny buttons in the VC Mod team channel
- Each voicemod (full role) gets one vote
- Vote has a defined window (48 hours) — if no majority by deadline, the application is auto-declined
- If simple majority (>50%) approves, the vote is escalated to VC Mod Lead for final sign-off
- VC Mod Lead can approve or deny at any time (no time limit)
- All votes are logged with voter identity and timestamp

---

## 5. Bot Commands

### Trial Mod Commands

| Command | Description |
|---------|-------------|
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

### VC Mod Team / Lead Commands

| Command | Description |
|---------|-------------|
| `/onboard-start @user` | (VC Mod Lead or Shadow only) Initiates the onboarding process for a user after email funnel |
| `/applications` | Lists pending applications awaiting review |
| `/review <application_id>` | Pulls up a specific application for review |
| `/trial-status <user>` | Shows a trial mod's progress (slots claimed, completed, reports filed, channels covered) |
| `/reports <user>` | Shows all reports filed by a specific trial mod |
| `/onboarding-stats` | Dashboard: active trials, pending votes, recent outcomes |
| `/voc-promote <user>` | (VC Mod Lead only) Final approval for a promotion after voicemod team vote passes |
| `/voc-deny <user>` | (VC Mod Lead only) Deny a promotion after voicemod team vote passes |

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
1. Posts a summary in the VC Mod team channel with:
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
- If majority approves → escalated to VC Mod Lead for final sign-off via `/voc-promote` or `/voc-deny` commands
- Results are logged and announced in the VC Mod team channel, mod log channel, and via DM to applicant

---

## 8. Notification System

Notifications are delivered through multiple channels:
- **Bot DMs** to the applicant/trial mod for all process updates
- **Public announcements channel** for promotions and Clawtributor grants
- **Mod log channel** for all events (audit trail)

### Privacy Policy for Trials

**Trial moderators remain private until promotion is confirmed.** This avoids potential embarrassment if they back out or fail the trial. Trial-related notifications go only to:
- Private VC Mod team channel
- DMs to the trial mod
- Mod log channel (private)

**No public visibility** of trial mods until they are promoted to full Voicemod.

### DM Notifications

| Event | DM Content |
|-------|------------|
| Onboarding initiated | "You've been invited to apply for the VC Moderator trial! Please complete the application form." (Sent via modal interaction) |
| Application submitted | "Your application has been received. You'll be notified when it's reviewed." |
| Application auto-rejected | "Your application could not be processed. [Reason: server tenure/previous warnings/cooldown]. You may reapply after 7 days." |
| Application approved | "Congratulations! You've been approved for a voice mod trial. [Guidelines link] [How to claim slots] [⚠️ Zero tolerance: missed slots = trial failure, 7-day cooldown]" |
| Application denied | "Your application has been reviewed and was not approved. You may reapply after 7 days." |
| Slot claimed | "You've claimed [slot time]. Remember: missing this slot will result in trial failure." |
| Slot completed | "Slot [time] completed ✅ Progress: [X/6+] hours done." |
| Slot missed | "You missed your claimed slot [time]. Your trial has ended. You may reapply after 7 days." |
| Trial week complete | "You've completed all your claimed slots! Your promotion is now under review by the voicemod team." |
| Vote passed, awaiting Lead | "The voicemod team has approved your promotion! Awaiting final sign-off from VC Mod Lead." |
| Promoted | "🎉 Congratulations! You've been promoted to Voice Moderator. [New permissions overview]" (Also posted in public announcements channel) |
| Declined (vote failed) | "After review, the voicemod team has decided not to proceed with your promotion. You may reapply after 7 days." |
| Declined (Lead denied) | "After final review, your promotion was not approved. You may reapply after 7 days." |

### VC Mod Team / Mod Log Notifications

**In VC Mod Team Channel:**
- New application received (after `/onboard-start` and auto-checks) → ping for review
- Trial mod missed a slot → alert posted
- Trial mod completed all slots → summary posted + vote opened
- Vote concluded → results announced
- Awaiting VC Mod Lead's final approval → notification with `/voc-promote` and `/voc-deny` instructions

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
  initiated_by: Discord snowflake (tracks who ran /onboard-start — VC Mod Lead or Shadow)
  reviewer_id: Discord snowflake (nullable, tracks who approved/denied the application)
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
| User emails Shadow but isn't in Discord server | Shadow directs them to join the Discord server first. `/onboard-start` will fail with error if user is not in server. |
| Trial mod misses a claimed slot | Immediate failure. Role removed. DM sent. Status → TRIAL_FAILED. Can reapply after 7 days. |
| Trial mod goes offline mid-slot | If presence drops below threshold (45 min of 60), slot is marked MISSED → failure. |
| Trial mod tries to unclaim a slot that already started | Denied. Slot is locked once start time passes. |
| Trial mod claims fewer than 6 hours | They cannot proceed. Bot reminds them of the minimum. Trial week clock doesn't start until ≥6 hours are claimed. |
| Voicemod votes after the vote window closes | Vote is rejected. Window is enforced. |
| No voicemods vote within the window | Auto-decline. Applicant can reapply after 7 days. |
| User who was previously declined tries to reapply within 7 days | `/onboard-start` checks database. If in cooldown, bot notifies initiator: "User must wait [X days] before reapplying." |
| User tries to reapply after 7-day cooldown | Allowed. Previous application history is logged but does not block new application. |
| Voicemod team approves but VC Mod Lead never responds | Application stays in "awaiting final approval" state indefinitely. No timeout. |
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

### Documentation Structure

**Primary documentation source:** [openclaw/community](https://github.com/openclaw/community) GitHub repository (public).

**Discord guidelines channel:** Quick-reference with links to the full GitHub docs. The channel contains:
- Links to the full documentation on GitHub
- Quick command reference (most commonly used commands)
- Biggest rules summary (top violations to watch for)
- Emergency escalation contacts

**Private trial onboarding materials:** Per the meeting policy, onboarding trial moderators is kept private. Trial-specific materials may be hosted in the private VC Mod team channel or DM'd to trial mods.

### GitHub Documentation Sections

The openclaw/community repository should contain comprehensive voice moderation documentation:

#### Section 1: Voice Channel Rules
- What behavior is acceptable/unacceptable
- Noise, music, harassment, hate speech policies
- Age-restricted content policies
- Channel-specific rules (if different VCs have different norms)

#### Section 2: Moderation Tools & Commands
- How to use `/report <user|message> <reason>`
- What constitutes a reportable incident
- Examples of good vs. poor reports

#### Section 3: Escalation Paths
- When to report vs. when to act (for full mods)
- How to escalate to VC Mod Lead or Shadow
- Emergency procedures (raids, doxxing, threats)

#### Section 4: Real Scenario Examples
- Example scenarios with correct responses
- Common mistakes new moderators make
- "What would you do?" practice scenarios

#### Section 5: Trial Period Expectations
- Reiteration of zero-tolerance slot policy
- How presence is tracked
- What the promotion vote considers
- Timeline and next steps

**Note:** OpenClaw already has existing voice moderation guidelines that will be migrated to the GitHub repository.

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
| `VC_MOD_LEAD_USER_ID` | (configured) | Discord user ID of VC Mod Lead with final promotion authority |
| `VC_MOD_CHANNEL_ID` | (configured) | Channel ID for VC Mod team channel (review, votes, notifications) |
| `PROTECTED_VOICE_CHANNELS` | (configured) | Array of voice channel IDs where Clawtributor role grants speak permissions |
| `NOMINATION_CHANNEL_ID` | (configured) | Channel where Clawtributor nominations are posted |
| `PUBLIC_ANNOUNCEMENT_CHANNEL_ID` | (configured) | Channel for promotion/Clawtributor announcements |
| `MOD_LOG_CHANNEL_ID` | (configured) | Channel for audit trail logging |
| `DOCS_REPO_URL` | https://github.com/openclaw/community | URL to the primary documentation repository |
| `COMMUNITY_STAFF_ROLE_ID` | (configured) | Role ID for Community Staff umbrella role (granted on promotion) |
