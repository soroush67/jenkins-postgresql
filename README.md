# pg-stack

PostgreSQL 18 + postgres_exporter, deployable to any destination via
`Jenkins + Ansible + Docker Compose` - same shape as this user's other
Jenkins/Ansible projects (`gitlab-backup-with-jenkins-ansible`,
`CockroachDB-Cluster`). Started from a hand-written `docker-compose.yml`
+ config files that had never actually been run together, and hardened
against a real failed Jenkins run along the way - see "Design notes"
below for every real bug found and fixed, not just the ones caught
before the first deploy.

## Quickstart

```
ansible-playbook playbooks/deploy.yml \
  -e pg_password='...' \
  -e pg_exporter_password='...'      # idempotent - safe to rerun

ansible-playbook playbooks/status.yml   # read-only health/connectivity check
```

`pg_password`/`pg_exporter_password` are required and never committed -
`roles/preflight` refuses to deploy with either one empty. Real
deployments should pass these via Ansible Vault or (as the Jenkinsfile
does) a Jenkins credential binding, not typed on a command line that
ends up in shell history.

`inventory/hosts.ini` ships with a placeholder `localhost
ansible_connection=local` entry, used as the fallback when no
`target_host` is given. "Deploy to any destination" means passing
`-e target_host=<ip-or-hostname>` per run - it does **not** need to
already exist in `inventory/hosts.ini` first (see "Design notes" for why
this isn't `--limit` or `-i "<ip>,"`):

```
ansible-playbook playbooks/deploy.yml -e target_host=10.0.1.50 -e pg_password='...' -e pg_exporter_password='...'
```

Add `-e pg_deploy_path=/opt/pg-stack` (Jenkins: `DEPLOY_PATH`) to pin
exactly where `docker-compose.yml` + `data`/`config`/`init`/`logs` get
created - the only path this ever writes to. Defaults to the repo
checkout directory when not set.

Testing against the public images instead of the internal registry
(this sandbox can't reach `docker-hosted.hamainsurance.net`):

```
-e pg_postgres_image=postgres:18.4-alpine3.22 \
-e pg_exporter_image=prometheuscommunity/postgres-exporter:v0.18.1
```

## Design notes

Real issues found while wrapping the original hand-written stack in
Ansible - the same "verify by actually running it" discipline used
throughout this user's other infra projects; every fix here was
confirmed against a real running stack, not assumed from reading the
files.

**The exporter's DB username never matched what it tried to authenticate
as.** `init/01-create-exporter-user.sql` created a role named
`pg_exporter`, but `docker-compose.yml`'s `DATA_SOURCE_NAME` connected as
`postgres_exporter` - the exporter would have failed to authenticate on
every single boot. Neither file was ever actually run together to catch
this. Fixed by standardizing on `postgres_exporter` (`pg_exporter_user`
in `inventory/group_vars/all.yml`), used to render *both* the init SQL
and the connection string, so they can't drift again.

**The exporter was never actually on the same Docker network as
postgres.** The original `postgres_exporter` service had no `networks:`
key at all - under the Compose spec, a service with no explicit
`networks:` joins the auto-created "default" project network, not
whatever custom networks (`back-net`/`db-net`) other services declare.
It could never have reached `postgres:5432` by that hostname. Fixed by
adding explicit `networks: [db-net, back-net]` to the exporter service
too.

**`local all all trust` in `pg_hba.conf` meant unauthenticated,
passwordless, superuser-equivalent access to anyone who could get a
shell in the container** (`docker exec`) - confirmed directly: before
the fix, `docker exec postgresql psql -U appuser -d appdb` connected
with zero authentication; after, it's correctly rejected
("fe_sendauth: no password supplied"). Hardened to `scram-sha-256`,
matching the other two rules that were already password-protected. The
`0.0.0.0/0` rule (fully open, but password-required) was left as-is,
since a published port suggests that's an intentional "reachable from
outside this host" design - not touched without being asked, but now
configurable (`pg_hba_allowed_cidr`) if that reach was never actually
intended.

**PostgreSQL 18's official image rejects the old
`/var/lib/postgresql/data` mount point outright** ("there appears to be
PostgreSQL data in: /var/lib/postgresql/data (unused mount/volume)") -
18+ uses `pg_ctlcluster`-style major-version-specific subdirectories
under a single `/var/lib/postgresql` mount instead. The original file
already had this right (`./data:/var/lib/postgresql`); confirmed
directly rather than assumed, since this is exactly the kind of thing
worth verifying instead of copying blind.

**Chowning the top-level `data`/`config`/`init`/`logs` directories
recursively to the postgres image's own uid (70) locked this deploying
user out of even listing them again.** Ansible always runs unprivileged
here (`become: false`, matching this user's other projects) and can't
chown a path to a uid it doesn't own - the fix shells out to a throwaway
container to do it, the same trick already used to inspect image UIDs.
The first attempt applied that `chown -R` to the *directories themselves*
too, not just their contents - broke `ls` on them immediately and every
subsequent Ansible run that needed to manage those directories. Fixed
with two different strategies depending on whether the container needs
to *write* into a path or just *read* it:
- `data`/`logs` (read-write mounts): created once, handed over to uid 70
  entirely (owner + `0700`), and never touched again - checked via
  `stat` first (which needs no permission on the target itself, only its
  parent), so this deploying user only ever confirms they *exist* on
  reruns, never manages their contents.
- `config`/`init` (read-only mounts): stay owned by this deploying user;
  `0755` (not `0751` - the postgres entrypoint `ls`'s
  `/docker-entrypoint-initdb.d/` to discover init scripts, which needs
  *list*, not just *traverse*, confirmed directly by the entrypoint
  failing with "Permission denied" under `0751`). Files with no secrets
  (`postgresql.conf`, `pg_hba.conf`, `postgres_exporter.yml`) are
  world-readable (`0644`). The one file that *does* hold a secret (the
  exporter-user init SQL, plaintext password) is instead chowned to uid
  70 with `0400` (owner-read-only) - its *filename* is visible via
  directory listing, but its *contents* aren't, even to this deploying
  user. Removed and re-rendered on every deploy first, so a changed
  `pg_exporter_password` doesn't get stuck behind its own previous
  `0400` lockout on a later run.

**`docker-entrypoint-initdb.d` scripts only run once, against a
genuinely empty data directory - and a redeploy with a changed
`pg_password` couldn't recover from it at all.** Confirmed via a second
real failed Jenkins run: a redeploy against an existing data directory,
with the `PG_PASSWORD` credential rotated to a new value, failed
outright with `FATAL: password authentication failed for user
"appuser"`. The original fix (a `PGPASSWORD`-based `ALTER USER` for just
the exporter role) only ever covered `pg_exporter_password`, and even
that relied on *already* authenticating as `appuser` with the *current*
`pg_password` - which doesn't help when `pg_password` itself is what
drifted. Since `POSTGRES_USER=appuser` makes `appuser` the *only*
superuser here (the default `postgres` role doesn't even exist), there
was no other account to recover through - and reintroducing `trust` just
to enable password resets would have undone the whole point of removing
it.

Fixed properly: `pg_hba.conf`'s `local` rule is `peer map=pgmap` instead
of `scram-sha-256` (see `templates/pg_ident.conf.j2`) - peer auth still
needs no password, but only for a connecting process that is actually
running as this exact container's own `postgres` OS user (uid 70), i.e.
`docker exec --user postgres` access to *this specific container* - not
"any local host user" the way `trust` was. Both `appuser`'s own password
and the exporter role's are now resynced through that local peer
connection on every deploy (`docker exec --user postgres ... psql -U
appuser -d appdb -c "ALTER USER ..."`, no `-h`, no remembered password
needed at all), regardless of whether this is a first deploy or a
redeploy after a password rotation. Verified for real: deployed with one
password, redeployed with a different one, confirmed the new password
works, the old one is now rejected, and that `docker exec --user root`
(a different OS identity) does *not* get authenticated as `appuser` -
the peer mapping is scoped to exactly the `postgres` uid, not the whole
container.

**No way to pin exactly where the stack gets deployed on the target,
independent of wherever Jenkins happened to check out the repo.**
`pg_project_root` (`{{ playbook_dir }}/..`) was being used for two
different things at once: locating this repo's own `templates/*.j2`
sources, *and* where `data`/`config`/`init`/`logs`/`docker-compose.yml`
get created - fine as long as those were always the same place, but
wrong the moment a real deployment needs the stack to live somewhere
stable (e.g. `/opt/pg-stack`) instead of inside an ephemeral Jenkins
workspace. Split into two variables: `pg_project_root` stays the repo
checkout (template sources only, never touched by a deploy target
change), `pg_deploy_path` (Jenkins: `DEPLOY_PATH`) is the one path
everything else derives from and gets created under - nothing created
outside it, no per-host subdirectory. Defaults to `pg_project_root` when
not overridden, so local/manual testing needs no extra flag. Verified
directly: a deploy with a custom `pg_deploy_path` created
`docker-compose.yml`/`data`/`config`/`init`/`logs` only under that one
path, nothing at the repo root.

**`--limit <ip>` against a static `inventory/hosts.ini` fails outright
for a destination that was never added to it** - confirmed for real via
an actual failed Jenkins run: `[WARNING]: Could not match supplied host
pattern, ignoring: 192.168.12.98` /
`[ERROR]: Specified inventory, host pattern and/or --limit leaves us
with no hosts to target.` This broke the entire "deploy to any
destination" premise, since a real workflow means typing in a different
IP on every run, not maintaining a growing static inventory file
per-target. The obvious-looking fix - `-i "<ip>,"` (Ansible's ad-hoc
single-host inventory syntax) instead of `--limit` - was tried and
confirmed *not* to work either: `inventory/group_vars/all.yml` never got
picked up at all (every `pg_*` variable came back undefined), because
Ansible's `group_vars`/`host_vars` auto-discovery is relative to a real
inventory *file*'s directory, which an inline ad-hoc string doesn't
have. The actual fix: a small bootstrap play at the top of both
`playbooks/deploy.yml` and `playbooks/status.yml` that uses `add_host`
to inject `target_host` into the *existing* file-based inventory's
`postgres` group for that one run, then targets exactly that host
(`hosts: "{{ target_host | default('postgres') }}"`, never the whole
group, so it doesn't also touch the static placeholder entry). This
keeps the real inventory file as the `group_vars` discovery root while
still accepting any destination at runtime - confirmed directly, not
assumed, including that a `target_host` run touches only itself and a
target_host-less run correctly falls back to whatever's in
`inventory/hosts.ini`.

**Verified end-to-end, not just "should work":** `postgres_exporter`'s
own `/metrics` endpoint returns real `pg_up 1` (not just "container
running") after every deploy; two consecutive `deploy.yml` runs left
both containers' `docker inspect --format '{{.State.StartedAt}}'`
completely unchanged; direct `psql` connections confirmed both
`appuser` and the now-correctly-named `postgres_exporter` role work with
their real passwords; the `trust`-auth removal was confirmed by trying
(and now failing) an unauthenticated local connection, not just by
reading the changed config line.
