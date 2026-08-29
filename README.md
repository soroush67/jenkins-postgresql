# pg-stack

PostgreSQL 18 + postgres_exporter, deployable to any destination via
`Jenkins + Ansible + Docker Compose` - same shape as this user's other
Jenkins/Ansible projects (`gitlab-backup-with-jenkins-ansible`,
`CockroachDB-Cluster`). Started from a hand-written `docker-compose.yml`
+ config files that had never actually been run together - three real
bugs found and fixed along the way, see "Design notes" below.

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
ansible_connection=local` entry - replace it (or add more) with real
hosts before a real deployment. "Deploy to any destination" is just
`--limit <host>` (or pointing at a different inventory) against whatever
host is in there - no code changes per target.

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
genuinely empty data directory** - a redeploy with a changed
`pg_exporter_password` would otherwise silently desync from the actual
DB role's password (the role already exists, so the init SQL is never
re-run to pick up the new value). Fixed with a separate `ALTER USER ...
WITH PASSWORD ...` step that runs on every deploy, idempotently
resyncing the role's password with whatever's currently declared,
regardless of whether this is a first deploy or a rerun.

**Verified end-to-end, not just "should work":** `postgres_exporter`'s
own `/metrics` endpoint returns real `pg_up 1` (not just "container
running") after every deploy; two consecutive `deploy.yml` runs left
both containers' `docker inspect --format '{{.State.StartedAt}}'`
completely unchanged; direct `psql` connections confirmed both
`appuser` and the now-correctly-named `postgres_exporter` role work with
their real passwords; the `trust`-auth removal was confirmed by trying
(and now failing) an unauthenticated local connection, not just by
reading the changed config line.
