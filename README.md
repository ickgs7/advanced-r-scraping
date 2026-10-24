# web scraping r: rvest covers the easy part — here's how to handle JavaScript, anti-bot walls, and real costs

Most tutorials about web scraping with R stop at the same place: install `rvest`, parse a static HTML table, done. That works fine for Wikipedia pages and price lists that load server-side. The part that actually costs you time is everything after — pages that render content with JavaScript, sites behind Cloudflare or Datadome, and the question of what a large scrape job will really cost you. This article walks through the R-side setup, explains where `rvest` alone stops being enough, and shows how to plug a scraping API like ScraperAPI into R when you hit those walls, including its full plan list and some pricing rules most reviews skip.

> **Quick take:** if your targets are static pages, `rvest` alone is free and you can stop reading after the setup section. If you're fighting JavaScript rendering or bot protection, the practical split is: R handles parsing, an API handles fetching. ScraperAPI has a 7-day trial with 5,000 credits (no credit card) and paid plans starting at $49/month — 👉 [claim the 5,000-credit 7-day trial](https://www.scraperapi.com/signup?fp_ref=coupons) and test it against your actual target sites before deciding anything.

## Setting up your R environment without the usual stumbling blocks

The setup order matters more than most guides admit: install R first, then RStudio. RStudio needs R to already exist on your machine, and installing them the other way around produces confusing startup errors that look like broken installations. Grab R from CRAN, keep the default settings, then install the free RStudio Desktop from Posit.

The core package stack for web scraping with R:

- **rvest** — extracts data from HTML using CSS selectors or XPath
- **httr2** — handles HTTP requests, which is what you'll use to call any scraping API
- **xml2** — parses HTML/XML and sits underneath rvest anyway
- **chromote** (optional) — controls a real Chrome browser when a site needs local JavaScript rendering

One older package you'll see in search results: Rcrawler. It can crawl many pages and even has a published paper behind it, but it's outdated and doesn't install on the newest R versions. Skip it.

## The rvest workflow everyone covers (and why it's still worth two minutes)

For static pages, `rvest` is genuinely all you need. The pattern is short: read the page, select elements, pull text or attributes, build a tibble.

r
library(rvest)

page <- read_html("https://example.com/products")
titles <- page |> html_elements("h2.product-title") |> html_text2()
links  <- page |> html_elements("a") |> html_attr("href")


The two functions you'll use constantly: `html_text2()` extracts readable text and trims whitespace for you, and `html_attr()` pulls attributes like `href`, `src`, or `title` off an element. Real-world selectors are rarely this clean — ratings encoded in class names, dates buried in paragraphs needing a `str_extract()` with a regex afterward — but that's parsing work, not fetching work.

If every site you care about behaves this way, congratulations, your total cost is zero dollars. The rest of this article is for the other case.

## Why your scraper "doesn't see the data you see in the browser"

This is the classic failure mode: your script runs without errors and returns an empty data frame, while the same URL in Chrome shows a full page. The cause is almost always JavaScript — the content loads after the initial HTML, or the site requires a request context (headers, cookies, a real browser fingerprint) that `read_html()` doesn't provide.

You have three options at this point:

1. **Local headless browser.** The newer `read_html_live()` (built on chromote) loads the page in headless Chrome, runs its JavaScript, and lets you call `wait_for()` until a selector appears before reading the DOM. RSelenium is the older approach for the same job.
2. **Reverse-engineer the underlying API.** Sometimes the data comes from a JSON endpoint you can call directly with `httr2`. Free, but brittle and not always possible.
3. **Offload fetching to a scraping API.** The API's servers render JavaScript, rotate residential IPs, solve CAPTCHAs, and retry — then hand your R code the finished HTML for `rvest` to parse.

Option 1 works and is free, but headless Chrome on your machine consumes RAM, breaks on heavy bot protection, and gives you zero IP rotation. Option 3 trades money for not maintaining any of that infrastructure yourself.

## Calling ScraperAPI from R: no official SDK, and that's fine

Here's something the ScraperAPI docs page doesn't shout about: its official SDKs cover Python, Node.js, PHP, Ruby, and Java — **there is no R SDK**. R users connect through the two lower-level methods instead, both fully documented.

**Method 1: the proxy port.** ScraperAPI exposes a standard HTTP proxy at `http://scraperapi:APIKEY@proxy-server.scraperapi.com:8001`. This is the cleanest fit for R, because `httr2` already speaks proxy:

r
library(httr2)
library(rvest)

page <- request("https://target-site.com/page") |>
  req_proxy("http://proxy-server.scraperapi.com:8001",
            user = "scraperapi",
            password = Sys.getenv("SCRAPERAPI_KEY")) |>
  req_timeout(70) |>
  req_perform() |>
  resp_body_html()


Your R code then parses `page` with `rvest` exactly as before. The fetching logic — rendering, retries, IP rotation — lives entirely on the API side.

**Method 2: the API endpoint.** Send the target URL as a parameter to `https://api.scraperapi.com` along with your API key, and it returns the HTML. Extra features are enabled by adding GET parameters, the same ones the SDKs use: `render` for JavaScript rendering, `premium` for premium pools, `country_code` for geotargeting. They combine in a single request.

One number worth remembering from the docs: set a **70-second timeout** in your application regardless of how you call the service. It's the official recommendation for the best success rates, especially on hard-to-scrape domains. There's also a 50MB request size limit, which matters if you're pulling large files rather than HTML.

This split — R for parsing, API for fetching — means your existing `rvest` code barely changes when a target gets harder.

## ScraperAPI plans: the full list, current pricing

All prices below are from the official pricing page, in USD per month. Annual billing takes 10% off. Every plan includes JavaScript rendering, premium residential and mobile IPs, CAPTCHA and anti-bot handling, JSON auto-parsing, custom headers and sessions, automatic retries, unlimited bandwidth, DataPipeline access, and pay-as-you-go support.

| Plan | Monthly | Annual (per mo) | API credits | Threads | Get it |
| --- | --- | --- | --- | --- | --- |
| Free plan | $0 | — | 1,000 | 5 | [ Start free](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | $49 | $44.10 | 100,000 | 20 | [ Try Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [ Try Startup](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [ Try Business](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | $475 | $427.50 | 5,000,000 | 200 | [ Try Scaling](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [ Try Professional](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [ Try Advanced](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom | — | 22,000,000+ | 500+ | [ Contact sales](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

A few differences don't fit in the table:

- **Geotargeting:** Hobby and Startup can only target US and EU locations. Country-level targeting outside those regions starts at Business ($299).
- **Analytics panel:** the lower two plans only show the last 30 days of usage data; Business and up get unlimited history.
- **Trial:** every plan tier offers a 7-day trial with 5,000 credits, no credit card required. The free plan (1,000 credits, 5 concurrent connections) exists separately and doesn't expire — if you need more test credits, support can grant additional ones for testing purposes.

The trial is the honest way to evaluate this. 👉 [See all plans and start the 7-day trial](https://www.scraperapi.com/pricing/?fp_ref=coupons), then point it at the sites you actually scrape.

## What a request actually costs: the credit math nobody explains

"100,000 credits per month" sounds like 100,000 pages. It isn't, and this is the biggest gap in most reviews of scraping APIs. ScraperAPI prices requests by domain difficulty:

- A standard page: **1 credit**
- Amazon: **5 credits**
- Google and Bing (including their subdomains): **25 credits**
- LinkedIn: **30 credits**
- Sites behind Cloudflare, Datadome, or PerimeterX: **+10 credits per request** when the bypass succeeds

So Hobby's 100,000 credits is 100,000 ordinary pages, but only about 4,000 Google search pages, or roughly 6,600 Amazon product pages. Turn on `render=true` and you're paying the JavaScript-rendering cost on top of the domain cost. Before you pick a plan, do this: open the **Domain Cost Estimator** in the dashboard, paste in your actual target URLs, and get the exact per-request price. You can also set a `max_cost` per scrape so a single request can't burn more credits than you intend — a useful guardrail when experimenting with unknown sites.

Run that math before comparing plans, and the plan tiers start making obvious sense. Someone scraping 50,000 standard pages monthly is fine on Hobby at $49. Someone pulling 10,000 Google SERPs monthly is at 250,000 credits before any rendering — Startup territory at minimum.

## The fine-print rules that change how you use it

Three behaviors are documented but rarely mentioned in roundups:

1. **Concurrency overages return 429, not errors you pay for.** Exceed your plan's thread limit and the request is simply turned away — "no extra charge for going over, the request is simply turned away rather than queued." Your R code should catch the 429 and retry with backoff. Hobby's 20 threads is a real ceiling for parallel `httr2` requests; don't fork 100 workers against it.
2. **Running out of credits pauses scraping by default.** Work continues at renewal. To keep going uninterrupted, Hobby through Professional users can enable **auto-upgrade** (automatically moves you to the next plan when credits run out, opt-in), while Scaling and higher can use **pay-as-you-go** at a fixed per-credit rate up to a monthly cap you set. Annual plans can also renew early for a fresh batch of credits.
3. **The trial is genuinely a trial of everything.** JavaScript rendering, structured data endpoints, the works — 5,000 credits is enough to test against real protected sites, not just toy pages.

None of these are dealbreakers. They're just the difference between a budget that holds up and one that gets torched by a week of Google scraping you assumed cost 1 credit per page.

## How this setup compares with the alternatives

A quick, link-free look at the other routes R users take:

- **ScrapingBee** operates on the same model — an API returns rendered HTML, R parses it. Its free tier gives 1,000 credits, and its own R tutorial confirms that's enough to run their sample code but not much more. Comparable positioning; compare per-credit pricing for your target domains before committing either way.
- **Local chromote / RSelenium** cost nothing and keep data on your machine. You pay in maintenance: managing proxies yourself, solving CAPTCHAs yourself, and eating the RAM of a headless Chrome per session. Reasonable for a handful of sites; painful at scale.
- **Plain rvest + httr2** remains the right answer for static or API-backed targets. Paying for a scraping service to fetch a page that `read_html()` already handles is waste, plain and simple.

## Who shouldn't pay for any of this

A scraping API solves fetching problems. If you don't have fetching problems, it's a subscription for nothing. Skip paid plans entirely if:

- your targets are static HTML or expose clean JSON endpoints;
- your monthly volume is under a few thousand standard pages — the free plan's 1,000 credits plus occasional testing credits from support covers that;
- you're doing one-off academic scrapes where a slow, polite `rvest` loop with `Sys.sleep()` between requests is perfectly acceptable.

Pay when at least one of these is true: content loads via JavaScript, the site sits behind known bot protection, or you need country-level targeting and hundreds of concurrent requests on a schedule.

## FAQ

**Can rvest scrape JavaScript-rendered pages?**
Not with `read_html()` — it only sees the initial HTML. Use `read_html_live()` from the chromote-based workflow to render locally, or send the URL through a rendering-capable API and parse the returned HTML with `rvest` as usual.

**How do I use ScraperAPI with R if there's no R SDK?**
Through the proxy port (`scraperapi:APIKEY@proxy-server.scraperapi.com:8001`) with `httr2`'s proxy support, or by calling `https://api.scraperapi.com` with your API key and target URL as parameters. Both are first-class methods in the official docs; the SDK is just a convenience wrapper.

**Is the free plan enough for a real project?**
For testing and small one-off scrapes, yes — 1,000 credits with 5 concurrent connections, no expiry mentioned on the official page. For recurring jobs, one protected-domain request can cost 10+ credits, and the math stops working quickly.

**What happens if I blow past my plan's limits?**
Concurrency overages get a 429 response with no charge — just retry within your limit. Credit exhaustion pauses scraping until renewal unless you've enabled auto-upgrade or pay-as-you-go.

## Picking a plan, concretely

Here's the decision path, using the verified numbers:

- **Static targets, under ~1,000 pages/month:** free plan. Don't pay.
- **JS-heavy targets you scrape a few hundred times a month:** the 7-day, 5,000-credit trial will tell you your real per-request cost — start there before choosing anything.
- **Standard pages at volume (tens of thousands monthly):** Hobby at $49, or $44.10 on annual billing.
- **Google/Bing/Amazon work or country-level targeting outside US/EU:** Business at $299 is the floor for national geotargeting; use the Domain Cost Estimator to check whether your credit math actually clears that tier.

One last check before checkout: confirm your concurrency needs against the thread limits (20 on Hobby is easy to outgrow with parallel R requests), and set `max_cost` on your first requests against unknown domains. If the numbers add up, 👉 [start with the 5,000-credit trial and upgrade only when the math says so](https://www.scraperapi.com/?fp_ref=coupons).
