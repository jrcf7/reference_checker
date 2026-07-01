# Reference Checker

A browser-based tool that extracts the reference list from a PDF and verifies whether each reference actually exists — checking not just that a DOI resolves, but that the title matches what is claimed in the document.

---

## Purpose

Academic papers, reports, and theses sometimes contain references with errors: wrong DOIs, misattributed titles, broken links, or references that do not exist at all. Manually checking a bibliography of 50–100 entries is tedious and error-prone.

This tool automates that process. Upload a PDF, and it will check every reference in the bibliography against a global academic database, flagging anything suspicious.

---

## How it works

### 1. Extract text from the PDF
The tool reads the PDF directly in your browser — nothing is uploaded to any server. It pulls out all the text, page by page, and tries to locate the bibliography section (looking for headings like "References", "Bibliography", "Works Cited", etc.).

### 2. Split into individual references
Once the bibliography is found, the tool separates it into individual entries. It handles the most common academic formats: numbered lists like `[1]`, `[2]` or `1.`, `2.`, as well as unnumbered author–year styles.

### 3. Look up each reference
For each reference, the tool checks a large public database of academic publications called **CrossRef**, which indexes over 150 million journal articles, books, and conference papers.

- If the reference contains a **DOI** (a persistent identifier like `10.1000/xyz`), that DOI is looked up directly.
- If no DOI is present, the full reference text is used to search the database for the closest match.

### 4. Verify the title — not just the DOI
A DOI resolving to *something* is not enough. The tool compares the title returned by CrossRef with the title written in the reference. Only if they match closely enough is the reference marked as **Verified**.

This catches common errors like a DOI that points to a different paper, or a reference where the title was misquoted.

---

## Result statuses

| Status | Meaning |
|---|---|
| ✅ Verified | DOI found and title matches |
| ⚠️ Title mismatch | DOI found but the title does not match — review manually |
| ❌ Not found | DOI present but not registered in CrossRef, or no match found |
| ℹ️ No DOI | No DOI in the reference; a text search was attempted but no confident match found |

---

## Limitations

- **Scanned PDFs** (images of pages) contain no extractable text and cannot be processed. Run them through an OCR tool first.
- **Paywalled or preprint DOIs** may not be in CrossRef. Other databases (PubMed, arXiv) are not currently checked.
- **Title extraction from the PDF** is heuristic — for unusual reference formats it may extract the wrong fragment, leading to false mismatches. The raw extracted text is shown at the bottom of the page for inspection.
- Verification requires an internet connection (CrossRef API calls).

---

## Deploying on GitHub Pages

1. Fork or push this repository to your GitHub account.
2. Go to **Settings → Pages → Source → Deploy from branch → main / root**.
3. The tool will be live at `https://<your-username>.github.io/<repo-name>/`.

No server, no build step, no dependencies to install.

---

## Technical details

### PDF parsing
Text extraction uses [PDF.js](https://mozilla.github.io/pdf.js/) (v3.11.174), running entirely in the browser. Text items are grouped by their vertical (y) coordinate to reconstruct lines. Soft hyphens at line breaks are removed by detecting a trailing `-` followed by a lowercase continuation.

### Bibliography detection
The tool scans for the *last* occurrence of common section headings (`References`, `Bibliography`, `Works Cited`, `Literature Cited`, `Bibliographie`, `Références`, `Literatur`) and treats everything after it as the bibliography. If no heading is found, the last 35% of the document is used as a fallback.

### Reference splitting
Four strategies are tried in order:
1. **Bracket numbers** — matches `[1]`, `[2]` anywhere in text (not just at line starts), validates they form an ascending sequence starting near 1.
2. **Dot numbers** — matches `1.`, `2.` followed by a capital letter, at line starts or inline.
3. **Blank lines** — splits on empty lines between entries.
4. **Hanging indent** — each line beginning with a capital letter after sufficient accumulated text is treated as a new entry.

### DOI extraction
Uses the regex `10\.\d{4,9}/[^\s,;'")\]>–—·]+`, which covers the full DOI syntax. Trailing punctuation (periods, commas, brackets) is stripped.

### CrossRef API
Lookups use the public [CrossRef REST API](https://api.crossref.org/):
- **DOI lookup**: `GET https://api.crossref.org/works/{doi}`
- **Bibliographic search**: `GET https://api.crossref.org/works?query.bibliographic={text}&rows=5`

No API key is required. Providing your email address in Settings adds you to CrossRef's "polite pool", which gives priority access and better rate limits. The email is sent as a `mailto` query parameter and is never stored.

### Title similarity
Both the extracted title and the CrossRef title are:
1. Lowercased
2. Stripped of punctuation
3. Tokenised by whitespace
4. Filtered to remove stop words (`the`, `of`, `in`, `and`, etc.) and tokens of 2 characters or fewer

The score is a **Jaccard-style word overlap**:

```
score = matching content words / max(words in A, words in B)
```

The default threshold is **60%**. This can be adjusted in Settings (range: 30%–90%).

A score below the threshold with a valid DOI is flagged as ⚠️ **Title mismatch** rather than verified, since this may indicate the DOI points to a different paper.

### Rate limiting
A configurable delay (default: 300 ms) is inserted between API calls to avoid overwhelming CrossRef. For large bibliographies, reducing this below 100 ms is not recommended.
