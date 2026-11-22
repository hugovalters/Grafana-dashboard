# Grafana Dashboard

A ready-made Grafana dashboard for ValtersIT CVE scan results — CVE trend,
CVSS distribution, vendor/ecosystem breakdown, most affected packages and a
drill-down table of critical and high findings across all your hosts. Open
source, auditable, privacy-first — powered by the ValtersIT CVE API.

> Part of the ValtersIT platform — CVE & Threat-IP data you can query or run yourself.
> Explore both API and scanners: https://www.valtersit.com/platform/
> Scanner plans & pricing: https://www.valtersit.com/cve/pricing/#scanners

## What it shows

`valtersit-cve-dashboard.json` reads the `valtersit-cve-*` Elasticsearch
indices written by
[`elastic-integration`](https://gitlab.com/valtersit/elastic-integration):

| Panel | Type | What it answers |
|---|---|---|
| Distinct CVEs / Critical CVEs / High CVEs / Affected hosts | Stat | Current exposure — the most recent day with scan data |
| CVE trend by severity | Stacked bars | Is patching keeping up? Findings per day by severity |
| CVSS distribution | Histogram | How severe are the matched CVEs overall (bucket size 1.0) |
| Vendor / ecosystem breakdown | Donut | Where findings come from — OS packages, WordPress, npm, PyPI, Composer, NuGet, container images, … |
| Most affected packages | Bar gauge | Top 15 packages by number of distinct CVEs |
| Critical & high findings | Table | Every CVSS ≥ 7.0 finding with host, package, installed and fixed version — CVE ids link to the ValtersIT write-up |
| Matched CVEs per scan, by host | Time series | Per-host trend from the scan summary documents |

Variables at the top filter every panel by **Host** and **Scanner**
(multi-select, "All" by default). Default time range: last 30 days.

## Your data stays yours

- **The dashboard itself sends nothing anywhere** — it only queries your
  own Elasticsearch through your own Grafana. The only outbound link is the
  CVE id link in the table, which opens the public ValtersIT write-up in
  your browser when you click it.
- What ValtersIT receives from the scans that feed it is described in each
  scanner's README: component name + version and an opaque system
  identifier — never hostnames, IP addresses or file contents.

## Requirements

- Grafana 10.0 or newer (tested with 11.3).
- An Elasticsearch data source pointing at the `valtersit-cve-*` indices,
  time field `@timestamp`.
- Data indexed by `valtersit-elastic.py` with its `index-template.json`
  installed (so `host.name`, `vulnerability.*`, `package.*` are keyword /
  float fields).

## Install

### 1. Add the data source

Grafana › Connections › Data sources › Add › **Elasticsearch**:

| Setting | Value |
|---|---|
| URL | your Elasticsearch, e.g. `https://es.example.internal:9200` |
| Authentication | an API key / user with **read** on `valtersit-cve-*` |
| Index name | `valtersit-cve-*` |
| Time field name | `@timestamp` |

Or provision it:

```yaml
# /etc/grafana/provisioning/datasources/valtersit.yaml
apiVersion: 1
datasources:
  - name: ValtersIT ES
    type: elasticsearch
    access: proxy
    url: https://es.example.internal:9200
    jsonData:
      index: "valtersit-cve-*"
      timeField: "@timestamp"
    secureJsonData:
      apiKey: "<read-only API key>"
```

### 2. Import the dashboard

Dashboards › New › **Import** › upload `valtersit-cve-dashboard.json` (or
paste its raw URL:
`https://gitlab.com/valtersit/grafana-dashboard/-/raw/main/valtersit-cve-dashboard.json`),
then choose your Elasticsearch data source when asked.

Via the API:

```bash
curl -fsSLO https://gitlab.com/valtersit/grafana-dashboard/-/raw/main/valtersit-cve-dashboard.json
python3 - <<'EOF' > import.json
import json
print(json.dumps({"dashboard": json.load(open("valtersit-cve-dashboard.json")), "overwrite": True,
                  "inputs": [{"name": "DS_VALTERSIT", "type": "datasource",
                              "pluginId": "elasticsearch", "value": "<data source uid>"}]}))
EOF
curl -fsS -X POST -H "Authorization: Bearer $GRAFANA_TOKEN" -H "Content-Type: application/json" \
  --data @import.json https://grafana.example.internal/api/dashboards/import
```

Or provision it as a file next to your other dashboards — replace
`${DS_VALTERSIT}` in the JSON with your data source uid first.

## Feeding it data

On each host, schedule a scanner through the Elastic integration, e.g.:

```
30 3 * * * VALTERSIT_API_KEY=vit_v1_xxx ES_URL=https://es:9200 ES_API_KEY=xxx /usr/bin/python3 /opt/elastic-integration/valtersit-elastic.py --run /usr/bin/python3 /opt/linux-cve-scanner/valtersit-linux-scan.py --yes
```

Each scanner's README describes its own set-up and the one-time
authorised-use confirmation. The dashboard fills in from the first run;
the trend panels become meaningful after a few days of scans.

## Customising

- **Severity colours** — overrides on the "CVE trend by severity" panel.
- **CVSS threshold of the table** — edit its query
  (`vulnerability.score.base:>=7`).
- **Alerting** — create a Grafana alert rule from the "Critical CVEs"
  stat query (`vulnerability.severity:"critical"`, unique count of
  `vulnerability.id`) with a threshold of `> 0`.

## Authorised use only

The scanners that produce this data must only be run against systems you
own, or for which you have explicit written authorisation to assess — each
one asks you to confirm exactly that before its first scan.

Licensed under Apache-2.0 (see `LICENSE` and `NOTICE`); provided without
warranty of any kind. "ValtersIT" and the ValtersIT logo are trademarks of
Valters Capital, SIA and are not licensed for use. Grafana is a trademark
of Grafana Labs; Elasticsearch is a trademark of Elasticsearch B.V.; this
project is not affiliated with or endorsed by either.

## Questions or a bug?

Please **don't** open GitHub/GitLab issues — I don't monitor them.
Reach me through the contact form instead: https://www.valtersit.com/contacts/

Maintained by Hugo Valters
