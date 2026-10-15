# ansible-role-elastic-fleet

An Ansible role that installs Elastic Agent 9.x from the RPM package and enrolls it as a self-managed **Fleet Server**, on RHEL, AlmaLinux and Rocky Linux 9 and 10.

What it does:

- **Installs Elastic Agent** from Elastic's repository, a Red Hat Satellite / Katello repository, or whatever repositories the host already has.
- **Sets up Fleet in Kibana** through the Fleet API: runs Fleet setup, creates the Fleet Server policy when it is missing, registers the URL agents use as the Fleet Server host, and optionally points the default Elasticsearch output at your cluster with its CA.
- **Enrolls each host as a Fleet Server** with `elastic-agent enroll`, using a service token that never appears on a command line.
- **TLS from `ssl_certificates` or [ansible-role-elastic-tls](https://github.com/jmaughmer/ansible-role-elastic-tls)**, with checks that the key matches the certificate, the certificate chains to the CA, and the names agents connect to are in it.
- **Checks before it changes anything:** the service token is tried against every Elasticsearch node through the configured CA, and Elasticsearch and Kibana must not be older than Fleet Server.
- **Rolling changes.** Running Fleet Servers are upgraded, re-enrolled or restarted a batch at a time, and each batch has to report `HEALTHY` before the next one starts.

## Requirements

- **Control node:** ansible-core 2.15 or later, plus the `ansible.posix` collection:

  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

- **Target hosts:** RHEL, AlmaLinux or Rocky Linux 9 or 10, with `openssl` (installed by default).
- **Elasticsearch and Kibana 9.x**, at least the version of Fleet Server, with security enabled and Kibana's `xpack.encryptedSavedObjects.encryptionKey` set (Fleet will not start without it).
- **A service token** for the `elastic/fleet-server` service account, and Kibana credentials allowed to manage Fleet — see [Credentials](#credentials).

## Supported platforms

| Distribution | `ansible_facts['distribution']` | Major versions |
|---|---|---|
| Red Hat Enterprise Linux | `RedHat` | 9, 10 |
| AlmaLinux | `AlmaLinux` | 9, 10 |
| Rocky Linux | `Rocky` | 9, 10 |

Preflight stops the run on any other distribution or major version; add others to `elastic_fleet_supported_distributions` at your own risk.

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

When the settings change (another URL, policy or token) a running Fleet Server is re-enrolled with `--force` in its batch. Re-enrolling gives the agent a **new id in Fleet**; the old one shows as offline until you remove it (or Fleet's inactivity timeout does). Set `elastic_fleet_reenroll_on_change: false` to be told instead.

An agent that was enrolled by hand, or as an ordinary agent, has no record, so the role enrolls it as a Fleet Server on the first run.

## Upgrades

Change `elastic_fleet_version`. Fleet cannot upgrade an agent installed from an RPM, so the role upgrades the package with dnf, a batch at a time, and waits for each batch to report healthy.

- **Order:** Elasticsearch, then Kibana, then Fleet Server, then the agents. Preflight refuses a Fleet Server newer than Elasticsearch or Kibana.
- **Downgrades and major upgrades** are refused.
- `dnf update` never upgrades Elastic Agent behind the role's back: the repository file is written disabled and enabled only for the role's own tasks (or set `elastic_fleet_satellite_repo_state: disabled` with Satellite).

## Package source

| `elastic_fleet_repo_source` | Packages come from |
|---|---|
| `elastic` (default) | `elastic_fleet_repo_baseurl`, Elastic's public repository or a mirror of it |
| `satellite` | `elastic_fleet_satellite_repo_id`, a repository delivered by Red Hat Satellite or Katello. The role checks it is in the host's content view, can set its enabled state and content overrides, and checks that the requested version has been promoted. |
| `none` | Whatever the host has enabled |

These work exactly as in [ansible-role-kibana](https://github.com/jmaughmer/ansible-role-kibana); the variables have the `elastic_fleet_` prefix.

## Firewall

Port `elastic_fleet_port` (`8220/tcp`) is opened in `elastic_fleet_firewall_zone` when firewalld is running. Set `elastic_fleet_manage_firewall: false` to leave the firewall alone.

Fleet Server also needs to reach Elasticsearch (9200) and agents need to reach Fleet Server and Elasticsearch; Kibana is only reached from the first Fleet Server host, during the run.

## Slow and unreliable networks

Downloads run as async jobs that survive a dropped SSH session, and every task that talks to a repository, Elasticsearch or Kibana is retried:

| Variable | Default | |
|---|---|---|
| `elastic_fleet_task_retries`, `elastic_fleet_task_delay` | `3`, `15` | Retries for downloads and API calls |
| `elastic_fleet_download_async_timeout`, `_poll` | `3600`, `15` | Package download time limit and poll interval |
| `elastic_fleet_repo_timeout`, `elastic_fleet_repo_retries` | `120`, `10` | dnf per-request timeout and retries |
| `elastic_fleet_api_timeout` | `30` | Per-request HTTP timeout |
| `elastic_fleet_enroll_timeout` | `600` | How long enroll waits for Fleet Server to start |
| `elastic_fleet_ready_retries`, `elastic_fleet_ready_delay` | `60`, `5` | Waiting for `HEALTHY` |

## Hardened hosts

- **FIPS.** Encrypted keys in the traditional PEM format (`Proc-Type: 4,ENCRYPTED`) use an MD5 key derivation and cannot be read in FIPS mode; preflight refuses them. ansible-role-elastic-tls writes PKCS#8, which is fine. For FIPS-validated cryptography in Fleet Server itself, Elastic ships separate FIPS builds of Elastic Agent; install that package through `elastic_fleet_repo_source: none` or a Satellite repository.
- **fapolicyd.** Elastic Agent runs binaries from `/var/lib/elastic-agent/data/`, which a default fapolicyd policy blocks. Trust that directory (or the package's files) before enrolling.
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

[`molecule/default/`](molecule/default/) runs the role against three systemd containers on one
container network: `kes-1`, a single Elasticsearch node and Kibana at the role's default
`elastic_fleet_version`, and two Fleet Servers, `fs-1` and `fs-2`. [`prepare.yml`](molecule/default/prepare.yml)
makes a test CA and a certificate for every host, starts Elasticsearch with TLS on HTTP and Kibana
with TLS, and creates service account tokens for `elastic/kibana` and `elastic/fleet-server`. `fs-1`
takes its certificates as files on the host, as ansible-role-elastic-tls leaves them; `fs-2` takes
them inline with an encrypted key and its passphrase. Fleet is managed with the default output and
two agent policies, one with every per-policy setting changed. The sequence is:

1. **converge**, then **idempotence**: a second converge must change nothing;
2. **side effect** ([`side_effect.yml`](molecule/default/side_effect.yml)):
   - an enrollment setting changes with `elastic_fleet_reenroll_on_change: false`: nothing is
     re-enrolled or restarted;
   - the same change with re-enrollment: both Fleet Servers get a new agent id, one per batch,
     and the new settings are recorded;
   - the converge settings again: both are re-enrolled back;
   - `elastic_fleet_force_restart` restarts both, one per batch, keeping their agent ids;
3. **verify** ([`verify.yml`](molecule/default/verify.yml)):
   - Elastic Agent is running and enabled at the default version, and its repository is disabled;
   - the TLS material, service token and enrollment record have their owners and modes, the key is
     encrypted and has a passphrase file exactly when it has a passphrase, and no secret is in the
     enrollment record or on a command line;
   - every Fleet Server reports `HEALTHY` through its own URL and every URL agents are given,
     checked against the test CA;
   - each host's agent is online in Fleet;
   - the Fleet Server policy, the agent policies, the Fleet Server host and the output are what
     the inventory asks for;
   - every host publishes the enrollment URLs and one active token per agent policy, with the
     token's name.

There is no check-mode step: the role's Fleet API calls are skipped in check mode.

Other scenarios:

| Scenario | What it covers |
|---|---|
| `upgrade` | Two Fleet Servers at an older 9.x release are upgraded to the default version one at a time: the second stops only once the first is back. Going back to the older release is refused. Fleet is managed with an API key instead of a password. |
| `preflight` | Settings preflight must refuse, each in its own check, without printing a secret. These include an unsupported distribution, a host outside `elastic_fleet_group`, an inexact or 8.x version, a bad package source or Satellite override, http or missing URLs, an unknown client authentication, a bad port or batch size, missing Elasticsearch hosts, service token or Kibana credentials, malformed agent policies, Fleet settings that differ between hosts, and missing TLS material. Nothing is installed. |
| `docker` | The default scenario on Docker instead of Podman. |

```bash
pip install -r molecule/requirements.txt
```

```bash
for s in default upgrade preflight; do molecule test -s "$s" || break; done
```

The scenarios use Podman (rootless works), except `docker`, which needs Docker. They download
Elasticsearch, Kibana and Elastic Agent from Elastic's repository, and Kibana downloads the Fleet
Server and System integrations from the Elastic Package Registry. Elasticsearch, Kibana and two
Fleet Servers need about 5 GB of memory. `MOLECULE_DISTRO` picks the image (default
`rockylinux9`; also `rockylinux10`):

```bash
MOLECULE_DISTRO=rockylinux10 molecule test -s preflight
```

Lint with `yamllint .` and `ansible-lint` (configured by [`.yamllint`](.yamllint) and
[`.ansible-lint`](.ansible-lint)). The tests run locally; there is no CI workflow.

Not covered by containers: firewalld rules, FIPS mode, fapolicyd and Satellite repositories.
These need virtual machines with firewalld running, FIPS or fapolicyd enabled, or a Satellite
registration.

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
└── molecule/                   # default, upgrade, preflight (Podman) and docker scenarios
```

## Related roles

- [ansible-role-elastic-tls](https://github.com/jmaughmer/ansible-role-elastic-tls): certificates from a private CA or Let's Encrypt, published for this role.
- [ansible-role-elasticsearch](https://github.com/jmaughmer/ansible-role-elasticsearch): the cluster Fleet Server writes to.
- [ansible-role-kibana](https://github.com/jmaughmer/ansible-role-kibana): Kibana, where Fleet lives.
- [ansible-role-logstash](https://github.com/jmaughmer/ansible-role-logstash): Logstash, which Fleet can use as an output.
- [ansible-role-keepalived](https://github.com/jmaughmer/ansible-role-keepalived): a virtual address for `elastic_fleet_public_urls`.

## License

MIT. See [LICENSE](LICENSE).
