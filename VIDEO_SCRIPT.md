# TrustGate — Demo Video Script (TIGHT cut, click-by-click)

~250 spoken words · lands ~2:40 · English narration (or NL audio + English subtitles).
`[DO]` = click/move · `[POINT]` = cursor target · `[SAY]` = exact words.

> **Why this is shorter:** trimmed wording AND — the real time killer — **talk OVER every tab switch and while things load. Never wait in silence.** The previous full version ran 3:30; this lands ~2:40.

> **Note:** Fivetran REST (`fivetran_rest_live`) is shown on the **dashboard receipt** (Beat 6), not as a raw tab — avoids the `405` line on camera.

---

## Tab order (open in THIS order so Ctrl+1…Ctrl+7 line up)

| Shortcut | Tab | Pre-set |
| --- | --- | --- |
| Ctrl+1 | Dashboard — https://trustgate-24801890031.us-central1.run.app | Click **Reset demo** |
| Ctrl+2 | Agent Builder — Customer Recovery Agent (Agent Platform → Studio → Agents) | **Flow** view, Tools panel ready |
| Ctrl+3 | Fivetran — connection `fulfill_pageant` Status page | Status tab, banners closed |
| Ctrl+4 | https://trustgate-24801890031.us-central1.run.app/api/fivetran/mcp-evidence | Refreshed |
| Ctrl+5 | https://trustgate-24801890031.us-central1.run.app/api/bigquery/evidence | Refreshed |
| Ctrl+6 | Architecture diagram (GitHub README Mermaid, or mermaid.live) | Diagram on screen |
| Ctrl+7 | GitHub repo — https://github.com/MoAz06/trustgate-ai-agents | MIT license visible |

## Pre-record checklist

- [ ] Close other apps. Focus Assist on (no notifications).
- [ ] Dismiss yellow Google trial banner + blue Fivetran "activate" banner.
- [ ] Hide bookmarks bar (Ctrl+Shift+B). Zoom 100–110%. Record 1080p.
- [ ] **Ctrl+1:** click **Reset demo**.
- [ ] Copy prompt: `Review customer C-1042, angry about late delivery, consider a $75 refund and ask TrustGate.`
- [ ] **Warm-up:** dashboard → **Run Gemini agent** once → **Reset demo** again (kills cold-start lag).
- [ ] **Golden rule: keep talking during every switch and load.** Target 2:35–2:50.

---

# THE SCRIPT

## Beat 1 — Hook · 0:00–0:09
**Window:** Ctrl+1 (Dashboard).
1. `[POINT]` Hover across the dashboard.
2. `[SAY]` *"AI agents take real actions on data they can't vouch for. TrustGate stops an agent until the data behind its action is actually trustworthy."*

---

## Beat 2 — The agent + the tool · 0:09–0:31
**Window:** Ctrl+2 (Agent Builder, Flow). **Start talking as the tab loads.**
1. `[POINT]` Hover the **Customer Recovery Agent** node, then the **Instructions** box on the right.
   `[SAY]` *"This Customer Recovery Agent in Google Cloud Agent Builder can't approve a refund on its own — it must call TrustGate first."*
2. `[DO]` Open the **Tools** panel (click the tool / **+** on the node).
3. `[POINT]` Point at **MCP display name: TrustGate** → **Endpoint URL …/mcp** → **Authentication: None**.
   `[SAY]` *"Its only tool is TrustGate, added as an MCP server — this endpoint. So its one job is to ask TrustGate whether an action is safe."*

---

## Beat 3 — Run it → ALLOW · 0:31–0:58
**Window:** Ctrl+2 → Preview.
1. `[DO]` Click **Preview** → **+ New Session** → click the input box → **Ctrl+V** → **Enter**.
2. `[SAY]` *(as it runs)* *"Watch — Gemini doesn't just answer. It calls TrustGate first and waits for a verdict."*
3. `[POINT]` Point at **#2 proposeTrustGateAction** → **#3 ✓** → in **#4**: **ALLOW**, **Receipt ID**, **fivetran_rest_live**, **fulfill_pageant**.
   `[SAY]` *"The result is an ALLOW decision with an auditable receipt — the Fivetran evidence source, and the connection it relied on, fulfill_pageant. All from live data, nothing staged."*

---

## Beat 4 — Real pipeline · 0:58–1:09
**Window:** Ctrl+3 (Fivetran). **Talk as the tab loads.**
1. `[POINT]` Point at **Google Sheets → Warehouse**, the green **Active** badge, then a green **"Successful sync"** row.
   `[SAY]` *"That data is real — Google Sheets synced through Fivetran into BigQuery. Here's the live connection and a successful sync, not a hardcoded answer."*

---

## Beat 5 — The flip (CORE PROOF) · 1:09–1:38
**Window:** Ctrl+1 (Dashboard).
1. `[DO]` Click **"Simulate new customer tier"**.
2. `[POINT]` Point at the badge **"Next action tier: retention_experiment"**.
   `[SAY]` *"Now the proof that data drives the outcome. I change one thing — to an unseen customer tier."*
3. `[DO]` Click **"Run Gemini agent"**. *(keep talking)*
   `[SAY]` *"Same agent, run again —"*
4. `[POINT]` Point at **APPROVAL_REQUIRED**, then the rule **`unseen_customer_tier_enum`**.
   `[SAY]` *"— and the decision flips to approval-required, fired by the rule unseen_customer_tier_enum. Same agent, different data, different answer."*

---

## Beat 6 — Live, openable proof · 1:38–1:58
**Windows:** Ctrl+1 receipt → Ctrl+4 → Ctrl+5. **Talk through both switches.**
1. `[POINT]` On the receipt, point at **`fivetran_rest_live`**.
   `[SAY]` *"It's all verifiable live — the Fivetran REST evidence right here in the receipt,"*
2. `[DO]` **Ctrl+4** → `[POINT]` **`fivetran_mcp_live`**, **`details_verified: true`**.
   `[SAY]` *"Fivetran's own MCP server with the connection verified,"*
3. `[DO]` **Ctrl+5** → `[POINT]` **`bigquery_rest_live`**, **`customer_id: C-1042`**.
   `[SAY]` *"and a live BigQuery row for customer C-1042."*

---

## Beat 7 — Honest architecture · 1:58–2:20
**Window:** Ctrl+6 (diagram). **Talk as it loads.**
1. `[POINT]` Trace: **Gemini → TrustGate (/mcp) → Fivetran MCP + REST + BigQuery → policy engine**.
   `[SAY]` *"The architecture, honestly: Gemini calls TrustGate's MCP; TrustGate checks Fivetran's MCP server, Fivetran REST, and BigQuery; then a deterministic policy engine returns allow, approval-required, or block. The risk score is a sum of rule weights — no black-box ML."*

---

## Beat 8 — Repo + pay-off · 2:20–2:38
**Window:** Ctrl+7 (GitHub) → quick cut to Ctrl+1.
1. `[DO]` **Ctrl+7**. `[POINT]` repo name, then **MIT license**.
   `[SAY]` *"It's open source under MIT, solo-built, and live at this URL."*
2. `[DO]` **Ctrl+1** (1-second final cut to the dashboard).
   `[SAY]` *"TrustGate turns 'the agent said so' into 'here's the data that earned it.'"*

---

## Still over 3:00? Emergency cuts (each ~5–8s)
- Beat 4: drop the whole sentence after "successful sync."
- Beat 6: skip Ctrl+5 (BigQuery tab) — the receipt + MCP tab already prove "live."
- Beat 7: drop *"The risk score is a sum of rule weights — no black-box ML."*

## Bare narration (read straight through)

1. AI agents take real actions on data they can't vouch for. TrustGate stops an agent until the data behind its action is actually trustworthy.
2. This Customer Recovery Agent in Google Cloud Agent Builder can't approve a refund on its own — it must call TrustGate first. Its only tool is TrustGate, added as an MCP server — this endpoint. So its one job is to ask TrustGate whether an action is safe.
3. Watch — Gemini doesn't just answer. It calls TrustGate first and waits for a verdict. The result is an ALLOW decision with an auditable receipt — the Fivetran evidence source, and the connection it relied on, fulfill_pageant. All from live data, nothing staged.
4. That data is real — Google Sheets synced through Fivetran into BigQuery. Here's the live connection and a successful sync, not a hardcoded answer.
5. Now the proof that data drives the outcome. I change one thing — to an unseen customer tier. Same agent, run again — and the decision flips to approval-required, fired by the rule unseen_customer_tier_enum. Same agent, different data, different answer.
6. It's all verifiable live — the Fivetran REST evidence right here in the receipt, Fivetran's own MCP server with the connection verified, and a live BigQuery row for customer C-1042.
7. The architecture, honestly: Gemini calls TrustGate's MCP; TrustGate checks Fivetran's MCP server, Fivetran REST, and BigQuery; then a deterministic policy engine returns allow, approval-required, or block. The risk score is a sum of rule weights — no black-box ML.
8. It's open source under MIT, solo-built, and live at this URL. TrustGate turns 'the agent said so' into 'here's the data that earned it.'
