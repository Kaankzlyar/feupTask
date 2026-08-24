# Setup

There are two machines in this project, and what you install on each one is
different. Getting them confused will cost you hours, so start with the layout.

Verified on 2026-08-24.

## The two machines

```
Windows PC                Ubuntu server            Debian 12 container
(your workstation)  SSH   (Docker host)            (plays the server role)
88.230.1.9         ----->  kernel 7.0.0-28-generic  172.17.0.2
                                                    hostname 4071f78a27e2
```

Your Windows machine is the workstation. You write code there, open the
browser there, and SSH out from there.

The Debian 12 container plays the server. GraphDB, the backend, and nginx all
run inside it. This is not a WSL2 distribution and not Docker Desktop: there is
no `/mnt/c`, `WSL_DISTRO_NAME` is empty, and the kernel is an Ubuntu server
build. The container runs on a separate Linux machine that you reach over the
internet via SSH.

That split works in your favor. Step one of the real thesis task is
"configure the InfoLab-DEI server", which means setting up a remote machine
over SSH. What you already have mirrors that exactly. If you install
everything on Windows and skip the server side, you skip the part of the
practice project that has the most to teach you.

## What to install on Windows

Short list. You do not need Node, Java, or GraphDB on Windows. All of that
runs in the container.

| Tool | Needed | Note |
|---|---|---|
| OpenSSH client | yes | already present on Windows 10 (1809+) and 11 |
| VS Code + Remote - SSH extension | recommended | edit files directly inside the container |
| Windows Terminal | optional | nicer than a bare PowerShell window |
| Browser | yes | you have one |
| Git for Windows | no | git is installed in the container, work there |
| Node.js for Windows | no | same reason |

Confirm OpenSSH is there. In PowerShell:

```powershell
ssh -V
```

If you get a version string, you are set. If not, add OpenSSH Client from
Settings under Optional Features.

For VS Code, the only thing to install is the Remote - SSH extension. Connect
to the server from the command palette and you edit files in the container
directly. No copying, no syncing, and the integrated terminal runs on the
server rather than on Windows.

## Port forwarding

Do not skip this section. The only port the container exposes is SSH.
GraphDB's web interface, your backend API, and the frontend are all
unreachable from your Windows browser until you open a tunnel.

The port map, which issue #7 will reuse:

| Port | What runs there | Who reaches it |
|---|---|---|
| 7200 | GraphDB workbench | only you, for administration |
| 3000 | backend API | nginx, plus you during development |
| 80 | nginx (fronts the frontend and the API) | the real entry point |

A one-off connection from PowerShell:

```powershell
ssh -L 7200:localhost:7200 -L 3000:localhost:3000 -L 8080:localhost:80 kutay@SERVER -p PORT
```

Replace `SERVER` and `PORT` with the address and port you already use to
connect. While that tunnel is open, `http://localhost:7200` in your Windows
browser opens GraphDB and `http://localhost:8080` opens nginx.

Typing that every time gets old. Put this in
`C:\Users\<you>\.ssh\config` on Windows instead:

```
Host feup
    HostName SERVER
    Port PORT
    User kutay
    LocalForward 7200 localhost:7200
    LocalForward 3000 localhost:3000
    LocalForward 8080 localhost:80
```

After that, `ssh feup` is the whole command. VS Code Remote - SSH reads the
same config file, so picking that host in VS Code sets up the tunnels too.

## What the container already has

Environment: Debian 12 (bookworm), x86_64, 125 GB RAM, 648 GB free under
`/workspace`, passwordless sudo available.

| Tool | Version | Where you need it |
|---|---|---|
| git | 2.39.5 | everywhere |
| Node.js | 22.22.2 | backend, frontend, ETL |
| npm | 10.9.7 | packages |
| GitHub CLI (`gh`) | 2.94.0 | issue tracking |
| OpenJDK (JDK, includes javac) | 17.0.19 | GraphDB's runtime |
| curl | 7.88.1 | testing APIs and the SPARQL endpoint |
| unzip, tar | - | unpacking GraphDB, taking backups |

GraphDB 10.x wants Java 11 or 17. The installed JDK 17 satisfies that, so
there is no Java to install.

## What the container is missing

| Tool | When you need it | Install |
|---|---|---|
| jq | now (for reading SPARQL JSON output) | `sudo apt-get install -y jq` |
| iproute2 (`ss`) | M2 (#15, checking ports) | `sudo apt-get install -y iproute2` |
| nginx | M1 (#7), M2 (#12) | `sudo apt-get install -y nginx` |
| rsync | M1 (#9) | `sudo apt-get install -y rsync` |
| cron | M1 (#9) | `sudo apt-get install -y cron` |
| pm2 | M2 (#14) | `npm install -g pm2` |
| GraphDB | end of M1 / start of M2 | manual download, see below |

Docker and Python 3 are not installed and you will not need either one.

## Two constraints in the container

First: systemd is not running. PID 1 is `sshd` and
`systemctl is-system-running` returns `offline`. The practical consequence is
that you cannot do issue #14's process management with systemd unit files.
You will use pm2. Write the systemd procedure anyway, since the real
InfoLab-DEI server will have systemd, but note in the issue that you could not
run it here.

For the same reason, services start with `service` rather than `systemctl`:

```bash
sudo service nginx start
sudo service cron start
```

Second: there is no Python 3. If you write the Excel-to-RDF ETL (#23, #24) in
Node, you never need to install it. Which libraries you use is a decision for
issue #3.

## Step by step

### Do this now

On Windows, confirm `ssh -V` works and add the `feup` entry above to your
`.ssh\config`. In the container:

```bash
sudo apt-get update
sudo apt-get install -y jq iproute2
```

Your `gh` token has no `project` scope (current scopes: `gist`, `read:org`,
`repo`). If you want to add issues to a Projects board from the command line:

```bash
gh auth refresh -s project
```

Skip it if you are not using the board. The issue list alone is enough.

### During M1

```bash
sudo apt-get install -y nginx rsync cron
npm install -g pm2
sudo service nginx start
```

With the tunnel open, check that `http://localhost:8080` in your Windows
browser shows the nginx welcome page. If it does not, the problem is the
tunnel and not nginx. Look there first.

### Installing GraphDB

GraphDB is not in the apt repository, so you download it by hand. Ontotext's
download page has a registration form and sends the link by email.

You will download it on Windows and install it in the container. To move the
file across, from PowerShell:

```powershell
scp -P PORT graphdb-dist.zip kutay@SERVER:/workspace/
```

With the `.ssh\config` entry in place, `scp graphdb-dist.zip feup:/workspace/`
does the same thing. Then, in the container:

```bash
mkdir -p /workspace/opt
unzip /workspace/graphdb-*-dist.zip -d /workspace/opt/
/workspace/opt/graphdb-*/bin/graphdb -d
curl -sS http://localhost:7200/rest/repositories | jq .
```

If that last command returns an empty array (`[]`), GraphDB is up and has no
repositories yet. Creating one is issue #10's job.

To see the workbench interface, open `http://localhost:7200` in your browser
while the tunnel is up.

If the registration form or the download is blocked for you, the fallback is
Apache Jena Fuseki. It needs no registration, downloads directly, runs on the
same JDK 17, and speaks the SPARQL 1.1 protocol. As long as your backend talks
to the SPARQL endpoint over HTTP, switching between the two is a few lines of
configuration. The real thesis task specifies GraphDB, so use Fuseki only if
you get stuck, and record the deviation in issue #10.

### During M3 and M4

Everything you install in these phases is an npm package that lands in the
project directory, with `package.json` keeping the record. Which packages
depends on the technology decision in issue #3, so do not write the list
before you make that decision.

## Verifying

You will know setup is done from these four checks. The first three run in the
container, the last one in your Windows browser:

```bash
node -v                                        # 22.x
java -version                                  # 17.x
curl -sS http://localhost:7200/rest/repositories | jq .
```

Then, with the tunnel open, `http://localhost:7200` should open the GraphDB
workbench. Once it does, you can move on to M2.
