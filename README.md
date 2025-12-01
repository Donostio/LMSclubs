LMS Volunteer Sign-up (Handover)

This repo hosts the front-end for the LMS PTA volunteer rota (GitHub Pages). It talks to a Google Apps Script (GAS) web API that reads/writes a protected Google Sheet on the PTA Drive.

If you change the sheet headers, tab names, or the deployed GAS URL without updating the code, you will break it. Don’t.

How it works (in one page)

Front-end: a single index.html served by GitHub Pages. It renders filters (year/group, sub-stall, time), shows live availability, and submits sign-ups.

Data store: a Google Sheet on the PTA Drive (private). This is the single source of truth for slots, bookings and descriptions.

API: a GAS web app (Execute as Me, access Anyone) that exposes JSON endpoints the front-end calls:

GET ?action=ping — health check

GET ?action=meta — lists stalls, times, classes, descriptions

GET ?action=slots[&stall=…&time=…&includeFull=1] — grouped capacity/booked/remaining

GET ?action=summary — per-stall totals

POST / — saves a sign-up (writes to the sheet with a lock)



Auth model (pragmatic):
The GAS app checks a shared API_KEY passed as ?key=…. The front-end uses a password gate (volunteers type the key), then stores it in localStorage and sends it with every request. This is not cryptographic security; it’s a gate to keep randoms out. The sheet itself remains private.


---

Setup (from zero or after a rotation)

1. Google Sheet (private; PTA Drive)

Tabs required (names must match exactly):

Stalls data — the live rota (see schema below)

Descriptions — 2 columns: Year-Stall, Description

Class list — single column of class codes (one per row)


Protect the document; only the GAS service account owner needs edit.



2. Google Apps Script (GAS)

Create a script bound to nothing (standalone).

Paste the current Code.gs (your working version).

Script Property: set API_KEY to a new random string (this is the “password” volunteers will use at the gate).

Deploy → New deployment → Web app

Execute as: Me

Who has access: Anyone


Copy the deployed Web App URL.



3. GitHub Pages site

In index.html, set:

API_BASE = 'YOUR_GAS_WEB_APP_URL'


Commit to main (or the Pages branch) and ensure Pages is enabled.

Do not commit any keys to the repo.



4. Sanity checks

Visit YOUR_GAS_WEB_APP_URL?action=ping&key=THE_API_KEY → expect { ok:true, now:… }.

Visit …?action=meta&key=THE_API_KEY → stalls/times/classes/descriptions populated.
If empty: you’ve mis-named tabs or headers (most common cause).





---

Secrets & access

Where is the key stored?
In GAS Script Properties (API_KEY). Not in GitHub. Not in the sheet.

How do volunteers get in?
They enter the same string at the front-end gate. The key is then kept in their browser’s localStorage.

Rotating the key
Update API_KEY in GAS → redeploy a new version. Tell reps the new password. Old sessions will fail the next call and be forced to re-enter.



---

Google Sheet schema (don’t freestyle)

Tab: Stalls data (column headers must match exactly; no trailing spaces)

URN — unique ID per slot (string; not reused)

Year-Stall — label shown to users (e.g. Y9: Bakery and Spin the Wheel)

Role — e.g. Lead, Lead – Bottle Tombola, Stall co-ordinator – Grotto, Bake: Donations, Student helper

Slot — ordinal or text used for capacity grouping (e.g. 1, 2, 3, or Various times)

Time — one of the accepted strings used for ordering/filters (e.g. 10.00am -12.00pm, Stall Lead, Baking, FRIDAY Anytime between 1pm - 6pm)

Book — free text (not used by the API logic)

Name — blank = available; non-blank = booked

Class (youngest if >1) — class code (mandatory on submit)

Mobile number (for Whatsapp) — mandatory on submit

Email — mandatory on submit

Notes/questions — optional

When — timestamp (the code writes yyyy-MM-dd HH:mm:ss)

Sub stall — helper category for the second filter (e.g. Y9 Bakery, Y9 Spin the Wheel, Any year: Artisan Foods)

(Optional in other tools) GS start, GS end — numeric datetimes for matching logic in Google Sheets


Tab: Descriptions

Year-Stall — must exactly match the values used in Stalls data

Description — rich text shown in the UI


Tab: Class list

Single column of class codes (e.g. 7KM, 7KL, …, 12/13ALB, N/A)


Non-negotiables

Headers must be spelled exactly; no trailing spaces (we’ve been bitten by Email vs Email).

Keep time strings to the known list. If you invent new labels, update the front-end time ordering array or accept that sorting will be odd.



---

Front-end config hotspots (in index.html)

Search for these constants:

API_BASE — the deployed GAS web app URL.

CLASS_LIST — hard-coded fallback (prevents blank drop-down if meta fails).

YEAR_LINKS — WhatsApp group links per year and Leads.

You can also add Any year links here (e.g. Any year: Artisan Foods).


TIME_ORDER — array ordering of time sections (includes Stall Lead, Baking, and day/time bands).

Password gate text and the Santa loader can be edited in the markup.


Deep link filters
The page supports URL parameters so reps can share pre-filtered views:

?stall=Y9%3A%20Bakery%20and%20Spin%20the%20Wheel

?time=10.00am%20-12.00pm

?sub=Y9%20Bakery They will only apply after a successful password gate.



---

Adding/changing data without breaking things

Add a new year-group stall

1. Add rows in Stalls data with a consistent Year-Stall label.


2. Add the exact same label to Descriptions (with a helpful blurb).


3. If you want a separate WhatsApp link for it, add an entry in YEAR_LINKS or reuse the year link.


4. If you want to filter by sub-stall, set Sub stall values consistently (e.g. Y8 Chocolate Tombola).



Add a new sub-stall (filter value)

Use one canonical label per sub-stall in Sub stall.

If a row applies to two sub-stalls (e.g. “X & Y”), either:

Preferred: create two rows (duplicate the slot) with each sub-stall; or

Keep the “X & Y” text and ensure the front-end mapping includes both (messy; error-prone).



Leads / Co-ordinators

The current code copes with Lead and suffixed versions like Lead – Bottle Tombola.

We removed the hard aggregation of “Co-ordinator” roles when Leads replaced them. The UI still shows remaining/selected correctly. You can leave the coordinator regex in place; it does no harm now.


Do not:

Change header names mid-event.

Merge cells.

Put dates/times in random formats (stick to the known set or you’ll get odd ordering).



---

Operations: common failures & fast fixes

“Network error talking to the rota”
The GAS deployment expired (new version not deployed), or wrong URL in API_BASE, or CORS temporary wobble.
Action: Deploy > Manage deployments > New version, confirm Execute as Me & Anyone, copy URL, update API_BASE, commit.

Meta returns empty
You changed a tab name or typo’d a header. Fix the sheet, then reload ?action=meta&key=….

All slots appear booked
There’s some non-blank text in Name (even a space). Clear the cells.

Volunteers see “Some positions are no longer available: …”
Normal race—two people clicked the last slot. They must re-select from what remains.

Password seems to “do nothing”
Old key cached in localStorage or wrong key. Clear localStorage entry lms_key, or rotate API_KEY and redeploy.

Mobile layout looks truncated
Hard-refresh (Safari caching), or assets path wrong. The UI is responsive; if cards stop short on the right, it’s usually stale CSS.



---

Backups (recommended)

Add a time-driven trigger (e.g. 03:00) in GAS to copy Stalls data to a backup (either a dedicated backup spreadsheet, or a new sheet named with the date). Verify the script creates a new file/sheet rather than deleting anything in the live doc. Test it.



---

What we made too complicated (and how to fix next time)

Honest take:

We let labels drive logic. Suffixes like “Lead – Spin the Wheel” were introduced mid-stream and then parsed in code. That’s fragile.

We added Sub stall halfway through and accepted values like “X & Y”, which then needed special casing.

We changed time strings repeatedly, forcing front-end ordering hacks.


Next time: design first, code once.

Freeze a vocabulary up front and publish it:

Year-Stall (human label)

Role (enum: LEAD, HELPER, DONATION, STUDENT)

SubStall (enum values; no “&”)

TimeCode (enum; machine ordering), TimeLabel (human)


Keep an ID column per grouping (e.g. GroupID) so cards don’t depend on fuzzy text.

Stick to one time canonical set; map to text for display.

Maintain a small Lookups tab that the API returns, so the front-end never needs code changes to learn new sub-stalls.



---

Handover checklist

[ ] GAS URL in API_BASE is current; site builds and loads.

[ ] API_KEY set; reps know the “password”.

[ ] Sheet tabs and headers match this README.

[ ] Descriptions populated for every Year-Stall.

[ ] YEAR_LINKS includes all live WhatsApp groups (including “Any year” groups like Artisan Foods).

[ ] Time order array updated if you introduced a new label (or, better, you didn’t).

[ ] A 03:00 backup trigger exists and writes to a backup, not the live sheet.



---

Support boundaries
This is a simple, reliable static+GAS setup. It is not a login-based CRM and it won’t be. If you try to turn it into one, you’ll create risk and burn time. If you need more, pick a product and budget for it.


This is a simple, reliable static+GAS setup. It is not a login-based CRM and it won’t be. If you try to turn it into one, you’ll create risk and burn time. If you need more, pick a proper product next year and budget for it.

Good luck.
