# Running the shell API and Keycloak on a Windows VM

For a Windows machine that already has a copy of this folder. It covers the
backend only: PostgreSQL, Keycloak, and the shell API. The four frontends are
unchanged — `npm run dev` in each — and are not part of this guide.

Everything is run from **PowerShell**, from the folder you copied to. That
folder is written as `C:\SOW2` below; substitute your own path.

---

## 0. Prepare what you copied

### 0a. Re-copy, if you copied before today

Several files changed on the Linux machine today, and some of them break the
setup silently if they are stale. **Copy these two folders across again**, over
the top of what you already have:

```
HW-Backend\
scripts\
```

Plus `WINDOWS_SETUP.md` — this file.

What changed, so you can spot it if you would rather check than re-copy:

| Change | Symptom if the old version is used |
| --- | --- |
| `unified-portal-shell-api\.env` gained the session settings | Login fails at the callback; `SESSION_DATABASE_URL` is empty |
| Four modules renamed (`kc_admin`→`core\keycloak_admin`, `oidc_bff`→`login_flow`, `dev_swagger`→`local_email_sign_in`, `dev_login`→`local_login_token`) | `ModuleNotFoundError` at startup if the folders are mixed |
| New `keycloak\import_realm_roles.py`, `export_realm.py`, `realm-roles-export.json` | Step 7 has nothing to run |
| New `scripts\windows\env.ps1` | Step 4 has nothing to dot-source |

If you mix old and new files you will get a half-renamed `app\` folder, which
fails on import. If in doubt, delete `C:\SOW2\HW-Backend\unified-portal-shell-api\app`
before re-copying rather than merging into it.

---

### 0b. Three things the copy brought that you must delete

A file copy carries files, not state. Three of the folders you copied are Linux
build output and will either fail or mislead you.

```powershell
cd C:\SOW2

# A Linux virtualenv. Its bin\ holds ELF binaries; Windows needs Scripts\.
Remove-Item -Recurse -Force HW-Backend\env

# PID files from the Linux run scripts, pointing at processes on another machine.
Remove-Item -Recurse -Force var -ErrorAction SilentlyContinue

# Keycloak's embedded H2 database, left over from before it was pointed at
# PostgreSQL. Unused, but confusing to find later.
Remove-Item -Recurse -Force keycloak-26.7.0\data\h2 -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force keycloak-26.7.0\data\transaction-logs -ErrorAction SilentlyContinue
```

**Do not delete `keycloak-26.7.0\providers\keycloak-magic-link-0.75.jar`.** The
realm's login flow uses the `login-token-verifier` authenticator from that jar,
and `setup_local_dev.py` aborts without it. Keep `data\import\` too.

### What did not travel at all

Your realm's roles, permissions and users live in Keycloak's PostgreSQL
database on the Linux machine. `keycloak-26.7.0\data\import\realm-export.json`
is a 2.5 KB stub — realm settings, the `hw-bff` client, and two placeholder
users with no roles. It does **not** contain:

- the three job roles `admin`, `developer`, `developers`
- the ten `permission:*` roles
- the composites that make `developers` grant its four permissions
- your real accounts and what they hold

Step 7 restores all of that from `HW-Backend\keycloak\realm-roles-export.json`,
which was exported from the Linux realm and is in the copy you already have.

---

## 1. Install the prerequisites

| Software | Version | Notes |
| --- | --- | --- |
| JDK | **21** | Temurin or Microsoft Build of OpenJDK. Keycloak 26.7 requires it. |
| Python | **3.12** | Tick "Add python.exe to PATH" in the installer. |
| PostgreSQL | 16 or 17 | Note the `postgres` password the installer asks for. |

Confirm each, in a new PowerShell window:

```powershell
java -version        # openjdk version "21..."
py -3.12 -V          # Python 3.12.x
psql --version       # psql (PostgreSQL) 16.x or 17.x
```

If `psql` is not found, add PostgreSQL's `bin` to `PATH`, e.g.
`C:\Program Files\PostgreSQL\17\bin`.

Set `JAVA_HOME` — `kc.bat` will not start without it:

```powershell
$env:JAVA_HOME = 'C:\Program Files\Eclipse Adoptium\jdk-21.0.5.11-hotspot'
```

Make it permanent so new windows have it:

```powershell
[Environment]::SetEnvironmentVariable('JAVA_HOME', $env:JAVA_HOME, 'User')
```

---

## 2. Create the two databases

Keycloak and the shell API each get their own database and their own owner.
This is not optional: since PostgreSQL 15 a non-owner cannot create tables in
`public`, and both applications create their tables on first run.

`sudo -u postgres` does not exist here; connect as `postgres` with the password
from the installer.

```powershell
psql -U postgres
```

At the `postgres=#` prompt:

```sql
CREATE ROLE keycloak   LOGIN PASSWORD 'local-keycloak-db';
CREATE ROLE shell_auth LOGIN PASSWORD 'local-shell-session-db';
CREATE DATABASE keycloak   OWNER keycloak;
CREATE DATABASE shell_auth OWNER shell_auth;
\q
```

Verify both roles can actually create tables — a wrong owner surfaces much
later as a confusing runtime error:

```powershell
$env:PGPASSWORD='local-keycloak-db'
psql -h 127.0.0.1 -U keycloak -d keycloak -c "CREATE TABLE _probe(i int); DROP TABLE _probe;"

$env:PGPASSWORD='local-shell-session-db'
psql -h 127.0.0.1 -U shell_auth -d shell_auth -c "CREATE TABLE _probe(i int); DROP TABLE _probe;"

Remove-Item Env:\PGPASSWORD
```

Both must print `DROP TABLE`.

---

## 3. Create the Python environment

```powershell
cd C:\SOW2\HW-Backend
py -3.12 -m venv env
.\env\Scripts\python.exe -m pip install --upgrade pip
.\env\Scripts\pip.exe install -r unified-portal-shell-api\requirements.txt
```

Upgrading pip first matters: an old pip may try to compile `cryptography` from
source instead of downloading the prebuilt wheel, which then demands a Rust
toolchain you do not need.

Check the imports resolve:

```powershell
.\env\Scripts\python.exe -c "import fastapi, uvicorn, asyncpg, jwt, cryptography, pydantic_settings, httpx; print('ok')"
```

---

## 4. Load the Keycloak environment

```powershell
cd C:\SOW2
. .\scripts\windows\env.ps1
```

The leading dot is required — it runs the script in your current window instead
of a child process, so the variables survive. **These last for this window
only.** Every new PowerShell window that runs Keycloak or a setup script needs
this line again.

If PowerShell refuses to run the file:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

That relaxes the policy for this window alone, not the machine.

---

## 5. Start Keycloak

Give Keycloak its own PowerShell window. It runs in the foreground; Ctrl-C
stops it. There is no start/stop script on Windows and none is needed.

```powershell
cd C:\SOW2
. .\scripts\windows\env.ps1
.\keycloak-26.7.0\bin\kc.bat start-dev --import-realm --health-enabled=true
```

First boot takes a minute or two — it builds the Quarkus image and imports the
realm. Wait until this returns `200`, in a **second** window:

```powershell
curl.exe -s -o NUL -w "%{http_code}`n" http://localhost:8080/realms/honeywell
```

> **`curl` in PowerShell is an alias for `Invoke-WebRequest`** and does not
> understand `-s`, `-o`, `-w` or `-H`. Always type **`curl.exe`**. This is the
> single most common way these commands appear broken on Windows.

Poll `/realms/honeywell`, not `/realms/master` — master answers before the
realm import has finished.

The admin console is at <http://localhost:8080/admin/> — `admin` / `admin`.

---

## 6. Configure the realm

In the second window:

```powershell
cd C:\SOW2
. .\scripts\windows\env.ps1
cd HW-Backend\keycloak
..\env\Scripts\python.exe setup_local_dev.py
```

This creates the `hw-bff` and `hw-users` clients, the local login flow, the
`developer@example.com` account and the `admin` role. It is idempotent — run it
again any time. It should end with:

```
Application clients in the realm: hw-bff, hw-users
Local development SSO is ready.
```

The scripts use only the Python standard library, so plain `python` works too
if you would rather not use the venv.

---

## 7. Restore the roles, permissions and users

Still in `HW-Backend\keycloak`:

```powershell
..\env\Scripts\python.exe import_realm_roles.py
```

This reads `realm-roles-export.json` and recreates the 13 roles (three job
roles plus ten permissions), the `developers` composite, and each user with the
roles it held. Also idempotent — `=` means already present, `*` means created.

No passwords are set. Sign-in is by email address against Keycloak, so the
accounts work without one.

To refresh the export later, run `export_realm.py` on the machine whose realm
is correct and copy the resulting JSON across.

---

## 8. Start the shell API

**The `.env` needs no Windows edits.** `HW-Backend\unified-portal-shell-api\.env`
already points at `127.0.0.1:5432` and `localhost:8080`, both correct here.

A third PowerShell window — and note it does **not** need `env.ps1`, because
the API reads its own `.env`:

```powershell
cd C:\SOW2\HW-Backend\unified-portal-shell-api
..\env\Scripts\python.exe -m uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Windows Firewall will prompt on first run because of `--host 0.0.0.0`. Allow it
for private networks, or drop the flag to bind loopback only if nothing off the
VM needs to reach the API.

Swagger is at <http://localhost:8000/docs>.

---

## 9. Verify it actually works

Processes being up proves nothing — a mistyped client secret only fails at
login. Run all four, in a spare window:

```powershell
# 1. the process and its database
curl.exe -s -o NUL -w "ready:    %{http_code}`n" http://localhost:8000/health/ready

# 2. sign in by email, then read back the live roles
curl.exe -s -H "X-Dev-Email: developer@example.com" http://localhost:8000/api/v1/auth/me

# 3. a guarded endpoint: dev login -> hw-users service account -> role check
curl.exe -s -o NUL -w "users:    %{http_code}`n" -H "X-Dev-Email: developer@example.com" "http://localhost:8000/api/v1/users?max=1"

# 4. the same endpoint with no credential
curl.exe -s -o NUL -w "no auth:  %{http_code}`n" "http://localhost:8000/api/v1/users?max=1"
```

Expected: `200`, a JSON profile whose `roles` include `admin`, `200`, then
`401`. Together those exercise the whole chain — email sign-in, the service
account credential, live role lookup from Keycloak, and enforcement.

Then confirm step 7 landed:

```powershell
curl.exe -s -H "X-Dev-Email: developer@example.com" http://localhost:8000/api/v1/users/permissions
```

Ten permissions, `permission:admin:view` through `permission:dashboard:view`.

---

## Daily routine, once set up

Two windows:

```powershell
# window 1
cd C:\SOW2 ; . .\scripts\windows\env.ps1 ; .\keycloak-26.7.0\bin\kc.bat start-dev

# window 2
cd C:\SOW2\HW-Backend\unified-portal-shell-api
..\env\Scripts\python.exe -m uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

`--import-realm` is only needed on a first boot against an empty database.
Ctrl-C in each window to stop. PostgreSQL runs as a Windows service and looks
after itself.

---

## Troubleshooting

**`Missing Keycloak provider(s): login-token-verifier`**
`keycloak-26.7.0\providers\keycloak-magic-link-0.75.jar` is missing. Copy it
back and restart Keycloak — the provider list is read at boot.

**`curl: The remote name could not be resolved` / `A parameter cannot be found that matches parameter name 's'`**
You used `curl`, which is an alias for `Invoke-WebRequest`. Use `curl.exe`.

**`JAVA_HOME is not set` or `kc.bat` exits immediately**
Set `JAVA_HOME` to a JDK **21** directory, and check `java -version` agrees.

**Keycloak starts but `/realms/honeywell` stays 404**
Read the `kc.bat` window first — the import logs there, and it usually says
exactly what went wrong (a malformed `realm-export.json`, or a realm of that
name already present so the import was skipped). Fix what it reports. Only if
it confirms the realm already exists, and you are certain that database holds
nothing you want, recreate it:
`DROP DATABASE keycloak; CREATE DATABASE keycloak OWNER keycloak;` then start
again with `--import-realm`. That erases every user and role in the realm, so
make sure step 7's export file is current before you do it.

**`FATAL: password authentication failed for user "keycloak"`**
The role password does not match `KC_DB_PASSWORD` in `env.ps1`. Reset it:
`ALTER ROLE keycloak PASSWORD 'local-keycloak-db';`

**`invalid_client` at login**
The client secrets in `scripts\windows\env.ps1` and in
`unified-portal-shell-api\.env` have drifted apart. They must match.

**`Port 8080 already in use`**
Docker Desktop running the Compose stack is the usual cause:
`docker compose stop`. Never `docker compose down -v` — that deletes the
database volumes.

**Setup scripts fail with a connection error**
Keycloak is not up yet, or you opened a new window and forgot to dot-source
`env.ps1`.
