# Domain Skill: alabamapublicnotices.com
**Learned:** 2026-04-19 (Session 2)
**Difficulty:** Hard — ASP.NET WebForms, AJAX UpdatePanels, reCAPTCHA on detail pages

## What this site does
Alabama Press Association's public notices aggregator. Newspapers upload notices daily.
Covers: foreclosures, zoning hearings, ordinances, bids, public hearings, contractor notices.
Data range: 8/25/2024 to today (current search). Older via Archive Search.

## Extraction approach: 2-step HTTP POST chain (not browser clicks)

Browser-level form interaction fails due to ASP.NET UpdatePanel complexity.
Use `urllib` HTTP POST with shared `CookieJar` inside harness exec.

```python
import urllib.request, urllib.parse, re as re2, http.cookiejar

jar = http.cookiejar.CookieJar()
opener2 = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(jar))

# STEP 1: GET initial page → extract ViewState
r0 = opener2.open(urllib.request.Request(
    "https://www.alabamapublicnotices.com/default.aspx",
    headers={'User-Agent': 'Mozilla/5.0'}
))
html1 = r0.read().decode('utf-8', errors='replace')
s1 = r0.geturl()
vs1 = re2.search(r'id="__VIEWSTATE"[^>]+value="([^"]*)"', html1).group(1)
vsg1 = re2.search(r'id="__VIEWSTATEGENERATOR"[^>]+value="([^"]*)"', html1).group(1)

# STEP 2: POST to select county (EVENTTARGET = county checkbox)
# Madison = index 44; other counties: find index in alphabetical list
county_post = urllib.parse.urlencode({
    '__EVENTTARGET': 'ctl00$ContentPlaceHolder1$as1$lstCounty$44',
    '__EVENTARGUMENT': '', '__LASTFOCUS': '',
    '__VIEWSTATE': vs1, '__VIEWSTATEGENERATOR': vsg1,
    'ctl00$ContentPlaceHolder1$as1$ddlPopularSearches': '0',
    'ctl00$ContentPlaceHolder1$as1$txtSearch': '',
    'ctl00$ContentPlaceHolder1$as1$rdoSearchType': 'rdoAllWords',
    'ctl00$ContentPlaceHolder1$as1$txtExclude': '',
    'ctl00$ContentPlaceHolder1$as1$dateRange': 'rbLastNumDays',
    'ctl00$ContentPlaceHolder1$as1$txtLastNumDays': '60',  # last N days
    'ctl00$ContentPlaceHolder1$as1$txtLastNumWeeks': '52',
    'ctl00$ContentPlaceHolder1$as1$txtLastNumMonths': '12',
    'ctl00$ContentPlaceHolder1$as1$txtDateFrom': '2/19/2026',
    'ctl00$ContentPlaceHolder1$as1$txtDateTo': '4/19/2026',
    'ctl00$ContentPlaceHolder1$as1$hdnCountyScrollPosition': '-1',
    'ctl00$ContentPlaceHolder1$as1$lstCounty$44': 'on',  # the checked county
}).encode()
r1 = opener2.open(urllib.request.Request(s1, data=county_post,
    headers={'User-Agent': 'Mozilla/5.0', 'Content-Type': 'application/x-www-form-urlencoded', 'Referer': s1}))
html2 = r1.read().decode('utf-8', errors='replace')
s2 = r1.geturl()
vs2 = re2.search(r'id="__VIEWSTATE"[^>]+value="([^"]*)"', html2).group(1)
vsg2 = re2.search(r'id="__VIEWSTATEGENERATOR"[^>]+value="([^"]*)"', html2).group(1)

# STEP 3: POST to run search (EVENTTARGET = '' + btnGo)
search_post = urllib.parse.urlencode({
    '__EVENTTARGET': '', '__EVENTARGUMENT': '', '__LASTFOCUS': '',
    '__VIEWSTATE': vs2, '__VIEWSTATEGENERATOR': vsg2,
    'ctl00$ContentPlaceHolder1$as1$ddlPopularSearches': '0',
    'ctl00$ContentPlaceHolder1$as1$txtSearch': 'zoning',  # search keyword
    'ctl00$ContentPlaceHolder1$as1$rdoSearchType': 'rdoAllWords',
    'ctl00$ContentPlaceHolder1$as1$txtExclude': '',
    'ctl00$ContentPlaceHolder1$as1$dateRange': 'rbLastNumDays',
    'ctl00$ContentPlaceHolder1$as1$txtLastNumDays': '60',
    'ctl00$ContentPlaceHolder1$as1$txtLastNumWeeks': '52',
    'ctl00$ContentPlaceHolder1$as1$txtLastNumMonths': '12',
    'ctl00$ContentPlaceHolder1$as1$txtDateFrom': '2/19/2026',
    'ctl00$ContentPlaceHolder1$as1$txtDateTo': '4/19/2026',
    'ctl00$ContentPlaceHolder1$as1$hdnCountyScrollPosition': '-1',
    'ctl00$ContentPlaceHolder1$as1$lstCounty$44': 'on',
    'ctl00$ContentPlaceHolder1$as1$btnGo': '',
}).encode()
r2 = opener2.open(urllib.request.Request(s2, data=search_post,
    headers={'User-Agent': 'Mozilla/5.0', 'Content-Type': 'application/x-www-form-urlencoded', 'Referer': s2}))
html3 = r2.read().decode('utf-8', errors='replace')
search_url = r2.geturl()

# Results are now in html3 — redirected to Search.aspx
```

## Parsing results

```python
# Extract notice IDs (needed to construct detail URLs)
notice_ids = re2.findall(r'ID=(\d+)', html3)

# Extract session ID for detail URLs
sid = re2.search(r'Details\.aspx\?SID=([^&"\']+)', html3).group(1)

# Extract notice blocks: (ID, county, preview_text)
blocks = re2.findall(
    r'ID=(\d+)&#39;.*?County: (Madison)[^<]*</div>.*?<td colspan="3"[^>]*>(.*?)</td>',
    html3, re2.DOTALL | re2.IGNORECASE
)
for nid, county, raw in blocks:
    text = re2.sub(r'<[^>]+>', '', raw)
    text = re2.sub(r'\s+', ' ', text).strip()
    text = text.replace(" click 'view' to open the full text.", "").strip()
    print(f"ID={nid} | {county} | {text[:400]}")

# Detail URL (CAPTCHA-gated — for reference only)
base = search_url.split('/Search.aspx')[0]
detail_url = f"{base}/Details.aspx?SID={sid}&ID={nid}"
```

## County index map (key counties)
| County | Index |
|--------|-------|
| Madison | 44 |
| Limestone | 41 |
| Marshall | 47 |
| Morgan | 51 |
| Jackson | 35 |
| DeKalb | 24 |

Form field name: `ctl00$ContentPlaceHolder1$as1$lstCounty${INDEX}` = 'on'

## Search keywords for Alabama IQ signals
| Keyword | Notice types returned |
|---------|----------------------|
| `zoning` | Zoning hearings, ordinances, adjustments |
| `rezoning` | Rezoning requests |
| `variance` | Development variances |
| `commercial` | Commercial development notices |
| `development` | General development activity |
| `ordinance` | City/county ordinances |

## Popular search categories (no keyword needed)
| Name | Value |
|------|-------|
| Public Hearings | 25 |
| Notice to Contractors | 28 |
| Request for Proposals | 26 |
| Ordinances | 24 |
| Bids | 27 |

To use: POST Step 2 with `ddlPopularSearches = '25'` as EVENTTARGET.

## Known limitations
1. **reCAPTCHA on detail pages** — full notice text requires CAPTCHA solve or SmartSearch subscription
2. **js() helper returns None** — use `cdp("Runtime.evaluate", ...)` directly instead
3. **No function definitions in harness exec** — all code must be flat/inline (Python exec() scope issue)
4. **Session ID in URL** — every fresh GET creates a new session; carry cookies between requests
5. **Date range** — site only stores from 8/25/2024 to today; older notices need Archive Search endpoint

## Archive Search
URL: `/Archive/ArchiveSearch.aspx` — different form, same pattern.
Not tested in Session 2.
