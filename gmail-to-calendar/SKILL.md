---
name: gmail-to-calendar
description: Find a specified or implied Gmail message and create the corresponding Google Calendar event with exact source-derived details. Use for meeting invitations, appointments, webinars, classes, tickets, hotel stays, flights, trains, 新幹線, reservations, and requests such as「このメールをカレンダーに登録」「昨日予約したホテルを予定に追加」. Use connected Gmail and Google Calendar apps to identify the source email, extract and normalize event fields, preserve complete online-meeting URLs, check duplicates and conflicts, choose the intended calendar without guessing, preview when required, create the event, and verify the saved result.
---

# Gmail to Calendar

Convert an email into a verified calendar event. Treat the email as the source of truth and use the connected Gmail and Google Calendar apps; do not substitute browser scraping when those apps are available.

## Workflow

1. Identify the source email.
   - Fetch the message or thread directly when the user supplies it.
   - Otherwise search with the user's date, sender, subject, organization, venue, or reservation clues. Resolve relative dates against the current date and time zone.
   - Narrow broad results by sender, subject, and event identity. If several plausible messages remain, show concise candidates and ask which one to use.
   - For a thread, inspect the latest authoritative message plus reschedules, cancellations, and corrections. Do not treat quoted obsolete details as current.

2. Extract the event.
   - Extract title, start, end, time zone, location, complete online-meeting URL(s), organizer, attendees, notes, and source identifiers.
   - Inspect the subject, plain/HTML body, headers, attachments exposed by the app, and structured reservation or invitation fields.
   - Normalize relative or partial dates to absolute values only when surrounding evidence makes the result unambiguous.
   - Preserve every complete Zoom, Microsoft Teams, Google Meet, Webex, or other join URL verbatim in the description. Do not shorten, rewrite, open, or omit it.
   - Read [references/event-rules.md](references/event-rules.md) when shaping fields, handling travel/lodging, or resolving edge cases.

3. Resolve ambiguity conservatively.
   - Never invent a date, time, duration, time zone, location, attendee, or target calendar.
   - Use stated start and end times exactly, including minute precision. Do not round or replace them with a customary duration.
   - Ask for the smallest missing fact before any write when the date, start, end, time zone, event identity, or target calendar cannot be established reliably.
   - Use an all-day event only when the source explicitly describes one or supplies dates without meaningful times and the event type clearly spans full dates.
   - If the source contains multiple independent events, keep them separate and confirm which events to create.

4. Choose the target calendar.
   - Honor an explicitly named calendar or established user instruction.
   - Inspect available calendars when the target is not explicit. Select automatically only when exactly one writable calendar is a valid target or the app exposes an unambiguous user-designated default.
   - If multiple writable calendars are plausible, ask the user. Do not infer the calendar from the sender, organizer, email account, or event topic alone.

5. Check duplicates and conflicts before creation.
   - Search the intended calendar over an explicit time range covering the proposed event, using its resolved time zone.
   - Compare source identifiers first; then compare title, start/end, location, organizer, reservation number, and meeting URL.
   - Treat a matching source identifier or a strong match on identity and time as a probable duplicate. Do not create another event without explicit user direction.
   - Report overlapping events as conflicts. A conflict is not automatically a duplicate and does not by itself authorize changing either event.

6. Build the proposed event.
   - Use the source's formal event name or subject for meetings.
   - Make travel titles scannable, such as `新幹線: 東京 → 新大阪`, while retaining the official service and reservation details in the description.
   - Put the physical venue or address in `location`. Also retain online-meeting URLs in the description even if an app places one elsewhere.
   - Record organizer and explicitly listed attendees. Do not add the organizer as an attendee merely because they sent the email.
   - Avoid sending invitations unless the user's request and the source clearly establish that intent; if attendee handling could notify people, preview it and obtain any required approval.
   - Do not add a new Google Meet link unless requested.

7. Preview and create.
   - Immediately before a write, present the title, target calendar, start/end with time zone, all-day status, location, online-meeting URL(s), organizer, attendees, and relevant duplicate/conflict findings.
   - If the app or permission policy requires confirmation, stop after the preview and request it. Otherwise, when the user has already clearly requested creation, proceed without an extra conversational confirmation.
   - Create exactly the proposed event with the connected Google Calendar app.

8. Verify the saved event.
   - Read the created event back from Google Calendar.
   - Verify calendar, title, start, end, time zone, all-day status, location, description, complete meeting URLs, attendees, and conference-link state.
   - Correct a material mismatch only when doing so is within the original request and does not create a new ambiguity; otherwise report it.
   - Report completion only after successful read-back. Include the event link when the app returns one.

## Failure handling

- If Gmail or Google Calendar is not connected or lacks required access, identify the missing connection or permission and stop before writing.
- If a message is cancelled, superseded, or internally contradictory, do not create an event until the authoritative state is clear.
- If creation times out or returns an indeterminate result, search/read the calendar before retrying so a retry cannot create a duplicate.
