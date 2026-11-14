# ansible-role-elastic-fleet

An Ansible role that installs Elastic Agent 9.x from Elastic's RPM or Debian package and enrolls it as a self-managed **Fleet Server**, on RHEL, AlmaLinux and Rocky Linux 9 and 10, and on Ubuntu 24.04 and 26.04.

What it does:

- **Installs Elastic Agent** from Elastic's repository, a Red Hat Satellite / Katello repository, or whatever repositories the host already has.
- **Sets up Fleet in Kibana** through the Fleet API: runs Fleet setup, creates the Fleet Server policy when it is missing, registers the URL agents use as the Fleet Server host, and optionally points the default Elasticsearch output at your cluster with its CA.
- **Enrolls each host as a Fleet Server** with `elastic-agent enroll`, using a service token that never appears on a command line.
- **TLS from `ssl_certificates` or [ansible-role-elastic-tls](https://github.com/jmaughmer/ansible-role-elastic-tls)**, with checks that the key matches the certificate, the certificate chains to the CA, and the names agents connect to are in it.
- **Checks before it changes anything:** the service token is tried against every Elasticsearch node through the configured CA, and Elasticsearch and Kibana must not be older than Fleet Server.
- **Rolling changes.** Running Fleet Servers are upgraded, re-enrolled or restarted a batch at a time, and each batch has to report `HEALTHY` before the next one starts.

## Requirements

- **Control node:** ansible-core 2.15 or later, plus the `ansible.posix` and `community.general` collections:

  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

- **Target hosts:** RHEL, AlmaLinux or Rocky Linux 9 or 10, or Ubuntu 24.04 or 26.04, with `openssl` (installed by default).
- **Elasticsearch and Kibana 9.x**, at least the version of Fleet Server, with security enabled and Kibana's `xpack.encryptedSavedObjects.encryptionKey` set (Fleet will not start without it).
- **A service token** for the `elastic/fleet-server` service account, and Kibana credentials allowed to manage Fleet — see [Credentials](#credentials).

## Supported platforms

| Distribution | `ansible_facts['distribution']` | Versions |
|---|---|---|
| Red Hat Enterprise Linux | `RedHat` | 9, 10 |
| AlmaLinux | `AlmaLinux` | 9, 10 |
| Rocky Linux | `Rocky` | 9, 10 |
| Ubuntu | `Ubuntu` | 24.04, 26.04 |

Elastic Agent comes from Elastic's RPM packages through dnf on Enterprise Linux, and from its Debian packages through apt on Ubuntu; everything else runs the same tasks on both. On Ubuntu the firewall is ufw, and Satellite is not available.

Preflight stops the run on any other distribution or version; add others to `elastic_fleet_supported_distributions` at your own risk.

## Installation

```yaml
# requirements.yml
roles:
  - name: elastic_fleet
    src: https://github.com/jmaughmer/ansible-role-elastic-fleet.git
    scm: git
    version: main
  - name: elastic_tls            # optional, to issue the certificates
    src: https://github.com/jmaughmer/ansible-role-elastic-tls.git
    scm: git
    version: main

collections:
  - name: ansible.posix
    version: ">=1.5.0"
  - name: community.general
    version: ">=8.0.0"
  - name: community.crypto       # for elastic_tls
    version: ">=2.10.0"
```

```bash
ansible-galaxy install -r requirements.yml
```

## Quick start

```yaml
- name: Issue certificates and deploy Fleet Server
  hosts: fleet_server
  become: true
  any_errors_fatal: true
  # Do NOT set `serial`: restarts and upgrades are batched by the role.
  roles:
    - role: elastic_tls
    - role: elastic_fleet
```

```yaml
# group_vars/fleet_server/main.yml
elastic_fleet_version: "9.5.4"
elastic_fleet_public_urls: ["https://fleet.example.com:8220"]
elastic_fleet_service_token: "{{ vault_elastic_fleet_service_token }}"
elastic_fleet_kibana_url: "https://kb-1.example.com:5601"
elastic_fleet_kibana_api_key: "{{ vault_elastic_fleet_kibana_api_key }}"

elastic_tls_certificates:
  - name: "{{ inventory_hostname }}"
    dns: ["{{ inventory_hostname }}", fleet.example.com]
    usage: [server, client]
```

The inventory needs a `fleet_server` group, plus `elasticsearch` and `kibana` groups (or set `elastic_fleet_elasticsearch_hosts` and `elastic_fleet_kibana_url` directly). [`examples/`](examples/) has a complete project.

## How a run works

1. **Every host in parallel:** gather state, preflight, configure the repository, install the package on new hosts, open the firewall, install certificates and the service token, and probe Elasticsearch with the token.
2. **Once, from the first host:** Fleet settings in Kibana (setup, policy, Fleet Server host, output, agent policies and their enrollment tokens).
3. **Hosts not yet enrolled** are enrolled and started in parallel, then waited on until `/api/status` reports `HEALTHY`.
4. **Hosts already serving** that need something — a new version, new enrollment settings, or new certificates — are handled in batches of `elastic_fleet_restart_batch_size`.

A run with nothing to change changes nothing.

## Credentials

**Service token** (`elastic_fleet_service_token`). Fleet Server uses it to talk to Elasticsearch, permanently, so keep it valid. Create one per deployment with an admin user and vault the value:

```bash
curl -u elastic --cacert ca.crt -X POST \
  "https://es-data-1.example.com:9200/_security/service/elastic/fleet-server/credential/token/fleet-prod"
```

The token is written to `/etc/elastic-agent/fleet-server/service-token` (root, `0600`) and passed to enroll as a file. Changing it re-enrolls every Fleet Server.

**Kibana credentials**, used only to manage Fleet settings: either an API key (`elastic_fleet_kibana_api_key`, the `encoded` value) or `elastic_fleet_kibana_username` / `elastic_fleet_kibana_password`. The user needs the **Fleet** and **Integrations** `All` privileges in Kibana. With `elastic_fleet_manage_fleet_settings: false` no Kibana credentials are needed, but the policy named by `elastic_fleet_policy_id` must already exist and contain the Fleet Server integration.

## TLS

The role looks for the entry named `elastic_fleet_ssl_certificate_name` (by default the inventory hostname) in `ssl_certificates` first and then in `elastic_tls_ssl_certificates`, so a manual entry always wins. Each of `certificate`, `key` and `ca` can be inline `content` (vault the key) or the `path` of a file already on the host.

```yaml
# host_vars/fleet-1.example.com.yml
ssl_certificates:
  - name: fleet-1.example.com
    certificate: {content: "{{ vault_fleet1_cert }}"}
    key: {content: "{{ vault_fleet1_key }}", passphrase: "{{ vault_fleet1_key_passphrase }}"}
    ca: {content: "{{ vault_internal_ca }}"}
```

The material is copied to `elastic_fleet_ssl_dir` (`/etc/elastic-agent/fleet-server`, root only) and Fleet Server is pointed at those copies.

| CA | Used for | Comes from |
|---|---|---|
| Fleet Server CA | Agents (and the readiness probe) verifying Fleet Server | the entry's `ca`, or `elastic_fleet_ssl_ca_certificate[_path]` |
| Elasticsearch CA | Fleet Server verifying Elasticsearch, and the output when managed | `elastic_fleet_elasticsearch_ca_certificate[_path]`, else the Fleet Server CA |
| Kibana CA | The Fleet API calls | `elastic_fleet_kibana_ca_certificate[_path]`, else the Fleet Server CA |

With ansible-role-elastic-tls's private CA all three are the same file, so nothing needs setting.

**Names in the certificate.** Agents connect to the URLs in `elastic_fleet_public_urls` and verify the name, so every host name there — typically a load balancer or virtual address — has to be a SAN in every Fleet Server's certificate, as well as the host's own name used in `elastic_fleet_url`. The role warns when one is missing.

**Renewal.** A renewed certificate at the same path only needs a restart, which the role does in batches. Elastic Agent does not reload certificates on its own.

## Fleet settings

With `elastic_fleet_manage_fleet_settings: true` (the default), the first host of the play calls the Kibana Fleet API:

| Object | Id variable | What the role does |
|---|---|---|
| Fleet setup | — | `POST /api/fleet/setup` (idempotent) |
| Fleet Server policy | `elastic_fleet_policy_id` | Created with the Fleet Server integration when missing (plus the System integration with `elastic_fleet_policy_system_monitoring`). An existing policy is not modified; edit it in Fleet. |
| Fleet Server host | `elastic_fleet_server_host_id` | Created or updated so its URLs are `elastic_fleet_public_urls`. |
| Elasticsearch output | `elastic_fleet_output_id` | Only with `elastic_fleet_manage_output: true`: hosts set to `elastic_fleet_output_hosts`, CA set to the Elasticsearch CA. |
| Agent policies | `elastic_fleet_agent_policies[].id` | Created when missing. An existing policy is not modified. |
| Enrollment tokens | `token_name` / `elastic_fleet_enrollment_token_name` | One active token per agent policy, reused by name or created, and published for the elastic-agent role. |

Objects preconfigured in `kibana.yml` (`xpack.fleet.fleetServerHosts`, `xpack.fleet.outputs`) cannot be changed through the API; the role stops with an explanation when they differ from what you asked for.

## Agent policies and enrollment tokens

Ordinary Elastic Agents enroll into the policies in `elastic_fleet_agent_policies`, which the role creates alongside the Fleet Server policy. By default that is one policy, `agent-policy`:

```yaml
elastic_fleet_agent_policies:
  - id: agent-policy
    name: Agent Policy
  - id: linux-servers
    name: Linux servers
    namespace: prod
    monitoring_enabled: [logs]   # default [logs, metrics]
    system_monitoring: true      # add the System integration on creation
    token_name: ansible          # default elastic_fleet_enrollment_token_name
```

For each policy the role looks for an active enrollment token named `token_name` (Fleet stores it as `<name> (<uuid>)`) and creates one when there is none. The default name, `Default`, is the token Fleet creates with every new policy, so nothing extra is created. Revoke a token in Fleet and the next run creates a new one.

The role then publishes two facts on every host of the play, for [ansible-role-elastic-agent](https://github.com/jmaughmer/ansible-role-elastic-agent) to read in a later play:

| Fact | Value |
|---|---|
| `elastic_fleet_enrollment_urls` | `elastic_fleet_public_urls` |
| `elastic_fleet_enrollment_tokens` | `{policy id: enrollment token}` for every agent policy |

```yaml
# For example, in the elastic-agent play
elastic_agent_fleet_url: "{{ hostvars[groups['fleet_server'] | first].elastic_fleet_enrollment_urls | first }}"
elastic_agent_enrollment_token: "{{ hostvars[groups['fleet_server'] | first].elastic_fleet_enrollment_tokens['agent-policy'] }}"
```

The facts only exist when the Fleet settings tasks ran in the same `ansible-playbook` run, so keep the Fleet Server play in the playbook (and the `elastic_fleet_settings` tag selected) when deploying agents. An enrollment token only lets an agent join its policy, but it is still a secret: with fact caching enabled it is written to the cache.

## Enrollment and re-enrollment

The role records a salted SHA-256 of the enrollment settings — the `enroll` arguments, the service token and the key passphrase — in `/etc/elastic-agent/.ansible-fleet-server-state.json`. A host is enrolled when that record and Elastic Agent's own `/etc/elastic-agent/fleet.enc` both exist.

When the settings change (another URL, policy or token) a running Fleet Server is stopped, re-enrolled with `--force` and started again, in its batch. Re-enrolling gives the agent a **new id in Fleet**; the old one shows as offline until you remove it (or Fleet's inactivity timeout does). Set `elastic_fleet_reenroll_on_change: false` to be told instead.

An agent that was enrolled by hand, or as an ordinary agent, has no record, so the role enrolls it as a Fleet Server on the first run.

## Upgrades

Change `elastic_fleet_version`. Fleet cannot upgrade an agent installed from a package, so the role upgrades the package with dnf or apt, a batch at a time, and waits for each batch to report healthy.

- **Order:** Elasticsearch, then Kibana, then Fleet Server, then the agents. Preflight refuses a Fleet Server newer than Elasticsearch or Kibana.
- **Downgrades and major upgrades** are refused.
- `dnf update` never upgrades Elastic Agent behind the role's back: the repository file is written disabled and enabled only for the role's own tasks (or set `elastic_fleet_satellite_repo_state: disabled` with Satellite). On Ubuntu the package is held instead, so `apt upgrade` skips it; `apt-mark unhold elastic-agent` to upgrade it by hand.

## Package source

| `elastic_fleet_repo_source` | Packages come from |
|---|---|
| `elastic` (default) | `elastic_fleet_repo_baseurl`, Elastic's public repository or a mirror of it: the yum repository on Enterprise Linux, the apt one on Ubuntu (key in `/etc/apt/keyrings/elastic.asc`, source in `/etc/apt/sources.list.d/elastic-9.x.sources`, shared with the other Elastic roles) |
| `satellite` | Enterprise Linux only. `elastic_fleet_satellite_repo_id`, a repository delivered by Red Hat Satellite or Katello. The role checks it is in the host's content view, can set its enabled state and content overrides, and checks that the requested version has been promoted. |
| `none` | Whatever the host has enabled |

These work exactly as in [ansible-role-kibana](https://github.com/jmaughmer/ansible-role-kibana); the variables have the `elastic_fleet_` prefix.

## Firewall

Port `elastic_fleet_port` (`8220/tcp`) is opened in `elastic_fleet_firewall_zone` when firewalld is running, or allowed in ufw when ufw is active (Ubuntu). The role never starts or enables a firewall. Set `elastic_fleet_manage_firewall: false` to leave the firewall alone.

Fleet Server also needs to reach Elasticsearch (9200) and agents need to reach Fleet Server and Elasticsearch; Kibana is only reached from the first Fleet Server host, during the run.

## Slow and unreliable networks

Downloads run as async jobs that survive a dropped SSH session, and every task that talks to a repository, Elasticsearch or Kibana is retried:

| Variable | Default | |
|---|---|---|
| `elastic_fleet_task_retries`, `elastic_fleet_task_delay` | `3`, `15` | Retries for downloads and API calls |
| `elastic_fleet_download_async_timeout`, `_poll` | `3600`, `15` | Package download time limit and poll interval |
| `elastic_fleet_repo_timeout`, `elastic_fleet_repo_retries` | `120`, `10` | dnf per-request timeout and retries (Enterprise Linux) |
| `elastic_fleet_api_timeout` | `30` | Per-request HTTP timeout |
| `elastic_fleet_enroll_timeout` | `600` | How long enroll waits for Fleet Server to start |
| `elastic_fleet_ready_retries`, `elastic_fleet_ready_delay` | `60`, `5` | Waiting for `HEALTHY` |

## Hardened hosts

- **FIPS.** Encrypted keys in the traditional PEM format (`Proc-Type: 4,ENCRYPTED`) use an MD5 key derivation and cannot be read in FIPS mode; preflight refuses them. ansible-role-elastic-tls writes PKCS#8, which is fine. For FIPS-validated cryptography in Fleet Server itself, Elastic ships separate FIPS builds of Elastic Agent; install that package through `elastic_fleet_repo_source: none` or a Satellite repository.
- **fapolicyd.** Nothing to do with the default rules. The package installs Elastic Agent and Fleet Server under `/var/lib/elastic-agent/data/`, where the rpm trust covers them, and the agent runs as root; tested with 9.5.4 and fapolicyd enforcing on AlmaLinux 10.
- **SELinux.** Elastic Agent runs unconfined from its packaged unit; nothing needs labelling.

## Notes and caveats

- **Check mode** is not supported for first installs: the certificate checks read files that do not exist yet.
- **One policy per play.** Fleet settings are managed once per run, so every host in the play must use the same `elastic_fleet_policy_id`, `elastic_fleet_public_urls` and `elastic_fleet_agent_policies`.
- **Removing a host** from the group leaves its agent enrolled and running. Unenroll it in Fleet and remove the package.
- **The service token stays on the host** in a root-only file; Elastic Agent also keeps it, encrypted, in its own state.

## Variables

[`defaults/main.yml`](defaults/main.yml) documents every variable. The main ones:

| Variable | Default | Purpose |
|---|---|---|
| `elastic_fleet_version` | `9.5.4` | Exact Elastic Agent release |
| `elastic_fleet_group` | `fleet_server` | Inventory group of Fleet Servers |
| `elastic_fleet_port` | `8220` | Port Fleet Server listens on |
| `elastic_fleet_url` | `https://<inventory_hostname>:8220` | URL this host's agent enrolls through |
| `elastic_fleet_public_urls` | every host in the group on 8220 | URLs agents are given |
| `elastic_fleet_policy_id` | `fleet-server-policy` | Policy the Fleet Servers enroll into |
| `elastic_fleet_service_token` | `""` | `elastic/fleet-server` service token (required, vault) |
| `elastic_fleet_elasticsearch_hosts` | the `elasticsearch` group on 9200 | Elasticsearch nodes to check and use for the output |
| `elastic_fleet_elasticsearch_url` | first of the above | URL Fleet Server bootstraps against |
| `elastic_fleet_ssl_certificate_name` | `{{ inventory_hostname }}` | Entry in `ssl_certificates` / `elastic_tls_ssl_certificates` |
| `elastic_fleet_manage_fleet_settings` | `true` | Manage Fleet settings through Kibana |
| `elastic_fleet_kibana_url` | first `kibana` host on 5601 | Kibana for the Fleet API |
| `elastic_fleet_kibana_api_key` | `""` | Kibana credentials (or username and password) |
| `elastic_fleet_manage_output` | `false` | Also manage the default Elasticsearch output |
| `elastic_fleet_agent_policies` | one, `agent-policy` | Agent policies to create, each with a published enrollment token |
| `elastic_fleet_enrollment_token_name` | `Default` | Enrollment token reused or created in each agent policy |
| `elastic_fleet_restart_batch_size` | `1` | Fleet Servers changed at a time |
| `elastic_fleet_reenroll_on_change` | `true` | Re-enroll when enrollment settings change |

Useful tags: `elastic_fleet_install`, `elastic_fleet_tls`, `elastic_fleet_settings`, `elastic_fleet_enroll`, `elastic_fleet_service`. Fact gathering and preflight always run.

## Testing

[`molecule/default/`](molecule/default/) runs the role against three real KVM VMs cloned from
the Proxmox test lab's fleet onto its isolated labnet network — SELinux enforcing, firewalld
running, the paths containers never could: `fleet-default-es` (a single Elasticsearch node at
the role's default `elastic_fleet_version`) and `fleet-default-kb` (Kibana) are the fixtures,
`fleet-default-fleet` is the Fleet Server the role is applied to.
[`prepare.yml`](molecule/default/prepare.yml) gets the VMs onto the labnet resolver, makes a
test CA and a certificate for every host with ansible-role-elastic-tls, starts Elasticsearch
with TLS on HTTP and Kibana with TLS, and creates service account tokens for `elastic/kibana`
and `elastic/fleet-server`. Converge is the repo's own [`examples/playbook.yml`](examples/playbook.yml):
issue certificates, then deploy Fleet Server — with the default output and the Fleet settings
from the scenario's inventory. The sequence is destroy → syntax → create → prepare → converge →
idempotence → verify → destroy, and `verify.yml`:

- Elastic Agent (running Fleet Server) is up, enabled and at the default version;
- the agent reports HEALTHY and its `fleet-server` component is healthy — Fleet Server answers
  on 8220 with the test CA;
- Kibana is available and its `fleet_server_hosts` record points at this Fleet Server's address.

There is no check-mode step: the role's Fleet API calls are skipped in check mode.

Tests run on the lab's molecule control machine: `molecule`, `molecule-proxmox` and the role's
collections are preinstalled there, as are the `elastic_tls` role checkout the fixture chain
needs; the driver's connection secrets and SSH key live only in the control machine's own
config files. Nothing lab-specific is in this repo beyond the fleet template name and the
default VMID band (9308–9310); hosts, tokens and subnets never leave the control machine. The
Test Lab project's `molecule-labnet/` directory documents the lab itself. The three VMs need
about 13 GB of memory between them. `LAB_TEMPLATE` picks the guest release (default
`template-alma9`; also `template-alma10`):

```bash
molecule test                                   # AlmaLinux 9, SELinux enforcing
LAB_TEMPLATE=template-alma10 molecule test      # the other EL release
```

The VMIDs come from `LAB_VMID_0..2` and default to the pinned 9308–9310 band, so a plain
`molecule test` means one lab run of this repo at a time. For several runs of this repo at
once, use the lab's `mlab` wrapper, which claims a free VMID band (and an isolated ephemeral
directory) per run:

```bash
/root/labnet/mlab test -s default               # parallel-safe run on the control machine
```

Lint with `yamllint .` and `ansible-lint` (configured by [`.yamllint`](.yamllint) and
[`.ansible-lint`](.ansible-lint)). There is no CI; run the lint and Molecule from the control
machine.

Not covered by the scenario: the rolling-upgrade path (upgrading two Fleet Servers one at a
time, refusing a downgrade, managing Fleet with an API key — the removed `upgrade` scenario);
settings preflight must refuse, each without printing a secret (the removed `preflight`
scenario); inline TLS material with an encrypted key and passphrase, and the re-enrollment,
force-restart and batching checks of the old `side_effect.yml`; and the Ubuntu guest paths
(the removed `kvm-ubuntu*` scenarios — Fleet Server does ship for Ubuntu, but the tests now
run EL guests only). FIPS mode, fapolicyd and Satellite repositories remain uncovered:
they need hosts with FIPS or fapolicyd enabled, or a Satellite registration.

## Role layout

```
├── defaults/main.yml           # every tunable, documented
├── vars/main.yml               # internal, lazily evaluated values
├── meta/main.yml
├── files/repo_lookup.py        # reads Satellite repository definitions
├── tasks/
│   ├── main.yml
│   ├── facts.yml               # installed version, service and enrollment state
│   ├── preflight.yml           # validation before anything is written
│   ├── repository.yml          # Elastic or Satellite repository
│   ├── package_version_check.yml
│   ├── install.yml
│   ├── firewall.yml
│   ├── certificates.yml        # TLS material, service token, checks
│   ├── elasticsearch_check.yml # token, CA and version, from every host
│   ├── fleet_settings.yml      # Fleet API: setup, policy, host, output
│   ├── agent_policies.yml      # agent policies and enrollment tokens
│   ├── enrollment.yml          # what each host needs
│   ├── service.yml             # enroll and start new Fleet Servers
│   ├── enroll_host.yml
│   ├── rolling_restart.yml     # batches of upgrades, re-enrollments, restarts
│   ├── restart_batch.yml
│   └── wait_ready.yml          # /api/status must be HEALTHY
├── examples/
└── molecule/                   # the lab default scenario
```

## Related roles

- [ansible-role-elastic-tls](https://github.com/jmaughmer/ansible-role-elastic-tls): certificates from a private CA or Let's Encrypt, published for this role.
- [ansible-role-elasticsearch](https://github.com/jmaughmer/ansible-role-elasticsearch): the cluster Fleet Server writes to.
- [ansible-role-kibana](https://github.com/jmaughmer/ansible-role-kibana): Kibana, where Fleet lives.
- [ansible-role-logstash](https://github.com/jmaughmer/ansible-role-logstash): Logstash, which Fleet can use as an output.
- [ansible-role-keepalived](https://github.com/jmaughmer/ansible-role-keepalived): a virtual address for `elastic_fleet_public_urls`.

## License

MIT. See [LICENSE](LICENSE).
