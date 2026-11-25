# Python web scraping: requests + BeautifulSoup, when you actually need a headless browser, and when a scraping API is cheaper than your own stack

Most Python web scraping projects never need a browser or a paid API. `requests` plus BeautifulSoup will get you through static pages, and the whole script fits in about 20 lines. You start paying for infrastructure when the site answers with 403s or CAPTCHAs, when the data only appears after JavaScript runs, or when you need thousands of pages a day through rotating IPs.

Below you get a working script, the three parsing mistakes that cost beginners the most time, a way to tell static from dynamic pages before you install anything heavy, and the full ScraperAPI plan table with the credit math that decides which plan is actually cheaper for your workload.

> **Quick pick:** static pages and a few thousand requests a month? Write the script yourself. Hitting bot protection, needing JS rendering, or not wanting to maintain a proxy pool? ScraperAPI starts at **$49/mo for 100,000 credits**, and a free signup gives you 1,000 credits and 5 concurrent connections to test with. 👉 [Check the current plans and prices](https://scraperapi.com/pricing/?fp_ref=coupons)

## ScraperAPI plans compared

Every plan includes JS rendering, rotating proxy pools, premium residential and mobile IPs, anti-bot bypass, CAPTCHA handling, autoparse, custom headers and automatic retries. What changes between tiers is credits, concurrency, geotargeting and what happens when you run out.

| Plan | Credits / month | Concurrency & geotargeting | Price (monthly / annual) | Get it |
| --- | --- | --- | --- | --- |
| Free trial | 1,000 | 5 connections | $0 | [Start with the free credits](https://scraperapi.com/pricing/?fp_ref=coupons) |
| Hobby | 100,000 | 20 threads, US & EU only | $49 / $44.10 per month | [Open Hobby](https://scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | 1,000,000 | 50 threads, US & EU only | $149 / $134.10 per month | [Open Startup](https://scraperapi.com/pricing/?fp_ref=coupons) |
| Business | 3,000,000 | 100 threads, global country-level | $299 / $269.10 per month | [Open Business](https://scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 threads, global, pay-as-you-go | $475 / $427.50 per month | [Open Scaling](https://scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | 10,500,000 | 300 threads, global, pay-as-you-go, priority support | $975 / $877.50 per month | [Open Professional](https://scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 threads, global, pay-as-you-go, priority routing | $1,975 / $1,777.50 per month | [Open Advanced](https://scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ threads, dedicated team and Slack support | Custom quote | [Request an Enterprise quote](https://scraperapi.com/pricing/?fp_ref=coupons) |

Annual billing takes exactly 10% off, which is where the second price in each row comes from. Analytics history is capped at 30 days on Hobby and Startup and unlimited from Business up. Pay-as-you-go overage is not available below Scaling.

## What one credit actually buys

A credit is not a page. A standard request costs 1 credit, but the domain sets a multiplier: Amazon, Walmart and eBay cost 5 credits, Google and Bing cost 25, LinkedIn costs 30, and any request where ScraperAPI activates bot-protection bypass (Cloudflare, Datadome, PerimeterX) adds 10. Parameters stack on top: `render=true` adds 10, `premium=true` adds 10, `ultra_premium=true` adds 30, and `ultra_premium` combined with rendering lands at 75 credits per request. `country_code`, `session_number`, `device_type`, `output_format`, `wait_for_selector`, `keep_headers` and `autoparse` add nothing.

That multiplier changes which plan is worth buying. Hobby's 100,000 credits look generous until you divide by 5:

| Plan | $ per 1,000 credits | Amazon pages included | Cost per 1,000 Amazon pages |
| --- | --- | --- | --- |
| Hobby | $0.49 | 20,000 | $2.45 |
| Startup | $0.15 | 200,000 | $0.75 |
| Business | $0.10 | 600,000 | $0.50 |
| Scaling | $0.095 | 1,000,000 | $0.48 |
| Professional | $0.093 | 2,100,000 | $0.46 |
| Advanced | $0.092 | 4,300,000 | $0.46 |

Hobby is a testing plan at roughly **five times Startup's per-page cost**. If your scraper works and you know your monthly volume, the jump from Hobby to Startup at $149 buys 10× the credits for 3× the money. Rendering your pages pushes each standard request to 11 credits, so divide the page counts above by 11 rather than 5 when `render=true` is in your URL.

Two limits matter more than the credit number. Unused credits are lost at each renewal date; nothing carries over. And on Hobby, Startup and Business, exhausting credits gets you a **429 response, not an extra charge**: requests are refused rather than queued, so size the plan with headroom or move to Scaling where pay-as-you-go caps the overflow at a per-credit rate you set. Auto-upgrade to the next tier is opt-in on Hobby through Professional.

One footnote for negotiated contracts: accounts on custom deals pay a higher credit cost on ultra-premium domains to hold a floor of $3 per 1,000 unrendered requests and $7 with rendering. The dashboard's Domain Cost Estimator is the number to trust for any specific URL, and `max_cost` lets you cap a single scrape.

## A working scraper: requests, BeautifulSoup, CSV

The flow is three steps: get the HTML, parse it into a Python object, save what you extracted. This version pulls the first 30 stories from Hacker News.

python
import csv
import requests
from bs4 import BeautifulSoup

res = requests.get("https://news.ycombinator.com/news", timeout=20)
print(res.status_code)  # 200 = success, 400-500 range = something broke

soup = BeautifulSoup(res.content, "html.parser")
rows = []
for item in soup.select("tr.athing")[:30]:
    link = item.select_one(".titleline a")
    rows.append({
        "rank": item.select_one(".rank").text.strip(". "),
        "title": link.text,
        "url": link["href"],
    })

with open("hn.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["rank", "title", "url"])
    writer.writeheader()
    writer.writerows(rows)


`requests` returns the HTML without launching a browser, which is the point: images, CSS and JavaScript files just make the page look right, and you don't need them for data. Check `res.status_code` before parsing anything, because a 403 or 429 will otherwise hand BeautifulSoup an error page and you'll debug the wrong layer.

The `newline=""` argument in `open()` is not optional. Skip it and your CSV picks up blank rows between records on Windows.

BeautifulSoup handles encoding for you by converting to UTF-8, so you rarely need to think about it. It also tolerates broken markup, which matters because a lot of real pages have unclosed tags.

## Three parsing traps that waste an afternoon

**`class` is a Python reserved word.** You cannot pass it as a keyword argument. Use `class_` or a dictionary:

python
soup.find("div", class_="article-body")
soup.find("div", attrs={"class": "article-body", "id": "main"})


The dictionary form is the only way to filter on class and id together.

**Nested search is how you target list structures.** `find` and `find_all` return elements you can search inside again, which is what you want for a table of contents or a product grid:

python
toc = soup.find("div", id="toc")
for li in toc.find_all("li"):
    print(li.text.strip())


Searching `soup.find_all("li")` on the whole document returns every list item on the page, including navigation and footers.

**Parser choice is a real tradeoff.** `html.parser` ships with Python, `lxml` is faster and supports XPath, `html5lib` parses the way a browser does and is the most forgiving. Because BeautifulSoup sits on top of them, you can swap the parser string without rewriting your extraction code. Start with `lxml` on large pages, fall back to `html5lib` when the markup is a mess.

Also worth knowing: `re.search()` returns a MatchObject rather than a string, and Python's `*` quantifier is greedy by default, so tag-stripping regexes need `.*?` to stop at the first closing bracket.

## Static or dynamic? Check before you install a browser

The test is simple. Fetch the page with `requests` and search the raw HTML for a value you can see in the browser. If the price or the review count is there in the source, the page is static and BeautifulSoup is all you need. If the HTML contains a placeholder and the data shows up only after scripts run, no amount of parser tuning will help.

For dynamic pages you need a real browser engine. Selenium is the standard option and still works fine:

python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from webdriver_manager.chrome import ChromeDriverManager

options = webdriver.ChromeOptions()
options.add_argument("--headless")
options.add_argument("--no-sandbox")
options.add_argument("--disable-dev-shm-usage")

driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=options)
driver.get(url)
# ... extract with driver.find_elements(By.CLASS_NAME, "title")
driver.quit()


`ChromeDriverManager().install()` grabs the driver version that matches your installed Chrome, `Service(...)` is the Selenium 4+ wrapper for that path, and `driver.quit()` releases the browser process. Skipping `quit()` in a loop leaks memory until the machine gives up.

Playwright is the newer alternative and renders just as well, but it is heavier and slower than an HTTP request plus a parser. Use it when you need it, not as a default. A headless browser costs you seconds and hundreds of megabytes per page; `requests` costs milliseconds.

## Going past one page: pagination and Scrapy

Pagination is where hand-rolled scripts start to sprawl. The usual pattern is a loop that builds each page URL, parses it, appends to a list, and stops when a "next" link disappears. That works to a few dozen pages.

Beyond that, Scrapy scales better than a custom loop: it handles the request queue, concurrency, retries and duplicate filtering for you. The catch that most framework recommendations skip is that **Scrapy does not render JavaScript out of the box**. Dynamic pages need the `scrapy-splash` plugin, which means running a Splash service alongside your spider. If your targets are JS-heavy, that extra service is often the point where a managed API becomes the simpler option.

## When your script starts returning 403s

Three things break a working scraper at volume: IP bans, CAPTCHAs, and rate limits. Fixing them yourself means a proxy pool, a retry strategy, header rotation and a CAPTCHA solver, plus the ongoing cost of keeping the pool clean.

There's a boundary question here too. Scraping can violate a site's terms of use, so read the acceptable use policy before pointing a crawler at anything. Repeated high-frequency requests also consume bandwidth and can slow a site down for everyone else, which is why sites like Google Maps throttle aggressive clients. Politeness settings and a rate limit are cheaper than getting blocked.

A scraping API folds proxies, retries, rendering and bypass into one HTTP endpoint, and you pay per credit instead of per gigabyte of proxy traffic. For a few thousand pages a month the DIY route is fine. For sustained volume against protected domains, the maintenance hours usually cost more than the subscription. 👉 [See which plan covers your monthly page volume](https://scraperapi.com/pricing/?fp_ref=coupons)

## Which plan fits your case

Run your own numbers against these rules:

- **Static pages, under ~20,000 requests a month.** Skip the API. `requests` plus BeautifulSoup on your own machine is enough.
- **You want to test before committing.** Sign up free, spend the 1,000 credits on your actual target URLs, and check the Domain Cost Estimator for the real credit cost per URL rather than assuming 1.
- **Standard pages or light e-commerce, up to 200,000 Amazon pages.** Startup at $149/mo. Hobby's per-page cost is too high once volume is predictable.
- **You need country-level geotargeting.** Business is the first tier that offers it; Hobby and Startup are US and EU only.
- **You expect spikes and don't want 429s.** Scaling, because it's the cheapest tier with pay-as-you-go overflow and a spending cap you control.
- **Millions of rendered pages or negotiated volume.** Professional and Advanced lower the per-credit cost slightly; Enterprise exists for 22M+ credits and dedicated support.

Two things to confirm before you pay: how many credits your specific targets consume (the multiplier, not the headline number), and what happens at the limit on your tier. On Hobby, Startup and Business the answer is a 429 and no charge; from Scaling up you can keep going at a fixed per-credit rate.

If the free 1,000 credits cover your test URLs without hitting a 429, you've found your tier. 👉 [Compare all ScraperAPI plans side by side](https://scraperapi.com/pricing/?fp_ref=coupons)
