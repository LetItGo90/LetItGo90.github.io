 # Scanning My Homelab: SAST in the Pipeline and ZAP Against the Running App

Security scanning is one of those things that sounds obvious until you actually try to wire it up properly. My goal here was straightforward: catch bad manifests before they reach the cluster, and then actively scan the running application after deployment to see what slipped through. For that I used Trivy and Kubescape for SAST and ZAP for the DAST side. It's also worth noting that this is part of my larger homelab security stack I've been building out, and it has been a learning process. This continues the Splunk post I already made since we'll be adding DAST data to the dashboards.

Here's how it's set up and what it actually produces.

---

## The Idea: Scan Before and After

There are two distinct moments where scanning makes sense.

**Before deployment**, while the code is still a PR. This is where you catch misconfigured manifests, known CVEs in images, and secrets accidentally committed. If something fails here, it never reaches the cluster and the merge is blocked.

**After deployment**, when the application is actually running. Static analysis can't tell you if your app responds to a SQL injection or exposes a sensitive header. That requires actually hitting it. This is where ZAP comes in.

These are often called SAST (Static Application Security Testing) and DAST (Dynamic Application Security Testing), but the labels matter less than understanding that they catch fundamentally different things.

---

## Part 1: Blocking Bad Code at the Gate

### The Tools

Two scanners run in GitHub Actions on every PR that touches the `infrastructure/` folder.

**Trivy** is an open source scanner from Aqua Security. In this setup it runs in config mode, meaning it's not scanning a container image but reading the Kubernetes YAML files themselves and looking for misconfigurations: missing resource limits, containers running as root, writable root filesystems, that kind of thing. It only flags `CRITICAL` severity to avoid noise. That's not realistic for a corporate environment, but this is a homelab.

**Kubescape** evaluates manifests against real security frameworks, specifically NSA/CISA Kubernetes Hardening Guidance and the MITRE ATT&CK matrix. Where Trivy is looking for known bad patterns, Kubescape is asking "does this manifest align with how a hardened cluster should look?" It's a useful second opinion because the two tools don't always flag the same things.

Both scanners upload their results in SARIF format to GitHub's Security tab, so findings are visible per PR without having to dig through logs. One practical note: both Trivy and Kubescape provide ready-made GitHub Action templates, so with minor adjustments you can be up and running fairly quickly.

### Branch Protection as the Enforcer

The scanners are only useful if a failed scan actually stops something. Branch protection rules on `main` require both checks to pass before a merge is allowed. The `gate` job in the workflow handles this, so if either Kubescape or Trivy fails, the gate exits with a non-zero code and the PR is blocked. You'll also get an email when a scan fails, and you can set up a webhook to route those alerts to Teams, Slack, or whatever paging system you use.

In practice this means ArgoCD can never sync a manifest that hasn't passed both scans, because that manifest can never land on `main`.

```
PR opened
  → Kubescape scans manifests against NSA + MITRE frameworks
  → Trivy scans for CRITICAL misconfigurations
     → Both pass? Gate opens, merge allowed, ArgoCD syncs
     → Either fails? Gate blocks the merge
```

### `infra-scan.yaml`

```yaml
on:
  push:
    paths:
      - 'infrastructure/**'
  pull_request:
    paths:
      - 'infrastructure/**'

jobs:
  kubescape:
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write
    steps:
    - uses: actions/checkout@v3
    - uses: kubescape/github-action@main
      continue-on-error: false
      with:
        format: sarif
        frameworks: nsa,mitre
        outputFile: results
    - name: Upload Kubescape scan results to Github Code Scanning
      uses: github/codeql-action/upload-sarif@v4
      with:
        sarif_file: results.sarif

  trivy:
    name: Trivy
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      security-events: write
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          scan-type: 'config'
          scan-ref: './infrastructure'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL'
      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v4
        with:
          sarif_file: 'trivy-results.sarif'

  gate:
    needs: [kubescape, trivy]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Check for Kubescape or Trivy scan failures
        run: |
          if [[ "${{ needs.kubescape.result }}" == "failure" ]]; then
            echo "Kubescape scan failed. Please fix the issues before merging."
            exit 1
          fi
          if [[ "${{ needs.trivy.result }}" == "failure" ]]; then
            echo "Trivy scan failed. Please fix the issues before merging."
            exit 1
          fi
          echo "All scans passed successfully."
```

### What This Does and Doesn't Catch

Worth being honest about this. Trivy and Kubescape are scanning the manifests, not running the application. They'll catch a container configured to run as root, an image with a known CVE, or a missing security context. They won't catch a SQL injection in your app, a broken authentication flow, or a misconfigured CORS header that only shows up when the service is actually responding to requests.

That's not a criticism, it's just what static analysis is. It's fast, it's cheap to run on every PR, and it stops obviously bad configs from ever touching the cluster. For the rest, you need a live scanner.

---

## Part 2: ZAP Against the Running Application

### What ZAP Is

[OWASP ZAP](https://www.zaproxy.org/) (Zed Attack Proxy) is a free, open source web application security scanner. It works by actually sending requests to a running application, crawling it, then actively probing it with known attack patterns: SQL injection attempts, XSS payloads, header manipulation. The kinds of things that would only show up if you're hitting a live service.

In this setup, ZAP runs against Juice Shop, an intentionally vulnerable Node.js app from OWASP that's designed to be attacked. It's a little unique in that I have it set up as a CronJob rather than wired directly into the CI/CD pipeline, because the goal was to get results into Splunk. There might be a way to pull results from workflow artifacts into Splunk, but that felt like overkill for what I needed here.

### How It's Deployed

The setup follows the approach documented in the official ZAP blog post [*Use ZAP with Flagger in Kubernetes*](https://www.zaproxy.org/blog/2024-12-24-use-zap-with-flagger-in-kubernetes/) by Trevor Mountney. The pattern of running ZAP as a suspended CronJob that gets triggered on demand is taken directly from there. Worth reading if you want to extend this to canary deployments where ZAP scans each new release before it goes live.

For this setup it's simpler: four resources managed by ArgoCD.

- An **ArgoCD Application** pointing at `infrastructure/zap` in the repo
- A **ConfigMap** with the ZAP Automation Plan
- A **suspended CronJob** that's the actual ZAP runner
- A **Service** exposing the ZAP API

The CronJob runs suspended, meaning it never fires on a schedule and gets triggered on demand instead, either manually or via a hook. For a homelab I'd recommend keeping it manual so you don't overwhelm log storage and system resources. Once the scan finishes the pod exits and that's it, no long-running process sitting idle consuming resources.

One other thing worth calling out: to get results into Splunk you have two main options, an HEC token or the OpenTelemetry collector. I was already using the OTel collector since it handles all the heavy lifting. The downside is there's no native stdout output from ZAP. After some experimenting, the approach that worked was writing the scan results to a JSON report and catting it out formatted with `jq`, which gives Splunk something it can actually extract fields from.

### `zap.yaml` — ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: zap
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/LetItGo90/HomeLab-Project.git
    targetRevision: main
    path: infrastructure/zap
  destination:
    server: https://kubernetes.default.svc
    namespace: zap-demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

ArgoCD manages the ZAP deployment the same way it manages everything else. Any change to `infrastructure/zap` on `main` syncs to the cluster automatically, after passing the SAST gate.

### `zap-configmap.yaml` — The Automation Plan

The Automation Plan is what ZAP actually executes. It's stored as a ConfigMap and mounted into the pod as `af-plan.yaml`. This particular plan does three things in sequence: spider the app to find endpoints, run an active scan against them, then write a JSON report.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/name: zap
  name: zap-config
  namespace: zap-demo
data:
  af-plan.yaml: |
    env:
      contexts:
      - authentication:
          parameters: {}
          verification:
            method: response
            pollFrequency: 60
            pollUnits: seconds
        includePaths:
        - http://juice-shop.juice-shop.svc.cluster.local:3000.*
        name: Default Context
        sessionManagement:
          method: cookie
          parameters: {}
        technology:
          exclude: []
        urls:
        - http://juice-shop.juice-shop.svc.cluster.local:3000
      parameters:
        failOnError: false
        failOnWarning: false
        progressToStdout: true
      vars: {}
    jobs:
    - name: spider
      type: spider
      parameters:
        context: Default Context
        url: http://juice-shop.juice-shop.svc.cluster.local:3000
        maxDuration: 5
    - name: activeScan
      type: activeScan
      parameters:
        context: Default Context
        maxAlertsPerRule: 0
        maxRuleDurationInMins: 0
        maxScanDurationInMins: 2
        policy: ""
        threadPerHost: 2
        user: ""
      policyDefinition:
        defaultStrength: medium
        defaultThreshold: medium
        rules: []
    - name: json-report
      type: report
      parameters:
        template: traditional-json
        reportDir: /tmp
        reportFile: zap-report
        reportTitle: ZAP Scanning Report
        reportDescription: ""
        displayReport: false
      risks:
      - medium
      - high
      confidences:
      - medium
      - high
      - confirmed
      sites: []
```

The scan duration is capped at 2 minutes and the spider at 5. For a demo target that's enough to surface real findings without running for an hour. In production you'd want to be more generous with those limits.

### `cronjob.yaml`

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: zap
  namespace: zap-demo
spec:
  schedule: "* * * * *"
  suspend: true
  jobTemplate:
    spec:
      backoffLimit: 0
      template:
        metadata:
          labels:
            app.kubernetes.io/name: zap
        spec:
          containers:
          - command: ["/bin/sh", "-c"]
            args:
            - "./zap.sh -cmd -autorun /zap/config/af-plan.yaml -host 0.0.0.0
              -config api.disablekey=true -config 'api.addrs.addr.name=.*'
              -config api.addrs.addr.regex=true;
              cat /tmp/zap-report.json | jq -c ."
            image: ghcr.io/zaproxy/zaproxy:stable
            name: zaproxy
            ports:
            - containerPort: 8080
              name: zaproxy
              protocol: TCP
            startupProbe:
              failureThreshold: 3
              httpGet:
                path: /
                port: 8080
                scheme: HTTP
              initialDelaySeconds: 60
              periodSeconds: 10
              successThreshold: 1
              timeoutSeconds: 3
            volumeMounts:
            - mountPath: /zap/config
              name: config
          restartPolicy: Never
          volumes:
          - name: config
            configMap:
              name: zap-config
```

After the automation plan finishes, the report gets piped through `jq -c .` and written to stdout as a single compact JSON line. That's the output the OpenTelemetry collector picks up and ships to Splunk.

### `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: zap
  name: zap
  namespace: zap-demo
spec:
  ports:
  - name: zaproxy
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/name: zap
```

For this setup the Service isn't strictly needed since the scan runs headlessly and exits, so it serves no real purpose in my lab. I added it anyway in case I wanted to experiment with webhooks later. It becomes useful if you extend this with Flagger webhooks, where something external needs to hit the ZAP API to trigger or control the scan mid-run.

---

## Part 3: Findings in Splunk

### Getting the Data In

The ZAP JSON report lands in Splunk via the OpenTelemetry collector reading the pod's stdout. It comes in as one long compact JSON string. Not pretty to look at raw, but Splunk has already parsed the nested structure and the fields are there, you just need to query them correctly.

---

![](/assets/Dast_Raw_Scan.png)

---

### Querying the Findings

ZAP's JSON structure is heavily nested: sites contain alerts, alerts contain fields. The key to making this readable in Splunk is `mvexpand`, which expands the nested arrays into rows.

```spl
index=* sourcetype="kube:container:zaproxy"
| mvexpand "site{}.alerts{}.alert"
| rename "site{}.@name" as siteName
| rename "site{}.alerts{}.alert" as alert
| rename "site{}.alerts{}.riskdesc" as riskdesc
| rename "site{}.alerts{}.confidence" as confidence
| rename "site{}.alerts{}.count" as count
| where isnotnull(alert) AND alert!=""
| dedup siteName, alert, riskdesc, confidence
| table siteName, alert, riskdesc, confidence, count
```

The `where` filter and `dedup` are important. Without them you get pages of blank rows from misaligned multi-value field expansion. A small thing that costs a lot of time to figure out.

---

![](/assets/dast_scan_query_splunk.png)
---

### Dashboard

Three panels built in Splunk Dashboard Studio.

---

![](/assets/splunk_dashboard.png)
---

**Panel 1 — Alert Summary Table**
```spl
index=* sourcetype="kube:container:zaproxy"
| mvexpand "site{}.alerts{}.alert"
| rename "site{}.@name" as siteName
| rename "site{}.alerts{}.alert" as alert
| rename "site{}.alerts{}.riskdesc" as riskdesc
| rename "site{}.alerts{}.confidence" as confidence
| rename "site{}.alerts{}.count" as count
| where isnotnull(alert) AND alert!=""
| dedup siteName, alert, riskdesc
| rex field=riskdesc "^(?P<riskLevel>\w+)\s+\((?P<confidenceLevel>\w+)\)"
| table siteName, alert, riskLevel, confidenceLevel, count
| sort riskLevel
```

**Panel 2 — Alerts by Risk Level**
```spl
index=* sourcetype="kube:container:zaproxy"
| mvexpand "site{}.alerts{}.alert"
| rename "site{}.alerts{}.riskdesc" as riskdesc
| where isnotnull(riskdesc) AND riskdesc!=""
| rex field=riskdesc "^(?P<riskLevel>\w+)"
| dedup "site{}.alerts{}.alert", riskLevel
| stats count by riskLevel
| sort -count
```

**Panel 3 — Alerts Over Time**
```spl
index=* sourcetype="kube:container:zaproxy"
| mvexpand "site{}.alerts{}.alert"
| rename "site{}.alerts{}.alert" as alert
| rename "site{}.alerts{}.riskdesc" as riskdesc
| rex field=riskdesc "^(?P<riskLevel>\w+)"
| where isnotnull(alert) AND alert!=""
| timechart span=5m count by riskLevel
```

One thing worth knowing about ZAP's `riskdesc` field: it combines risk and confidence into a single string like `Medium (High)`. That means `Medium (Medium)` and `Medium (High)` are both Medium risk findings, the difference is ZAP's confidence in the detection. The `rex` extraction above splits these cleanly so risk groupings don't get polluted by confidence variance in your charts.

### Splunk Alert — High Severity Findings

A Splunk alert fires on any `High` severity ZAP finding, wiring application security detections into the same alerting pipeline as the infrastructure threat detection.

```spl
index=* sourcetype="kube:container:zaproxy"
| mvexpand "site{}.alerts{}.alert"
| rename "site{}.alerts{}.riskdesc" as riskdesc
| rename "site{}.alerts{}.alert" as alert
| rex field=riskdesc "^(?P<riskLevel>\w+)"
| where riskLevel="High"
| stats count by alert, riskdesc
```

---

## What This Does and Doesn't Show

ZAP with a 5 minute spider and a 2 minute active scan is not a thorough penetration test. It will find low-hanging fruit: missing security headers, obvious injection points, known vulnerability patterns. It won't find logic flaws, authentication bypasses that require understanding the app flow, or anything that needs actual context about what the application is supposed to do.

What it does show is that the pipeline works end to end. Bad manifest goes in, SAST blocks it. Good manifest deploys, ZAP scans the running app and the findings land in Splunk where they can alert. That's the loop, and it's genuinely useful even if the scan depth is limited.

To get more out of ZAP, the next step is proxying integration test traffic through it so instead of spidering from scratch, ZAP has real authenticated requests to work from. The ZAP + Flagger blog post linked above covers exactly that pattern.

---

## Summary

| Stage | Tool | What It Catches | Blocks Deploy? |
|---|---|---|---|
| Pre-merge | Kubescape | Manifest hardening vs NSA/MITRE | Yes |
| Pre-merge | Trivy | CRITICAL misconfigs in YAML | Yes |
| Post-deploy | ZAP | Live app vulnerabilities | Alerts only |

The SAST gate means nothing that fails a security check can be merged, and therefore nothing that fails can be deployed. ZAP closes the loop on the other side. What the static scanners can't see, the live scanner probes directly. Neither is complete on its own, but together they cover both ends.
