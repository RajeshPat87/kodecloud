## Pod Disruption Budget — The Core Idea

A **PDB** is a safety floor for availability during **voluntary disruptions** (node drains, upgrades, autoscaler scale-downs). It works through the **Eviction API**: `kubectl drain` doesn't kill pods directly — it *requests* eviction, and the PDB either allows it or returns **HTTP 429** to make the drain wait until a healthy replacement is ready.

- **Without PDB** → evictions cascade freely → you hit a window with **zero healthy pods** → outage.
- **With PDB** → the eviction on the next pod is **blocked until App 3 is Ready** → the app is always serving → no disruption.

Here's the flow as a colorful diagram:The diagram traces both paths side by side. Reading it:

**Red path (no PDB):** the drain terminates App1, then App2, while App3 is still `pending`. Nothing checks whether a replacement exists first, so Step 3 lands on zero healthy pods → outage.

**Green path (with PDB):** the drain calls the Eviction API, which hits the **PDB gate**. If evicting would drop below `minAvailable`, the API returns **429** and the drain waits. App3 comes up Ready, *then* the next eviction is allowed → the app never goes dark.

The whole trick is that one blue decision node — the PDB doesn't stop disruptions, it just **paces** them so availability never dips below your declared floor.

Minimal manifest that produces this behavior for a 3-replica Deployment:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  maxUnavailable: 1        # at most one pod down at a time
  selector:
    matchLabels:
      app: my-app
```

One gotcha the diagram implies but is worth stating: if you set `maxUnavailable: 0` (or `minAvailable: 100%`), the gate **never** returns "safe" — the drain hangs forever and your node upgrade stalls. That's the most common PDB misconfiguration in production.

Want the AKS node-pool upgrade version of this, showing how `maxSurge` and the PDB interact during a surge upgrade?

flowchart TB
    subgraph WITHOUT["WITHOUT Pod Disruption Budget"]
        direction TB
        W1["Step 1<br/>Node1: App1 ok<br/>Node2: App2 ok<br/>Node3: empty"]:::healthy
        W2["Step 2 - drain starts<br/>Node1: App1 terminating<br/>Node2: App2 ok<br/>Node3: App3 pending"]:::warn
        W3["Step 3 - drain continues<br/>App1 terminated<br/>App2 terminating<br/>App3 NOT ready"]:::danger
        W4["No Healthy Pods Available<br/>= OUTAGE"]:::outage
        W1 --> W2 --> W3 --> W4
    end

    subgraph WITH["WITH Pod Disruption Budget"]
        direction TB
        P1["Step 1<br/>Node1: App1 ok<br/>Node2: App2 ok<br/>Node3: empty"]:::healthy
        P2["Step 2 - drain starts<br/>Node1: App1 terminating<br/>Eviction API checks PDB"]:::warn
        GATE{"PDB check:<br/>would this drop below<br/>minAvailable?"}:::gate
        BLOCK["Evict returns HTTP 429<br/>BLOCK termination on Node2<br/>until App3 is Ready"]:::block
        READY["App3 becomes Ready on Node3"]:::healthy
        P3["Step 3 - safe eviction<br/>App1 terminated<br/>App2 terminating<br/>App3 serving"]:::healthy
        P4["No Service Disruption"]:::success
        P1 --> P2 --> GATE
        GATE -->|unsafe| BLOCK --> READY --> P3
        GATE -->|safe| P3
        P3 --> P4
    end

    classDef healthy fill:#22c55e,stroke:#15803d,color:#ffffff,stroke-width:2px;
    classDef warn fill:#f97316,stroke:#c2410c,color:#ffffff,stroke-width:2px;
    classDef danger fill:#ef4444,stroke:#991b1b,color:#ffffff,stroke-width:2px;
    classDef outage fill:#7f1d1d,stroke:#450a0a,color:#ffffff,stroke-width:3px;
    classDef gate fill:#3b82f6,stroke:#1e40af,color:#ffffff,stroke-width:2px;
    classDef block fill:#eab308,stroke:#a16207,color:#1f2937,stroke-width:3px;
    classDef success fill:#16a34a,stroke:#14532d,color:#ffffff,stroke-width:3px;

    style WITHOUT fill:#fee2e2,stroke:#ef4444,stroke-width:2px,color:#7f1d1d
    style WITH fill:#dcfce7,stroke:#22c55e,stroke-width:2px,color:#14532d