## Hubble

Hubble is the observability layer of Cilium and can be used to obtain cluster-wide visibility into the network and security layer of your Kubernetes cluster.

```bash
cilium upgrade --set hubble.relay.enabled=true  --reuse-values
```

```bash
curl -s https://raw.githubusercontent.com/cilium/hubble/main/stable.txt
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/v1.18.6/hubble-darwin-arm64.tar.gz
sudo tar xzvfC hubble-darwin-arm64.tar.gz /usr/local/bin
rm -f hubble-darwin-arm64.tar.gz
```


```bash
hubble status -P
Current/Max Flows: 7,097/12,285 (57.77%)
Flows/s: 12.68
Connected Nodes: 3/3
```

```bash

cilium hubble port-forward


hubble list nodes
time=2026-03-04T10:01:14.731+01:00 level=WARN msg="Hubble CLI version is lower than Hubble Relay, API compatibility is not guaranteed, updating to a matching or higher version is recommended" hubble-cli-version=1.18.6 hubble-relay-version=1.19.0+g7c6667e1
NAME                      STATUS      AGE      FLOWS/S   CURRENT/MAX-FLOWS
kubernetes/kube-master1   Connected   13m16s   2.77      2253/4095 ( 55.02%)
kubernetes/kube-worker1   Connected   13m16s   5.10      4095/4095 (100.00%)
kubernetes/kube-worker2   Connected   13m16s   4.64      3793/4095 ( 92.63%)
```

```bash
hubble observe --to-ip 138.199.218.25
Mar  4 09:07:02.976: default/mediabot:57992 (ID:43545) -> 138.199.218.25:8081 (world) to-network FORWARDED (TCP Flags: SYN)
Mar  4 09:07:02.976: default/mediabot:57992 (ID:43545) -> 138.199.218.25:8081 (world) to-network FORWARDED (TCP Flags: SYN)
```

```bash
➜  ~ hubble observe --from-pod default/mediabot
Mar  4 09:07:02.976: default/mediabot:57992 (ID:43545) -> 138.199.218.25:8081 (world) to-network FORWARDED (TCP Flags: SYN)
Mar  4 09:07:02.976: default/mediabot:57992 (ID:43545) -> 138.199.218.25:8081 (world) to-network FORWARDED (TCP Flags: SYN)
```

## Hubble UI 

```bash
cilium hubble enable --ui
```