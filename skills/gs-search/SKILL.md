---
name: gs-search
description: Search Google Scholar for academic papers by keywords. Returns results with title, authors, journal, year, citation count, and full-text links. Use when the user wants to search Google Scholar.
argument-hint: "[search keywords]"
---

# Google Scholar Basic Search

Search Google Scholar for papers using keyword(s). Returns structured result list via DOM scraping.

## Arguments

$ARGUMENTS contains the search keyword(s).

## Steps

### 1. Navigate

Use `mcp__chrome-devtools__navigate_page`:

**Preferred URL** (use `.com.pr` regional domain — more likely to show solvable CAPTCHA rather than hard IP block):
- url: `https://scholar.google.com.pr/scholar?q={URL_ENCODED_KEYWORDS}&hl=en&num=10`

**Fallback** (if `.com.pr` fails):
- `https://scholar.google.com.pk/scholar?q={URL_ENCODED_KEYWORDS}&hl=en&num=10`
- `https://scholar.google.com/scholar?q={URL_ENCODED_KEYWORDS}&hl=en&num=10`

### 2. Extract results (evaluate_script)

Wait for results to load, check for CAPTCHA, then scrape the DOM:

```javascript
async () => {
  // Wait for results or CAPTCHA
  for (let i = 0; i < 20; i++) {
    if (document.querySelector('#gs_res_ccl') || document.querySelector('#gs_captcha_ccl')) break;
    await new Promise(r => setTimeout(r, 500));
  }

  // CAPTCHA check
  if (document.querySelector('#gs_captcha_ccl') || document.body.innerText.includes('unusual traffic')) {
    return { error: 'captcha', message: 'Google Scholar requires CAPTCHA verification. Please complete it in your browser, then tell me to continue.' };
  }

  const items = document.querySelectorAll('#gs_res_ccl .gs_r.gs_or.gs_scl');
  const results = Array.from(items).map((item, i) => {
    const titleEl = item.querySelector('.gs_rt a');
    const meta = item.querySelector('.gs_a')?.textContent || '';
    // Parse "Author1, Author2 - Journal, Year - publisher"
    const parts = meta.split(' - ');
    const authors = parts[0]?.trim() || '';
    const journalYear = parts[1]?.trim() || '';
    const citedByEl = item.querySelector('.gs_fl a[href*="cites"]');
    const relatedEl = item.querySelector('.gs_fl a[href*="related"]');
    const versionsEl = item.querySelector('.gs_fl a[href*="cluster"]');

    return {
      n: i + 1,
      title: titleEl?.textContent?.trim() || item.querySelector('.gs_rt')?.textContent?.trim() || '',
      href: titleEl?.href || '',
      authors,
      journalYear,
      citedBy: citedByEl?.textContent?.match(/\d+/)?.[0] || '0',
      citedByUrl: citedByEl?.href || '',
      dataCid: item.getAttribute('data-cid') || '',
      fullTextUrl: (item.querySelector('.gs_ggs a') || item.querySelector('.gs_or_ggsm a'))?.href || '',
      snippet: item.querySelector('.gs_rs')?.textContent?.trim()?.substring(0, 200) || '',
      relatedUrl: relatedEl?.href || '',
      versionsUrl: versionsEl?.href || '',
      versions: versionsEl?.textContent?.match(/\d+/)?.[0] || ''
    };
  });

  const totalText = document.querySelector('#gs_ab_md')?.textContent?.trim() || '';
  const currentUrl = window.location.href;
  return { total: totalText, resultCount: results.length, currentUrl, results };
}
```

### 3. Report

Present results as a numbered list:

```
Searched Google Scholar for "$ARGUMENTS": {total}

1. {title}
   Authors: {authors} | {journalYear}
   Cited by: {citedBy} | [Full text]({fullTextUrl})
   Data-CID: {dataCid}

2. ...
```

Always show the `dataCid` — it's the unique identifier used for citation export and "cited by" tracking.

If `fullTextUrl` is available, highlight it (means open-access PDF/HTML).

### 4. Follow-up

When the user wants to:
- **See more results**: use `gs-navigate-pages` to go to next page
- **See who cited a paper**: use `gs-cited-by` with the data-cid
- **Export to Zotero**: use `gs-export` with the data-cid(s)

## Blocking & CAPTCHA Handling

### Hard IP Block ("unusual traffic" sorry page)

**Detection**: page redirects to `https://www.google.com/sorry/index?...`, body contains "unusual traffic", and there is NO reCAPTCHA form (no `#gs_captcha_ccl`, no reCAPTCHA iframe). This is an **IP-level hard block** — it cannot be solved in-browser.

**Procedure** (validated against live incident, 2026-05):

1. Try regional domains **one at a time**, stop at the first that returns results:
   - `https://scholar.google.com.pr/scholar?q=...`
   - `https://scholar.google.com.pk/scholar?q=...`
   - `https://scholar.google.com/scholar?q=...`
   - `https://scholar.google.com.hk/scholar?q=...`
2. If a domain shows a **solvable reCAPTCHA** (see below) → ask the user to solve it, wait for confirmation, then retry extraction.
3. If **all four domains** return the hard sorry page with no form:
   - **STOP immediately. Do NOT retry automatically** — repeated attempts make the block worse and longer.
   - Tell the user: "Google Scholar is hard-blocking this network with an 'unusual traffic' sorry page (no CAPTCHA form on any of the four domains). Please wait 10–30 minutes, or switch networks (proxy / mobile hotspot), then tell me to retry."
   - **Fall back to a web-search tool** to complete the content part of the task (titles, authors, journals, years, DOIs are obtainable without Scholar). Explicitly defer citation counts and `data-cid` until Scholar is reachable.
   - Record which domains were attempted and state the block scope in the final report.
4. A single retry is allowed only after **user confirmation** or a **≥10-minute idle gap**.

### CAPTCHA (solvable)

If `#gs_captcha_ccl` exists or a reCAPTCHA form/iframe is present:
1. Tell the user: "Google Scholar requires human verification — please complete the CAPTCHA in your browser."
2. Wait for user confirmation
3. Retry the evaluate_script extraction

## Notes

- This skill uses 2 tool calls: `navigate_page` + `evaluate_script`
- Google Scholar has NO public API — all data extraction is via DOM scraping
- `data-cid` is the primary identifier (cluster ID) — used across all GS skills
- Keep request frequency low to avoid triggering CAPTCHA
- Default `num=10` results per page (max 20)
