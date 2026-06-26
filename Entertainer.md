# Entertainer Scheduling System — Full Project Walkthrough

## Overview
- **Employee App** (Android + iOS) — staff book their own shifts
- **Admin Dashboard** (Web) — management oversees everything
- Both apps sync in real time

---

## Part 1: Employee App (Mobile)

### Login & Authentication
1. User opens app → sees a single screen asking for phone number (no "New" or "Returning" buttons)
2. Enters phone number → OTP code sent via SMS → user enters code to verify
3. System automatically determines status and routes accordingly:

| Status | What Happens Next |
|--------|-------------------|
| Never seen before | Goes to sign-up/application form |
| Approved employee | Goes straight to booking shifts |
| Pending approval | "Your application is under review" — cannot proceed |
| Rejected | "Your application was not approved — please contact management" |
| Removed/Fired | Message to contact management — blocked permanently |

### New Applicant Registration
First-time applicant enters phone number, verifies with OTP, then sees the application form.

**Step 1 — Choose position:**
- Dancer Auditions
- Security
- Bartenders
- Others (case-by-case consideration)

**Step 2 — Fill out form (per position):**
- Name
- Phone number
- Age
- Years of experience
- Role-specific requirements (displayed as informational text)
- ID upload (optional — IDs are scanned in person at the door)

**After submission:**
- Application marked as pending
- Management notified immediately
- Applicant cannot book shifts
- If they try to log in again, they see "Your application is under review"

**Management review process:**
- Management contacts applicant directly to follow up
- **If approved:** Assigned to operational role (Dancer, Bartender, Security, or Cook) → Can now log in and book shifts
- **If rejected:** Sees message to contact management — cannot proceed

**Note on "Others" applicants:** This category exists for positions outside the standard list. Once management decides the right fit, the applicant is assigned to one of the existing operational roles — or, if it's a Manager/Housemom-type position, set up as a Standard Admin account instead (see Part 2).

### Operational Roles (4)
Day-to-day bookable staff fall under four roles, each with its own distinct color on the schedule:

| Role | Color on Schedule |
|------|-------------------|
| Dancer | Distinct color |
| Bartender | Distinct color |
| Cook | Distinct color |
| Security | Distinct color |

> **Important:** Managers and Housemoms are no longer bookable staff. They are Standard Admins (see Part 2) — they manage the schedule rather than appearing on it themselves.

### Returning Employee Login
1. Employee enters phone number (same screen as everyone else)
2. Receives and enters OTP code
3. If approved → goes straight to booking shifts

### Shift Booking
After login, employees see available shifts and can book directly — no separate "browse open shifts" step.

**Available shift times:**

| Days | Times Available | Applies To |
|------|-----------------|-------------|
| Monday – Friday | 12:00 PM, 5:00 PM, 7:00 PM | Dancer, Bartender |
| Saturday – Sunday | 5:00 PM, 7:00 PM | Dancer, Bartender |
| Monday – Sunday | Day Shift, Night Shift | Cook |
| Monday – Sunday | Day Shift, Night Shift | Security |

> **Note on Cooks & Security:** Cooks and Security staff choose between **Day Shift** or **Night Shift**. Informational text on the booking screen will explain the specific hours that each shift covers. Exact shift times will be finalized during development and updated in the informational text accordingly.

### The Monday/Tuesday/Sunday Rule
To book a shift on Wednesday, Thursday, Friday, or Saturday, an employee must first have booked at least one shift on Monday, Tuesday, OR Sunday of that same week.

- Applies to: Dancer, Bartender, Cook
- Security: TBD (to be confirmed with scheduling details)
- Resets every week (Monday–Sunday)

**Example:** An employee wants to book Friday. They must have already worked Monday, Tuesday, OR Sunday that week. If not, they must book one of those days first.

### Booking Actions

| Action | What Happens |
|--------|--------------|
| **Book a shift** | Management notified immediately + employee receives confirmation |
| **Cancel a shift** | Warning appears: cancelling <24hrs before may result in a fine → Once confirmed, management notified with clear <24hr flag + employee receives cancellation confirmation |
| **Day-of reminder** | Every day at 12:00 PM (noon), any employee scheduled that day gets a reminder notification |

### Monthly 5PM Requirement
Every employee, regardless of role, must work **two 5:00 PM shifts per month**.

- 5PM shifts are only available on Saturdays and Sundays
- Tracked automatically by the system
- Does **not** block bookings — simply tracks and reports who has met the requirement so management can follow up
- Applies to all operational roles (for Cooks and Security, "Night Shift" counts toward this requirement if it falls within 5PM hours — exact applicability TBD with shift times)

### Dancer-Specific Admin Alerts
This notification applies **only to dancers** — not bartenders, cooks, or security.

For **Sunday, Monday, and Tuesday** only, management receives alerts when:

| Alert Type | Trigger |
|------------|---------|
| Fully booked | 24 dancers booked for that day |
| Low booking | 2 days before the day, 15 or fewer dancers booked |

### Employee Privacy & History
- ✅ Employees can view their own past bookings and cancellations
- ❌ Cannot see the complete schedule
- ❌ Cannot see anyone else's bookings
- ❌ Cannot see anyone else's phone number

---

## Part 2: Admin Dashboard (Web)

### Admin Access Levels

| Level | Count | Capabilities |
|-------|-------|--------------|
| **Master Admin** | 1 | Full control over everything, including managing other admin accounts |
| **Standard Admin** | Up to 10 | Manage schedules, reports, employee approvals, day-to-day operations — but **cannot** add, edit, or remove other admin accounts |

**Managers & Housemoms = Standard Admins:**
- Set up with Standard Admin accounts (not as bookable staff)
- Full access to view schedules, see phone numbers, approve applicants, and manage shifts
- Count toward the 10 Standard Admin total
- Do **not** appear on the bookable staff schedule

**If a Manager/Housemom leaves:** Their admin account is removed and the position can be reassigned — for example, to an applicant from the "Others" category who fits that role.

### Master Admin — Managing Admin Accounts
The Master Admin has a dedicated "Admin Accounts" section that Standard Admins cannot see.

| Action | Details |
|--------|---------|
| **Add Standard Admin** | Enter username, password, display name → New admin can log in immediately |
| **Edit Standard Admin** | Update username, password, or display name at any time |
| **Remove Standard Admin** | Deactivate or delete account → Login no longer works. **Important:** Any schedule changes that admin made remain in the system — removing an admin does not undo their past actions |

### Admin Login
All admins (Master or Standard) log in with their own username and password.

### Reviewing New Applicant Submissions
Both Master Admin and Standard Admins can view pending applications. Each shows:
- Position applied for (Dancer Auditions, Security, Bartenders, or Others)
- Name, phone number, age, years of experience
- ID photo (if applicant chose to upload one)

**Process:**
1. Admin reviews application
2. Contacts applicant directly to follow up
3. Clicks **Approve** or **Reject**
4. **If approved:** Assigns them to an operational role (Dancer, Bartender, Security, or Cook) — or, for Manager/Housemom-type roles under "Others," sets them up with a Standard Admin account
5. Applicant is notified automatically of the decision

### Schedule Management
Admins can view the entire schedule for any day, week, or month — every employee, every role, every shift, in one place.

**Visual Features:**
- Each role displays in its own distinct color:
  - Dancers — one color
  - Bartenders — another color
  - Cooks — another color
  - Security — another color
- Phone numbers shown directly on the schedule next to each booked employee's name
- Cook and Security shifts display as "Day Shift" or "Night Shift" on the schedule

**Management Actions:**
- Manually add any employee to any shift
- Move an employee's shift to a different day or time
- Cancel any employee's booking
- Adjust capacity — set how many people are allowed per shift
- Generate upcoming weeks of available shifts in one click (automatically follows all time rules)
- Update informational text for Cook and Security shift times as details are finalized

### Activity Log
Every change made by an admin is automatically recorded. Any admin can view this log.

**Each entry shows:**
- **Who** made the change (which admin)
- **What** they did (e.g., removed employee from shift, moved booking, approved application, added admin)
- **When** it happened (exact date and time)

**Example:** *"Sara removed Rachel from Monday at 1:45pm"*

**Tracks all admin actions:**
- Adding, moving, or cancelling any employee's shift
- Approving or rejecting new applications
- Adding, editing, or removing employee profiles
- Adding, editing, or removing other admin accounts (Master Admin actions)

Viewable by day, week, or month.

### Employee Management
- Add and edit employee details and roles (Dancer, Bartender, Security, Cook)
- Manager/Housemom accounts managed under Admin Accounts instead
- Permanently block fired employees from logging in — once removed, that person can never access the app again
- Past schedule history is preserved for records
- Search/filter employee list by name, phone number, or role

### Reports & Printing
All reports viewable by **day, week, or month** and printable directly from the browser. Every report can be filtered by **role** — for example, "show only dancers' schedule" or "only bartenders' cancellations."

| Report | What It Shows |
|--------|---------------|
| **Full Schedule Report** | Everyone's shifts for the selected period, including phone numbers, color-coded by role |
| **Cancellations Report** | Every cancellation in the selected period, flagged clearly when it happened less than 24 hours before the shift |
| **Monday/Tuesday/Sunday Compliance Report** | For any given week: who worked at least one of those three days, and who did not — filterable by role |
| **Monthly 5PM Compliance Report** | For any given month: exactly how many of the required two 5PM shifts each employee completed |
| **Pending Applications Report** | List of new applicants awaiting review, with the position they applied for |

### Notifications to Management
Management automatically receives notifications for:

| Event | Notification Details |
|-------|---------------------|
| Employee books a new shift | Immediate notification |
| Employee cancels a shift | Immediate notification, with <24hr flag noted when applicable |
| New employee submits application | Notification to review |
| Dancers: 24 booked (Sun/Mon/Tue) | Day is full |
| Dancers: ≤15 booked, 2 days out (Sun/Mon/Tue) | Low booking warning |

### Privacy & Access Control
- Only Admins (Master or Standard) can view the full schedule
- No employee, regardless of role, can see the complete schedule or anyone else's bookings
- Only the admin dashboard can see employees' phone numbers and full schedules
- The employee app never exposes another employee's personal information

---

## Language
- **Current build:** English only
- **Future enhancement:** Bilingual English/Spanish (not included in this phase)

---

## Complete Flow Summary

### Applicant / Employee Journey
1. Downloads app → Opens it
2. Enters phone number → Receives and enters OTP code
3. **If new:** Chooses position (Dancer Auditions, Security, Bartenders, Others) → Fills out application (name, phone, age, experience; ID optional) → Application goes to management → Management contacts and reviews → Approved or rejected
4. **If approved:** Assigned operational role (Dancer, Bartender, Security, or Cook) → Sees available shifts → Books shifts (subject to Monday/Tuesday/Sunday rule) → Receives confirmation
5. Can cancel shifts (with <24hr warning about possible fine)
6. Receives day-of reminder at noon
7. Can view own booking history
8. Cannot see anyone else's schedule, bookings, or phone number

### Admin Journey
1. Master Admin creates up to 10 Standard Admin accounts (including Managers and Housemoms)
2. Admins log in with username/password
3. Review new applicant submissions → Contact applicants → Approve or reject
4. Approved applicants assigned to operational role (or Manager/Housemom = Standard Admin)
5. View full schedule with color-coded roles and phone numbers
6. Manually add, move, or cancel any employee's shift
7. Generate reports filtered by day/week/month and by role → Print reports
8. Manage employees (add, edit, permanently block fired employees)
9. View Activity Log to see who made which change and when
10. Receive real-time notifications for bookings, cancellations, applications, and dancer alerts
11. Master Admin can add/edit/remove other admin accounts

---

## Confirmation Checklist

### Authentication & Onboarding
- [ ] Phone number + OTP login (auto-routing — no manual new/returning choice)
- [ ] New applicants choose position (Dancer Auditions, Security, Bartenders, Others) and fill form (name, phone, age, experience); ID upload optional
- [ ] Applications require management review/contact before applicant can book shifts
- [ ] Once approved, assigned to Dancer, Bartender, Security, or Cook (each color-coded)
- [ ] Security is confirmed as 4th operational role with own color
- [ ] "Others" applicants assigned accordingly (operational role or Manager/Housemom Standard Admin)

### Scheduling & Rules
- [ ] Direct shift booking — no separate browse step
- [ ] Mon–Fri: 12PM, 5PM, 7PM | Sat–Sun: 5PM, 7PM (Dancer, Bartender)
- [ ] Cooks: Day Shift / Night Shift with informational text explaining times
- [ ] Security: Day Shift / Night Shift with informational text explaining times
- [ ] Monday/Tuesday/Sunday rule for Dancer, Bartender, Cook (Security TBD)
- [ ] 2× 5PM shifts per month required (tracked, non-blocking)
- [ ] Employees get noon reminder on shift day

### Notifications
- [ ] Management notified on every booking and cancellation (<24hr flag on cancellations)
- [ ] Dancer alerts: 24 booked = full, ≤15 = low warning 2 days out (Sun/Mon/Tue only)

### Admin System
- [ ] 1 Master Admin (full control, manages admin accounts) + up to 10 Standard Admins (cannot manage other admins)
- [ ] Managers/Housemoms = Standard Admins (not bookable, see full schedule/phone numbers, count toward 10 total)
- [ ] Removing admin does not undo their past schedule changes
- [ ] Admin schedule view shows phone numbers + color-coding (Dancer, Bartender, Security, Cook)
- [ ] Reports filterable by role and period; printable from browser
- [ ] Manual shift add/move/cancel + adjust capacity + generate upcoming weeks
- [ ] Fired employees permanently blocked; history preserved
- [ ] Activity Log records every admin action (who, what, when); viewable by day/week/month
- [ ] Pending Applications Report shows all new applicants with position applied for

### Privacy & Platform
- [ ] Only admins see full schedule and phone numbers — employees cannot see others' data
- [ ] English only (Spanish planned for later)
- [ ] Android + iOS for employees | Web browser for admins

---

*This document reflects all confirmed decisions and requested changes. No technical implementation details — pure functionality for your verification before development begins.*
