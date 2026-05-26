# Splunk Operator on K3s: GitOps Deployment with ArgoCD, Helm, and OTel Log Shipping

## The Stack

This post is one piece of a larger homelab project. The full thing is a 19-phase build documented at [HomeLab-Project](https://github.com/LetItGo90/HomeLab-Project). By the time Splunk goes in at Phase 9 the cluster already has a fair bit running.

The infrastructure layer is K3s on Ubuntu Server 22.04, Cilium as the CNI with Hubble for observability, HashiCorp Vault for secrets, cert-manager with an internal CA, and Keycloak handling SSO across the stack. Everything is deployed and managed through ArgoCD using an App of Apps pattern. Gitleaks runs on every push to catch credential leaks before they land in a public repo. Nothing gets deployed manually after the initial ArgoCD bootstrap.

On the security side, Falco is already running for runtime threat detection, Prometheus and Grafana handle metrics, and Juice Shop sits in the cluster as a live vulnerable target for testing the detection pipeline.

Splunk is the SIEM that ties the security layer together. It is where Falco alerts land, where Kubernetes audit logs go, and later where Tetragon eBPF events, MISP threat intel, and ZAP findings will all feed in. Running it on Kubernetes keeps it inside the same GitOps workflow as everything else rather than bolting a bare metal instance onto the side of the stack.

---

## Why Kubernetes Instead of Bare Metal

Everything else in this stack — Falco, Keycloak, Tetragon, cert-manager, Cilium — runs on Kubernetes. Running Splunk on bare metal would mean a second operational model, manual upgrades, and secrets management living outside the cluster. Keeping it on Kubernetes means one control plane, one GitOps workflow, and one place to look when something breaks.

The Splunk Operator handles the complexity. It manages the `SplunkStandalone` custom resource, handles rolling upgrades, and takes care of the underlying StatefulSets and PVCs.

---

## The Three ArgoCD Manifests

The whole deployment is three files committed to Git and synced by ArgoCD:

1. **splunk-operator** — the operator itself
2. **splunk-enterprise** — the actual Splunk instance
3. **splunk-otel-collector** — the OTel agent that ships logs to Splunk via HEC

### splunk-operator.yaml

yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: splunk-operator
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://splunk.github.io/splunk-operator
    chart: splunk-operator
    targetRevision: 3.1.0
    helm:
      valuesObject:
        splunkGeneralTerms: "--accept-sgt-current-at-splunk-com"
        watchNamespaces:
          - splunk
  destination:
    server: https://kubernetes.default.svc
    namespace: splunk-operator
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

The `splunkGeneralTerms` flag is required as of operator version 3.0.0 for Splunk Enterprise 10.x. Without it the container refuses to start and the logs are not obvious about why — this one cost me a while. `watchNamespaces` scopes the operator to the `splunk` namespace rather than cluster-wide.

### splunk-enterprise.yaml

yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: splunk-enterprise
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://splunk.github.io/splunk-operator
    chart: splunk-enterprise
    targetRevision: 3.1.0
    helm:
      valuesObject:
        splunk-operator:
          enabled: false
        standalone:
          enabled: true
          name: splunk
          extraEnv:
            - name: SPLUNK_GENERAL_TERMS
              value: "--accept-sgt-current-at-splunk-com"
          resources:
            requests:
              cpu: 250m
              memory: 2Gi
            limits:
              cpu: "1"
              memory: 3Gi
          etcVolumeStorageConfig:
            storageCapacity: 5Gi
          varVolumeStorageConfig:
            storageCapacity: 20Gi
  destination:
    server: https://kubernetes.default.svc
    namespace: splunk
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

A few things worth calling out here.

**`splunk-operator.enabled: false`** — The enterprise chart can deploy the operator as a sub-chart dependency. Since the operator already has its own ArgoCD application, this must be disabled. I missed this the first time and ended up with two operators conflicting. ArgoCD was not happy.

**Resource limits** — 2Gi request, 3Gi limit. Do not go below 2Gi or Splunk will OOMKill under any real indexing load. Learned this the hard way.

**Storage** — Two PVCs: `etcVolumeStorageConfig` for config (5Gi) and `varVolumeStorageConfig` for index data (20Gi). Size `varVolume` to your retention needs. Splunk's 500MB/day free ingest limit means 20Gi gives roughly 40 days of headroom before you need to think about it.

### splunk-otel-collector.yaml

yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: splunk-otel-collector
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://signalfx.github.io/splunk-otel-collector-chart
    chart: splunk-otel-collector
    targetRevision: 0.152.0
    helm:
      valuesObject:
        clusterName: "homelab"
        secret:
          create: false
          name: splunk-otel-secret
        splunkPlatform:
          endpoint: https://splunk-splunk-standalone-service.splunk.svc.cluster.local:8088
          insecureSkipVerify: true
          index: main
        logsEngine: otel
        logsCollection:
          containers:
            enabled: true
          journald:
            enabled: true
        clusterReceiver:
          enabled: true
        agent:
          enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: splunk
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

The collector runs as a DaemonSet on every node, reading container logs from `/var/log/pods/` and shipping them to Splunk over HEC.

**`insecureSkipVerify: true`** — Splunk is running with a self-signed cert internally. For intra-cluster traffic this is acceptable. It took me a bit to figure out why HEC connections were failing before I added this.

**`secret.create: false`** — The HEC token lives in a pre-existing secret called `splunk-otel-secret` rather than being passed through Helm values in plaintext. Create it once manually:

bash

```bash
kubectl create secret generic splunk-otel-secret \
  --from-literal=splunk_hec_token=<your-hec-token> \
  -n splunk
```

**`index: main`** — Everything goes to `main`. More on why below.

---

## HEC Token

I ended up with four HEC tokens from various test runs and had to work out which one was actually wired up. The one being used is `otel`, pointed at `main`, which matches the collector config. The others were leftovers from when I was testing per-source indexes before scrapping that approach.

To create one: navigate to **Settings > Data Inputs > HTTP Event Collector > New Token**, give it a name, and on the input settings page select `main` as the allowed index.

![](/assets/hec_token.png)
---

## Why Everything Is in index=main

The original plan was proper index separation — Falco in its own index, audit logs in their own index, everything clean and queryable per source. There is even a `falco-hec` token still sitting in HEC pointing at a dedicated `falco` index as a reminder of that attempt.

The problem was the custom routing in the OTel collector config. To route different sourcetypes to different indexes you need custom processors and routing connectors in the OTel pipeline. Getting those working meant pulling apart the default collector config, and the moment I started overriding the pipeline it broke the terms agreement flag handling, which meant Splunk stopped indexing entirely. Fixing the terms issue broke the routing. Going back and forth between the two never landed in a working state at the same time.

In the end everything went to `main` and sourcetype filtering handles the separation instead. The OTel collector automatically assigns sourcetypes based on the log source, so querying by sourcetype gets you exactly what you need without the routing headache. If you are running at scale, proper index separation is worth figuring out. For a homelab it is not worth the fight.

---

## Verifying the Pipeline

Trigger a Falco alert manually:

bash

```bash
kubectl exec -it -n falco $(kubectl get pod -n falco \
  -l app.kubernetes.io/name=falco \
  -o jsonpath='{.items[0].metadata.name}') -- cat /etc/shadow
```

Then search in Splunk:

```verilog
index=main sourcetype="kube:container:falco" | head 5 | table _time, rule, priority, output
```

![](/assets/etc_shadow.png)


If that returns results the full pipeline is working. Falco detects the event, writes to container stdout, the OTel agent reads from `/var/log/pods/`, ships to HEC, and it lands in `index=main`.
