# eu-sparql-search

Search EU legislation, CJEU case law, and publications via the **Publications Office SPARQL endpoint** (Cellar / EUR-Lex). This skill translates natural-language queries into SPARQL, executes them against the Cellar knowledge graph, and returns structured results with verifiable source links.

---

## What It Can Do

- Search **EU Regulations, Directives, Decisions, Opinions, Recommendations**
- Retrieve **CJEU Judgments and Orders** by date or CELEX number
- Filter by **language**, **date range**, **in-force status**, or **thematic EuroVoc concept**
- Return **direct download URLs** for PDF/HTML files (via the full Work→Expression→Manifestation→Item chain)
- Produce **clickable EUR-Lex and Cellar citations** with every answer

---

## Network Configuration — Required Domain Allowlist

This skill makes outbound HTTP requests to three EU Publications Office domains. All three must be present in your environment's **domain allowlist** (egress proxy / network policy) for the skill to work correctly.

### Required Domains

| Domain | Wildcard variant | Purpose |
|--------|-----------------|---------|
| `publications.europa.eu` | `*.publications.europa.eu` | SPARQL endpoint + Cellar REST file downloads |
| `eur-lex.europa.eu` | `*.eur-lex.europa.eu` | EUR-Lex canonical document links shown to users |
| `eurovoc.europa.eu` | `*.eurovoc.europa.eu` | EuroVoc thematic concept URIs (used in subject-based queries) |

> All three endpoints are **public and require no authentication**. HTTPS (port 443) is sufficient; no special headers or certificates are needed.

---

### How to Add Domains — Examples by Platform

#### Claude.ai / Anthropic Console (skills network settings)

In the skill's `SKILL.md` front-matter or your workspace network policy, add the allowed domains as a list:

```yaml
network:
  allowed_domains:
    - "*.eur-lex.europa.eu"
    - "eur-lex.europa.eu"
    - "*.eurovoc.europa.eu"
    - "eurovoc.europa.eu"
    - "*.publications.europa.eu"
    - "publications.europa.eu"
```

#### System Prompt / `<network_configuration>` block

If your deployment uses a `<network_configuration>` block in the system prompt, add all six entries to the `Allowed Domains` list:

```xml
<network_configuration>
  Enabled: true
  Allowed Domains: *.eur-lex.europa.eu, *.eurovoc.europa.eu, *.publications.europa.eu,
                   eur-lex.europa.eu, eurovoc.europa.eu, publications.europa.eu
</network_configuration>
```

#### Nginx egress proxy (`/etc/nginx/conf.d/allowlist.conf`)

```nginx
# EU Publications Office — eu-sparql-search skill
server {
    listen 8080;
    resolver 8.8.8.8;

    location / {
        proxy_pass $scheme://$host$request_uri;

        # Allow only the three required domains
        if ($host !~* "(publications\.europa\.eu|eur-lex\.europa\.eu|eurovoc\.europa\.eu)$") {
            return 403;
        }
    }
}
```

#### AWS Security Group / VPC Endpoint Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEUPublicationsOffice",
      "Effect": "Allow",
      "Action": "execute-api:Invoke",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:RequestedRegion": "eu-*"
        }
      }
    }
  ]
}
```

For outbound HTTPS, add these entries to your **VPC Network ACL** or **Security Group egress rules**:

| Type | Protocol | Port | Destination |
|------|----------|------|-------------|
| HTTPS | TCP | 443 | `publications.europa.eu` |
| HTTPS | TCP | 443 | `eur-lex.europa.eu` |
| HTTPS | TCP | 443 | `eurovoc.europa.eu` |

#### Squid Proxy (`/etc/squid/squid.conf`)

```squid
# EU Publications Office — eu-sparql-search skill
acl eu_publications dstdomain .publications.europa.eu publications.europa.eu
acl eu_eurlex       dstdomain .eur-lex.europa.eu      eur-lex.europa.eu
acl eu_eurovoc      dstdomain .eurovoc.europa.eu       eurovoc.europa.eu

http_access allow eu_publications
http_access allow eu_eurlex
http_access allow eu_eurovoc
http_access deny all
```

#### GitHub Actions (if running in a restricted runner network)

```yaml
- name: Allow EU Publications Office domains
  run: |
    sudo iptables -A OUTPUT -p tcp --dport 443 \
      -d publications.europa.eu -j ACCEPT
    sudo iptables -A OUTPUT -p tcp --dport 443 \
      -d eur-lex.europa.eu -j ACCEPT
    sudo iptables -A OUTPUT -p tcp --dport 443 \
      -d eurovoc.europa.eu -j ACCEPT
```

---

### Verifying Connectivity

Run these three checks from the environment where Claude executes `bash_tool`:

```bash
# 1. SPARQL endpoint (most critical)
curl -s -o /dev/null -w "%{http_code}" \
  "https://publications.europa.eu/webapi/rdf/sparql?query=SELECT+%2A+WHERE+%7B%7D+LIMIT+1&format=application%2Fsparql-results%2Bjson"
# Expected: 200

# 2. Cellar REST API
curl -s -o /dev/null -w "%{http_code}" \
  "https://publications.europa.eu/resource/authority/resource-type/REG"
# Expected: 200 or 303

# 3. EUR-Lex (for citation links)
curl -s -o /dev/null -w "%{http_code}" \
  "https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=celex:32016R0679"
# Expected: 200
```

If any check returns a non-2xx/3xx code or a `x-deny-reason` header, the domain is blocked by your egress proxy and must be added to the allowlist.

---

## Endpoint

```
https://publications.europa.eu/webapi/rdf/sparql
```

- **Authentication:** none — public endpoint
- **Methods:** HTTP GET and POST
- **Key parameters:**

| Parameter | Description |
|-----------|-------------|
| `query`   | SPARQL query string (URL-encoded) |
| `format`  | Output format (see table below) |
| `timeout` | Milliseconds — use `30000` for safety |

### Output Formats

| `format` value | Description |
|----------------|-------------|
| `application/sparql-results+json` | JSON — best for programmatic use |
| `text/csv` | CSV |
| `application/sparql-results+xml` | XML |
| `text/turtle` | Turtle RDF |
| `text/html` | HTML table |

---

## Data Model (CDM)

Every document lives at four nested levels:

```
Work            ← abstract document (e.g. "Regulation 2016/679")
  Expression    ← language version (e.g. Polish, English)
    Manifestation ← file format (pdfa2a, fmx4, xhtml)
      Item        ← downloadable file URL
```

Main ontology prefix: `cdm:` → `http://publications.europa.eu/ontology/cdm#`

### Standard Prefixes (include in every query)

```sparql
PREFIX cdm:   <http://publications.europa.eu/ontology/cdm#>
PREFIX annot: <http://publications.europa.eu/ontology/annotation#>
PREFIX skos:  <http://www.w3.org/2004/02/skos/core#>
PREFIX dc:    <http://purl.org/dc/elements/1.1/>
PREFIX xsd:   <http://www.w3.org/2001/XMLSchema#>
PREFIX rdf:   <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
```

### Key CDM Properties

| Property | Description | Example value |
|---|---|---|
| `cdm:work_has_resource-type` | Document type URI | e.g. `.../resource-type/REG` |
| `cdm:resource_legal_id_celex` | CELEX number | `"32016R0679"` |
| `cdm:work_date_document` | Document date | `"2016-04-27"^^xsd:date` |
| `cdm:resource_legal_in-force` | Currently in force | `"true"^^xsd:boolean` |
| `cdm:expression_uses_language` | Language resource | join with `dc:identifier` — ISO 639-3 (`POL`, `ENG`…) |
| `cdm:manifestation_type` | File format | `"pdfa2a"`, `"fmx4"`, `"xhtml"` |
| `cdm:do_not_index` | Hidden doc flag | always filter `NOT EXISTS` |
| `cdm:expression_belongs_to_work` | Expression → Work link | join predicate |
| `cdm:manifestation_manifests_expression` | Manifestation → Expression link | join predicate |
| `cdm:item_belongs_to_manifestation` | Item → Manifestation link | join predicate |

### Resource Types (most common)

| Code | Document type |
|------|--------------|
| `REG` | Regulation |
| `DIR` | Directive |
| `DEC` | Decision |
| `RECO` | Recommendation |
| `OPIN` | Opinion |
| `JUDG` | Judgment (CJEU) |
| `ORDER_CJEU` | Order (CJEU) |
| `CASELAW` | Case law (generic) |

Full URI pattern: `<http://publications.europa.eu/resource/authority/resource-type/REG>`

### Language Codes

> ⚠️ Cellar uses **ISO 639-3 three-letter codes**, not two-letter ISO 639-1 codes (e.g. `POL`, not `pl`).

| Language | Code | Language | Code |
|---|---|---|---|
| Polish | `POL` | Romanian | `RON` |
| English | `ENG` | Bulgarian | `BUL` |
| German | `DEU` | Croatian | `HRV` |
| French | `FRA` | Slovak | `SLK` |
| Italian | `ITA` | Swedish | `SWE` |
| Spanish | `SPA` | Finnish | `FIN` |
| Dutch | `NLD` | Danish | `DAN` |
| Czech | `CES` | Greek | `ELL` |
| Hungarian | `HUN` | Portuguese | `POR` |

Filter pattern:
```sparql
?expr cdm:expression_uses_language ?lang .
?lang dc:identifier "POL" .
```

---

## Query Patterns

### 1. Search by document type
```sparql
PREFIX cdm: <http://publications.europa.eu/ontology/cdm#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>

SELECT DISTINCT ?work ?celex ?date
WHERE {
  ?work cdm:work_has_resource-type
        <http://publications.europa.eu/resource/authority/resource-type/REG> .
  OPTIONAL { ?work cdm:resource_legal_id_celex ?celex }
  OPTIONAL { ?work cdm:work_date_document ?date }
  FILTER NOT EXISTS { ?work cdm:do_not_index "true"^^xsd:boolean }
}
ORDER BY DESC(?date)
LIMIT 20
```

### 2. Search by CELEX number
```sparql
PREFIX cdm: <http://publications.europa.eu/ontology/cdm#>
PREFIX dc:  <http://purl.org/dc/elements/1.1/>

SELECT DISTINCT ?work ?celex ?title
WHERE {
  ?work cdm:resource_legal_id_celex ?celex .
  FILTER(STR(?celex) = "32016R0679")  -- ⚠️ always use FILTER(STR()) — direct literal matching fails
  OPTIONAL {
    ?expr cdm:expression_belongs_to_work ?work ;
          cdm:expression_uses_language ?lang .
    ?lang dc:identifier "ENG" .
    ?expr cdm:expression_title ?title .
  }
}
```

### 3. Search by date range
```sparql
PREFIX cdm: <http://publications.europa.eu/ontology/cdm#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>

SELECT DISTINCT ?work ?celex ?date
WHERE {
  ?work cdm:work_has_resource-type
        <http://publications.europa.eu/resource/authority/resource-type/DIR> .
  ?work cdm:work_date_document ?date .
  OPTIONAL { ?work cdm:resource_legal_id_celex ?celex }
  FILTER (?date >= "2020-01-01"^^xsd:date AND ?date <= "2023-12-31"^^xsd:date)
  FILTER NOT EXISTS { ?work cdm:do_not_index "true"^^xsd:boolean }
}
ORDER BY DESC(?date)
LIMIT 50
```

### 4. Get downloadable file URLs (full Work→Item chain)
```sparql
PREFIX cdm: <http://publications.europa.eu/ontology/cdm#>
PREFIX dc:  <http://purl.org/dc/elements/1.1/>

SELECT DISTINCT ?celex ?fmt ?item
WHERE {
  ?work cdm:resource_legal_id_celex ?celex .
  FILTER(STR(?celex) IN ("32016R0679", "32018R1725"))
  ?expr cdm:expression_belongs_to_work ?work ;
        cdm:expression_uses_language ?lang .
  ?lang dc:identifier "POL" .
  ?manif cdm:manifestation_manifests_expression ?expr ;
         cdm:manifestation_type ?fmt .
  FILTER(STR(?fmt) = "pdfa2a")  -- ⚠️ "pdfa1a" does not exist; use "pdfa2a", "fmx4", or "xhtml"
  ?item cdm:item_belongs_to_manifestation ?manif .
}
```

### 5. CJEU case law by date range
```sparql
PREFIX cdm: <http://publications.europa.eu/ontology/cdm#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>

SELECT DISTINCT ?work ?celex ?date
WHERE {
  ?work cdm:work_has_resource-type
        <http://publications.europa.eu/resource/authority/resource-type/JUDG> .
  ?work cdm:work_date_document ?date .
  OPTIONAL { ?work cdm:resource_legal_id_celex ?celex }
  FILTER (?date >= "2023-01-01"^^xsd:date)
  FILTER NOT EXISTS { ?work cdm:do_not_index "true"^^xsd:boolean }
}
ORDER BY DESC(?date)
LIMIT 20
```

### 6. In-force legislation only
```sparql
PREFIX cdm: <http://publications.europa.eu/ontology/cdm#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>

SELECT DISTINCT ?work ?celex ?date
WHERE {
  ?work cdm:work_has_resource-type
        <http://publications.europa.eu/resource/authority/resource-type/REG> .
  ?work cdm:resource_legal_in-force "true"^^xsd:boolean .
  OPTIONAL { ?work cdm:resource_legal_id_celex ?celex }
  OPTIONAL { ?work cdm:work_date_document ?date }
  FILTER NOT EXISTS { ?work cdm:do_not_index "true"^^xsd:boolean }
}
ORDER BY DESC(?date)
LIMIT 20
```

---

## How to Execute Queries

### Preferred method: `bash_tool` + Python (urllib)

```python
import urllib.parse, urllib.request, json

def sparql(query):
    encoded = urllib.parse.quote(query)
    url = (
        "https://publications.europa.eu/webapi/rdf/sparql"
        f"?query={encoded}&format=application%2Fsparql-results%2Bjson&timeout=30000"
    )
    with urllib.request.urlopen(url, timeout=35) as r:
        return json.loads(r.read())["results"]["bindings"]

results = sparql("""
PREFIX cdm: <http://publications.europa.eu/ontology/cdm#>
SELECT DISTINCT ?work ?celex WHERE {
  ?work cdm:resource_legal_id_celex ?celex .
  FILTER(STR(?celex) = "32016R0679")
}
""")
for r in results:
    print(r["celex"]["value"])
```

If SSL certificate errors occur (transient), disable verification:
```python
import ssl
ctx = ssl.create_default_context()
ctx.check_hostname = False
ctx.verify_mode = ssl.CERT_NONE
# pass context=ctx to urlopen
```

### JSON Response Structure

```json
{
  "results": {
    "bindings": [
      {
        "work":  { "type": "uri",     "value": "http://publications.europa.eu/resource/cellar/..." },
        "celex": { "type": "literal", "value": "32016R0679" },
        "date":  { "type": "literal", "value": "2016-04-27" }
      }
    ]
  }
}
```

Access results: `data["results"]["bindings"]` — list of dicts, each key maps to `{type, value}`.

### Fetching Document Content from Cellar

Once you have an item URL from SPARQL, fetch the full document text using `curl` in `bash_tool`:

```bash
curl -s -L "<item_url>" -H "Accept: text/html" \
  | python3 -c "
import sys
from html.parser import HTMLParser

class TextExtractor(HTMLParser):
    def __init__(self):
        super().__init__()
        self.text = []
        self.skip = False
    def handle_starttag(self, tag, attrs):
        if tag in ('script', 'style', 'nav', 'header', 'footer'):
            self.skip = True
    def handle_endtag(self, tag):
        if tag in ('script', 'style', 'nav', 'header', 'footer'):
            self.skip = False
    def handle_data(self, data):
        if not self.skip and data.strip():
            self.text.append(data.strip())

p = TextExtractor()
p.feed(sys.stdin.read())
print('\n'.join(p.text)[:20000])
"
```

> ⚠️ **Do NOT use `web_fetch` for Cellar URLs returned by SPARQL.** Cellar item URLs from `bash_tool` SPARQL queries will be rejected by `web_fetch`. Always use `curl` in `bash_tool` instead.

---

## REST API — Direct File Download

Cellar also offers a simpler REST interface (no SPARQL needed):

```
http://publications.europa.eu/resource/cellar/{uuid}.{lang_code}.{fmt_code}/DOC_1
```

With query parameters:
```
http://publications.europa.eu/resource/cellar/{uuid}?language=POL&format=pdfa2a
```

The Cellar UUID comes from the `?work` URI returned by SPARQL queries.

### Other Cellar APIs

| API | URL |
|-----|-----|
| RSS/Atom feeds | https://op.europa.eu/en/web/cellar/cellar-data/rss-and-atom-feeds |
| Linked Data wizard (GUI) | https://op.europa.eu/en/linked-data |

---

## Citations — Always Provide Verifiable Links

Every answer based on fetched document content **must** include source links.

### Mandatory Citation Elements

1. **EUR-Lex link** (canonical, stable):
   ```
   https://eur-lex.europa.eu/legal-content/PL/TXT/?uri=celex:{CELEX}
   ```

2. **Direct Cellar download link** (from SPARQL `?item`):
   ```
   http://publications.europa.eu/resource/cellar/{uuid}.{lang_code}.{fmt_code}/DOC_1
   ```

3. **Article anchor** (when citing a specific article): prefer linking to the full document and referencing the article number explicitly (e.g. "Art. 30 ust. 2 lit. e)") — EUR-Lex anchors are not always stable.

### Citation Format in Responses

```
**Źródło:** DORA — Rozporządzenie (UE) 2022/2554
- EUR-Lex (PL): https://eur-lex.europa.eu/legal-content/PL/TXT/?uri=celex:32022R2554
- Pobierz plik (PL, xhtml): http://publications.europa.eu/resource/cellar/0caf473a-85bd-11ed-9887-01aa75ed71a1.0018.03/DOC_1
```

---

## Workflow

1. **Identify intent** — document type, CELEX, date range, language, in-force status, EuroVoc topic?
2. **Select query pattern** — use or combine the patterns above
3. **Execute via `bash_tool`** — run SPARQL with Python/urllib, parse JSON bindings
4. **Parse and present** — display as a readable table with CELEX numbers and dates
5. **Fetch content if needed** — use `curl` in `bash_tool` with the item URL from SPARQL
6. **Always cite sources** — include EUR-Lex and Cellar links for every result
7. **Offer next steps** — change language, expand date range, get file download URLs, etc.

---

## Rules & Gotchas

| Rule | Detail |
|------|--------|
| Always `SELECT DISTINCT` | Avoids duplicate triples |
| Always add `LIMIT` | Start with 20; unbound queries time out |
| Always filter `do_not_index` | `FILTER NOT EXISTS { ?work cdm:do_not_index "true"^^xsd:boolean }` |
| Use `FILTER(STR(?var) = "value")` | Direct literal matching silently returns 0 results |
| Language codes are ISO 639-3 | `POL`, `ENG`, `DEU` — **not** `pl`, `en`, `de` |
| No `pdfa1a` format | Use `pdfa2a`, `fmx4`, or `xhtml` |
| No `COM_PROP_REG` / `COM_PROP_DIR` | Use `PROP_REG`, `PROP_DIR`, `PROP_DEC` |
| EuroVoc not on JUDG | For thematic search use REG, DIR, DEC, or omit type filter |
| OPTIONAL + FILTER scoping | Never put `?date` in `OPTIONAL` then filter it — keep it as a required triple when filtering |
| Endpoint covers metadata only | For full-text search, use the EUR-Lex search UI |
| CELEX format | `3YYYYTNNNN` for legislative acts (R=Regulation, L=Directive, D=Decision) |
