# Ishuri — campus room status, announcements & assignments

**What this is:** a platform for students, CP/CPN, and staff:
- Students see which rooms are free/occupied, see announcements for their year+department, and submit assignments.
- CP/CPN mark rooms occupied/free (auto-tagged with their own year+department), write announcements for their year+department, post assignments, and download submissions from their year+department.
- Staff have full access across all years and departments.

**Built so far — Module 1, 2 & 3: Accounts + Room status + Announcements + Assignments**
- `index.html` — sign in / sign up (captures name, email, role, year, department)
- `dashboard.html` — room status board (CP/CPN and staff can add rooms and toggle free/occupied; students see it read-only)
- `announcements.html` — CP/CPN and staff post announcements; students see only posts matching their own year + department
- `assignments.html` — CP/CPN and staff post assignments; students in the matching year+department upload a submission file; CP/CPN downloads submissions
- `firebase-config.js` — your project's connection settings (already filled in for UR-CAFF-courses)
- `style.css` — shared styling

## Data model
```
users/{uid}
  name, email
  role: "student" | "cp_cpn" | "staff"
  year: "Year 1".."Year 4"
  department: free text (e.g. "Food Science & Technology")

rooms/{roomId}
  code: "Room 204"
  building: "Old Block"
  status: "free" | "occupied"
  occupiedBy: { uid, name, year, department } | null

announcements/{id}
  title, body
  year: "Year 1".."Year 4" | "all"   (CP/CPN always their own year; staff can pick "all")
  department: free text | "all"      (CP/CPN always their own dept; staff can pick "all")
  authorUid, authorName, authorRole
  createdAt

assignments/{id}
  title, body
  year, department                   (same scoping as announcements)
  authorUid, authorName, authorRole
  createdAt

submissions/{assignmentId_studentUid}
  assignmentId, studentUid, studentName
  year, department
  fileName, fileURL
  submittedAt
```

## Creating a staff account
Staff can't self-register through the sign-up form (that would let anyone grant themselves full access). To make someone staff:
1. They register normally as a Student first.
2. In Firebase Console → **Firestore Database → Data**, open the `users` collection.
3. Find their document (matches their `uid` — you can identify them by their `email` field).
4. Edit the `role` field from `student` to `staff`.
5. They'll need to refresh the app to see staff access.

## Firestore security rules
Go to **Firestore Database → Rules** in the Firebase console and use:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    match /users/{userId} {
      allow read: if request.auth != null;
      // Anyone can create their own profile, but only as "student" — CP/CPN
      // and staff are never self-assigned, only granted by an admin editing
      // Firestore directly (which bypasses these rules entirely).
      allow create: if request.auth != null && request.auth.uid == userId &&
        request.resource.data.role == "student";
      // A user can update their own profile, but can never change their own role.
      allow update: if request.auth != null && request.auth.uid == userId &&
        request.resource.data.role == resource.data.role;
    }

    match /rooms/{roomId} {
      allow read: if request.auth != null;
      allow create, update: if request.auth != null &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ["cp_cpn", "staff"];
      // Only staff can remove a room entirely — CP/CPN can only toggle status.
      allow delete: if request.auth != null &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == "staff";
    }

    match /announcements/{postId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ["cp_cpn", "staff"];
    }

    match /assignments/{assignmentId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ["cp_cpn", "staff"];
    }

    match /submissions/{submissionId} {
      allow read: if request.auth != null;
      allow create, update: if request.auth != null &&
        request.resource.data.studentUid == request.auth.uid;
    }

  }
}
```

This means: anyone signed in can see room status, but only CP/CPN and staff can add rooms, change status, or post announcements/assignments. A student can only ever edit their own profile, and can only ever write a submission under their own `studentUid`. (Visibility by year/department is filtered in the app itself, not the security rule — the rule only controls who can write.)

(Note: during setup we temporarily used a fully-open rule — `allow read, write: if request.auth != null` for everything — to get things working without errors. Once this module is confirmed working, switch to the rules above before real students start using it.)

## Storage security rules (for assignment submissions)
Go to **Storage → Rules** in the Firebase console and use:

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /submissions/{assignmentId}/{studentUid}/{fileName} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && request.auth.uid == studentUid;
    }
  }
}
```

This means: anyone signed in can download a submission file (so CP/CPN can grab them), but a student can only ever upload into their own folder.

## Running it locally
```
cd lms
python3 -m http.server 8000
```
Then open `http://localhost:8000`.

## Putting it online for real (free)
```
npm install -g firebase-tools
firebase login
firebase init hosting     # choose your project, public dir = current folder
firebase deploy
```
You'll get a real `https://ur-caff-courses.web.app` URL to share.

## What's built and what's next
1. ✅ Accounts (student / CP-CPN / staff, with year + department)
2. ✅ Room status board
3. ✅ Announcements — CP/CPN post is auto-scoped to their year+department; staff can target any year/department or post to everyone
4. ✅ Assignments — CP/CPN posts (auto-scoped like announcements), students upload a submission file, CP/CPN downloads submissions via "View submissions"

All four original pieces are now built. From here, natural next steps would be things like: editing/deleting rooms and assignments, due dates with reminders, or a notification when a new announcement/assignment is posted — build any of these whenever you're ready.

## Module 3: Assignments (link-based, no billing required)

`assignments.html` — CP/CPN post assignments (title, instructions, due date), auto-scoped to
their year+department. Students submit by pasting a **link** to their file (Google Drive,
Dropbox, etc.) rather than uploading it directly — this avoids Firebase Storage's requirement
to attach a billing card, while still letting CP/CPN open and download every submission.

**Data model addition:**
```
assignments/{id}
  title, description, dueDate
  year, department
  authorUid, authorName
  createdAt

submissions/{assignmentId}_{studentUid}
  assignmentId, studentUid, studentName
  year, department
  link
  submittedAt
```

**Firestore rules addition** — add these inside the same `match /databases/{database}/documents { ... }` block:
```
match /assignments/{id} {
  allow read: if request.auth != null;
  allow create: if request.auth != null &&
    get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ["cp_cpn", "staff"];
}

match /submissions/{id} {
  allow read: if request.auth != null;
  allow write: if request.auth != null && (
    request.resource.data.studentUid == request.auth.uid ||
    request.resource.data.submittedByUid == request.auth.uid
  );
}
```
Click **Publish** after adding these.

### Telling students how to get a shareable link (Google Drive)
1. Upload the file to Google Drive
2. Right-click it → **Share** → **General access** → change to **"Anyone with the link"**
3. Copy the link and paste it into the assignment's submit box

## All modules — status
1. ✅ Accounts (student / CP-CPN / staff, with year + department)
2. ✅ Room status board
3. ✅ Announcements
4. ✅ Assignments (submit via link + CP/CPN can view/download submissions)

The original request is now fully covered. Natural next steps if you want to keep going:
staff dashboard for promoting/managing users from inside the app (instead of the Firestore
console), push/email notifications, or a proper file-upload flow later if you're ever willing
to attach a card to unlock Firebase Storage's free tier.

## Module 3b: Groups + completion tracking

`groups.html` — CP/CPN create groups (name + members, picked from registered students in
their year/department). Every assignment can now be **Individual** or **Group**:

- **Individual assignments** show a live "Who's finished" list — everyone (students and
  CP/CPN) can see who has submitted, without seeing their actual file link.
- **Group assignments** show group-by-group progress — for each group, how many members
  have submitted out of the total, with a "Complete" badge once everyone in the group is done.

**Data model addition:**
```
groups/{id}
  name, year, department
  members: [{ uid, name }, ...]
  createdBy, createdAt

assignments/{id}
  ...(existing fields)...
  type: "individual" | "group"
```

**Firestore rules addition:**
```
match /groups/{id} {
  allow read: if request.auth != null;
  allow create, update, delete: if request.auth != null &&
    get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ["cp_cpn", "staff"];
}
```
*(Superseded below, once self-join was added — use the version in the final consolidated block instead.)*

Click **Publish** after adding this alongside your other rules.

**Setup order for group assignments:** create the group(s) on the Groups page first (with
their members ticked), *then* post the assignment with type = Group — the assignment
automatically finds groups matching its year/department, no manual linking needed.

## Resubmission limit
Both individual and group submissions store a `resubmitCount` field (starts at 0). A
student/group can update their submission twice after the first one (3 total versions),
then the app hides the update option and shows "Resubmission limit reached (2/2)." This is
enforced in the app's interface, not in Firestore's security rules — a technically
determined user could bypass it via browser dev tools, but it fully prevents accidental
over-editing, which was the actual goal (keeping CP/CPN from getting confused by too many
versions).

## Final step: one consolidated security rules block

Now that every module is built and tested, replace whatever is currently in Firestore →
**Rules** with this complete, locked-down version (this replaces the permissive
`allow read, write: if request.auth != null` rule we used during testing):

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    match /users/{userId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null && request.auth.uid == userId &&
        request.resource.data.role == "student";
      // A user can update their own profile but never their own role or
      // department — UNLESS they're staff, in which case they can update
      // anyone's role/department (this is what powers the in-app Admin page,
      // so no one needs to touch Firestore directly after the first staff
      // account is set up).
      allow update: if request.auth != null && (
        (request.auth.uid == userId &&
          request.resource.data.role == resource.data.role &&
          request.resource.data.department == resource.data.department) ||
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == "staff"
      );
      // Only staff can remove a profile (via the Admin page's "Remove" button).
      allow delete: if request.auth != null &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == "staff";
    }

    match /rooms/{roomId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ["cp_cpn", "staff"];
      allow update: if request.auth != null && (
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == "staff" ||
        (
          get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == "cp_cpn" &&
          (
            (resource.data.status == "free" && request.resource.data.status == "occupied") ||
            (
              resource.data.status == "occupied" &&
              request.resource.data.status == "free" &&
              resource.data.occupiedBy.year == get(/databases/$(database)/documents/users/$(request.auth.uid)).data.year &&
              resource.data.occupiedBy.department == get(/databases/$(database)/documents/users/$(request.auth.uid)).data.department
            )
          )
        )
      );
      allow delete: if request.auth != null &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == "staff";
    }

    match /announcements/{postId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ["cp_cpn", "staff"];
      allow delete: if request.auth != null && (
        resource.data.authorUid == request.auth.uid ||
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == "staff"
      );
    }

    match /assignments/{id} {
      allow read: if request.auth != null;
      allow create: if request.auth != null &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ["cp_cpn", "staff"];
    }

    match /submissions/{id} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && (
        request.resource.data.studentUid == request.auth.uid ||
        request.resource.data.submittedByUid == request.auth.uid
      );
    }

    match /groups/{id} {
      allow read: if request.auth != null;
      allow create, delete: if request.auth != null &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ["cp_cpn", "staff"];
      // CP/CPN and staff can change anything on a group. Everyone else can only
      // touch the "members" field — this is what lets students join/leave
      // themselves without being able to rename, delete, or edit other groups.
      allow update: if request.auth != null && (
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ["cp_cpn", "staff"] ||
        request.resource.data.diff(resource.data).affectedKeys().hasOnly(["members"])
      );
    }

    match /group_messages/{id} {
      allow read: if request.auth != null;
      allow create: if request.auth != null && request.resource.data.senderUid == request.auth.uid;
    }

    match /notifications/{id} {
      allow read: if request.auth != null;
      allow create: if request.auth != null &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ["cp_cpn", "staff"];
    }

    match /attendance_sessions/{id} {
      allow read: if request.auth != null;
      allow create: if request.auth != null &&
        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role in ["cp_cpn", "staff"];
    }

    match /attendance_records/{id} {
      allow read: if request.auth != null;
      allow create: if request.auth != null && request.resource.data.studentUid == request.auth.uid;
    }

  }
}
```

## Going live (Chromebook-specific steps)

1. Open Terminal, install Node.js if not already present:
   ```
   sudo apt update
   sudo apt install -y nodejs npm
   ```
2. Install the Firebase CLI:
   ```
   npm install -g firebase-tools
   ```
3. Log in — Crostini's Linux terminal has no browser inside it, so use:
   ```
   firebase login --no-localhost
   ```
   It will print a long URL — open that URL in your normal Chrome (outside the terminal),
   sign in, and it will show you a code. Copy that code back into the terminal and press Enter.
4. From inside your `lms` folder, run:
   ```
   firebase init hosting
   ```
   - "Use an existing project" → select **ur-caff-courses**
   - "What do you want to use as your public directory?" → type `.` (just a dot, meaning "this folder")
   - "Configure as a single-page app?" → **No**
   - "Set up automatic builds with GitHub?" → **No**
   - If it asks to overwrite `index.html` → **No**
5. Deploy:
   ```
   firebase deploy --only hosting
   ```
6. It will print a real URL like `https://ur-caff-courses.web.app` — that's the link to share
   with classmates. It works from any phone or computer, no terminal needed on their end.

## Module 4: Admin page (staff-only)

`admin.html` — any Staff account can search for a person and change their role
(Student / CP-CPN / Staff) right in the app. This means after the very first staff
account exists, you never need to open the Firestore console again to grant access.

**One-time bootstrap (can't be avoided — someone has to be "staff zero"):**
1. Register normally as a Student
2. Firebase Console → **Firestore Database → Data → `users`** → find your document
3. Change `role` from `student` to `staff`
4. Refresh the app — you'll now see an **Admin** tab in the nav

From that point on, that staff account (or any staff account they promote) can grant
CP/CPN or Staff to anyone else directly from the Admin page — search their name, pick
the new role from the dropdown, done.

**Security note:** the rules above only let someone update another user's role if their
*own* profile already says `role: "staff"` — this is what makes the Admin page safe to
expose to everyone (a student opening admin.html just sees "Staff access required").

## Rebrand + Home page

The platform is now called **UR-CAFF UPDATE** (was "Ishuri") across every page. A new
**`home.html`** page is the landing page after sign-in — it has your campus photo as a
hero banner and a short explanation of each section (Room status, Announcements,
Assignments, Groups), so new users immediately understand what the platform does instead
of landing straight on the room board with no context.

**New file:** `campus.jpg` — your campus photo, used in the sign-in page's side panel and
the Home page's hero. If you want to swap it for a different photo later, just replace
`campus.jpg` with a new image of the same filename (keep the `.jpg` extension, or update
the two `src="campus.jpg"` references in `index.html` and `home.html` if you use a
different file type).

## Rebrand + Home page + campus photo

- Renamed throughout from "Ishuri" to **UR-CAFF UPDATE**
- New **`home.html`** — the first thing people see after signing in. Explains what the
  platform is and links to each section, using your campus photo as the header image.
- The sign-in page's side panel now also uses the campus photo, with a plain-language
  explanation of the four features instead of a vague quote.

**Important:** both `index.html` and `home.html` reference an image file called
**`campus.png`** — this file needs to sit in your `lms` folder alongside the other files,
or the photo won't show (you'll just see a broken image icon). It's included as a download.

## Deleting an account (two-step, unavoidable platform limit)

A plain client-side app like this one **cannot delete another person's login credentials**
— only the person themselves, or Firebase's own admin tools, can do that. So removing
someone fully is two separate steps:

**Step 1 — Remove their profile (in-app, staff only):**
Go to the **Admin** page, find them, click **Remove**. This deletes their year/department/role
data, so they instantly lose access to Rooms, Announcements, Assignments, and Groups.

**Step 2 — Delete their login (Firebase console, manual):**
Their email/password can still technically sign in after Step 1 — it'll just show a broken,
empty experience since their profile is gone. To fully stop them from logging in at all:
1. Firebase Console → **Authentication → Users**
2. Find their row (by email), click the **⋮** menu → **Delete account**

Step 1 alone is usually enough in practice (no profile = no real access to anything). Step 2
is only necessary if you want their email/password combination to stop working entirely —
e.g. someone leaving the university, or a mistakenly-created duplicate account.

## Self-service profile editing

`profile.html` — every signed-in person now has a **Profile** tab where they can update
their own name, year, and department. Role stays locked (shown but greyed out) — the
existing security rules already prevented self-role-changes, so no rule changes were
needed for this one, just the missing page.

## Group chat

Each group card on the Groups page now has a **"Group chat"** toggle — visible only to
that group's members and to staff (not to CP/CPN in general, unless they're also a member).
Messages are live (update instantly for everyone with the panel open).

**Data model addition:**
```
group_messages/{id}
  groupId, senderUid, senderName, text, createdAt
```

**Firestore rules addition** — add this alongside the others, then **Publish**:
```
match /group_messages/{id} {
  allow read: if request.auth != null;
  allow create: if request.auth != null && request.resource.data.senderUid == request.auth.uid;
}
```

**Note on privacy:** like Rooms, Announcements, and Assignments, reads are open to any
signed-in user at the database level (consistent with the rest of this app) — the app's
interface only ever shows a group's chat toggle to that group's members or staff, but a
technically determined person could theoretically query the collection directly. This
matches the trust level already in place elsewhere in the app.

## Email verification

New accounts now must verify their email before using the platform — instead of typing
a code, they click a secure link Firebase sends automatically. This confirms real
ownership of the email with zero backend and zero cost.

**Flow:**
1. On sign-up, `sendEmailVerification()` fires automatically and the person is sent to
   `verify-email.html`.
2. Every protected page (Home, Room status, Announcements, Assignments, Groups, Admin,
   Profile) checks `user.emailVerified` — if false, it redirects to `verify-email.html`
   before showing anything.
3. On that page, "I've verified — continue" re-checks their status and lets them through
   once the link's been clicked; "Resend verification email" sends another one if needed.

**Important — accounts created before this update:** anyone who registered earlier never
had a verification email sent, so the next time they sign in they'll be sent straight to
`verify-email.html` and need to click **"Resend verification email"** once to get their
first one. This is expected, not a bug — just mention it if a CP/CPN or staff member asks
why they're suddenly asked to verify.

**Optional polish (not required):** by default, clicking the link lands on a plain
Firebase-hosted confirmation page, not back on your own site. This is safe to leave as-is;
customizing it further requires `actionCodeSettings` and isn't necessary for the feature
to work correctly.

## Auto-generate groups (for large classes)

The Groups page now has an **"Auto-generate groups"** panel above the manual form —
built for exactly this problem: manually ticking checkboxes for 200 students doesn't scale.

**How it works:** CP/CPN picks a group size (e.g. 5), optionally shuffles, clicks one
button. Every student in their year/department who isn't already in a group gets split
into evenly-sized groups automatically, named `Group 1`, `Group 2`, etc. — continuing
the numbering past any groups that already exist, so running it again later (as new
students register) only groups the newcomers.

The manual "tick each student" form also got a **search box**, since scrolling through
200 names to build one group manually was the other half of this problem.

No new Firestore rules needed — this uses the same `groups` create permission already in place.

## Groups became self-join

Based on real usage feedback, Groups changed from "CP/CPN builds each group by picking
members" to: **CP/CPN just creates empty group slots — students join themselves, one
group at a time.**

- **Manual creation:** name + optional max-members cap, no member picking.
- **Bulk creation:** "how many groups" + optional cap — creates that many empty, numbered
  groups at once (this replaced the old "auto-split existing students" version, since
  self-join makes pre-assigning unnecessary).
- **Students** see every group in their year/department with a live member count, and a
  **Join** button — greyed out with "Full" if the group hit its cap, or replaced with
  "You're already in another group" if they've already joined elsewhere.
- **Leaving:** a student in a group sees a **Leave group** button instead of Join.

This needed a security rule change (see the final consolidated block above) — students
can now update a group's `members` field only (to join/leave themselves), while renaming,
capping, or deleting a group is still CP/CPN/staff-only.

## Offline/in-person assignments

Assignments now have a **"How is it submitted?"** field: **Online (a link)** — the
existing behavior — or **In person / on paper**. Offline assignments show the title,
instructions, and due date like normal, but skip the submit box, completion tracking,
and "View submissions" entirely, since there's nothing to collect online. Existing
assignments created before this update default to "online" automatically.

## Notifications

A new **Notifications** tab (with a small red dot when there's something unread) shows
a live feed of:
- **Room updates** — whenever a CP/CPN marks a room occupied
- **New announcements**
- **New assignments**

...all scoped the same way as everything else — matching the viewer's year and department
(or visible to everyone if posted as "all"). Staff see everything. Clicking a notification
jumps to the relevant page (Room status, Announcements, or Assignments).

**How "unread" works:** the browser remembers the last time you opened the Notifications
page (via `localStorage`, not the database) and compares it to each notification's
timestamp. This is simple and has no backend cost, but it's per-browser — checking
notifications on your phone won't clear the dot on your laptop, and vice versa.

**Data model:**
```
notifications/{id}
  type: "room" | "announcement" | "assignment"
  title, body
  link (which page to open on click)
  year, department (same scoping rules as everything else)
  createdAt
```

**Firestore rule** (already included in the consolidated block above): only CP/CPN and
staff can create notifications — this matches who's allowed to trigger the actions that
generate them (marking rooms, posting announcements/assignments).

## Graduated accounts

Staff can now mark someone as **Graduated** from the Admin page's role dropdown — a
fourth option alongside Student, CP/CPN, and Staff. This is meant for students finishing
their final year (or anyone leaving the platform), without deleting their account or
losing their history.

**What happens when someone is graduated:**
- Signing in sends them straight to a simple **"Congratulations"** page instead of the
  normal app — no access to Room status, Announcements, Assignments, Groups, or
  Notifications.
- Their **Profile** page still works, in case they want to check or correct their info.
- They automatically stop appearing in the student pool for "Auto-generate groups" and
  the manual group picker (both already filter by `role == "student"`, so a graduated
  account is naturally excluded — no extra code needed there).
- Nothing is deleted — past submissions, group memberships, and chat messages stay
  exactly as they were, for historical record.
- If it was a mistake, a staff member can just change their role back from the Admin page.

**One known limitation:** if a graduated student was still in a group when marked
graduated, they stay listed as a member of that group's roster (since they can no longer
open Groups to leave it themselves). A CP/CPN or staff member would need to note this
manually for now — not automated.

## Department became a dropdown, and Admin gained Year/Department filters

**Why:** free-typed department names ("FST" vs "Food Science & Technology" vs a typo)
meant the same department showed up as several different values, making the Admin list
look mixed together and impossible to filter reliably.

**Sign-up and Profile** now use a dropdown with the real UR-CAFF department codes
(CROP, HORT, AEA, AGRO, FLM, EGM, AMC, ALI), plus an **"Other (type it in)"** option
for anything not listed.

**Admin page** now has two filter dropdowns next to the search box: **Year** and
**Department** (the department list fills in automatically from whoever's actually
registered). All three — search text, year, and department — combine together, so
staff can narrow straight down to "Year 2 · Agriculture" and see just those people.

## Getting to the site without typing or sharing the link every time

Three ways to solve "how do people reach this without the link":

**1. Install it like an app (free, already set up)**
Every page now links a `manifest.json` and app icons. Once someone visits the site in
Chrome, they can tap the **⋮ menu → "Add to Home screen" / "Install app"**. After that,
it behaves like a real app icon on their phone — one tap, no typing, no browser bar.
This only works *after* someone has visited at least once, so it doesn't solve
first-time discovery — that's what the QR code below is for.

**2. QR code (free, included as a download)**
`ur-caff-update-qr.png` — a scannable code pointing straight to
`https://ur-caff-courses.web.app`. Print it on posters, put it on a WhatsApp group
description, or hand it out — anyone with a phone camera can scan it straight in,
no typing, no needing someone to paste a link.

**3. A short custom domain (optional, small yearly cost)**
Instead of `ur-caff-courses.web.app`, you could use something like `urcaffupdate.rw`
if UR-CAFF has (or is willing to register) a domain. Firebase Hosting supports custom
domains on the free plan — the only cost is registering the domain itself
(roughly $10-15/year depending on the registrar), not Firebase. Worth considering if
this becomes official, not necessary for a pilot.

## Mobile-friendly navigation redesign

The horizontal row of tabs (which wrapped badly on phones) is now:

- **☰ Hamburger button** (top-left, next to the logo) — tap it to open a dropdown with
  Home, Room status, Announcements, Assignments, Groups, and Admin (staff only).
  Tapping any link, or tapping outside the dropdown, closes it again.
- **Notifications and Profile moved to fixed icons** in the top-right corner, on every
  page, always visible without opening the menu — a 🔔 bell (with the same red unread
  dot as before) and a small circular avatar showing your initials, which links to
  Profile.
- **Log out** stays in the top-right corner too.

Logging in still lands on Home first, same as before — this change is just about how
the same pages are reached, not what the flow is.

**Still pending:** swapping the "U" square logo for the real official UR logo, once
you're able to send the image file.

## Real UR logo (self-serve, no upload needed)

Every page now looks for a file called **`ur-logo.png`** in the `lms` folder and shows it
in the brand corner instead of the plain "U" square. If that file isn't there yet, it
automatically falls back to the "U" square — nothing breaks either way.

**To add the real logo yourself (I couldn't fetch it directly — my tools can only read
web pages as text, not download image files from arbitrary sites):**
1. Open **https://ur.ac.rw/** in Chrome
2. Right-click the UR logo top-left → **Save image as...**
3. Save it into your `lms` folder, named exactly **`ur-logo.png`**
4. Redeploy (`firebase deploy --only hosting`) — no code changes needed, it'll just start showing up

## Room status fixes after launch-day testing

- **Removed the "bulk-add rooms" panel** — it caused duplicate entries and wasn't
  needed anymore once real rooms were already added. Adding rooms one at a time is
  now the only way, on purpose.
- **Only staff can add rooms now** — CP/CPN can still mark rooms occupied/free (their
  core job), but the "Add room" form is staff-only. This matches the actual campus
  workflow: staff manage what rooms exist, CP/CPN manage their status.
- **Duplicate prevention** — adding a room with the same name + building as one that
  already exists is now blocked with a clear message, instead of silently creating
  a second copy.
- **Search box** — everyone (students and CP/CPN) can now search rooms by name or
  building at the top of the Room status page.
- **Remove button now shows the real error** if deleting a room fails, instead of
  doing nothing silently — if it still fails after redeploying, the message it shows
  will say exactly why (most likely: the Firestore rules weren't published yet).

**About your existing duplicate rooms:** these aren't cleaned up automatically — the
duplicate-prevention only stops *new* ones. Use the (now-fixed) Remove button to
manually delete the extra copies you already have.

## Access code + staff filtering (post-launch fixes)

**Campus access code:** sign-up now requires a shared code (currently `CAFF2026`, set in
`index.html` — search for `CLASS_JOIN_CODE` to change it any time). Since students use
personal emails, not official campus ones, this is the practical way to keep random
strangers who find the link from registering. It's a client-side check, not a hard
security barrier (visible in page source to anyone who looks) — but it stops casual
misuse, which was the actual problem. Share the code verbally, on the printed handout,
or in class — not on any public materials.

**Staff Year/Department filters:** Announcements and Assignments now have the same
filter dropdowns Groups and Admin already had — staff (who see everything, unlike
students/CP-CPN who are automatically scoped) can narrow down instead of seeing every
department and year mixed together in one list.

**Groups subject/year/department filtering, and student counts on Admin** — already
built in an earlier round, confirmed still working correctly.

## Attendance (QR code check-in)

**New pages:**
- `attendance.html` — CP/CPN (their own year/department) or staff (any year/department)
  start a session with a title. It generates a live QR code, shows a running count of
  who's scanned in, and has a **Download CSV** button — every student in that class,
  marked Present or Absent, compared against who actually scanned in.
- `attend.html` — where the QR code points. A logged-in student scanning it gets
  instantly marked present (blocked if it's not their own year/department, so people
  can't check into a class they're not part of).

**Data model:**
```
attendance_sessions/{id}
  title, year, department, createdBy, createdByName, createdAt

attendance_records/{sessionId_studentUid}
  sessionId, studentUid, studentName, year, department, scannedAt
```
The record ID is fixed per session+student, so scanning twice doesn't double-count.

**Uses a QR code library from a CDN** (`cdn.jsdelivr.net/npm/qrcode`) — same approach
as loading Firebase itself, no new setup needed on your end.

## Room status: only the occupying CP/CPN (or staff) can free it

Fixed both in the interface and the security rules (client-side hiding alone isn't
real protection): a CP/CPN can mark any free room occupied, but can only free a room
back up if it matches their own year + department. A different class's CP/CPN sees
a note instead of a button. Staff can always do both.

## Department locked to staff-only editing

Students and CP/CPN can still see their department on the Profile page, but it's now
greyed out — changing it (a real source of confusion when people switched departments
mid-use) requires a staff member, either via the new **Department dropdown on the
Admin page** (right next to the role dropdown) or by asking. Enforced in both the
interface and the security rules.

The "Other (type it in)" option was already removed from the department dropdowns
in an earlier round — confirmed still gone.
