# ms-stack

Microsoft SQL Server 2022 + a Prometheus exporter, deployable to any
destination via `Jenkins + Ansible + Docker Compose` - same shape and
same discipline as this user's `pg-stack`/`mongo-stack` projects
(Ansible-templated compose, hardened containers, `DEPLOY_PATH` +
`TARGET_HOST` Jenkins parameters, volumes under `/data/<name>`, never
`become`/sudo). Started from a hand-written `docker-compose.yml` that
had never actually been run hardened - see "Design notes" below for
every real bug/requirement found while getting it there, including the
exporter added later to match `pg-stack`/`mongo-stack`.

## Quickstart

```
ansible-playbook playbooks/deploy.yml --ask-vault-pass -e mssql_exporter_password='...'   # idempotent - safe to rerun
ansible-playbook playbooks/status.yml --vault-password-file .vault-pass                    # read-only health/connectivity check
```

`mssql_sa_password`/`mssql_exporter_password` are required -
`roles/preflight` refuses to deploy with either one empty, and checks
`mssql_sa_password` against SQL Server's own password policy (at least
8 characters, 3 of {uppercase, lowercase, digit, symbol}) before ever
touching Docker. `mssql_sa_password` already has an Ansible-Vault-
encrypted default committed in `inventory/group_vars/all.yml` (see
"Ansible Vault" below) - no `-e mssql_sa_password` needed for
local/manual runs, just the vault password. Jenkins doesn't use that
default - it passes its own value via a Jenkins credential binding
(see the Jenkinsfile), which always overrides it.
`mssql_exporter_password` has no committed default - pass it via `-e`
/ Vault / a Jenkins credential binding.

`-e target_host=<ip-or-hostname>` (Jenkins: `TARGET_HOST`) deploys to
any destination without needing to pre-add it to `inventory/hosts.ini`
first. `-e mssql_deploy_path=/opt/servers/mssql-prod-1` (Jenkins:
`DEPLOY_PATH`) pins exactly where `docker-compose.yml` gets created -
the actual data **volume** lives separately, under `/data/<the last
folder of that path>` (`/data/mssql-prod-1` for the example above).
Both default to sensible local-testing values when left unset.

## Ansible Vault

`mssql_sa_password` in `inventory/group_vars/all.yml` is Ansible-Vault-
encrypted - safe to commit and push as-is, since without the vault
password the file is just ciphertext.

```
ansible-playbook playbooks/deploy.yml --ask-vault-pass -e mssql_exporter_password='...'
# or, non-interactively:
echo 'the-vault-password' > .vault-pass && chmod 600 .vault-pass
ansible-playbook playbooks/deploy.yml --vault-password-file .vault-pass -e mssql_exporter_password='...'
```

**Never commit the vault password itself** (`.vault-pass`, if you
create one, is already covered by `.gitignore` - double check before
committing regardless). It's a separate secret from the admin password
it protects; whoever asked for this to be set up should already have it
out-of-band. Verified directly: a real deploy using only
`--vault-password-file` (no `-e mssql_sa_password`) decrypts correctly
and the resulting password actually authenticates as `sa`.

To rotate the encrypted value later:
```
ansible-vault encrypt_string --vault-password-file .vault-pass --stdin-name 'mssql_sa_password' <<< 'new-password-here'
```
paste the resulting `mssql_sa_password: !vault |` block over the
existing one - but see "Known limitation" below first: this alone does
**not** change the password on an already-running instance.

## Design notes

Real issues found and requirements confirmed while wrapping the original
hand-written stack in Ansible - same "verify by actually running it"
discipline as this user's other infra projects; nothing here was assumed
from Microsoft's docs alone.

**The original file used an external named Docker volume (`ms-data`),
not a bind mount.** Changed to a bind mount under `/data/<the last
folder of DEPLOY_PATH>`, matching `jenkins-postgresql`'s established
convention for this user's infrastructure (their own explicit
requirement there: all volumes live under `/data/`, namespaced by
deployment). Flag if the external-named-volume approach was actually
intentional for this stack specifically - it's a real behavior change
from the original file, not just a wrapper around it.

**`cap_drop: ALL` alone made `sqlservr` fail to even launch** -
`/opt/mssql/bin/launch_sqlservr.sh: line 20: /opt/mssql/bin/sqlservr:
Operation not permitted`, confirmed directly, with no other symptom (the
container's data directory was already correctly owned; this wasn't a
permissions-on-disk problem). Root cause, also confirmed directly via
`getcap`: the `sqlservr` binary has `cap_net_bind_service=ep` set as a
file capability - the kernel only honors a binary's own file
capabilities at exec time if that capability is also present in the
container's capability *bounding set*, which `cap_drop: ALL` empties
out entirely. Fixed with `cap_add: [NET_BIND_SERVICE]` - confirmed as
the complete, minimal fix by testing broader candidate sets first
(`CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `SETUID`, `SETGID`, `SYS_PTRACE`,
`SYS_RESOURCE`, `IPC_LOCK` - none of these fixed it on their own) before
finding the actual one via `getcap` rather than guessing further.

**The mssql image already bakes in `USER mssql` (uid 10001) in its own
Dockerfile** - unlike the postgres image `jenkins-postgresql` uses
(which defaults to root unless told otherwise, letting its entrypoint's
own privilege-dropping dance run). Confirmed directly: a throwaway
container used to `chown` a host directory for this image needs
`--user root` *explicitly* - simply omitting `--user` here still runs as
the image's own non-root default, not root, and the chown fails
("Operation not permitted"). Every privileged-container step in
`roles/mssql` passes `--user root` explicitly for this reason.

**`mssql-tools18`'s `sqlcmd` requires `-C` (trust server certificate)** -
confirmed directly: v18 encrypts connections by default and refuses an
unverified self-signed certificate without it. Used in both the
container healthcheck and every SA-connectivity check this role/the
status playbook run.

**SQL Server's own password policy needed checking up front, not left to
fail inside the container.** A password that doesn't meet the policy
(8+ characters, 3 of {uppercase, lowercase, digit, symbol}) makes the
container fail in a much less obvious way than a clean, up-front Ansible
assertion - `roles/preflight` checks this directly (confirmed: a
deliberately weak test password was correctly rejected before Docker was
ever touched).

**The rest of the permission/idempotency/`DEPLOY_PATH`/`target_host`
design is identical to `jenkins-postgresql`**, including two real bugs
already found and fixed there and inherited correctly here rather than
re-discovered: `realpath` (not a bare `basename`) is required to derive
`/data/<name>` from `mssql_deploy_path`'s own default value (which ends
in a literal `/..`); and `ignore_errors: true` (not `failed_when: false`)
is required for the "try directly, fall back to a privileged container"
pattern to actually detect a real permission failure - `failed_when:
false` silently rewrites the registered result's own `failed` key,
breaking the downstream `is failed` check that gates the fallback. See
that project's own README for the full story on both.

**Adding the exporter (to match `pg-stack`/`mongo-stack`): Microsoft
doesn't publish an official one** - `awaragi/prometheus-mssql-exporter`
is the actively maintained community exporter (confirmed via its own
source: queries `sys.dm_os_performance_counters`,
`sys.dm_io_virtual_file_stats`, `sys.dm_os_process_memory`, and similar
DMVs, all covered by a single `VIEW SERVER STATE` grant - never `sa`
itself, matching `postgres_exporter`'s/`mongo_exporter`'s own
separate-role pattern). Pinned by exact tag
(`awaragi/prometheus-mssql-exporter:v1.3.0`) rather than following this
project's own existing `mssql_image` convention of a moving tag -
confirmed directly this resolves to the identical image digest as
`:latest` at the time it was added. The image bakes in no dedicated
non-root user of its own (runs as root by default, confirmed directly) -
`mssql_exporter_run_uid`/`_run_gid` (1000:1000) is an arbitrary but
confirmed-working non-root uid, unlike `mssql_run_uid` which is the
`mssql` image's own real baked-in identity.

**Unlike Postgres/Mongo, SQL Server has no `docker-entrypoint-initdb.d`-
style once-only init mechanism at all**, so the exporter's login can't
be created that way. `templates/exporter-login.sql.j2` is instead
piped via stdin directly into `docker exec -i ... sqlcmd` (confirmed
directly this works cleanly - no file with the exporter password ever
touches disk, unlike the SQL/JS init-script files `pg-stack`/
`mongo-stack` render and then have to lock down with `chmod 0400`) and
is itself idempotent (`CREATE` if the login doesn't exist, `ALTER` to
resync the password if it does) - run fresh on every deploy,
authenticated as `sa`. This is actually simpler than either sibling
project here: `sa`'s own credential is always directly usable to manage
the exporter's login (no Postgres-style `peer` auth trick or
Mongo-style temporary-auth-disable procedure needed) - confirmed
directly that rotating just `mssql_exporter_password` on a redeploy
works cleanly every time, immediately, with no extra step.

**A real first-deploy race, found via an actual deploy, not
anticipated in advance**: the original `docker compose up -d` (with
`mssql_exporter` already in the same compose file, gated by
`depends_on: condition: service_healthy`) starts BOTH services once
`mssql` reports healthy - but the exporter's own login doesn't exist
yet at that point on a first deploy, since `roles/mssql` doesn't create
it until *after* confirming `mssql` is healthy. `mssql_exporter` would
start and fail to authenticate before its login was ever created.
Fixed by splitting the single `up -d` into two explicit steps: `up -d
mssql` first, then (after the exporter login is created/synced) `up -d
mssql_exporter` - confirmed directly a fresh deploy now succeeds
cleanly on the first attempt, exporter authenticated and scraping
immediately, no crash-loop-then-recover.

## Known limitation: `mssql_sa_password` rotation against existing data

**Not fixed here - found while building the exporter, flagged rather
than silently worked around.** Confirmed directly: a redeploy against
an EXISTING data volume with a changed `mssql_sa_password` does not
update the live `sa` password at all - `MSSQL_SA_PASSWORD` is only ever
read on a genuinely fresh `/var/opt/mssql`, the same one-time-init
limitation Postgres and MongoDB both have. Worse here: the compose
file's own healthcheck embeds `mssql_sa_password` directly (`sqlcmd -P
"{{ mssql_sa_password }}"`), so after such a redeploy the container
gets stuck reporting **unhealthy forever** - Docker's healthcheck is
now checking the new (wrong) password against an instance still
running the old one - even though SQL Server itself may be working
fine. `pg-stack` solved the equivalent problem with `peer`-auth local
sockets; `mongo-stack` solved it with a documented temporary-auth-
disable procedure. Neither trick has an obvious SQL-Server-native
equivalent verified yet (the closest documented approach is Microsoft's
own single-user-mode SA-password-recovery procedure) - out of scope for
"add an exporter," flagged here rather than either guessed at or
silently ignored. If this needs fixing, treat it as its own task: a
`playbooks/reset-sa-password.yml` mirroring `mongo-stack`'s
`reset-passwords.yml` shape, verified the same way, is the natural next
step.

**Verified end-to-end, not just "should work":** a real `sqlcmd -Q
"SELECT 1"` against the SA login succeeded after every deploy, not just
a "container running" check; `mssql_exporter`'s own `/metrics` returns
real `mssql_up 1` after a fresh deploy (not just "container running");
two consecutive `deploy.yml` runs left both containers'
`docker inspect --format '{{.State.StartedAt}}'` completely unchanged;
tested the permission-fallback path for real by resetting `/data` to
root-owned and confirming the deploy still completed without stopping;
tested a custom `DEPLOY_PATH` and confirmed `docker-compose.yml` and the
data volume landed in exactly the two separate places expected, nothing
else created anywhere; a real deploy using only `--vault-password-file`
(no `-e mssql_sa_password`) decrypted correctly and authenticated as
`sa`.
