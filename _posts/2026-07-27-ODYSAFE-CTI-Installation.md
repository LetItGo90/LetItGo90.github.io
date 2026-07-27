# ODYSAFE-CTI: A Practical Installation Guide and Platform Overview

## Introduction

I've been looking for a self-hosted CTI platform that doesn't require a cloud account, a vendor relationship, or a licensing conversation with someone's sales team. ODYSAFE-CTI scratches that itch pretty well. It's open source, runs entirely on-premise, stores everything in SQLite, and after install makes zero outbound network connections unless you explicitly configure integrations. For a homelab or a small SOC that wants to own its data, that's a solid starting point.

This post is a walkthrough of a real installation on an Ubuntu VM - including the parts that broke and why - plus a tour of the features that I've actually found useful day-to-day. Part 2 will cover forwarding logs from Splunk into the platform.

---

## What Is ODYSAFE-CTI?

At its core it's a Python/Flask app that gives you a structured workspace for everything that normally ends up scattered across browser tabs, random text files, and half-finished spreadsheets when you're working a threat. IOCs, MITRE mappings, structured reports, log analysis - all in one place, all in your own infrastructure.

The headline features:

- **IOC Extraction** - pulls 50+ IOC types from files, pasted text, or URLs. Handles defanged indicators like `hxxp://example[DOT]com` automatically, which matters more than it sounds when you're working with threat reports
- **IOC Management** - browse, filter, tag, validate (TP/FP), assign TLP levels, and link indicators to MITRE ATT&CK techniques
- **Log Analyzer** - upload security logs and automatically map events to MITRE ATT&CK with a Kill Chain visualization
- **STIX Graph** - interactive visualization of STIX 2.x bundles, fully client-side, nothing leaves the browser
- **Flash Reports (FLINT)** - a 13-step CTI report wizard with Excel export that includes Diamond Model diagrams
- **CTI Memory** - local semantic search and analyst notes using vector embeddings, completely offline after first use
- **MITRE ATT&CK** - full enterprise matrix stored locally, browsable by tactic, technique, and group with IOC linking

---

## System Requirements

The README is accurate here:

- **OS:** Debian or Ubuntu
- **Python:** 3.10 or higher (read the gotcha section below before you assume 3.10 is fine)
- **git, pip, openssl:** any recent version
- **Disk:** ~500MB base install, plus ~80MB on first CTI Memory use when the embedding model downloads
- **RAM:** The platform itself is lightweight - 8GB is plenty, I'm on 16GB because it was already there

My VM for reference: Ubuntu 22.04, 16GB RAM, 150GB disk, 4 vCPUs.

---

## Installation

### Step 1 - System Packages

Only step that needs sudo. Everything else runs as a normal user.

bash

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git openssl
```

### Step 2 - Clone the Repository

The README recommends checking out a release tag rather than running off `main`. Probably should do this.

bash

```bash
git clone https://github.com/Odysafe/ODYSAFE-CTI.git
cd ODYSAFE-CTI
git checkout v1.0.0
```

### Step 3 - Run the Installer

bash

```bash
./install.sh
```

This sets up the Python virtual environment, installs dependencies from the lock file, generates your SSL certificate locally, and optionally downloads pinned CTI resource snapshots - DeepDarkCTI, MITRE ATT&CK data, and the Ransomware Tool Matrix.

---

## The Gotcha - Python Version

This is the part I got stuck at. The installer completes fine and the venv looks healthy. Then you try to start the app and get:

```vbnet
ModuleNotFoundError: No module named 'flask'
```

The venv was created but the dependencies weren't installed properly. When you try to fix it manually:

bash

```bash
source venv/bin/activate
pip install -r scripts/requirements.lock
```

You get:

basic

```basic
ERROR: Could not find a version that satisfies the requirement numpy==2.4.6
```

The lock file pins `numpy==2.4.6`, which requires Python 3.11 minimum. Ubuntu 22.04 ships with Python 3.10. So you to fix just install python version 3.11 and rebuild from scratch:

bash

```bash
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install -y python3.11 python3.11-venv python3.11-pip

# Remove the broken venv
deactivate
rm -rf venv

# Rebuild with 3.11
python3.11 -m venv venv
source venv/bin/activate
pip install -r scripts/requirements.lock
```

The README says Python 3.10–3.12 is CI-tested, which is technically true, but the lock file as of v1.0.0 makes 3.10 unusable in practice. Use 3.11 and you're done. If you are reading this at a later time, maybe its fixed and this parts pointless.

### Step 4 - Start the Application

bash

```bash
./start.sh
```

You should see:

```vim
Access URLs:
  Local:     https://localhost:5001
  Network:   https://192.168.1.189:5001
```

Open the network URL in your browser. The self-signed certificate warning is expected, the cert was generated locally during install, no external CA was contacted. 

---

## First Things to Do After Install

### Enable Authentication Immediately

Default install has **no login**. Anybody on your network who can reach port 5001 gets full access to the platform. Go to **Settings → Authentication** and turn it on before you do anything else. If it in your homelab, its technically wouldn't really matter unless you have it hosted on the web or something. Mine's just in my lan so, not a big deal but be smart.

### Sort Out the Self-Signed Certificate

If you want to front the app with a cert from your internal CA or cert-manager instead of the self-signed one, drop your own certs into `cti-platform/ssl/` and the app will use them. For pure homelab use the self-signed cert is fine.

### Download CTI Resources

If you skipped the optional downloads during install, go to **Analysis → MITRE ATT&CK** and grab the enterprise matrix. Same for DeepDarkCTI and the Ransomware Tool Matrix if you want them. These are pulled from pinned commits defined in `scripts/pinned_sources.json`, you're getting known-good snapshots, not whatever's live upstream right now.

The CTI Memory embedding model (~80MB) downloads automatically the first time you actually use the Memory module.

---

## Platform Tour

### Dashboard

The home page is a genuine operational view - total sources, active IOC count, TP/FP split, TLP breakdown, temporal trends for the last 24h/7d/30d, and alerts surfaced for any IOCs tagged TLP:RED + True Positive. It's the kind of dashboard where you can actually tell at a glance whether anything needs attention, rather than just being decorative.

![Dashboard](/assets/homepage_overview.png)


### IOC Extraction

This is the module I use most. Navigate to **Workspace → Extraction** and you've got four ways to feed it:

- **File upload** - drop in `.txt`, `.html`, `.docx`, `.csv`, `.json`, `.log`, `.xml`, `.md` up to 100MB
- **Paste text** - paste any content and extraction runs immediately
- **URL import** - give it a URL, the platform fetches and extracts
- **Manual entry** - single IOC with type, value, source name, and context

The defanging handling is genuinely useful. `hxxp://malicious[.]domain[.]com` gets refanged automatically, which is nice not having to clean the text yourself before dropping it in.

**Quick Export** is handy for one-off jobs where you don't want to clutter your IOC database. Extract, download the results as a text file, done. Nothing gets saved.

To test this I put the Proofpoint TA458/Operation RoundPress report through Quick Export.The extractor pulled out all the SpyPress C&C domains (`upgybj[.]store`, `xsza[.]net`, `zxzaq[.]com`, `xwe[.]us`, `hgmydr[.]wiki`, `share-ya[.]space`) plus the SHA256 hashes for the exploit-laden emails targeting Zimbra, Roundcube, mDaemon and SOGo, so 30-odd IOCs out of a blog post in about two seconds. You can then take that text file straight into **Data → IOCs** to start enriching and validating.

![FQDN](/assets/adding_fqdn_to_data.png)

### IOC Management

**Data → IOCs** is where you'll spend most of your time. Everything extracted lands here with type, source, and extraction date. Per indicator you can mark TP/FP, set TLP and confidence, add analyst notes that get indexed into CTI Memory, and link the IOC to specific MITRE ATT&CK techniques. There are also one-click jump links to VirusTotal, AbuseIPDB, Shodan, ThreatFox, OTX, and BGP.he.net for enrichment without leaving the page.

Taking `upgybj[.]store` from that TA458 report as an example - you can add it manually, tag it as a SpyPress C&C domain, drop in the context from the Proofpoint report (first seen February 2026, linked to the mDaemon zero-day CVE-2025-3929 campaign), set TLP:AMBER, mark it as a true positive once you've confirmed it's not in your environment, and link it to MITRE T1071.001 (Application Layer Protocol: Web Protocols). That context then travels with the IOC everywhere in the platform. The VirusTotal and AbuseIPDB links are right there in the detail panel so you're not digging through tabs to cross-reference.

![IOC_Added](/assets/ioc_added.png)


### Log Analyzer

**Analysis → Log Analyzer** is one of the more immediately satisfying modules. Upload a log file (CSV, JSON, TXT/LOG, XML) and it scans every event against 200+ regex patterns covering all 14 MITRE ATT&CK tactics. It understands Syslog, Windows Event Logs, Apache/Nginx access logs, AWS CloudTrail, Azure Activity Logs, and IIS logs.

I tested it with a sample Tunna RDP tunnel IIS log from the EVTX-ATTACK-SAMPLES repository,a real IIS access log from a compromised server where Tunna was being used to tunnel RDP traffic over HTTP. The log records an attacker at `10.0.2.17` hitting `conn.aspx` on the server at `10.0.2.15`, starting with an initial GET request with the tell-tale Tunna query string `proxy&port=3389&ip=127.0.0.1`, then 4,000+ alternating GET and POST requests over the next two minutes as the RDP tunnel carries data back and forth. The platform mapped this correctly to Command and Control, flagged the beaconing pattern and the use of an application layer protocol to proxy C2 traffic, and surfaced the Kill Chain phase clearly in the visualization.

The output gives you a Kill Chain visualization highlighting which phases have detected events and which are blind zones, technique cards with IDs and event counts, a chronological timeline grouped by tactic, and a full events table with signal strength ratings. You can export the whole thing to JSON or CSV.

One thing worth remembering: a match flags an area for investigation. It doesn't confirm compromise. An analyst still has to look at it.

_(screenshot)_

### MITRE ATT&CK

**Analysis → MITRE ATT&CK** is the full enterprise matrix running locally - 15 tactics, 365 techniques, 493 sub-techniques, 189 groups, 824 software entries. Filter by group, platform, or search by name and ID. Link IOCs from your database directly to techniques, which surfaces both in the IOC detail view and on the technique page. TTP-type IOCs like `T1059` extracted from threat reports get automatically mirrored into the same link table so the connections build up without extra work.

_(screenshot)_

### Flash Reports (FLINT)

This is probably my favourite part of the platform and the one that will matter most if you've ever had to write a proper CTI deliverable. The Flash Report wizard walks you through 13 structured steps covering everything from metadata (reference number, TLP, PAP, priority, status) through summary, key takeaways, timeline, Diamond Model (Adversary/Capability/Infrastructure/Victim), IOCs, detection rules, recommended actions, gaps, assessment, sources, and distribution.

You can import IOCs directly from your database into the report - TLP, confidence, analyst notes, linked MITRE techniques and all. The Excel export produces a clean 7-sheet workbook with a dashboard sheet including KPI cards and a Diamond Model diagram, executive summary, technical analysis, IOC table, detection rules, recommendations, and sources. The kind of thing that would normally take an afternoon of copy-paste and formatting comes out ready to brief in a few minutes.

One thing to know: report drafts live in browser localStorage until exported. Don't close the tab mid-wizard without hitting export first or you'll lose your work.

_(screenshot)_

### CTI Memory

**Workspace → Memory** is the semantic search layer. It uses local vector embeddings (fastembed, fully offline after the ~80MB model downloads on first use) to let you search across everything in your workspace - IOCs, MITRE techniques, log incidents, sources, analyst notes. Press `Ctrl+K` anywhere in the app for global search across the same dataset.

The way notes work is particularly useful. You can attach context to any IOC, TTP, log incident, or source from anywhere in the platform, and those notes get indexed and become searchable. So if you tag `upgybj[.]store` with notes about the TA458 campaign context and link it to the SpyPress malware family, that context surfaces whenever you search for anything related to that infrastructure or threat actor. It builds up into something genuinely useful as your workspace grows.

_(screenshot)_

### STIX Graph

**Analysis → STIX Graph** imports any STIX 2.x bundle - either upload a JSON file or paste raw JSON directly - and renders it as an interactive network graph. Default layout is a hierarchical DAG which you can toggle to force-directed if you prefer. This is 100% client-side. Nothing gets sent back to the server, which makes it fine for sharing with air-gapped teams.

I tested it with the OASIS `indicators-for-C2-with-COA.json` example bundle. That file has an identity object (MITRE Corp/DHS Support Team), a Poison Ivy malware entry, two indicators (an IP address for a known C2 channel and a SHA-256 hash for a Poison Ivy variant), a course of action (blocking TCP port 80 traffic), and three relationships connecting them - two `indicates` relationships from the indicators to the malware, and a `mitigates` relationship from the course of action to the malware. The graph renders all of that cleanly, shows the object types, and lets you trace the relationships interactively. For correlating STIX-structured intel it's a nice lightweight alternative to firing up Maltego.

_(screenshot)_

---

## Data Architecture

Worth a quick look at how everything is stored, because it affects how you think about the platform.

Everything lives in a single SQLite file at `cti-platform/database/cti_platform.db`. No database server to manage. The IOC store is the authoritative source and other modules link to it rather than duplicating rows. CTI Memory lives separately at `cti-platform/data/zettelforge/` - it's the searchable context layer, not a second IOC database.

Cross-module linking uses stable reference keys (`odysafe:source:{id}`, `odysafe:ioc:{id}`, etc.) stored in a `cross_refs` table with content hashes so the Memory index doesn't re-process unchanged content.

All data stays in the installation directory:

|Type|Location|
|---|---|
|Uploads|`cti-platform/uploads/`|
|IOC exports|`cti-platform/outputs/iocs/`|
|Reports|`cti-platform/outputs/reports/`|
|Database|`cti-platform/database/`|
|CTI Memory|`cti-platform/data/zettelforge/`|
|SSL certs|`cti-platform/ssl/`|

---

## Backup and Restore

**Settings → Backup & Restore** exports the full workspace as a ZIP. The v2.1 format includes IOCs, sources, tags, groups, MITRE IOC links, the Memory index registry, log analyzer incidents, STIX models, CTI favorites, and CTI Memory. Restore merges into the existing workspace - same source + type + value updates lifecycle fields rather than creating duplicates.

It's worth automating this with a cron job via the API endpoint:

bash

```bash
curl -k -X POST https://localhost:5001/api/settings/export-zip \
  -o backup-$(date +%Y%m%d).zip
```

_(screenshot)_

---

## Suggested Workflow

Once you're settled in, the flow that makes sense to me is:

**Extraction → IOCs → Memory/Analysis → Flash Report → Export**

1. Import threat reports, log files, or paste IOCs via Extraction
2. Validate indicators in the IOC view - mark TP/FP, set TLP, add context notes, link MITRE techniques
3. Search your workspace via Memory or `Ctrl+K`, run Log Analyzer on relevant log files
4. Build a Flash Report if you need a structured deliverable
5. Export scoped IOC lists for firewalls, EDR, or downstream tooling
