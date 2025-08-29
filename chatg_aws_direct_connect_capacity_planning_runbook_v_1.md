# 1.0 Purpose

Define a repeatable, auditable process to monitor, forecast, and scale AWS Direct Connect (DX) capacity before user-visible saturation occurs. This runbook sets thresholds, reports, and step-by-step operational procedures to: (a) detect when to add capacity (ports/LAG members/VIFs), (b) verify resiliency, and (c) coordinate changes safely.

---

# 2.0 Subjects & Scope

**In scope**: DX Dedicated/Hosted connections, LAGs, Private/Transit/Public VIFs, DX Gateway (DXGW), MACsec, TGW/VGW routing considerations, CloudWatch metrics/dashboards/alarms, EventBridge/SNS notifications, BFD on edge routers, AWS Health maintenance notifications.

**Out of scope**: Last-mile carrier SLAs, physical cross-connect provisioning at the colocation (covered only as change prerequisites), router OS upgrades.

---

# 3.0 Architecture & Data Sources (Overview)

- **Physical**: One or more **Dedicated connections** (1/10/100/400 Gbps) optionally grouped into a **LAG**. Hosted connections (50 Mbps–10/25 Gbps) when provided by partners.
- **Logical**: Private/Transit/Public **Virtual Interfaces (VIFs)** terminating on VGW/TGW/DXGW.
- **Monitoring planes**:
  - **AWS/DX CloudWatch metrics**: Connection\* and VirtualInterface\* Bps/Pps, ConnectionState, ConnectionErrorCount, ConnectionLightLevel{Tx,Rx}, ConnectionEncryptionState.
  - **Routers (on-prem/SP PoP)**: SNMP (interface bps/pps/errors), BGP session state & prefixes, **BFD** timers.
  - **Events/Notifications**: AWS Health (DX maintenance), EventBridge rules, SNS/Chatbot.

---

# 4.0 What We Monitor (Parameters)

## 4.1 Utilization & Traffic Shape

- **Per Connection**: ConnectionBps{Ingress,Egress}, ConnectionPps{Ingress,Egress}.
- **Per VIF**: VirtualInterfaceBps{Ingress,Egress}, VirtualInterfacePps{Ingress,Egress}.
- **Aggregates**: LAG sum (Σ ConnectionBps…) and VIF groups per service domain (e.g., Prod vs NonProd).

## 4.2 Health & Errors

- **State**: ConnectionState (Up/Down), VIF operational state (via console/API), ConnectionEncryptionState (MACsec).
- **Errors**: ConnectionErrorCount (MAC-level errors; use **Sum** statistic), CRC trends.
- **Optics**: ConnectionLightLevelRx/Tx (by OpticalLaneNumber) for fiber health.

## 4.3 Control Plane & Routing (Observed)

- **BGP**: Session up/down (router side), route counts, flap rate. Note: DX does not emit BGP metrics—observe on routers and alert.
- **BFD**: Enabled and effective (DX supports async BFD; must be configured on CPE).

## 4.4 Quotas & Limits

- VIFs per dedicated connection/LAG, transit VIF counts, BGP route scale per VIF, LAG member count, TGW/DXGW associations.

---

# 5.0 Capacity Policy & Thresholds

## 5.1 Definitions

- **Port Speed (S)**: Connection nominal bandwidth in bits/s (e.g., 10 G = 10,000,000,000 bps).
- **Utilization % (U)**: 100 × (Bps ÷ S) for each direction (Ingress/Egress). Compute at 1–5 min granularity.
- **p95 (window)**: 95th percentile of U over rolling 7/30 days.
- **Time-Above-X**: % of periods in a lookback where U ≥ X.

## 5.2 Decision Thresholds (default)

> Tune per environment; start conservative to ensure headroom for bursts and maintenance/failover.

| Trigger ID | Metric Logic              | Default Threshold                                     | Lookback | Action                                                                                         |
| ---------- | ------------------------- | ----------------------------------------------------- | -------- | ---------------------------------------------------------------------------------------------- |
| T1         | p95(U\_e or U\_i)         | ≥ 60%                                                 | 7 days   | **Plan scale**: open capacity case; validate forecasts.                                        |
| T2         | p95(U\_e or U\_i)         | ≥ 70%                                                 | 30 days  | **Commit scale**: order additional port or add LAG member; target completion ≤ 30 days.        |
| T3         | Time-Above-80             | ≥ 20% of periods                                      | 7 days   | **Commit scale** (hot): add capacity or redistribute flows.                                    |
| T4         | Max(U\_e or U\_i)         | ≥ 85% for ≥ 3 consecutive periods                     | 24 h     | **Immediate**: enable traffic shaping/shift traffic; check single-flow limits; expedite scale. |
| T5         | Sum(ConnectionErrorCount) | > 0 for ≥ 3 intervals OR > 100 in any interval        | 1 h      | Investigate optics/cabling; escalate to DX provider; consider proactive optics swap.           |
| T6         | ConnectionLightLevelRx/Tx | deviation > 3 dB from 14-day median OR vendor redline | 24 h     | Open fiber incident; schedule hands-on check.                                                  |
| T7         | ConnectionState           | Down (1+ periods)                                     | N/A      | Failover to redundant path (BFD/BGP); incident mgmt.                                           |
| T8         | ConnectionEncryptionState | Down for any LAG member                               | 5 min    | Investigate MACsec (keys, XPN, SCI); fail closed per policy.                                   |

**Note on single-flow limits**: A single 5-tuple flow tops \~5 Gbps (Region) / \~2.5 Gbps (Local Zone). Apparent “headroom” may still experience per-flow throttling; design for **multipath/parallelism** and avoid interpreting single-flow saturation as link exhaustion.

---

# 6.0 Business-Hour Considerations (UK & US NY)

To ensure capacity planning aligns with **real business impact**, thresholds should be measured both **globally (24h)** and during **regional business hours**:

### UK Business Hours

- **Window:** 09:00–18:00 (Europe/London timezone)
- Track: p95 utilization (7d/30d), Time-Above-80 within this window, and Max load.

### US East (NY) Business Hours

- **Window:** 09:00–18:00 (America/New\_York timezone)
- Track: p95 utilization (7d/30d), Time-Above-80 within this window, and Max load.

### Reporting & Alerts

- Create **separate dashboards** per time zone.
- Alarms can be configured as **composite alarms** that trigger only when both utilization is high AND the current time is within business hours.
- Weekly scorecards should show three series: **UK hours**, **US hours**, and **24h all-day**.
- Capacity decisions are made on the **highest** of the three metrics.

---

# 7.0 Scaling Options & Design Patterns

Design to keep **headroom during failures and maintenance** while avoiding over‑provisioning. Use these patterns together.

## 7.1 Add LAG Members (Scale Out)

- **When**: Any of T1–T4 triggers, or when p95 exceeds the post‑failure safe threshold (see §7.3).
- **How**: Order an additional connection of the same speed and attach to the existing LAG; keep optics/patch panels symmetrical.
- **Pros**: Linear capacity, no IP/BGP changes, fast turn‑up once cross‑connect is ready.
- **Cons**: Hashing variance; single‑flow limits remain.

## 7.2 Upgrade Port Speed (Scale Up)

- **When**: LAG member slots are constrained, or single‑flow performance is a requirement.
- **How**: Order a higher‑speed port (e.g., 10G→100G), migrate VIFs (or rebuild LAG at new speed). Burn‑in ≥7 days before decom.
- **Pros**: Higher per‑flow throughput potential; fewer members to manage.
- **Cons**: Longer lead time; potential MACsec capability changes; router optics/line‑card requirements.

## 7.3 Set LAG **Minimum Links** Correctly

Goal: ensure **post‑failure capacity ≥ observed p95**.

- Given **N** members at speed **S**, total capacity = **N×S**. After one failure: **(N−1)×S**.
- If `p95_LAG_util % > ((N−1)/N) × 100`, a **single failure will congest** → **add a member** or **upgrade speed**.
- Reference thresholds:
  - 2×10G → safe p95 ≤ **50%**; if p95>50% add a 3rd link.
  - 3×10G → safe p95 ≤ **66.7%**; if p95>66.7% add a 4th link.
  - 4×10G → safe p95 ≤ **75%**; if p95>75% scale out/up.
- **Min Links setting**: choose `MinLinks = N−1` for maintenance survival **only if** `(N−1)×S ≥ p95_peak_bps`. Otherwise keep `MinLinks < N−1` and pre‑shift traffic before maintenance.

## 7.4 Split Traffic by VIFs & Policy

- Separate **Prod** vs **NonProd** VIFs; apply **BGP Local‑Pref/Communities** to steer latency‑sensitive vs bulk transfers across LAGs/locations.
- Use **per‑VIF MTU** and QoS (CPE) to protect critical flows.

## 7.5 Multi‑Location Resiliency (A/A)

- Deploy DX in **two locations** with ECMP; prefer **High/Max** models.
- Advertise same prefixes with equal preference for A/A; use longer/shorter prefixes or communities for primary/secondary steering where needed.

## 7.6 Transit VIF + TGW Hub

- For multi‑VPC scale, use **Transit VIF to TGW** and control growth via TGW route tables and summarization.

## 7.7 VPN as Continuity Path

- Maintain **Site‑to‑Site VPN** backup with enough bandwidth to absorb at least **p95** of critical traffic. Advertise with lower preference.

---

# 8.0 Operational Procedures (Step‑by‑Step)

## 8.1 Establish Baselines

1. Create 30/90‑day dashboards (per **Connection** and **VIF**) showing U% (ingress/egress), p95 lines, and business‑hour overlays (UK/US).
2. Produce the weekly **Capacity Scorecard** (see §11.0) and store in the runbook space.

## 8.2 Build CloudWatch Dashboards

- Panels: ConnectionBps{Ingress,Egress}, VirtualInterfaceBps{Ingress,Egress}, **U%** (metric math), **ErrorCount (Sum)**, **LightLevelRx/Tx**, **EncryptionState**.
- Add **LAG aggregate** panels (sum of members ÷ total capacity) and separate **UK/US business‑hours** panels (see §9.12–§9.13).

## 8.3 Configure Alarms

- **Link Down**: ConnectionState==0 (1 period).
- **Hot Utilization**: Time‑Above‑80 ≥20% over last 5 hours (5‑min periods; `DatapointsToAlarm=12` of 60).
- **p95 Headroom**: Daily p95(U%) ≥70% (via Metrics Insights → Ops review notification).
- **Error Burst**: Sum(ConnectionErrorCount) ≥100 in 5 min.
- **Optics Anomaly**: LightLevelRx anomaly detection per lane.
- **MACsec Down**: ConnectionEncryptionState==0 (1 period).
- **Business‑Hours Variants**: composite alarms `(UtilizationHigh AND UKBusinessHours)` and `(UtilizationHigh AND USBusinessHours)`.

## 8.4 Capacity Change – **Add LAG Member**

1. Validate port availability with provider; ensure same bandwidth/optic type.
2. Raise CRQ with risk/backout; set **MinLinks** so traffic stays online during add.
3. Turn‑up & attach new connection to LAG; verify hashing distribution and error‑free counters.
4. Post‑change tests: BFD/BGP up, U% trends normal, optics stable; update diagrams & dashboards.

## 8.5 Capacity Change – **New Connection / New LAG or Speed Upgrade**

1. Order via **Resiliency Toolkit** (choose High/Max as required).
2. Receive LOA‑CFA; coordinate cross‑connect with colo.
3. Validate light levels, MTU, and MACsec (if used).
4. Create/attach VIFs; align BGP policy; stage at lower local‑pref; controlled switchover.
5. Burn‑in ≥7 days; then decommission old path.

## 8.6 Routing & Resiliency Controls

- Enable **BFD** (async; \~300 ms / multiplier 3) on CPE; avoid graceful‑restart with BFD.
- Prefer **ECMP** across active paths; use communities/local‑pref for traffic engineering.
- Validate **VPN backup** route preferences; run quarterly failover tests.

## 8.7 Maintenance Windows

1. Subscribe to DX/AWS Health notices; confirm scope.
2. Pre‑shift bulk traffic; verify that `(N−1)×S` covers p95 or reduce load.
3. Monitor ConnectionState, ErrorCount, LightLevel; keep provider bridge open.
4. Post‑checks: optics deltas, error counters, p95 re‑baseline; reset any temporary routing policies.

---

# 9.0 Implementation Guide (How-To)

(Already expanded with alarms, dashboards, queries, business-hours logic)

---

# 10.0 Run of Show (Capacity Event)

1. **Detect** (alarm fires T2/T3/T4) → **Assess** current U/p95, burst patterns, per‑flow behavior (UK/US windows and all‑day).
2. **Contain**: shift bulk jobs; confirm backup path; increase Min Links if safe; rate‑limit noisy flows if needed.
3. **Decide**: choose scale option (LAG add vs speed upgrade vs new DX PoP). Open change request.
4. **Execute**: follow §8.4/§8.5/§9.8/§9.9.
5. **Verify**: post‑change KPIs (U%, errors=0, optics stable), run failover test.
6. **Record**: update Scorecard and review at weekly capacity meeting.

---

# 11.0 KPIs & Scorecard (template)

- p95 U% (Ingress/Egress) by Connection & VIF
- Time‑Above‑80 (% of periods)
- Max U% (24 h)
- ErrorCount Sum (24 h / 7 d)
- Optics delta vs 14‑day median (dB)
- BGP flaps (#/day)
- MACsec uptime (%)
- Forecast (p95 next 4 weeks)
- **Business Hour KPIs (UK & US):** p95, Time‑Above‑80, Max

---

- p95 U% (Ingress/Egress) by Connection & VIF
- Time-Above-80 (% of periods)
- Max U% (24 h)
- ErrorCount Sum (24 h / 7 d)
- Optics delta vs 14-day median (dB)
- BGP flaps (#/day)
- MACsec uptime (%)
- Forecast (p95 next 4 weeks)
- **Business Hour KPIs (UK & US):** p95, Time-Above-80, Max

---

# 12.0 Compliance & Change Control

**Change policy**

- All capacity increases require a **CRQ** with: risk analysis, test plan, roll‑back plan, owner approvals, and maintenance window.
- Validate **LAG MinLinks** vs observed p95 (see §7.3) before any member maintenance.
- Attach AWS Health/DX notices and LOA‑CFA to the CRQ.

**Security**

- If using **MACsec**: document key IDs (CKN/CAK), rotation cadence, XPN settings, and acceptance tests (EncryptionState==1).
- Ensure CloudWatch Logs, dashboards, alarms, and SNS topics are **encrypted at rest** with organization KMS keys.

**Documentation**

- Update network diagrams, IP/BGP peering tables, and DX inventory after each change.
- Keep the **Capacity Scorecard** history for 12 months for audits.

---

# 13.0 Appendix A – Quick Reference Thresholds

- **Plan** at p95 ≥ **60%** (7d)
- **Commit** at p95 ≥ **70%** (30d) **or** Time‑Above‑80 ≥ **20%** (7d)
- **Hot** when Max ≥ **85%** for ≥15 min
- **Errors**: any persistent non‑zero; **burst** ≥100 in 5 min
- **Optics**: >3 dB deviation or vendor redline
- **MACsec**: any down

---

# 14.0 Appendix B – Terraform Snippets

```hcl
variable "dx_connection_id" {}
variable "port_speed_bps" { default = 10000000000 }

resource "aws_sns_topic" "dx_alarms" { name = "dx-alarms" }

resource "aws_cloudwatch_metric_alarm" "dx_time_above_80" {
  alarm_name          = "DX-conn-egress-80pct-20pct-of-time"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 60
  datapoints_to_alarm = 12
  threshold           = 1
  treat_missing_data  = "notBreaching"

  metric_query {
    id          = "m1"
    return_data = false
    metric {
      namespace   = "AWS/DX"
      metric_name = "ConnectionBpsEgress"
      period      = 300
      stat        = "Average"
      dimensions  = { ConnectionId = var.dx_connection_id }
    }
  }

  metric_query {
    id          = "e1"
    expression  = "IF((m1/${var.port_speed_bps})*100 >= 80, 1, 0)"
    label       = "hot"
    return_data = true
  }

  alarm_actions = [aws_sns_topic.dx_alarms.arn]
}
```

---

# 15.0 Appendix C – Router Snippets (Illustrative)

**Cisco IOS‑XE**

```
ip cef
bfd-template single-hop BFD_DX
 interval 300 min_rx 300 multiplier 3
!
interface TenGigabitEthernet0/0/0
 mtu 9001
 bfd interval 300 min_rx 300 multiplier 3
!
router bgp <ASN>
 neighbor <AWS_PEER_IP> remote-as <AWS_ASN>
 neighbor <AWS_PEER_IP> timers 30 90
 neighbor <AWS_PEER_IP> fall-over bfd
! (avoid graceful-restart when using BFD)
```

**Juniper (Junos)**

```
protocols bfd {
  interface xe-0/0/0 {
    minimum-interval 300;
    multiplier 3;
  }
}
protocols bgp {
  group AWS-DX {
    type external;
    neighbor <AWS_PEER_IP> {
      peer-as <AWS_ASN>;
      bfd-liveness-detection {
        minimum-interval 300;
        multiplier 3;
      }
    }
  }
}
```

---

# 16.0 Appendix D – Operations Checklist

-

---

# 17.0 Further Notes

- Respect **single‑flow limits** (≈5 Gbps in‑Region; ≈2.5 Gbps Local Zones). Favor parallelism/ECMP.
- Verify **hashing distribution** across LAG members; investigate skew.
- Keep optic spares and loopback plugs on‑site for rapid triage.
- Re‑evaluate thresholds after major topology or traffic pattern changes.

