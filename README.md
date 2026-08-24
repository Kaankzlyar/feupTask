# DEI InfoLab: IT asset and service catalogue

A web system that keeps a university department's labs, hardware, software
licenses, services, and staff in a single RDF graph. The data lives in GraphDB,
queries are written in SPARQL, and records are loaded from Excel files.

It has two interfaces. The end-user side searches and browses the catalogue.
The backoffice side manages the records.

## About this repository

This is a practice project. It mirrors the four steps of a FEUP/MEIC thesis
continuation as milestones, so that the same technologies get walked through
once in the same order before the real work starts.

| Milestone | Issues | Equivalent in the real task |
|---|---|---|
| M0: Foundation | #1 - #4 | preparation, not a separate step in the real task |
| M1: Server configuration | #5 - #9 | setting up the InfoLab-DEI server |
| M2: System deployment | #10 - #15 | putting GraphDB and the software into service |
| M3: Software update | #16 - #21 | end-user and backoffice interfaces |
| M4: Data creation | #22 - #26 | loading data from Excel files |

## Working environment

Two machines. Your Windows PC is the workstation, and a Debian 12 container
running on a remote Ubuntu server plays the server role. You connect over SSH,
GraphDB and the backend run there, and browser access goes through an SSH
tunnel.

This arrangement mirrors step one of the real task, configuring a remote
InfoLab-DEI server. Details and port forwarding are in 00-setup.md.

## Where to start

Read these in order:

1. [docs/orientation.md](docs/orientation.md), what transfers from a
   JavaScript and React background and what is new. Start here if you have not
   worked with RDF or run a Linux server before.
2. [docs/00-setup.md](docs/00-setup.md), what to install on each machine and
   when. The container's two constraints are described here and both affect
   later steps.
3. [docs/01-roadmap.md](docs/01-roadmap.md), the 26 issues as ordered steps,
   each with the artifact it produces and a test for when it is finished.

Then begin with M0, issue #1.

## Technology

The backend is Node.js. The data store is GraphDB, running on JDK 17. The
frontend framework, SPARQL client, and Excel reader get chosen in issue #3.

Nothing gets installed on Windows except an SSH client and, preferably, the
VS Code Remote - SSH extension.

## Issue tracking

```bash
gh issue list --milestone "M0: Hazırlık & Temel"
gh issue view 1
```

Milestone and issue titles on GitHub are in Turkish. The documentation in this
repository is in English.
