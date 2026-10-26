<p align="center"><a href="https://apify.com/DevilScrapes/france-sirene-companies-scraper"><img src="assets/banner.svg" alt="France SIRENE Company Registry Scraper — JavaScript example" width="100%"></a></p>

# SIRENE API Scraper for JavaScript and Node.js: French company lookups as JSON

<p align="center">
<a href="https://apify.com/DevilScrapes/france-sirene-companies-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="JavaScript" src="https://img.shields.io/badge/Node.js-20%2B-339933?logo=nodedotjs&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/sirene-api-scraper-javascript/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/sirene-api-scraper-javascript/actions/workflows/ci.yml/badge.svg"></a>
</p>

Use this repository to query France's company register from Node.js through the **[France SIRENE Company Registry Scraper](https://apify.com/DevilScrapes/france-sirene-companies-scraper)**. The module sends a SIREN, a name fragment or a set of filters, such as an activity code plus a department, and receives one flat object per company with identifiers, status, size class, sector labels, VAT number and head-office address. Paging, retries and unpacking of the registry's nested JSON happen in the Actor on Apify, so your onboarding service, Lambda function or CRM worker deals only with plain objects. If your stack already speaks REST and JSON, there is nothing French about the integration except the data.

> **Actor:** [France SIRENE Company Registry Scraper](https://apify.com/DevilScrapes/france-sirene-companies-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in Python:** [sirene-api-scraper-python](https://github.com/devilscrapes/sirene-api-scraper-python)

## 🔥 What you get

- **One row per legal unit**: each company is keyed by its nine-digit `siren`, with the head-office SIRET, address, postal code, commune and department lifted out of the nested registry block onto the row itself.
- **The registry's own filters**: free text that matches a name, SIREN or SIRET, a NAF/APE code in `NN.NNL` form, an INSEE department or commune code, a workforce band and active or ceased status, all combined server-side.
- **Codes with readable labels**: the one-letter NACE section arrives with its English name, and the INSEE size class (`GE`, `ETI`, `PME`, `TPE`) comes with a decoded label.
- **Compliance-friendly columns**: creation and closure dates, the year behind the headcount band, total and still-open establishment counts, the first registered VAT number and a link to the company's public directory page.
- **Mistakes caught before the meter starts**: a NAF code in the wrong format, an out-of-range `maxResults` or a form with every filter cleared is refused without the start fee.
- **Straight answers from the API**: rate limits and server errors are retried with exponential backoff; a filter value the registry rejects ends the run with the registry's own error text instead of five pointless retries.
- **Empty is not broken**: filters that match no company still end in a `SUCCEEDED` run, whose status line lists what was searched, and a malformed entry is skipped and counted rather than ending the page.

## 🎯 Use cases

### KYB check during customer onboarding

When a French business signs up, your backend starts a run with the SIREN they entered in `nameQuery` and compares `legal_name`, `is_active` and `postal_code` with the form. Mismatches go to manual review; matches pass, and the `url` to the public directory page is stored as evidence.

### CRM enrichment on record creation

A webhook from HubSpot, Salesforce or Pipedrive enqueues a lookup for each new French account. The worker writes back NAF code, size class, headcount band and creation year, so segmentation does not depend on what a sales rep typed.

### Lead lists routed by territory

Pull a sector per department, group the objects by `department_code`, and push each group to the regional rep who owns that area. The recipe below produces exactly that structure with a few flags attached to each account.

### Counterparty status monitoring

A weekly serverless job looks up every supplier SIREN with status `C`. Any hit means a counterparty has ceased trading, and the job posts its name and `closure_date` to the finance channel before the next invoice run.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/sirene-api-scraper-javascript.git
cd sirene-api-scraper-javascript
npm install
export APIFY_TOKEN=your_token_here
node index.js                       # uses input.json
node index.js my-input.json --out data --max-charge 2
```

The script starts the [France SIRENE Company Registry Scraper](https://apify.com/DevilScrapes/france-sirene-companies-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

This is the complete integration. `call()` waits for the run to finish, and `listItems()` reads the resulting company objects from the default dataset.

```javascript
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: 'YOUR_APIFY_TOKEN' });
const run = await client.actor('DevilScrapes/france-sirene-companies-scraper').call({"nafCode": "62.01Z", "departmentCode": "69", "workforceRange": "12", "administrativeStatus": "A", "maxResults": 10, "proxyConfiguration": {"useApifyProxy": false}});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(items);
```

## 📥 Input

`index.js` sends `input.json` exactly as written. The sample filter asks for up to ten active software companies in the Rhône department within one headcount band, enough to see real objects without a noticeable bill. In a backend you would usually build the input per request, either a single SIREN in `nameQuery` for a lookup or a NAF code with a department for a list.

```json
{
  "nafCode": "62.01Z",
  "departmentCode": "69",
  "workforceRange": "12",
  "administrativeStatus": "A",
  "maxResults": 10,
  "proxyConfiguration": {
    "useApifyProxy": false
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `nameQuery` | string | no | — | — |
| `nafCode` | string | no | — | — |
| `departmentCode` | string | no | — | — |
| `communeCode` | string | no | — | — |
| `workforceRange` | string | no | — | — |
| `administrativeStatus` | string | no | `"A"` | `A`, `C` |
| `maxResults` | integer | no | `100` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": false}` | — |

## 📤 Output

Each object describes one company with its head office folded in. Identifiers (`siren`, `siret_headquarters`, `vat_number`) are strings, establishment counts are integers or `null`, `is_active` is a boolean, and dates are `YYYY-MM-DD` strings. Expect `null` in optional fields like `acronym` and `closure_date`; type them as optional in TypeScript.

| Field | Type | Always present |
|---|---|---|
| `siren` | string or null | no |
| `siret_headquarters` | string or null | no |
| `company_name` | string or null | no |
| `legal_name` | string or null | no |
| `acronym` | string or null | no |
| `legal_form_code` | string or null | no |
| `company_size_category` | string or null | no |
| `company_size_label` | string or null | no |
| `naf_code` | string or null | no |
| `naf_section_code` | string or null | no |
| `naf_section_label` | string or null | no |
| `workforce_range_code` | string or null | no |
| `workforce_range_year` | string or null | no |
| `administrative_status` | string or null | no |
| `is_active` | boolean or null | no |
| `creation_date` | string or null | no |
| `closure_date` | string or null | no |
| `last_updated` | string or null | no |
| `street_address` | string or null | no |
| `postal_code` | string or null | no |
| `city` | string or null | no |
| `department_code` | string or null | no |
| `establishment_count` | integer or null | no |
| `open_establishment_count` | integer or null | no |
| `vat_number` | string or null | no |
| `url` | string or null | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.004 |

Prices as configured on 2026-08-26. The [Actor page](https://apify.com/DevilScrapes/france-sirene-companies-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in JavaScript

The script below needs nothing beyond Node.js. It groups companies by department, attaches simple review flags (ceased, no VAT number, no open establishment) and writes `territories.json`, ready for a routing job or an API response.

```javascript
import { readFile, writeFile } from 'node:fs/promises';

const rows = JSON.parse(await readFile('results.json', 'utf8'));

const territories = {};
for (const row of rows) {
    const key = row.department_code ?? 'unknown';
    territories[key] ??= { department: key, companies: 0, accounts: [] };
    const flags = [];
    if (!row.is_active) flags.push(`ceased ${row.closure_date ?? '(date not given)'}`);
    if (!row.vat_number) flags.push('no VAT number on file');
    if ((row.open_establishment_count ?? 0) === 0) flags.push('no open establishment');
    territories[key].companies += 1;
    territories[key].accounts.push({
        siren: row.siren,
        name: row.company_name ?? row.legal_name,
        city: row.city,
        postalCode: row.postal_code,
        sector: row.naf_section_label,
        registryPage: row.url,
        flags,
    });
}

const payload = Object.values(territories).sort((a, b) => b.companies - a.companies);
await writeFile('territories.json', `${JSON.stringify(payload, null, 2)}\n`);
console.table(payload.map(({ department, companies, accounts }) => ({
    department,
    companies,
    flagged: accounts.filter((a) => a.flags.length).length,
})));
```

## ❓ FAQ

### Why not call recherche-entreprises.api.gouv.fr directly?

You can. The Actor saves you the paging, the retry logic, the flattening of the nested head-office block and the code-to-label mapping, and it returns the same object shape every time. Choose the raw API when you want full control; choose this when you want a stable contract.

### How fast is a single SIREN lookup?

A lookup is one registry page, but every call also starts an Apify run, so plan for seconds rather than milliseconds. For interactive forms, run the check asynchronously and update the account when the webhook arrives.

### What happens when a filter value is invalid?

A malformed NAF code is refused before any charge. A value the registry itself rejects, such as an unknown department code, fails the run with the registry's error message, which your code can surface to the user.

### Can I search ceased and active companies together?

Not in one run; status is either `A` or `C`. Start two runs in parallel with `Promise.all` and merge the arrays by `siren`.

### What does it cost?

A start fee per run plus a fee per company row, both shown under Pricing. Single-SIREN lookups are dominated by the start fee, so batch them into a daily job where your product allows it.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/france-sirene-companies-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [France SIRENE Company Registry Scraper on Apify Store](https://apify.com/DevilScrapes/france-sirene-companies-scraper)
- [Python version of this example](https://github.com/devilscrapes/sirene-api-scraper-python)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Companies House UK Scraper](https://apify.com/DevilScrapes/companies-house-uk-scraper) — the UK register, searchable by SIC code, town and incorporation date.
- [KVK Company Search Scraper](https://apify.com/DevilScrapes/kvk-company-search-scraper) — Dutch Chamber of Commerce records with KVK and branch numbers.
- [GLEIF LEI Scraper — Legal Entity Identifier Records](https://apify.com/DevilScrapes/gleif-lei-records-scraper) — LEI codes for French and foreign entities that trade in financial markets.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/france-sirene-companies-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with INSEE, the French administration or data.gouv.fr.
