# INTR

A single-user task tracker for people with more work than time.  Named after
the interrupt request pin on the Intel 8088 — because something is always
driving that pin high.

The queue is priority-ordered top to bottom, but tasks don't have to be
handled in priority order.  You grab whatever makes sense to work on, and
the top slot shows what you're currently doing.  Finished tasks move to a
yearly done archive.  Co-workers can see your queue without logging in.
Only you can change anything.

## Files

```
tasks.py            Main task queue (CGI script)
done.py             Completed task archive (CGI script)
intr-banner.svg     Page banner — tasks page
intr-done-banner.svg  Page banner — done page
intr-logo.svg       64px page header logo
intr-favicon.svg    32px favicon (SVG)
favicon.ico         Favicon (ICO, 16x16 + 32x32)
```

The banners are embedded directly in the Python source as string constants
so no external image files need to be deployed alongside the scripts.  The
standalone SVG files are provided for use in documentation, wikis, or
anywhere else you want the artwork.

## Requirements

- Apache 2.4 with `mod_cgi` and `mod_rewrite`
- Python 3 (stdlib only — no third-party packages)
- A machine on a trusted internal network

## Data files

The scripts maintain two kinds of JSON files in a data directory that must
**not** be under the web root:

```
/var/www/taskdata/
    tasks.json          Current queue — auto-created on first write
    done_2026.json      Completed tasks for the year — auto-created
    done_2025.json      Previous years accumulate here automatically
    ...
```

`tasks.json` structure:

```json
{
    "current":      { "id": "...", "name": "...", "notes": "...", "created_at": "..." },
    "idle_active":  false,
    "idle_notes":   "",
    "queue":        [ { "id": "...", "name": "...", "notes": "...", "created_at": "..." } ]
}
```

`queue[0]` is the highest priority waiting task.  `current` is what is being
worked on right now.  When `idle_active` is true the Idle state is active and
`current` is null.

`done_YYYY.json` is a flat array of completed task objects, each with
`name`, `notes`, `created_at`, and `completed_at` fields.

## Configuration

At the top of both `tasks.py` and `done.py`, adjust these two path constants
to match your deployment:

```python
TASKS_FILE = '/var/www/taskdata/tasks.json'
DONE_DIR   = '/var/www/taskdata'
```

## Installation

### 1. Enable Apache modules

```bash
sudo a2enmod cgi rewrite
sudo systemctl restart apache2
```

### 2. Make scripts executable and symlink into place

```bash
chmod 755 /path/to/tasks.py /path/to/done.py
ln -s /path/to/tasks.py /var/www/cgi-me/tasks.py
ln -s /path/to/done.py  /var/www/cgi-me/done.py
```

`/var/www/cgi-me/` is the script directory used in the example Apache config
below.  Adjust to taste.

### 3. Create the data directory

```bash
sudo mkdir -p /var/www/taskdata
sudo chown www-data:www-data /var/www/taskdata
sudo chmod 750 /var/www/taskdata
```

### 4. Apache VirtualHost config

GETs are served freely.  POSTs are silently rewritten to an auth-protected
location.  Both aliases point at the same script directory so the same files
handle both.  The `LocationMatch` block scopes the rewrite to only these two
scripts — other CGI on the server is unaffected.

Add the following inside your `<VirtualHost>` block:

```apache
# Both paths serve from the same script directory.
# /cgi-bin/      -> open to all (GET)
# /cgi-bin-auth/ -> requires Basic Auth (POST target)
ScriptAlias /cgi-bin/      "/var/www/cgi-me/"
ScriptAlias /cgi-bin-auth/ "/var/www/cgi-me/"

<Directory "/var/www/cgi-me">
    Options +ExecCGI +FollowSymLinks
    AddHandler cgi-script .py
    Require ip 127.0.0.1 ::1
    Require host .example.com
</Directory>

# Rewrite POSTs to the auth-protected path.
# Scoped to just these two scripts; other CGI is unaffected.
<LocationMatch "^/cgi-bin/(tasks|done)\.py$">
    RewriteEngine On
    RewriteCond %{REQUEST_METHOD} POST
    RewriteRule ^/cgi-bin/(tasks|done)\.py$ /cgi-bin-auth/$1.py [PT,L]
</LocationMatch>

# Auth gate for the POST target.
<Location "/cgi-bin-auth/">
    AuthType Basic
    AuthName "INTR"
    AuthUserFile /etc/apache2/taskstack.passwd
    Require valid-user
</Location>
```

Adjust the `Require ip` and `Require host` lines to match the networks and
domains you want to allow read access from.

### 5. Create the password file

```bash
sudo htpasswd -c /etc/apache2/taskstack.passwd yourusername
sudo chown root:www-data /etc/apache2/taskstack.passwd
sudo chmod 640 /etc/apache2/taskstack.passwd
```

After this, `GET /cgi-bin/tasks.py` is open to anyone matching the `Require`
directives in the `Directory` block.  POST requests trigger a Basic Auth
challenge.  `REMOTE_USER` is set by Apache after a successful login and the
scripts check it before acting on any write operation.

## Usage

### Task queue (tasks.py)

| Action | Description |
|--------|-------------|
| Push new task | Add a task at the top (becomes current), at position #2, or at the bottom |
| Work on | Grab any queued task — it becomes current, old current returns to top of queue |
| ▲ / ▼ | Nudge a task up or down in the queue without touching the current task |
| Mark done | Completes the current task, logs it to `done_YYYY.json`, promotes the next queued task |
| Done (from queue) | Complete a queued task without grabbing it first |
| Drop | Delete a task permanently with no log entry |
| Edit | Edit the name or notes of any task inline |
| Set idle | Parks the current task back at the top of the queue, sets state to Idle (HLT) |

All write actions require HTTP Basic Auth.  Read access is open.

### Done archive (done.py)

Completed tasks grouped by ISO calendar week, newest week first.  Columns:
task name, date completed, time spent in the stack.

A year selector appears automatically when more than one year's data is
present.  No cron job or manual action is needed at year-end — the scripts
detect the current year and create a new file automatically.

## Network and security notes

This tool is designed for use on a **trusted internal network**.  Do not
expose it to the public internet without HTTPS.  Adding SSL via Let's Encrypt
or a self-signed cert is recommended even internally if the machine is
accessible across subnets.

The scripts do not implement CSRF protection.  On an internal network behind
a firewall this is generally acceptable for a single-user tool, but be aware
of the tradeoff if your threat model requires it.

## Year rollover

At year-end, `done_YYYY.json` stays in place and a new file is created for
the incoming year.  The done page auto-detects all `done_*.json` files in the
data directory and shows a year selector when more than one is present.
Nothing needs to be done manually.

## AI disclosure

> This code was generated with Claude, which is an Artificial Intelligence
> service provided by Anthropic. Though design and development was orchestrated
> by a human, reviewed by a human and tested by a human, most of the actual code
> was composed by an AI.
>
> It is completely reasonable to forbid AI generated software in some contexts.
> Please check the contribution guidelines of any projects you participate in.
> If the project has a rule against AI generated software then DO NOT INCLUDE
> THIS FILE, in whole or in part, in your patches or pull requests.
