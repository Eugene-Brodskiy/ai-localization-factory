# AI Localization Factory

An n8n workflow that turns a product spreadsheet into SEO-ready product content in 5 languages, with glossary rules, automatic checks, an AI quality review and import files for Magento 2 and Shopify.

```
 Google Sheet (products, EN) ──> Enrich EN ──> Localize x5 ──> Validate ──> QA Judge ──> Results sheet
            Glossary sheet ─────────────────────────┘          (code)      (AI + back-       │
                                                                             translation)    ├──> Magento 2 import CSV
                                                                                             └──> Shopify translations CSV
```

## Why

I spent 2 years running content for a 10,000+ SKU garden catalog in 12 languages. Plain machine translation kept breaking the same things:

- cultivar and brand names got translated ('Gloria Dei' is a rose, not a phrase)
- store terminology drifted between products and languages
- keywords were translated, not localized: people in Germany search differently
- meta titles and Amazon fields silently went over character limits
- nobody saw the errors until a customer did

This workflow automates the part that is repetitive and checks the part that usually goes wrong.

## What it does

1. **Reads products** with `status = new` from a Google Sheet (name, specs, target keywords).
2. **Enrich EN:** an LLM writes English SEO content using only facts from the specs: SEO title, meta title and description, alt text, 3 bullets, HTML description.
3. **Localize:** the content is adapted, not translated word for word, into German, French, Polish, Spanish and Italian. Glossary rules come from a separate sheet: names that must never be translated and terms with a fixed translation.
4. **Validate (code, no AI):** character limits, glossary terms, cultivar names in quotes, allowed HTML tags.
5. **QA Judge:** a second LLM call back-translates the text into English, compares it with the source and scores it from 1 to 5.
6. **Decide:** a row is `done` only if all checks pass and the score is 4 or higher. Everything else is `needs_review` for a human.
7. **Export:** results go to a sheet, plus a Magento 2 import CSV and a Shopify translations CSV.

## Results on the demo catalog

Demo data: 20 fictional garden products (plants, seeds, tools, planters), 5 languages, 100 localized product pages.

| Metric | Value |
| --- | --- |
| Product pages generated | 100 (20 products × 5 languages) |
| Passed all checks automatically | 78 (78%) |
| Sent to human review | 22 (12 low AI review score, 10 failed code checks) |
| Average AI review score | 4.38 / 5 |
| Review rate by language | DE 3/20 · ES 3/20 · FR 5/20 · PL 5/20 · IT 6/20 |
| Run time | ~2 hours over 4 batches, mostly deliberate pauses to stay within free-tier rate limits |
| API cost | $0 (Google Gemini Flash-Lite, free tier) |

**What the review queue caught:** the glossary rule "never translate 'Genovese'" made the model keep the whole English phrase "Genovese Basil Seeds" in German and Italian text. Code checks passed it, the AI reviewer scored it 2/5. A rule meant to protect a name ended up blocking the translation, which is exactly the kind of case a human should see.

## Example output

All files in `examples/` come from the real demo run:

- `results.csv`: all 100 localized pages with QA scores, issues and back-translations
- `magento-import.csv`: 78 rows that passed, one per product per store view
- `shopify-translations.csv`: the same 78 pages as 312 Shopify translation rows (4 fields each)

## Quality controls

| Check | How |
| --- | --- |
| Facts | Prompts allow only facts from the specs; QA Judge flags anything added |
| Glossary | Code checks "never translate" terms and fixed translations, with simple stemming for inflected languages |
| Cultivar names | Code checks they keep their single quotes |
| Length limits | SEO title 70, meta title 60, meta description 155, alt text 125, bullets 90 characters |
| HTML | Only `<p>`, `<ul>`, `<li>`; inline tags are unwrapped, broken tags are flagged |
| Meaning | QA Judge back-translates and compares with the English source |

## Output formats

- **Magento 2:** one row per product per store view (`sku`, `store_view_code`, `name`, `description`, `short_description`, `meta_title`, `meta_description`, `meta_keywords`). `url_key` is left out on purpose to avoid URL rewrite conflicts. Store view codes must match your Magento setup.
- **Shopify:** Shopify imports translations by filling its own export (Settings > Languages > Export). The demo file has the right structure, with `SKU:...` as a placeholder for the product ID. In a real store the workflow fills your export by Identification + Field + Locale.

## Run it yourself

1. Import `workflow/AI-Localization-Factory.json` into n8n.
2. Create credentials: Google Sheets OAuth2 and Google Gemini (or swap the model nodes for OpenAI) and select them in the nodes.
3. Import `data/products.csv`, `data/glossary.csv` and `data/results-header.csv` into one Google Sheet as tabs `products`, `glossary` and `Results`. Replace `YOUR_GOOGLE_SHEET_ID` in the Google Sheets nodes with your sheet.
4. Set **Limit** to 5 for a test run, then Execute workflow. Each run takes the next products with `status = new`.

## Limitations

- Translations were checked by code and by an AI reviewer, **not by native speakers**. If you speak German, French, Polish, Spanish or Italian, feedback is welcome.
- `local_keywords` are AI suggestions, not search volume data. Validate them in a keyword tool before relying on them.
- The free Gemini tier allowed only 20 requests per day per model during testing, which is not enough for a full run (about 11 requests per product).
- Shopify needs the store's own translation export to produce an importable file.

## Stack

n8n · Google Gemini Flash-Lite · Google Sheets · JavaScript (n8n Code nodes)

## Author

Yevhenii Brodskyi, e-commerce SEO and content lead building AI automation. [LinkedIn](https://www.linkedin.com/in/yevhenii-brodskyi)
