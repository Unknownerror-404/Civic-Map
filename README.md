# Civic-Map

Civic-Map connects community problems with government moderators, universities, student teams, and industry partners.

Features:

- Citizen issue reporting on a Leaflet/OpenStreetMap map
- Jharkhand districts, blocks, and societal domains
- MySQL storage for issues, users, proof images, proposals, assignments, teams, universities, and industry offers
- Duplicate detection and image GPS verification
- Admin moderation and university assignment
- University dashboards for decisions, teams, and proposals
- Industry dashboards for support offers

This is a local demo built on Python's standard-library HTTP server. Login sessions are kept in memory and are lost when the server stops. Records in `accounts.csv` are imported into MySQL on startup.

## Quick start

Create a MySQL database and an application user:

```powershell
mysql -u root -p
CREATE DATABASE civic_map;
CREATE USER 'civic_app'@'localhost' IDENTIFIED BY 'change-this-password';
GRANT ALL PRIVILEGES ON civic_map.* TO 'civic_app'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Set the connection variables in PowerShell, install the dependencies, and start the server:

```powershell
$env:CIVIC_MAP_DB_USER = "civic_app"
$env:CIVIC_MAP_DB_PASSWORD = "change-this-password"
$env:CIVIC_MAP_DB_NAME = "civic_map"
python -m pip install -r requirements.txt
python map.py
```

Without these variables, the app connects to database `sih26` as MySQL user `root`. `storage.py` creates the tables and applies schema migrations on startup.

The browser opens automatically. If it doesn't, go to:

```text
http://127.0.0.1:8000
```

### Demo accounts

| Role | Email | Password |
|---|---|---|
| Citizen | `citizen@example.com` | `map123` |
| Verified professional | `engineer@example.gov` | `gov12345` |

Create your own account for anything beyond a demo.

The administrator account is allowlisted as `admin@jharkhand.gov.in`. Register that email with a password of at least 8 characters before opening `/admin`.

University and industry accounts use the contact email stored in their MySQL profile. Register one of these emails before logging in:

```text
innovation@bitmesra.ac.in
innovation@cuj.ac.in
innovation@nitjsr.ac.in
partner@jin.example
connect@alf.example
innovation@etm.example
```

## Using the app

1. Start the server with `python map.py` and sign in or create an account.
2. Click the map to place a report pin. You can drag it to adjust the position.
3. Pick a Jharkhand district, then enter a block or city, a title, a domain, and a description.
4. Optionally attach a JPEG, PNG, or WebP image as proof, then submit.
5. On the Community page, browse nearby issues and support the ones you agree with.
6. Use the `Propose` link to pick one of the top-voted problems and submit a solution with a title, a description, and an optional visual.
7. Stop the server with `Ctrl+C`.

### Roles

- **Verified professionals** open `/professionals` to review proposals. The demo professional account is marked as a verified government professional.
- **The administrator** opens `/admin` to approve, reject, or archive pending reports (a reason is required), and `/universities` to add or edit university profiles: name, district, domains, departments, laboratories, incubation facilities, and contact email.
- **University accounts** linked to a profile's contact email open `/university-dashboard`. There they accept or reject assigned challenges with a reason, create faculty-led student teams, submit proposals, update the project stage (`Team Formed`, `Prototype`, `Pilot`, `Deployed`, `Impact Measured`), and add milestones with due dates and deliverables.
- **Industry accounts** linked to a partner profile's contact email open `/industry-dashboard` to browse approved challenges and offer mentorship, funding, prototyping, testing, or deployment support.

## Solution proposals

`/proposals` ranks issues by supporter count and lists the most-supported ones as targets for proposals. A signed-in user can submit:

- the civic issue it addresses
- a title
- a description
- an optional JPEG, PNG, or WebP visual, up to 8 MB

Proposals, visuals, votes, and professional reviews are stored in the `proposals` table. Visuals are saved as `LONGBLOB` values and served from stable URLs, so they survive a restart.

Issue support is recorded per user in the `issue_supporters` table, so each user can support an issue once. The demo supporter totals shown on seeded issues don't count toward voting eligibility. Only recorded supports do.

Voting is available on `/community` and `/proposals`. You have to support the related issue first, and each eligible user has one active solution choice per issue. Vote totals are stored in MySQL. The per-user eligibility map is still kept in memory.

Each item shows a status label:

| Label | Meaning |
|---|---|
| Awaiting Consideration | Ranked problem, nothing happening yet |
| Awaiting Solution | The leading problem |
| Awaiting Approval | New user proposal |
| Solution Approved | Approved by a professional |
| Marked Non-Feasible | Rejected as not feasible |
| Revision Requested | Needs changes |

## Professional portal

`/professionals` is limited to the `VERIFIED_PROFESSIONALS` list in `login_users.py`. Other users get a forbidden response. Verified users see proposals ranked by votes, ten at a time, and submit one of these decisions with an explanation:

- Under review
- Approved
- Non-feasible
- Needs revision

Reviews are saved with the proposal and show the reviewer's name, the decision, and the explanation. The professional list is a local allowlist. In production it should be replaced with role records approved by an administrator.

## Image proof

The report form accepts images up to 8 MB. Accepted images are stored as MySQL `LONGBLOB` values and remain available after a restart.

If Pillow is installed, `AI_model.py` reads the EXIF GPS data and compares it with the map pin:

- Within 100 metres: marked location verified.
- More than 100 metres away: the report is rejected.
- No GPS data: accepted as photo proof, marked location unverified.
- Unreadable or invalid metadata: marked location unverified.

Many messaging and social apps strip EXIF data, so use the original camera file when you need verification. GPS metadata is evidence of where a photo was taken, but it can be edited or faked.

## Duplicate detection

`AI_model.py` works in two stages:

1. Filter candidates by category and distance.
2. Compare text with `intfloat/multilingual-e5-small` through `sentence-transformers`.

If the library or model can't be loaded, it falls back to token overlap so the app keeps working.

Each new report gets one of three results:

- `new`: a new issue is created.
- `duplicate`: the report is merged into the existing issue and its supporter count goes up.
- `possible_duplicate`: no issue is created, and the reporter is asked to support the existing one.

The search radius depends on the category. Roads use a tight radius, while water outages use a wider one because a single outage affects a whole neighbourhood.

## Spam protection

`spam.py` adds basic checks at the API boundary:

- Per-user and per-IP rate limits on reports, proposals, issue support, solution votes, and professional reviews
- Rejection of repeated submissions in a short window
- Minimum title and description lengths
- Checks for control characters and too many links
- Detection of repeated filler text
- One vote per user for issue support and solution voting

For production, keep the guard state in Redis or the database, add CAPTCHA or device-risk checks, moderate comments and uploaded media, scan images for malware, and log moderation actions. Login, registration, password reset, comments, notifications, and professional verification requests also need rate limits.

## Project files

```text
SIH26043/
  AI_model.py       Duplicate detection and EXIF GPS verification
  community.py      Shared issues, matching, community page, upvotes
  login_users.py    CSV-backed accounts and password hashing
  map.py            HTTP server, map page, API routes, proof image serving
  spam.py           Rate limits and content checks
  storage.py        MySQL connection, schema setup, and queries
  accounts.csv      Legacy accounts, imported into MySQL if present
  .env.example      MySQL configuration template
  README.md         This file
```

`accounts.csv` holds password hashes and salts, not plain-text passwords. Keep it out of any public repository.

## Routes

| Route | Access | Purpose |
|---|---|---|
| `/` | Signed-in users | Issue map and report form |
| `/community` | Signed-in users | Browse, filter, and support issues |
| `/my-issues` | Signed-in users | Track your own reports |
| `/proposals` | Signed-in users | Submit and vote on proposals |
| `/professionals` | Verified professionals | Review proposals |
| `/admin` | Administrator | Moderate pending issues |
| `/universities` | Administrator | Edit university profiles and assign issues |
| `/university-dashboard` | Linked university account | Review assignments, create teams, submit proposals |
| `/industry-dashboard` | Linked industry account | Browse approved challenges and submit support offers |
| `/government-dashboard` | Administrator | Analytics on issues, institutions, partners, and projects |
| `/notifications` | Signed-in users | In-app notifications |
| `/messages` | Signed-in users | Project messages |

## MySQL tables

`storage.py` creates and migrates these tables:

- `issues`: challenges, locations, moderation state, proof image blobs
- `issue_supporters`: who supported which issue
- `accounts`: email identities, password hashes, salts
- `proposals`: solutions, visuals, votes, professional reviews
- `universities`: institutions and their capabilities
- `issue_assignments`: issue-to-university routing and responses
- `project_teams`: faculty-led teams linked to assigned issues
- `team_members`: student memberships
- `industry_partners`: startup, MSME, and CSR partner profiles
- `support_offers`: partner offers on approved challenges, with commitment status and notes for admin follow-up
- `milestones`: milestones, due dates, deliverable notes
- `project_status_history`: timestamped record of project stage changes
- `notifications`: workflow notifications
- `messages`: role-based project messages

## Checks

Syntax check, from the repository root:

```powershell
python -m py_compile AI_model.py community.py login_users.py map.py storage.py
```

Dependency check:

```powershell
python -c "import PIL, sentence_transformers, mysql.connector; print('Dependencies available')"
```

The first duplicate check downloads the embedding model from Hugging Face, which can take several minutes. Later runs use the local cache.

## Troubleshooting

**The map is blank.** Check that the server is running, refresh with `Ctrl+F5`, and look for JavaScript errors in the browser console. Leaflet and the OpenStreetMap tiles load from external URLs, so you also need an internet connection.

**The port is already in use.** Stop the old `python map.py` process, or change `PORT` in `map.py`.

**The first duplicate check is slow.** That's the one-time model download. Install `sentence-transformers` before starting the server and let the download finish.

**A geotagged image shows as unverified.** The file probably lost its EXIF GPS data. Screenshots, edited images, and images saved from many websites have it removed.

## Before deploying

The app is a prototype. A real deployment needs:

- PostgreSQL with PostGIS for issues and radius queries
- Object storage (S3-compatible) for proof images
- A background worker for embeddings and image processing
- A full web framework such as FastAPI or Django
- HTTPS, persistent secure sessions, and CSRF protection
- Image MIME validation, re-encoding, malware scanning, and a policy on EXIF data
- Human review for borderline duplicate decisions
- Tests for deduplication, proof verification (EXIF, missing GPS, location mismatch, file size), and the API (login, report, merge, errors)
- A `.gitignore` covering `accounts.csv`, uploads, caches, and secrets

Planned features:

- Faculty and student role records, and an approval workflow for partners
- Sessions and rate limits stored in persistent, secure infrastructure

## Requirements

```text
Pillow>=10.0
sentence-transformers>=3.0
mysql-connector-python
```

Install with `python -m pip install -r requirements.txt`. Pin exact versions once you've tested on the deployment machine. The embedding model isn't part of the repository. `sentence-transformers` downloads it separately.

## Privacy notes

- Proof images can contain faces, vehicle plates, or location metadata.
- Get the required consent before publishing images.
- Decide whether to keep or strip EXIF data once a report has been verified.
- Don't treat automatic duplicate detection as a final moderation decision.
- Use a persistent database and authenticated user IDs before relying on supporter counts.
