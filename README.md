# TrueScale

A single-purpose Android application for tracking daily weight. No account server, no cloud sync, and no advertising: every entry lives in a local SQLite database on the device.

Built in Java for SNHU CS 360: Mobile Architecture and Programming, across Projects One through Three.

- **Package:** `com.example.truescale_shadwick`
- **Language:** Java
- **minSdk:** 24 (Android 7.0)
- **compileSdk / targetSdk:** 34 (Android 14)

---

## Features

- Local account creation and login. Passwords are stored as a SHA-256 hash over a per-user random salt, never in readable form.
- Full create, read, update, and delete over weigh-ins, displayed as a date, weight, and change grid.
- The change since the previous weigh-in is computed when the list loads rather than stored, so inserting or deleting an entry cannot leave stale values behind.
- Optional SMS alert when a new entry reaches the user's goal weight. Each alert fires once for the entry that triggered it. Denying the permission disables only this feature, and the notification appears inside the application instead.

## Repository contents

- `TrueScale_Shadwick_Week_7.zip` : the complete Android Studio project, including the finalized UI from Project Two and the functional code from Project Three

---

## Project Reflection

### Requirements and goals

TrueScale addresses a narrow problem that larger fitness applications treat as an afterthought. Competitive analysis of MyFitnessPal and Lose It! showed that both bury weight logging inside a much larger calorie and macro counting tool, and that some data views sit behind a subscription. The users I designed for did not want a nutrition platform. They wanted to record one number a day and see whether it was moving.

Three user types drove the requirements. A goal-oriented dieter needs to see progress toward a target without navigating a feature set built for something else. A medically monitored user needs a reliable, complete history to bring to a physician. A maintenance-focused user needs to notice drift early, which means the change between entries matters more than any single reading. All three are served by the same design: a grid that opens directly to the history, two actions available from it, and a notification when the goal is reached. An additional constraint came from the stakeholder interview conducted during Project One, which established that health data should stay on the device rather than syncing to a server. That constraint shaped the architecture and became the application's main differentiator.

### Screens and features

Four screens support those needs. The login and create account screen is the entry point and the only gate. The weight log is the home screen, showing every entry with its date, weight, and change since the previous weigh-in, plus the current weight, the goal, and the distance remaining. The add and edit screen collects a single entry. The goal and alerts screen holds the goal weight, the alert phone number, and the SMS permission state.

The user-centered decisions were mostly about removing work and removing the chance of error. Buttons stay disabled until the fields they depend on have content, so the user is never invited to submit something incomplete. The date field opens a calendar dialog rather than accepting typed text, clamped so a future date cannot be chosen, which makes an invalid date impossible to enter rather than something to catch afterward. Deleting an entry asks for confirmation, because a weigh-in cannot be recovered. Interactive targets meet the 48 density independent pixel minimum from Material Design. Focus order is defined explicitly through `imeOptions` and `nextFocusDown` so the keyboard moves through a form the way a user expects.

The choice I consider most successful was giving the SMS permission its own screen rather than raising the system dialog from somewhere in the middle of a task. A permission prompt with no visible context reads as an interruption, and users deny interruptions. Placing the request on a screen that first explains what the alert does, and triggering it from a switch the user deliberately turns on, means the request always arrives with a reason attached.

### Approach to coding

I built the application in vertical slices rather than writing several files and discovering at the end which one broke. The database layer came first, because every other piece needed something real to talk to. Login followed, then the grid and its CRUD operations, then the goal and the SMS trigger. Each stage was compiled and run on the emulator before the next one started.

Working this way surfaced problems in the right order. Because the database schema existed before any screen consumed it, three defects inherited from the Project Two prototype became visible immediately: dates were being stored in a display format that SQLite would sort incorrectly across a year boundary, the change between entries was stored rather than derived and would go stale on any insert or delete, and the delete callback identified rows by screen position rather than by record. Fixing all three cost minutes at that point. Any of them would have been considerably harder to diagnose after four screens were built on top.

The general strategy that carries forward is to build the layer everything else depends on first, and to treat each layer as unfinished until it has actually run. It applies to any project where persistence, business logic, and interface have to agree with each other.

### Testing

Testing was done on the Android Emulator throughout, exercising each feature the way a user would rather than confirming that the code compiled. Both branches of every conditional path were tested deliberately, which for the permission flow meant revoking SEND_SMS through the emulator's application settings and running the denial case as carefully as the granted one.

This process matters because a passing build proves almost nothing about behavior. An earlier module in this course produced a button whose enable logic was exactly inverted after an IDE quick-fix silently dropped a negation. The code compiled and ran without a warning, and only functional testing exposed it.

The most valuable defect this project surfaced came from reading the build configuration rather than from running the application. The system service lookup used to obtain `SmsManager` requires API 31, and returns null below it. With a declared `minSdk` of 24, every device from Android 7.0 through Android 11 would have thrown a null pointer exception when the goal alert fired, and the exception type meant the existing catch block would not have caught it. The emulator ran a current image, so the failure was completely invisible to testing. Auditing framework calls against the minimum SDK the application claims to support, rather than against the device in front of me, is the practice I took away from this.

### Innovation

Two problems required designing something rather than applying a known pattern.

The first was making the goal alert fire exactly once. The obvious implementation, checking the newest weight against the goal whenever the log loads, resends the congratulation every time the user returns to the home screen. Suppressing it by time or by a session flag would break as soon as the application was closed. The solution was to store the identifier of the entry that triggered the alert on the user's record, so the condition becomes whether *this specific weigh-in* has already been announced. A new qualifying entry produces a new alert; revisiting the screen produces nothing. The state that needed to persist turned out to be an identity rather than a flag.

The second was reconciling two date formats. SQLite compares dates stored as text lexically, which means the American convention of month first sorts incorrectly the moment a year boundary is crossed. Storing dates in ISO form and converting for display at the point of rendering solved it, and keeping both formats inside the entry class rather than scattering conversions through the interface meant the stored value and the shown value cannot drift apart.

### Component of particular success

The SMS goal alert is the component that best demonstrates what I learned, because it is the one place where every layer of the application has to cooperate. A new weigh-in enters the database, a query determines whether the goal has been met, the user's stored settings decide whether a text is wanted, the runtime permission state decides whether one can be sent, the platform version decides which API to call, and the stored trigger identifier decides whether it has already happened. A failure anywhere in that chain would have been silent.

Getting it working also meant taking the denial case as seriously as the success case. The requirement is not simply that the application sends a text. It is that a user who declines the permission keeps a fully working application and still finds out they reached their goal, through a different channel. Treating a denied permission as a normal outcome rather than an error condition is the design lesson from this project I expect to reuse most.

---

## AI Acknowledgment

All application requirements, design decisions, screen structure, emulator testing, and verification of the login flow, database operations, and SMS delivery were performed by the author. Claude, an AI assistant developed by Anthropic, was used to assist with generating and troubleshooting portions of the application code and with drafting the written documentation in this repository.

Anthropic. (2026). *Claude* (August 2026 version) [Large language model]. https://claude.ai
