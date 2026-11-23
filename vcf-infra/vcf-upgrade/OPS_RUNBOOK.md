# Running a VCF upgrade from the Ops appliance

This runbook covers deploying the `saltext-vcf` wheel and the `vcf-upgrade` formula onto the
Ops appliance (the Salt minion), then running `salt-call` there to upgrade SDDC Manager, NSX,
vCenter and ESXi hosts through SDDC Manager.

Supported components: `SDDC_MANAGER`, `HCX`, `NSX_T_MANAGER`, `NSX_EDGE`, `VCENTER`, `HOST`.

> **A real run starts a real upgrade.** SDDC Manager upgrades commit immediately, and NSX,
> vCenter and host upgrades reboot appliances or hosts. Always do a `test=True` run first and
> take a snapshot or backup of SDDC Manager (and the other management VMs) before a real run.

## 1. Prerequisites

- The Ops appliance runs a Salt minion and can reach SDDC Manager over HTTPS (port 443).
- An SDDC Manager account that can create upgrades (for example `administrator@vsphere.local`).
- The target bundles are in the SDDC Manager catalog (section 6).
- You can `scp` to the appliance as `root`.

## 2. Build the wheel (on your workstation)

```bash
cd saltext-vcf
python3 -m venv .venv-build && source .venv-build/bin/activate
pip install --upgrade build
python -m build --wheel --outdir dist/
ls -t dist/*.whl | head -1        # the wheel to copy
```

Build from the branch that has the upgrade fixes (depot sync, fixed component order, duplicate
bundle handling, readable HTTP errors). The version string carries the commit and a `d<date>`
suffix when the tree has uncommitted changes.

## 3. Copy and install the wheel on Ops

```bash
scp dist/saltext_vcf-<version>-py2.py3-none-any.whl root@<ops>:/tmp/
```

On Ops:

```bash
salt-pip install --no-deps --force-reinstall /tmp/saltext_vcf-<version>-py2.py3-none-any.whl
systemctl restart salt-minion
salt-call --local sys.list_state_modules | grep vcf_upgrade     # confirms the extension loads
```

- Always use `--no-deps --force-reinstall`. Without `--no-deps`, `salt-pip` tries to reinstall
  Salt and every dependency, which fails behind the proxy and must not be done.
- If `salt-pip` cannot run (for example `/tmp` is mounted `noexec`), unpack the wheel and copy the
  package over the installed one:
  ```bash
  cd /tmp && rm -rf whl && mkdir whl && cd whl && python3 -m zipfile -e /tmp/saltext_vcf-*.whl .
  cp -r saltext/vcf/. /opt/saltstack/salt/extras-3.14/saltext/vcf/
  ```
- Check that the right code is installed:
  `grep -c sync_depot /opt/saltstack/salt/extras-3.14/saltext/vcf/clients/vcf_upgrade.py`
  should print a number above 0.
- The minion may restrict loadable modules with `whitelist_modules` / `whitelist_state_modules`
  (check `salt-call --local config.get whitelist_modules`). If it is set, add `vcf_upgrade` and
  `vcf_sddc_bundles`. Confirm with the appliance owners before changing it.

## 4. Copy the formula onto Ops

The formula is the `vcf-infra/vcf-upgrade/` folder: `vcf-upgrade.sls`, `map.jinja`,
`defaults.yaml`. The SLS imports `vcf-upgrade/map.jinja`, so the folder name matters.

```bash
scp -r vcf-infra/vcf-upgrade root@<ops>:/tmp/
```

On Ops:

```bash
mkdir -p /srv/salt/vcf-upgrade
cp /tmp/vcf-upgrade/vcf-upgrade.sls /tmp/vcf-upgrade/map.jinja /tmp/vcf-upgrade/defaults.yaml /srv/salt/vcf-upgrade/
chmod 755 /srv/salt /srv/salt/vcf-upgrade
chmod 644 /srv/salt/vcf-upgrade/*
sed -i 's/^domain_id:.*/domain_id: null/' /srv/salt/vcf-upgrade/defaults.yaml
```

- `salt-call` runs as the `salt` user (`salt-call --local config.get user`). If `/srv/salt` or
  the folder is not readable by that user you get `No matching sls found`. Directory mode 755 and
  file mode 644 fix it.
- `domain_id` must stay `null` in `defaults.yaml` unless you set a real SDDC Manager domain ID. An
  ID such as `domain-c9` is a vCenter cluster ID and gives `GET /v1/domains/domain-c9 failed: HTTP 404`.
- `salt-call --local config.get file_roots` must list `/srv/salt`.

## 5. Connection pillar and proxy

The SDDC Manager connection is passed as pillar:

```json
{"saltext.vcf": {"sddc_manager": {"host": "<sddc-manager>", "username": "administrator@vsphere.local",
                                  "password": "<pw>", "verify_ssl": false}}}
```

- Use `"verify_ssl": true` with a trusted CA outside a lab.
- A password given on the command line is visible in shell history and in the Salt job record.
  For real environments keep the connection block in Salt pillar and pass only the per-run inputs.
- If the appliance has `HTTPS_PROXY` set, SDDC Manager calls may go through it and fail. Make sure
  the SDDC Manager host is covered by `NO_PROXY` for the minion process.

## 6. Find the exact bundle build

`target_build` must match the catalog exactly, as `<release>-<build>`. A build number belongs to one
release, so `9.2.0-25874375` does not exist if the catalog lists `9.1.2-25874375`.

```bash
TOKEN=$(curl -sk --noproxy '*' -X POST https://<sddc-manager>/v1/tokens -H 'Content-Type: application/json' \
  -d '{"username":"administrator@vsphere.local","password":"<pw>"}' | python3 -c 'import sys,json; print(json.load(sys.stdin)["accessToken"])')

curl -sk --noproxy '*' -H "Authorization: Bearer $TOKEN" https://<sddc-manager>/v1/bundles | python3 -c '
import sys,json
for b in json.load(sys.stdin).get("elements",[]):
    print(b.get("version"), b.get("downloadStatus"), [c.get("type") for c in b.get("components",[])])' | sort -u
```

Add `| grep NSX_T_MANAGER` (or another type) to narrow it. Pick a build whose release matches your
upgrade path, and follow the documented order: SDDC Manager, then NSX, then vCenter, then hosts, then NSX Edge.
The same build can appear twice. The code uses the downloaded entry, or the first one if they share a status.

## 7. Run the upgrade with salt-call

Always run `test=True` first. The dry run checks the configuration and that the bundle exists. It
does not run the real prechecks. Drop `test=True` for the real run.

### One component

```bash
salt-call --local state.apply vcf-upgrade.vcf-upgrade test=True pillar='{
  "saltext.vcf": {
    "sddc_manager": {"host": "<sddc-manager>", "username": "administrator@vsphere.local", "password": "<pw>", "verify_ssl": false},
    "vcf_upgrade": {
      "depot": {"type": "online"},
      "component_type": "<COMPONENT>",
      "target_build": "<release>-<build>",
      "checkpoint_file": "/var/cache/salt/minion/vcf-upgrade-<component>.json"
    }
  }
}'
```

| `component_type` | Extra input |
|---|---|
| `SDDC_MANAGER` | none |
| `HCX` | none. Runs after `SDDC_MANAGER` and before `NSX_T_MANAGER`. Uses a draft upgrade with prechecks. Post-upgrade validation checks only that the domain is `ACTIVE`, not the HCX version. |
| `NSX_T_MANAGER` | `"options": {"nsx_mp_upgrade": true, "nsx_datapath_appliance_upgrade": false}`. The defaults upgrade the management plane only; set the options to what you intend. |
| `NSX_EDGE` | none. Datapath (Edge) phase of the NSX bundle (management plane skipped; `isEdgeOnlyUpgrade` is not used, VCF 9.0+ rejects it); **must run after the ESX host (`HOST`) upgrade**. |
| `VCENTER` | `"options": {"vcenter_upgrade_mechanism": "InPlace"}` (or `ReducedDowntimeMigration`). Temporary networking is `"vcenter_network_mode": "AUTOMATIC"` (DHCP, default) or `"STATIC"`, which also needs `vcenter_temporary_ip`, `vcenter_temporary_subnet_mask` and `vcenter_temporary_gateway`. |
| `HOST` | needs a `sequence` entry with `clusters` (section below): SDDC Manager `cluster_id` and `personality_id`. |

### Several components in one command

A `sequence` always runs in the fixed order **SDDC_MANAGER, HCX, NSX_T_MANAGER, VCENTER, HOST, NSX_EDGE**,
whatever order you list. Only the components you list run.

```bash
salt-call --local state.apply vcf-upgrade.vcf-upgrade test=True pillar='{
  "saltext.vcf": {
    "sddc_manager": {"host": "<sddc-manager>", "username": "administrator@vsphere.local", "password": "<pw>", "verify_ssl": false},
    "vcf_upgrade": {
      "depot": {"type": "online"},
      "sequence": [
        {"type": "SDDC_MANAGER",  "target_build": "<release>-<build>"},
        {"type": "HCX",           "target_build": "<release>-<build>"},
        {"type": "NSX_T_MANAGER", "target_build": "<release>-<build>",
         "options": {"nsx_mp_upgrade": true, "nsx_datapath_appliance_upgrade": false}},
        {"type": "VCENTER",       "target_build": "<release>-<build>",
         "options": {"vcenter_upgrade_mechanism": "InPlace"}},
        {"type": "HOST",          "target_build": "<release>-<build>",
         "clusters": [{"name": "<cluster>", "cluster_id": "<SDDC Manager cluster UUID>", "personality_id": "<personality UUID>"}]},
        {"type": "NSX_EDGE",      "target_build": "<release>-<build>"}
      ]
    }
  }
}'
```

- **`NSX_EDGE` must run after the ESX host upgrade.** SDDC Manager keeps the NSX datapath (Edge)
  phase `PENDING` until the hosts are upgraded; submitting it earlier fails with
  `UPGRADE_SPEC_BUNDLE_NOT_AVAILABLE`. Use the same NSX `target_build` as `NSX_T_MANAGER`. When run
  on its own, check that `GET /v1/upgradables/domains/<domain-id>` shows `NSX_T_DATAPLANE` as
  `AVAILABLE` first.
- The whole sequence is validated, including a bundle lookup for every entry, before anything is submitted.
- `cluster_id` is the SDDC Manager cluster ID from `GET /v1/clusters`, not the vCenter `domain-c9` ID.
  `personality_id` must be an already imported personality for the target ESXi build.
- Each component runs its prechecks first. A failed precheck stops the run, and force or
  bypass options are not supported.

### Depot handling

| `depot` | Behaviour |
|---|---|
| `{"type": "online"}` | Triggers a depot sync on SDDC Manager and waits for it to finish before the bundle lookup. A failed sync stops the run with SDDC Manager's error. Set `"sync": false` to skip it. Dry runs never start a sync. |
| `{"type": "offline", "bundle_file_path": ..., "manifest_file_path": ..., "signature_file_path": ...}` | Registers the staged offline bundle files first. Set `"upload": false` on a rerun after the bundle is registered. |

## 8. Checkpoints and resuming

Progress is saved in `checkpoint_file` (default `/var/cache/salt/minion/vcf-upgrade.json`) with the
remote upgrade IDs.

- To resume an interrupted run, rerun with the same endpoint, targets, options and checkpoint file.
- Use a separate `checkpoint_file` for each separate upgrade. A different target against an old
  checkpoint fails with `Checkpoint belongs to a different endpoint, target or sequence`.
- A request that SDDC Manager rejects outright (HTTP 400, 403, 404, 422) leaves no pending mark. A
  server error or timeout leaves the checkpoint in `CREATING` and the next run stops with
  `submission outcome is unknown`. Check `GET /v1/upgrades` on SDDC Manager. If nothing new was
  created, move the checkpoint aside (`mv ... .bak`). If an upgrade exists, keep the checkpoint
  and resolve it first.
- Do not delete a checkpoint while its remote upgrade is running or half finished.

## 9. Long runs and monitoring

An SDDC Manager upgrade restarts SDDC Manager, so the API is unreachable for a while. The state keeps
polling and tolerates that, for up to the upgrade timeout (12 hours by default;
`lifecycle: {upgrade_timeout_seconds: ...}`). Run it from RaaS (section 10) or the master with `--async` if you
do not want a terminal tied up. Check progress with `GET /v1/upgrades` or the SDDC Manager UI.

## 10. Running the job from the RaaS UI

The same state can run as a RaaS job instead of `salt-call` on Ops. RaaS sends `state.apply` to the Ops
minion through the Salt master, so the formula is served from the master's file server, not from
`/srv/salt` on Ops. The wheel (section 3) still has to be installed on the Ops minion. Menu names
below follow the usual RaaS layout and can differ between versions.

### 10.1 Put the formula in the RaaS file server

In the RaaS UI open the file server for the `base` environment and create the same layout as in section 4,
with the same relative paths and file contents:

```
vcf-upgrade/vcf-upgrade.sls
vcf-upgrade/map.jinja
vcf-upgrade/defaults.yaml        # keep domain_id: null
```

The SLS imports `vcf-upgrade/map.jinja`, so the folder name must match. Update these files in RaaS
whenever the formula changes in the repo. The minion's log shows what it fetched, for example
`Fetching file from saltenv 'base', ** done ** 'vcf-upgrade/vcf-upgrade.sls'`.

### 10.2 Create the job

Create a job (or an ad-hoc run) with these fields:

| Field | Value |
|---|---|
| Name | for example `sddc-upgrade-dryrun` |
| Targets | the Ops minion, by minion ID (a compound target such as `L@<minion-id>` works too) |
| Function | `state.apply` |
| Arguments (`mods`) | `vcf-upgrade.vcf-upgrade` |
| Keyword argument `test` | `True` for a dry run. Leave it out, or `False`, for a real run. |
| Keyword argument `pillar` | the JSON below |
| Environment | `base` |

`pillar` is one JSON object, the same one used with `salt-call` in section 7:

```json
{"saltext.vcf": {
  "sddc_manager": {"host": "<sddc-manager>", "username": "administrator@vsphere.local", "password": "<pw>", "verify_ssl": false},
  "vcf_upgrade": {
    "depot": {"type": "online"},
    "component_type": "SDDC_MANAGER",
    "target_build": "<release>-<build>",
    "checkpoint_file": "/var/cache/salt/minion/vcf-upgrade-sddc.json"
  }
}}
```

For several components, replace `component_type` and `target_build` with the `sequence` list from section 7.
It still runs in the fixed order SDDC_MANAGER, NSX_T_MANAGER, VCENTER, HOST.

Equivalent command from a master shell:

```bash
salt '<minion-id>' state.apply vcf-upgrade.vcf-upgrade test=True pillar='{...same JSON...}'
```

### 10.3 Run it safely

- **Save the dry run and the real run as two separate jobs.** The saved job record decides whether `test` is
  set. A real job with no `test` argument starts an actual upgrade, which is easy to miss when reusing a job.
- Run the dry run first, read its result, then run the real job. Take the snapshot or backup first.
- Check the job's arguments in the run history before you start it: `test` must be present for a dry run.

### 10.4 Watch the result and stop a job

- Open the job's run in the RaaS activity page. A long upgrade shows as running until the state ends,
  and the result appears then. The minion log (`/var/log/salt/minion` or the appliance's own log) shows
  progress lines such as `VCF upgrade <id> status=INPROGRESS`.
- From the master: `salt-run jobs.active`, and `salt-run jobs.lookup_jid <jid>` for a finished job.
- Stopping the Salt job does not stop the upgrade on SDDC Manager. It only ends the polling:
  ```bash
  salt '<minion-id>' saltutil.kill_job <jid>
  ```
  The checkpoint keeps the remote upgrade ID, so a later run with the same inputs resumes it.
- While SDDC Manager restarts during its own upgrade, the log shows `VCF status temporarily unreachable`.
  That is expected. It keeps polling up to the upgrade timeout (section 9).

### 10.5 Credentials

The job record stores the `pillar` argument and the run output, so the SDDC Manager password is visible
to anyone who can read the job. Prefer RaaS pillar for the `saltext.vcf:sddc_manager` block, targeted at
the Ops minion, and pass only `vcf_upgrade` (depot, targets, sequence) in the job.

## 11. Troubleshooting

| Message | Likely cause | Fix |
|---|---|---|
| `No matching sls found` | folder missing, wrong name, or not readable by the `salt` user | section 4 |
| `ID ... multiple state declarations of the same type` | an old formula with several steps under one ID | use the current formula |
| `...GET /v1/domains/domain-c9 failed: HTTP 404` | `domain_id` set to a vCenter cluster ID | `domain_id: null` |
| `No bundle found for <component>` | build not in the catalog, or wrong release-build pair | section 6, sync the depot |
| `Checkpoint belongs to a different endpoint, target or sequence` | old checkpoint for another request | section 8 |
| `submission outcome is unknown` | create call may have reached SDDC Manager | section 8 |
| `HTTP 400 (UPGRADE_SPEC_BUNDLE_NOT_AVAILABLE ...)` | bundle not applicable to the domain, for example NSX 9.2.0 while SDDC Manager is on 9.1.x | choose a bundle that matches the current release, or upgrade SDDC Manager first |
| `HOST requires cluster_id and personality_id` | incomplete HOST entry | section 7 |
| `depot sync failed: ... compatibility data download failed` | SDDC Manager cannot reach the depot | fix its outbound access, or `depot.sync: false` |
| `Unsupported options: ['__pub_fun', ...]` | an old wheel | install the current wheel |
| `salt-pip` errors about `timelib` or `failed to map segment` | missing `--no-deps`, or `/tmp` is noexec | section 3 |
| `salt-pip` output ends with `Uninstalling attrs-26.1.0` and a dependency conflict with `config-modules-vmware-vcf-ops` | the install ran without `--no-deps`, so pip replaced `attrs` | restore it: `salt-pip install --no-deps attrs==26.1.0` (use the appliance's index with `--index-url` if needed), then reinstall the wheel with `--no-deps --force-reinstall` |
| connection or 5xx errors during an SDDC Manager upgrade | the appliance is restarting | wait; check the SDDC Manager LCM logs if it never returns |
