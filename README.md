<div align="center">

<img src="https://readme-typing-svg.demolab.com?font=Orbitron&size=42&duration=3000&pause=1000&color=00F5FF&center=true&vCenter=true&multiline=true&width=900&height=120&lines=SANDI+RIDWAN;Data+Automation+Engineer" alt="Sandi Ridwan" />

<br/>

<img src="https://capsule-render.vercel.app/api?type=waving&color=gradient&customColorList=0,2,2,5,30&height=120&section=header&text=Summer%20Leads%20Automation&fontSize=32&fontColor=00F5FF&animation=fadeIn&fontAlignY=65" width="100%"/>

<br/>

![Make.com](https://img.shields.io/badge/Make.com-6D00CC?style=for-the-badge&logoColor=white)
![Calendly](https://img.shields.io/badge/Calendly-006BFF?style=for-the-badge&logo=calendly&logoColor=white)
![Monday.com](https://img.shields.io/badge/Monday.com-FF3D57?style=for-the-badge&logo=monday&logoColor=white)
![Gmail](https://img.shields.io/badge/Gmail_API-EA4335?style=for-the-badge&logo=gmail&logoColor=white)
![Status](https://img.shields.io/badge/Status-LIVE-00FF88?style=for-the-badge)

<br/>

<img src="https://readme-typing-svg.demolab.com?font=Fira+Code&size=16&duration=2000&pause=500&color=00F5FF&center=true&vCenter=true&multiline=true&width=700&height=100&lines=2+Automated+Scenarios+Running+24%2F7;Zero+Manual+Entry+|+100%25+Real-Time;Duplicate+Guard+|+Mountain+Time+|+Smart+Routing" alt="Stats" />

</div>

---

## What This Does

> **One booking in Calendly = One perfectly filled lead in Monday.com. Automatically. Always.**

This automation pipeline eliminates **100% of manual data entry** for summer camp and intensive program leads. Every booking, cancellation, and status change flows through a bulletproof Make.com scenario architecture — no missed leads, no duplicate entries, no wrong timezones.

---

## Architecture

```
SCENARIO A — NEW BOOKING
=========================

[Calendly] invitee.created
        |
        v
[Make.com] Search by Email
        |
   EXISTS? ----YES----> Update Call Date only (reschedule)
        |
        NO
        |
        v
[Monday.com] Create Item
    Name        = invitee.name
    Email       = invitee.email
    Phone       = location.location (Text column)
    Program     = IF(contains(event,"Camp"),"Camp","Intensive")
    Call Date   = formatDate(UTC -> Mountain Time)
    Status      = "Call Scheduled"


SCENARIO B — CANCELLATION
==========================

[Calendly] invitee.canceled
        |
        v
[Make.com] Search by Email
        |
   +----+----+
   |         |
FOUND    NOT FOUND
   |         |
   v         v
Update    Gmail Alert
Status    to Owner
= Lead
Notes
= "Cancelled on [date MT]"
```

---

## Monday.com Board Structure

| Column | Type | Source | Auto-filled |
|--------|------|--------|-------------|
| Name | Text | Calendly invitee.name | YES |
| Email | Email | Calendly invitee.email | YES |
| Phone | Text | Calendly location.location | YES |
| Program Type | Dropdown | IF formula from event name | YES |
| Call Date | Date | start_time converted to Mountain Time | YES |
| Status | Status | "Call Scheduled" on create | YES |
| Start Date | Date | — | Manual |
| Teacher/Placement | Text | — | Manual |
| Notes | Long Text | Cancel date on cancellation | Auto + Manual |

### Status Pipeline

```
Call Scheduled -> Form Completed -> Deposit Pending -> Enrolled -> Needs Scheduling
     ^
   Lead  <-- auto-set on cancellation
```

---

## Technical Details and Lessons Learned

### Bug 1 — Monday.com Phone Column Incompatibility

**Problem:** Monday.com native Phone column requires strict JSON format. Make.com EU server converts all dynamic variables inside JSON strings to `=>` format, making every payload invalid.

```
BROKEN:  {"phone" => "+62 821-9414-9596", "countryShortName" => "ID"}
WORKING: +62 821-9414-9596   (stored as Text column)
```

**Solution:** Change column type from Phone to Text. Full data preserved, only loses click-to-call UI.

---

### Bug 2 — Make.com EU Server Formula Separator

**Problem:** eu1.make.com uses comma not semicolon as separator.

```
FAILS:  {{replace(value; " "; "")}}
WORKS:  {{replace(value, " ", "")}}
```

---

### Bug 3 — Make.com Error Handler Cannot Chain Modules

**Problem:** Resume error handler is terminal — no button to connect downstream modules.

**Solution:** Use Router with filter conditions instead:

```
Search Items
    |
  Router
    |-- {{2.id}} EXISTS     --> Update Item
    |-- {{2.id}} NOT EXIST  --> Gmail Alert
```

---

### Advanced Technique — Duplicate Guard

Prevents duplicate Monday items when lead reschedules (triggers invitee.created again):

```
Search by Email
    EXISTS?  YES -> Update Call Date only
    EXISTS?  NO  -> Create new item
```

---

### Advanced Technique — Mountain Time Conversion

```
{{formatDate(1.scheduled_event.start_time, "YYYY-MM-DD", "America/Denver")}}
America/Denver auto-handles MST (UTC-7) and MDT (UTC-6) Daylight Saving Time
```

---

### Advanced Technique — Auto Program Type Detection

```
{{if(contains(1.scheduled_event.name, "Camp"), "Camp", "Intensive")}}

"Summer Camp Consultation" -> Camp
"Intensive Consultation"   -> Intensive
```

---

## Setup Guide

### Prerequisites

- Calendly account (Free plan works)
- Monday.com account (Free plan works)
- Make.com account (Free plan: 1,000 ops/month — enough for summer volume)
- Gmail account for error alerts

### Step 1 — Calendly Setup

1. Create two Event Types: Summer Camp Consultation and Intensive Consultation
2. Set meeting type to Phone Call (phone auto-captured in location field)
3. Add custom question: "What is your country code?" — Required: Yes
4. Generate Personal Access Token: Settings > Integrations > API & Webhooks

### Step 2 — Monday.com Board

Create board "Summer Leads 2025" with these columns:

```
Name          -> Text (default)
Email         -> Email
Phone         -> Text  (NOT Phone type — see Bug 1)
Program Type  -> Dropdown: Camp, Intensive
Call Date     -> Date
Start Date    -> Date
Status        -> Labels: Call Scheduled, Lead, Form Completed,
                         Deposit Pending, Enrolled, Needs Scheduling
Teacher       -> Text
Notes         -> Long Text
```

Get API token: Avatar > Developers > My Access Tokens

### Step 3 — Make.com Scenario A

```
Module 1: Calendly Watch Events (trigger: invitee.created)
Module 2: Monday Search Items by Email
Module 3: Router
    Route 1 - {{2.id}} does not exist -> Module 4: Create Item
    Route 2 - {{2.id}} exists         -> Module 5: Update Call Date
Enable Scheduling: ON
```

### Step 4 — Make.com Scenario B

```
Module 1: Calendly Watch Events (trigger: invitee.canceled)
Module 2: Monday Search Items by Email
Module 3: Router
    Route 1 - {{2.id}} exists         -> Update Status=Lead + Notes
    Route 2 - {{2.id}} does not exist -> Gmail alert to owner
Enable Scheduling: ON
```

---

## Test Matrix

| ID | Scenario | Expected Result | Status |
|----|----------|----------------|--------|
| T-01 | New Camp booking | Item created, Program=Camp, Status=Call Scheduled | PASS |
| T-02 | New Intensive booking | Item created, Program=Intensive | PASS |
| T-03 | Phone captured | Phone column filled from Calendly | PASS |
| T-04 | Mountain Time | Call Date in MT not UTC | PASS |
| T-05 | Cancellation | Status=Lead, Notes filled with date | PASS |
| T-06 | Reschedule detected | No duplicate, Call Date updated only | PASS |
| T-07 | Email not in Monday | Gmail alert sent to owner | PASS |
| T-08 | 24/7 operation | Both scenarios live and auto-triggering | PASS |

---

## Project Stats

| Metric | Value |
|--------|-------|
| Scenarios Built | 2 |
| Modules per Scenario | 4 to 5 |
| Manual Entry Eliminated | 100% |
| Fields Auto-filled per Booking | 6 |
| Error Handling | Router + Gmail Alert |
| Duplicate Protection | Active |
| Timezone | Mountain Time (America/Denver) |
| Make.com Operations per Booking | ~4 |
| Free Tier Capacity | ~250 bookings per month |

---

## About

<div align="center">

<img src="https://readme-typing-svg.demolab.com?font=Orbitron&size=22&duration=3000&pause=1000&color=00F5FF&center=true&vCenter=true&width=600&height=60&lines=Sandi+Ridwan+|+Data+Automation+Engineer" alt="About" />

Automation Engineer specializing in no-code/low-code workflows, web scraping, and AI integration pipelines. 17 projects completed across 10+ countries.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Sandi_Ridwan-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/sandi-ridwan)
[![GitHub](https://img.shields.io/badge/GitHub-sandiridwan-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/sandiridwan)
[![Email](https://img.shields.io/badge/Email-sandyzvoster%40gmail.com-EA4335?style=for-the-badge&logo=gmail&logoColor=white)](mailto:sandyzvoster@gmail.com)

</div>

---

<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=gradient&customColorList=0,2,2,5,30&height=100&section=footer&animation=fadeIn" width="100%"/>

**Built by Sandi Ridwan — Palu, Central Sulawesi, Indonesia**

If this helped you, drop a star. It means a lot.

</div>
