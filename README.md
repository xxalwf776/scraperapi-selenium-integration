# ScraperAPI Selenium Integration: How to Scrape Dynamic Websites Without Getting Blocked — Setup Guide, Proxy Config, Code Examples & Full Plan Breakdown (With Free Trial)

So you've built a Selenium scraper. It works fine on your machine, against your test sites, with your home IP. Then you point it at a real target — Amazon, Google, some heavily guarded e-commerce site — and within minutes you're staring at a CAPTCHA page, a 403, or that uniquely demoralizing blank screen that just means "we know what you are."

Welcome to the actual problem with web scraping at scale. Selenium handles the browser automation part beautifully. It doesn't handle proxy rotation, IP bans, CAPTCHA solving, or anti-bot fingerprinting. That's where most scraping projects quietly die.

This guide walks through exactly how to use **ScraperAPI with Selenium** — the right way, not the common wrong way — plus everything you need to know about plans, pricing, and whether it's actually worth it for your use case.

---

## Why Selenium Alone Isn't Enough for Serious Scraping

Selenium is genuinely great at what it was built for: controlling a real browser, executing JavaScript, handling dynamic page loads, simulating user interactions like clicks and form fills. If you need to scrape a single-page app that renders its data client-side, Selenium is one of the most straightforward tools available.

The problem isn't the browser automation. The problem is the IP.

When you run Selenium from a single machine, every request comes from the same IP address. Modern anti-bot systems — Cloudflare, Datadome, PerimeterX, and the homegrown rate limiters that major sites maintain — don't just look at request headers. They look at behavioral patterns, request velocity, IP reputation, and about forty other signals. A single residential IP hammering the same domain repeatedly is a textbook bot signature, no matter how much you randomize your user agent string.

The fix isn't a clever Selenium trick. It's proxy rotation. And proxy rotation at scale, without spending weeks building and maintaining your own infrastructure, is exactly what ScraperAPI handles.

---

## What ScraperAPI Actually Does (The One-Paragraph Version)

ScraperAPI sits between your Selenium script and the target website. When your browser sends a request, it goes through ScraperAPI's infrastructure first — a pool of **40+ million IPs across 50+ countries** — before reaching the target. The service handles proxy rotation automatically, solves CAPTCHAs, retries failed requests, spoofs headers, and manages session persistence. Your Selenium code stays almost exactly the same. You just route traffic through the proxy.

The key word is *almost*. There's a right way and a wrong way to integrate it with Selenium, and the wrong way is more common than you'd think.

---

## The Right Way: Using Selenium-Wire + ScraperAPI Proxy Port

The most common mistake people make when trying to use ScraperAPI with Selenium is pointing their scraper at ScraperAPI's API endpoint (`api.scraperapi.com`) as if it were a URL. This sounds logical but breaks immediately once Selenium tries to load page assets (CSS, JS, images) that reference relative paths — they'll all resolve to the ScraperAPI domain instead of the target site, and the page either loads broken or doesn't load at all.

The correct approach is to use ScraperAPI as a **proxy**, not as a URL wrapper. The easiest way to do this is with `selenium-wire`, which extends standard Selenium with interceptor capabilities and makes proxy configuration clean.

**Step 1: Install the dependencies**

bash
pip install selenium-wire webdriver-manager


**Step 2: Configure the proxy and initialize the driver**

python
from seleniumwire import webdriver
from webdriver_manager.chrome import ChromeDriverManager
from selenium.webdriver.chrome.options import Options

API_KEY = 'YOUR_SCRAPERAPI_KEY'  # Get yours at scraperapi.com

proxy_options = {
    'proxy': {
        'http': f'http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001',
        'https': f'http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001',
        'no_proxy': 'localhost,127.0.0.1'
    }
}

chrome_options = Options()
chrome_options.add_argument('--headless')
chrome_options.add_argument('--no-sandbox')
chrome_options.add_argument('--disable-dev-shm-usage')

driver = webdriver.Chrome(
    ChromeDriverManager().install(),
    options=chrome_options,
    seleniumwire_options=proxy_options
)


**Step 3: Use Selenium exactly as you normally would**

python
driver.get('https://quotes.toscrape.com')

# Find elements, extract data, navigate — all normal Selenium code
quotes = driver.find_elements('css selector', '.quote .text')
for quote in quotes:
    print(quote.text)

driver.quit()


That's the entire integration. Once the proxy is configured, every request Selenium makes — including the page assets, XHR calls, and navigation events — routes through ScraperAPI's infrastructure automatically. Your IP rotates. CAPTCHAs get handled. You just get data.

---

## Advanced Configuration: Passing ScraperAPI Parameters via Proxy

One of the useful things about the proxy-port integration is that you can still pass ScraperAPI-specific parameters by embedding them in the proxy username string. This is how you enable JavaScript rendering, set a country, use premium proxies, or trigger other features.

python
# Enable JavaScript rendering via proxy username parameters
proxy_options = {
    'proxy': {
        'http': f'http://scraperapi.render=true.country_code=us:{API_KEY}@proxy-server.scraperapi.com:8001',
        'https': f'http://scraperapi.render=true.country_code=us:{API_KEY}@proxy-server.scraperapi.com:8001',
        'no_proxy': 'localhost,127.0.0.1'
    }
}


Available parameter options you can chain this way include:

- `render=true` — enables full JavaScript rendering (costs +10 credits)
- `country_code=us` — routes through US proxies specifically
- `premium=true` — enables premium residential proxies (+10 credits)
- `keep_headers=true` — passes your custom headers through to the target

This lets you fine-tune exactly what the proxy does on a per-request basis without rewriting your scraper architecture.

---

## A Complete Working Example: Scraping a Product Page

Here's a realistic end-to-end example — a scraper that pulls product information from a page, with proper waits and error handling:

python
from seleniumwire import webdriver
from webdriver_manager.chrome import ChromeDriverManager
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import time

API_KEY = 'YOUR_SCRAPERAPI_KEY'

proxy_options = {
    'proxy': {
        'http': f'http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001',
        'https': f'http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001',
        'no_proxy': 'localhost,127.0.0.1'
    }
}

chrome_options = Options()
chrome_options.add_argument('--headless')
chrome_options.add_argument('--no-sandbox')
chrome_options.add_argument('--disable-dev-shm-usage')
chrome_options.add_argument('--ignore-certificate-errors')

driver = webdriver.Chrome(
    ChromeDriverManager().install(),
    options=chrome_options,
    seleniumwire_options=proxy_options
)

try:
    driver.get('https://quotes.toscrape.com')
    
    # Wait for quotes to load
    wait = WebDriverWait(driver, 10)
    wait.until(EC.presence_of_element_located((By.CSS_SELECTOR, '.quote')))
    
    quotes = driver.find_elements(By.CSS_SELECTOR, '.quote')
    
    for quote in quotes:
        text = quote.find_element(By.CSS_SELECTOR, '.text').text
        author = quote.find_element(By.CSS_SELECTOR, '.author').text
        print(f'"{text}" — {author}')
        
finally:
    driver.quit()


This is the structure you'd adapt for any real target. The proxy handles the anti-bot layer; Selenium handles the browser interaction layer. They don't step on each other.

---

## Why Not Just Use the ScraperAPI Endpoint Directly?

For completeness: yes, there's another integration pattern where you wrap your target URL in a ScraperAPI endpoint URL and pass that to `driver.get()`. It looks like this:

python
from urllib.parse import urlencode

API_KEY = 'YOUR_SCRAPERAPI_KEY'

def scraperapi_url(url):
    params = {'api_key': API_KEY, 'url': url}
    return 'http://api.scraperapi.com/?' + urlencode(params)

driver.get(scraperapi_url('https://example.com'))


ScraperAPI's own documentation explicitly flags this as **not recommended** for Selenium. The issue is that when Selenium receives the HTML from `api.scraperapi.com`, any relative links or assets on that page (`/static/main.css`, `/images/logo.png`, etc.) resolve against the ScraperAPI domain — not the original site's domain. This means page assets fail to load, relative navigation breaks, and you end up with a hobbled, partial page experience that defeats much of the purpose of using a headless browser in the first place.

The proxy-port method avoids this entirely because Selenium remains connected to the real target domain — ScraperAPI just handles the IP layer transparently.

---

## ScraperAPI Plans: Full Comparison Table

Here's the complete current plan lineup. The key differentiators between tiers are monthly credit volume, concurrent thread limits, geotargeting scope, and pay-as-you-go overflow availability.

| Plan | Monthly Price | Annual Price (10% off) | Credits/Month | Threads | Geotargeting | Purchase |
|---|---|---|---|---|---|---|
| **Free Trial** | $0 (7 days) | — | 5,000 one-time | 5 | Limited | [ Start Free Trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Hobby** | $49/mo | $44.10/mo | 100,000 | 20 | US & EU | [ Get Hobby Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Startup** | $149/mo | $134.10/mo | 1,000,000 | 50 | US & EU | [ Get Startup Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Business** | $299/mo | $269.10/mo | 3,000,000 | 100 | Global | [ Get Business Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Scaling** ⭐ Most Popular | $475/mo | $427.50/mo | 5,000,000 | 200 | Global | [ Get Scaling Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Professional** | $975/mo | $877.50/mo | 10,500,000 | 300 | Global + PAYG | [ Get Professional Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Advanced** | $1,975/mo | $1,777.50/mo | 21,500,000 | 500 | Global + PAYG | [ Get Advanced Plan](https://www.scraperapi.com/?fp_ref=coupons) |
| **Enterprise** | Custom | Custom | 22M+ | 500+ | Global + PAYG | [ Contact Sales](https://www.scraperapi.com/?fp_ref=coupons) |

**A few things worth flagging from this table:**

- **Geotargeting is gated by tier.** Hobby and Startup only support US and EU proxy pools. If you need country-specific targeting outside of those regions — say, scraping prices from a Japanese retailer or a Brazilian marketplace — you need at minimum the Business plan.

- **Pay-as-you-go overflow starts at Scaling.** On Hobby, Startup, and Business, running out of credits mid-month means either upgrading or stopping. From Scaling upward, you can keep going at a fixed per-credit rate with a configurable spending cap so you never get surprised by an unexpected bill.

- **Credits don't roll over.** Unused credits reset at renewal. Don't overbuy "just in case."

- **The new Growth plans** (Professional at 10.5M credits with 300 threads, and Advanced at 21.5M credits with 500 threads) launched in May 2026 with limited-time bonus credits — 250K extra on Professional and 500K extra on Advanced — for teams that need high volume without a custom enterprise negotiation.

---

## The Credit Multiplier System: What You Actually Pay Per Page

The plan prices make a lot more sense — or a lot less, depending on your target — once you understand how credits actually get consumed.

The base rate is **1 credit per request** for a standard, unprotected page. But the cost scales up depending on what you're scraping:

| Target / Parameter | Credit Cost |
|---|---|
| Standard unprotected page | 1 credit |
| Amazon, major e-commerce | 5 credits |
| Google / Bing search results | 25 credits |
| LinkedIn | 30 credits |
| Cloudflare / Datadome bypass | +10 credits |
| `render=true` (JS rendering) | +10 credits |
| `premium=true` (premium proxies) | +10 credits |
| `ultra_premium=true` | +30 credits |

What this means practically: if you're scraping Amazon product pages with JavaScript rendering enabled, each successful request costs you 5 (Amazon tier) + 10 (rendering) = **15 credits**. Your 100,000-credit Hobby plan now covers roughly 6,600 pages, not 100,000.

The good news: you're only charged for successful requests. Failed scrapes (anything that doesn't return a 200 or 404) don't burn credits, so ScraperAPI's own failures don't come out of your budget.

Before committing to a plan, run a batch of test requests against your actual target sites during the free trial and check your credit consumption in the dashboard. That number — not the headline credit count — is what tells you which plan actually fits.

---

## Which Plan Is Right for Your Selenium Setup?

The answer depends almost entirely on what you're scraping and how often. Here's a plain-language breakdown:

**Hobby ($49/mo)** is genuinely fine for personal projects, side experiments, or prototyping. If you're monitoring a handful of standard pages that aren't Amazon or Google, 100,000 base credits goes a long way. If you're hitting hard targets with rendering enabled, do the math first.

**Startup ($149/mo)** makes sense for small teams or lightweight SaaS products — when you've moved past "this is a test" but aren't running production infrastructure yet. The 10x jump in credits from Hobby is real, and 50 concurrent threads is meaningful for parallel Selenium jobs.

**Business ($299/mo)** is where global geotargeting unlocks, which matters if your targets require non-US/EU IP addresses. The 100-thread limit and 3M monthly credits put this squarely in "production-grade small operation" territory.

**Scaling ($475/mo) and above** is for when the question has shifted from "which plan" to "how do we stay unblocked at volume." The PAYG overflow means you're never hard-stopped mid-month, and the 200+ thread limits start to matter when you're running large parallel Selenium browser pools.

👉 [Start your free 7-day trial — 5,000 credits, no credit card needed](https://www.scraperapi.com/?fp_ref=coupons)

---

## What the People Actually Running This Say

ScraperAPI sits at **4.5/5 on Trustpilot** based on verified user reviews. The consistent themes across positive reviews: clean documentation (the Selenium guide is one of the more thorough in the space), genuinely simple integration that slots into existing scraper code without a major rewrite, and support that responds quickly and actually solves problems.

The most common complaint, echoed across multiple independent review sites, isn't about reliability or uptime — it's about the credit multiplier math being non-obvious at first. Specifically: people sign up for the Hobby plan expecting 100,000 requests, then discover their actual target costs 5x or 15x the base rate. It's not hidden, but it requires reading the docs before you commit.

The other recurring note from technical reviewers is that performance against highly dynamic, aggressively protected sites varies. ScraperAPI consistently performs well against major e-commerce platforms, SERP data, and standard web properties. For niche sites with custom, frequently-rotating anti-bot setups, results are less predictable — which, to be fair, is true of every proxy-based scraping service.

---

## Common Troubleshooting: ScraperAPI + Selenium Issues

**SSL/Certificate errors**: When routing through the proxy, Selenium may complain about SSL certificates. Add `--ignore-certificate-errors` to your Chrome options, or configure selenium-wire to handle SSL verification appropriately.

**Slow page loads**: If pages are timing out, it's often the proxy authentication handshake adding latency. Increase your `WebDriverWait` timeout from the default 10 seconds to 30-60 seconds for the first few requests.

**Still getting blocked**: Make sure you're actually routing through the proxy — add a quick test to print `driver.requests` and verify requests are going through `proxy-server.scraperapi.com`. Also check whether the target site requires `premium=true` or `ultra_premium=true` for higher success rates.

**Page assets not loading**: If you're seeing this, you may have accidentally used the API endpoint method instead of the proxy method. Switch to the `proxy-server.scraperapi.com:8001` configuration described above.

---

## Final Thought

The ScraperAPI + Selenium combination is genuinely one of the most practical setups for scraping dynamic, JavaScript-heavy sites at scale — not because either tool is uniquely magical on its own, but because they cover each other's gaps cleanly. Selenium handles the browser layer. ScraperAPI handles the IP and anti-bot layer. Neither has to do a job it wasn't built for.

The free trial is real, gives you 5,000 credits with no credit card required, and is enough to test against your actual targets before spending a dollar. If you're building anything serious involving dynamic pages and you're tired of managing your own proxy infrastructure, it's worth 20 minutes to run the integration and see your actual credit consumption against your actual targets.

[👉 Try ScraperAPI free — 5,000 credits, no card required](https://www.scraperapi.com/?fp_ref=coupons)
