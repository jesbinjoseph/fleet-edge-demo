# Ansible onboarding (push mode, SSH into each node)

Turns a plain Ubuntu/Debian machine into a single-node K3s cluster registered with the
Fleet hub. Everything inside the cluster is then delivered by Fleet from this repo.

```
ansible/
  inventory/edge.yml      hosts + fleet_labels  (the only file you edit per fleet)
  group_vars/all.yml      pinned versions, hub context
  site.yml                hub-token -> prereqs -> k3s -> fleet-agent -> verify
  roles/
    hub-token             reads the ClusterRegistrationToken secret from the hub (control node)
    edge-prereqs          OS checks, chrony, no sleep/auto-reboot, swap off, sysctl
    edge-k3s              pinned K3s server, traefik/servicelb off, helm
    edge-fleet-agent      fleet-agent chart with labels from inventory
    hub-verify            waits until the hub sees each cluster
```

## Requirements (control node)
- `kubectl` with access to the hub context in `group_vars/all.yml`
- Ansible + `ansible.posix` collection. On NixOS/nix-darwin:
  `nix-shell -p ansible kubectl --run 'ansible-galaxy collection install ansible.posix && ansible-playbook site.yml'`
- Hub already has a `ClusterRegistrationToken` named `edge-token` in `fleet-default`
  (see the hub repo, `edge/hub/`).

## Run
```bash
cd ansible
ansible-playbook site.yml --check -l edge-01   # dry run one host
ansible-playbook site.yml                      # all hosts in [edge]
```
The hub play runs once and passes the token in memory; nodes get it as
`/etc/rancher/fleet/agent-values.yaml` (0600, root). Nothing sensitive is written in this repo.

## Multi-NIC hosts
K3s advertises the IP of the default-route interface unless told otherwise. On hosts with a
NAT/management NIC plus a reachable NIC, `edge-k3s` resolves the interface that owns
`ansible_host` and passes `--node-ip/--node-external-ip/--flannel-iface` for it. Override with
`k3s_node_ip` / `k3s_iface` host vars when `ansible_host` is not the address you want in-cluster.

## Day 2
| task | how |
|---|---|
| add a machine | add it to `inventory/edge.yml`, `ansible-playbook site.yml -l <name>` |
| rotate the registration token | recreate the CRT on the hub, `ansible-playbook site.yml --tags fleet-agent` |
| bump K3s | change `k3s_version`, re-run; or hand it to `system-upgrade-controller` via Fleet |
| remove a machine | on node `/usr/local/bin/k3s-uninstall.sh`; on hub `kubectl delete clusters.fleet.cattle.io <name> -n fleet-default` |

Workloads never go through Ansible. Push to `main`, Fleet polls (60s) and reconciles
every cluster whose labels match the bundle's `targetCustomizations`.
