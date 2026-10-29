# fleet-edge-demo

Demo Fleet bundles for edge K3s clusters managed from the Rancher/Fleet hub.
No secrets live here. Hub-side GitRepo + registration live in the private `rancher-fleet` repo.

## Bundles
| path | what | target |
|---|---|---|
| `nginx-nodeport/` | nginx 1.27, 1 replica, Service NodePort **30080** | clusters labelled `env=edge` |

After the bundle is Ready: `curl http://<edge-node-ip>:30080/` → the hostname page.
