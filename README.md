# Ai-Proctor-Meet
# AI Interview Live Monitoring Tool (Java)

A desktop proctoring tool for live online interviews. It locks the candidate's
window into a fullscreen session and watches for signals that the candidate
has moved attention away from the interview — losing window focus, resizing
into a split-screen layout, connecting a second monitor, the webcam losing
sight of a face, or the feed going still for too long. Every event streams
live to an interviewer dashboard, and the session auto-terminates once too
many violations pile up.

## Important, honest limitation

No software — this one included — can literally detect "the candidate opened
ChatGPT." What it *can* detect are behavioral proxy signals that correlate
with a candidate looking away from the interview screen: focus loss, window
resizing/snapping, extra monitors, and webcam presence/motion. The interviewer
sees these flagged as **suspicious activity**, not a confirmed accusation —
build your interview process around that distinction.

Also worth knowing before deploying this: recording someone's webcam and
monitoring their screen during an interview raises consent and privacy-law
questions that vary by country/state. Get explicit candidate consent and
check local requirements before using this in a real hiring process.

## Project structure

```
ai-interview-proctor/
├── pom.xml
├── src/main/java/com/proctor/
│   ├── Protocol.java                        # shared wire protocol
│   ├── candidate/
│   │   ├── CandidateApp.java                # main entry point, locked window
│   │   ├── FocusMonitor.java                # focus/resize/monitor detection
│   │   ├── MotionDetector.java              # webcam motion + face detection
│   │   └── AlertClient.java                 # socket client -> interviewer
│   └── interviewer/
│       └── InterviewerDashboard.java        # main entry point, alert viewer
```

## Requirements

- JDK 17+
- Maven 3.8+
- A webcam (for the candidate side)

## Build

```bash
cd ai-interview-proctor
mvn clean package
```

This produces `target/ai-interview-proctor.jar`, a runnable fat jar containing
JavaCV/OpenCV natives for webcam access.

<img width="1395" height="1026" alt="imagesScreenshot2026-09-14-14214635 png" src="https://github.com/user-attachments/assets/3602b5b6-0ce4-474b-9921-45bce8dbba0a" />

## Modern Web Meeting Application (Frontend)

The project includes a **modern web meeting application** frontend inspired by Google Meet and Zoom, featuring real-time video feeds, AI proctoring HUD, an in-call code assessment editor, chat, and interviewer controls.



### Quick Start (Zero-Setup Browser Launch)
Simply open the web frontend directly in any modern browser:
```bash
# Double-click or open in browser:
frontend/index.html
```

Or serve via Python/Node if preferred:
```bash
# Via Python:
python -m http.server 8080 --directory frontend

# Or via Java Embedded Web Server:
java -cp target/ai-interview-proctor.jar com.proctor.web.ProctorWebServer --port 8080
```
Then visit: `http://localhost:8080`

<img width="1077" height="876" alt="imagesLogin Screenshot 2026-09-15 160341 png" src="https://github.com/user-attachments/assets/97c8e138-e111-4f83-a012-3713fcb2f7a7" />

---

## Run Desktop & Proctoring Stack

**1. Start the interviewer dashboard first** (runs the TCP socket server + embedded web UI):

```bash
java -cp target/ai-interview-proctor.jar com.proctor.interviewer.InterviewerDashboard --port 5050
```

This launches both the desktop dashboard on port 5050 and the Web Meeting Frontend at `http://localhost:8080`.

<img width="1917" height="926" alt="imagesDashboard Screenshot 2026-09-15 160400 png" src="https://github.com/user-attachments/assets/533f671f-c055-4730-ae64-17479879e73e" />


**2. Start the candidate app**, pointing at the interviewer's machine:

```bash
java -cp target/ai-interview-proctor.jar com.proctor.candidate.CandidateApp \
  --host 192.168.1.10 --port 5050 --name "Jane Doe"
```

(Use `--host 127.0.0.1` if testing both on the same machine.)

<img width="1917" height="1023" alt="imagesCandidate Screenshot 2026-09-15 160747 png" src="https://github.com/user-attachments/assets/ca08deb9-6070-48a4-b52d-a606431d428d" />


The candidate window locks into a maximized, always-on-top, undecorated
window. It cannot be closed with a normal window-close action.

## What triggers an alert

| Signal | Source |
|---|---|
| Window loses OS focus (alt-tab, another app clicked) | `FocusMonitor` |
| Window minimized | `FocusMonitor` |
| Window resized/moved off its original bounds (split-screen/snap) | `FocusMonitor` |
| A second monitor connected mid-session | `FocusMonitor` |
| No face visible in webcam for 8+ seconds | `MotionDetector` |
| No meaningful motion for 15+ seconds | `MotionDetector` |
| Webcam disconnected or unavailable | `MotionDetector` |

Every alert is sent to the interviewer immediately. After **3 violations**
(tunable via `VIOLATION_THRESHOLD` in `CandidateApp.java`), the candidate
session automatically ends and the interviewer is notified.

## Tuning

- `CandidateApp.VIOLATION_THRESHOLD` — how many flags before auto-termination
- `MotionDetector.NO_MOTION_ALERT_MS` / `NO_FACE_ALERT_MS` — sensitivity of
  webcam-based checks
- `FocusMonitor` polling interval (`Timer(1500, ...)`) — how often bounds/
  monitor checks run as a backup to event listeners

## What this does *not* do (be upfront about this with users/candidates)

- It does not read clipboard contents, browser history, or other application
  windows' content.
- It does not perform true OS-level "block alt-tab" enforcement — that needs
  a kernel-level hook/driver and admin privileges, which a normal desktop app
  shouldn't request. Focus-loss is *detected and reported*, not physically
  prevented.
- It does not identify *what* app the candidate switched to.
#
