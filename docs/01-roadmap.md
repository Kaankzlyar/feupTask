# Roadmap

26 issues across five milestones. This document sets out the order you work
them in, what each step produces, and how you know a step is finished.

Issue numbers follow the order they were created on GitHub, but the working
order is not always the numeric order. Where I changed it, the reason is
written next to it.

New to this stack? Read [orientation.md](orientation.md) first. It maps what
transfers from React Native and what genuinely is new.

## How to work

Open one issue at a time. Create a branch per issue, something like
`feature/<number>-short-name`, finish the work, open a PR, read your own PR,
merge it. Skipping the PR step will be tempting because this is a practice
project. Do not skip it. The trail it leaves is part of what the real thesis
work will be judged on.

Every milestone should leave the repository in a working state. After M1 you
have procedures written down. After M2 you have a system that runs. After M3
you have an interface people can use. After M4 the graph has real data in it.

You write the code inside the container, not on your Windows machine. Connect
with the VS Code Remote - SSH extension and files open directly on the server,
which spares you any copying. Setup, port forwarding, and the division of
labor between the two machines are covered in [00-setup.md](00-setup.md).

## Step 0: an hour with SPARQL before you start

This step is not an issue and produces nothing you keep. Do it anyway if you
have never worked with RDF.

Install GraphDB, create a throwaway repository, and load one of the sample
datasets that ships with it. Then write five or six SPARQL queries in the
workbench: list everything of one type, filter by a property, follow a
relationship between two things, count some rows, sort them.

The reason is issue #2. You are about to design an ontology, and designing a
data model in a query language you have never written is guesswork. An hour
of playing first will change the decisions you make, and throwing the
repository away afterwards costs you nothing.

## M0: Foundation (#1 - #4)

Before writing code you need to know what you are modeling. This entire
milestone is writing, and it is the part you will most want to skip. Skip it
and you will be redesigning the ontology partway through M3.

### Step 1, issue #1: vision, scope, and architecture

Write down what you are building and what you are not. List the out-of-scope
items explicitly, because this project can quietly grow into a full ITSM
product.

The architecture section needs at minimum the components (GraphDB, backend
API, frontend, ETL), the arrows between them, and how data travels from Excel
into the graph.

Produces: `docs/02-architecture.md`.
Done when: someone who has not seen the project can read it and explain what
the system does.

### Step 2, issue #2: the domain ontology

An RDF/RDFS schema. The things you model are roughly these: lab, hardware
asset, software, license, service, staff member. Then the relationships
between them and the properties on each class.

Pick your namespace at the start and never change it. Type date fields such as
license expiry as `xsd:date`, because the expiry dashboard in #18 will compare
against them and string comparison will not work.

Produces: `ontology/dei-infolab.ttl` plus a short note explaining the schema.
Done when: the Turtle file loads without a syntax error.

### Step 3, issue #3: technology stack and repository skeleton

The backend is Node.js, that much is settled. What you have to decide: the
HTTP framework, the SPARQL client, the frontend framework, the Excel reader,
and the RDF serialization library.

Write one sentence of reasoning per choice, and name the alternative you
rejected. When the ETL library lets you down halfway through M4, you will want
to know why you picked it.

Fix the directory layout in the same issue. Moving things later makes the PR
history unreadable.

Produces: `docs/03-tech-stack.md`, an initial `package.json`, and the
directory skeleton.
Done when: `npm install` runs without errors.

### Step 4, issue #4: the SPARQL query catalogue

Do not start this before the ontology is done. Write one query per screen you
plan in M3: asset list, filtered search, asset detail, lab detail, service
catalogue, expiring licenses.

Keep the queries in files rather than embedded in code, and have the backend
read them from disk. That way the SPARQL console in #21 can reuse the same
files.

Produces: `queries/*.rq` and an index describing what each one returns.
Done when: every query runs against an empty graph without a syntax error.

## M1: Server configuration (#5 - #9)

The equivalent of step one of the real task. What you produce here is not a
running system but a repeatable procedure. If a server were wiped, these
documents should be enough to rebuild it.

### Step 5, issue #5: base server setup

Operating system choice, users and groups, SSH configuration, firewall, patch
policy. Do not run services as root; define a separate system user for the
application.

The good part of this issue is that the container you already connect to is
exactly its subject. The current `sshd_config` has `PermitRootLogin no` (good)
and `PasswordAuthentication yes` (needs hardening). Generate a key on Windows
with `ssh-keygen`, add the public key to `~/.ssh/authorized_keys` in the
container, confirm you can log in with the key, then turn password
authentication off.

You can lock yourself out doing this. Keep a second SSH session open the whole
time, and only close it once you have confirmed key login works from a fresh
third session.

systemd is not running here, so parts of the procedure cannot be verified.
Separate the commands you actually tested from the ones you only wrote down.

Produces: `docs/runbook/01-server-setup.md`.

### Step 6, issue #6: Java runtime and GraphDB installation procedure

JDK 17 is already installed in the container, so the Java part is verification
and version pinning. The GraphDB part needs a manual download, with the steps
in 00-setup.md. Move them here with the server context added: which directory
it installs into, which user runs it, what the memory settings are.

Produces: `docs/runbook/02-graphdb-install.md`.

### Step 7, issue #7: reverse proxy, ports, and TLS

nginx puts GraphDB (7200), the backend API (3000), and the frontend behind a
single port. The port map is already fixed in 00-setup.md. Stay consistent
with it, and keep it matching the `LocalForward` lines in your `.ssh/config`.

Do not expose the GraphDB workbench to the outside. Leave it as an internal
service that only the backend reaches, and expose only your own API.

Because SSH is the container's only externally reachable port, every browser
test you run goes through the tunnel. When a page does not load, check whether
the tunnel is still up before you start editing nginx config.

Plan TLS, but you will not get a real certificate: the container has no
outward-facing domain name. Write the certificate issuing and renewal steps as
procedure and test with a self-signed certificate.

Produces: `docs/runbook/03-reverse-proxy.md` and example config under
`deploy/nginx/`.

### Step 8, issue #8: environment variables, secrets, and config

This issue is labeled `priority:high` and deserves it. GraphDB credentials,
the backoffice session key, service addresses: all of it comes from the
environment.

Keep a `.env.example` in the repository and never a `.env`. Add `.env` to
`.gitignore` during this step rather than later. Removing a committed secret
from history is a separate and annoying job.

Produces: `.env.example`, a `.gitignore` update, `docs/runbook/04-config.md`.
Done when: `git log -p` contains no real passwords.

### Step 9, issue #9: backup and restore

Two things need backing up: the contents of the GraphDB repository and the
configuration files. Taking a backup is the easy half. The real work is
restoring one and showing that it worked.

cron does not run under systemd here, so you will have to start it by hand
with `service cron start`. Alternatively, run the backup script manually,
practice the restore, and write the scheduling part as procedure.

Produces: `scripts/backup.sh`, `scripts/restore.sh`,
`docs/runbook/05-backup.md`.
Done when: you can delete the repository and bring it back from a backup.

## M2: System deployment (#10 - #15)

Step two of the real task. Apply the procedures you wrote in M1 and get the
system running.

### Step 10, issue #10: GraphDB deployment and repository

Install GraphDB, start it, create a repository, and load the M0 ontology into
it.

Record the repository name and settings. The endpoint address the backend
connects to comes out of this step.

Done when: a SPARQL query over `curl` returns the list of classes from your
ontology.

### Step 11, issue #11: backend API deployment

The service that reads your M0 query catalogue and serves it over HTTP. You do
not need every endpoint at this stage. A health check and one real query are
enough; the actual endpoints arrive in M3.

Done when: the service answers through nginx.

### Step 12, issue #12: frontend deployment

Serve the static build output from nginx. The content does not matter yet. One
page that proves it can reach the backend is enough.

Done when: with the SSH tunnel open, `http://localhost:8080` loads in your
Windows browser and the page can fetch data from the backend.

### Step 13, issue #13: the Omega-S analogue

This issue is labeled `needs-info` and should stay that way. Nobody knows what
"Omega S" is in the real task. In this practice project it is modeled as a
separate data and integration platform.

My advice: leave this step until the end of M2 and keep it narrow. Build one
integration point that pulls data from outside, say a periodic fetch from a
CSV or JSON source. When the real definition of Omega S turns up you will
rewrite this step. Designing a large integration layer now will most likely be
wasted effort.

When you close the issue, write down which assumption you worked from.

### Step 14, issue #14: process management and health checks

There is a deviation here, so document it. The issue says "systemd/pm2" but
systemd does not run in this container. Use pm2.

Manage the backend with pm2, and GraphDB too if you want, set it to come back
up after a restart, add a `/health` endpoint, and have pm2 watch it.

Write the systemd unit file anyway and keep it under `deploy/systemd/`. That
is what the real InfoLab-DEI server will use.

Done when: the system recovers on its own after `pm2 restart`.

### Step 15, issue #15: smoke test and deployment verification

The closing step of M2. A checklist to run after a fresh install, and a script
that runs it.

Put at least these in the checklist: are the expected ports listening
(`ss -tlnp`, once iproute2 is installed), does the GraphDB repository answer,
does the backend health check pass, does nginx route both paths correctly.

Produces: `docs/runbook/06-smoke-test.md` and `scripts/smoke-test.sh`.
Done when: the script runs in one command and every check passes.

## M3: Software update (#16 - #21)

Step three of the real task, and the longest part of the project. There are
two separate interfaces: the end-user side is read-oriented, the backoffice
side is write-oriented.

I am breaking the numeric order here. Backoffice authentication (#19) has to
come before CRUD (#20). Otherwise you build an interface that writes without
authorization, and adding security afterwards means going back through every
endpoint.

Suggested order: #16, #17, #18, #19, #20, #21.

### Step 16, issue #16: asset list, filters, and search

The first screen on the end-user side. Pagination, filtering by type and lab,
text search.

In SPARQL, pagination uses `LIMIT` and `OFFSET`, but the total row count needs
a separate `COUNT` query. Plan for that from the start.

### Step 17, issue #17: asset and lab detail pages

What opens when someone clicks a row. The asset's properties, the lab it
belongs to, the software installed on it, and the licenses attached.

The detail query pulls in relationships, so it will be a different query from
the list query in #16. Do not try to merge them into one.

### Step 18, issue #18: service catalogue and license expiry dashboard

A list of services, plus a dashboard of licenses that have expired or are
about to. Make the threshold configurable, 30 days for example.

This screen depends on the date typing in the ontology. If expiry dates were
not typed as `xsd:date`, comparison will not work and you will be back in #2.

### Step 19, issue #19: backoffice authentication and authorization

Labeled `priority:high`. Define at least two roles, one that can read and one
that can write. Decide session handling and password storage here.

Read the credentials from the config mechanism you built in #8.

Done when: an unauthorized request returns 401 or 403 and a test proves it.

### Step 20, issue #20: backoffice CRUD

Create, edit, and delete for assets, software, licenses, and staff records.

Updates in SPARQL are a `DELETE` paired with an `INSERT`, and a badly written
`DELETE WHERE` removes more than you meant. Before every write, run a query
that counts the triples it will affect, then delete. This is where the backup
procedure from #9 earns its keep.

### Step 21, issue #21: SPARQL console and audit log

A free-query screen for administrators, and a record of what changed.

Restrict the console to read queries only. Reject anything containing
`INSERT`, `DELETE`, `DROP`, or `CLEAR`. Writes go through the audited
endpoints from #20.

The audit log records who changed what and when. You can keep it inside the
graph in a separate named graph or in a plain file outside it. Either is
defensible; write down which you chose.

## M4: Data creation (#22 - #26)

Step four of the real task. Filling the graph from the Excel files the
department already has.

### Step 22, issue #22: Excel templates and ontology mapping

Define the template before writing any code. Which column maps to which class
or property, which columns are mandatory, how identifiers get generated.

Identifier generation is the part that matters. Loading the same row twice has
to produce the same URI, or the graph fills up with duplicates. If there is a
natural key such as an inventory number, use it.

Produces: `docs/04-excel-schema.md` and sample files under `templates/`.

### Step 23, issue #23: Excel to RDF ETL

A Node script that reads the template and emits Turtle. Keep conversion and
loading separate: write to a file first, eyeball it, then send it to GraphDB.
Load directly in one step and you will only find out about a mistake after the
graph is already wrong.

Done when: the Turtle generated from a sample Excel file loads without syntax
errors.

### Step 24, issue #24: triggering from the backoffice, and deduplication

Move the ETL off the command line and into the interface. File upload, run,
show the result.

Deduplication rests on the identifier generation from #22. Uploading the same
file twice must not increase the triple count. Pin that down with a test.

### Step 25, issue #25: data quality checks and load report

Checks that run before loading: is a mandatory field empty, is the date format
valid, does the referenced lab exist in the graph.

Rows that fail a check get skipped and listed in the report rather than
halting the load. At the end, show how many rows were processed, how many were
skipped, and how many triples were added.

### Step 26, issue #26: end-to-end test

Run the whole flow with a realistic volume of sample data: upload an Excel
file, let the ETL run, see the data in the end-user interface, edit a record
from the backoffice, and confirm the change landed in the audit log.

This is not the same as the smoke test in #15. That one checked the system was
up. This one checks the workflow is correct.

Done when: the whole scenario passes without manual intervention.

## Open questions

Nobody knows what Omega S actually is (#13). The practice project proceeds on
an assumption. When the real definition surfaces, #13 and probably #11 will
need revisiting.

systemd does not run in the container, so the systemd side of #14 can be
written but not verified. Testing that procedure is the first thing to do on
the real server.

SSH is the container's only externally exposed port. The TLS work in #7 and
the frontend access in #12 will therefore be tested through a tunnel rather
than against a real domain name. Both steps need a second look when you move
to the real InfoLab-DEI server.
