# Puppeteer proxy: launch flags, authentication and session rotation that stop the 407s and blocks

Search for "puppeteer proxy" and you'll find a hundred snippets that look identical. Most of them contain the same silent bug: credentials inside the `--proxy-server` flag. Chromium strips them. No error, no warning, just a wall of 407s and a script that "doesn't work for some reason."

The proxy side of Puppeteer isn't hard. It's just unforgiving about ordering and syntax. Below is what actually matters — where credentials go, how many proxies you can run in one process, who owns rotation, and what that costs once you stop using free lists.

## The flag that quietly throws your password away

This is the single most common failure:

js
args: ['--proxy-server=http://user:pass@proxy.example.com:823']


Chromium ignores the `user:pass@` portion. The connection goes out unauthenticated and the proxy answers with `407 Proxy Authentication Required`. Puppeteer reports nothing useful because from its point of view the launch succeeded.

Split it. Host and port go in the flag, credentials go through the page:

js
const browser = await puppeteer.launch({
  headless: 'new',
  args: ['--proxy-server=proxy.example.com:823']
});
const page = await browser.newPage();
await page.authenticate({ username: 'LOGIN', password: 'PASSWORD' });
await page.goto('https://ip-api.com/json');


Order matters: `authenticate()` before the first `goto()`. Reversing them sends the first request unauthenticated.

## Pick the scope before you pick a provider

Puppeteer gives you three levels at which a proxy can be attached. They cost very different amounts of memory, and they differ in what they can isolate.

**Whole browser.** One `--proxy-server` flag, one exit IP for every page and every tab. Fine for straightforward crawling.

**Per browser context.** One browser process, several isolated contexts, each with its own proxy and cookie jar. This is how you run parallel accounts without giving them a shared fingerprint or a shared IP:

js
const context = await browser.createIncognitoBrowserContext({
  proxyServer: 'gw.dataimpulse.com:823'
});
const page = await context.newPage();
await page.authenticate({
  username: 'LOGIN__cr.us;sid.acct07',
  password: 'PASSWORD'
});


**Per request.** Puppeteer can't do this natively — there's no per-request proxy in `puppeteer.launch()`. You need request interception plus a relay library, or you route every request through Node and pay for it in latency, since each request and response now round-trips between Puppeteer and your Node process. Worth it if you genuinely need a different IP per URL. Overkill if you don't.

Launching a fresh browser per job is the most expensive option and the cleanest isolation. If your jobs are short and your memory budget is generous, it also removes a whole class of state-leak bugs.

## Authenticating with proxy-chain instead

The alternative to `page.authenticate()` is to strip the credentials out of the equation entirely. `proxy-chain` starts a tiny local proxy that accepts unauthenticated connections and forwards them upstream with the real credentials attached:

js
const originalUrl = `http://${user}:${pass}@gw.dataimpulse.com:823`;
const newUrl = await proxyChain.anonymizeProxy(originalUrl);
// newUrl looks like http://127.0.0.1:45678


Then pass `newUrl` straight into `--proxy-server`. Two gotchas worth knowing. Every `anonymizeProxy()` call opens a local server on a random port, so close it in a `finally` block or you'll leak ports across long runs. And `proxy-chain` v3 dropped CommonJS support — if your project uses `require()`, install `proxy-chain@2` explicitly rather than pulling latest.

Which method wins? `page.authenticate()` is fewer moving parts and works everywhere. `proxy-chain` is the better choice when you can't call `authenticate()` — for example when a third-party library owns the page object, or when you want one set of credentials shared across a browser context without repeating the call on every page.

## Rotation isn't a Puppeteer feature

This trips people up constantly. Puppeteer has no rotation logic. Rotation lives in your proxy credentials — specifically in the session identifier you send with them.

Two modes, and the difference matters more than the branding suggests:

| Connection type | What happens | Where to use it |
| --- | --- | --- |
| Rotating | New exit IP on every request | Crawling many pages of one site, spreading load across the pool |
| Sticky | Same IP bound to a port for a set window | Logged-in sessions, checkout flows, parallel workers |

With DataImpulse, rotating traffic runs on **port 823 for HTTP(S)** and **824 for SOCKS5**. Sticky sessions use **ports 10000–20000**, with a rotation interval from 1 to 120 minutes and a default of 30 if you don't specify one.

The rule that actually prevents bans: never reuse one session ID across two accounts. Two accounts sharing an IP link themselves together far more reliably than any browser fingerprint does.

## Which proxy type belongs in your Puppeteer launch

Detection risk and speed trade off directly against cost here.

| Type | Detection risk | Speed | Typical 2026 price | Best for |
| --- | --- | --- | --- | --- |
| Residential | Very low | Medium | ~$1–8/GB | Protected targets: e-commerce, SERPs, social |
| Mobile (4G/5G) | Very low | Slow | ~$2–15/GB | Hardest targets, app and mobile-web data |
| Datacenter | High | Very fast | ~$0.50–3/GB | Unprotected targets, high throughput |
| Premium residential | Very low | Fast | ~$5+/GB | High-stakes scraping that must not fail |

Datacenter IPs are fast and cheap and get blocked on anything with real bot defenses. Residential is the default answer for scraping. Mobile is what you reach for when residential still fails.

## What DataImpulse actually charges, across all four product lines

DataImpulse prices everything pay-as-you-go per gigabyte. No subscriptions, no monthly minimum, and purchased traffic doesn't expire — unused gigabytes stay on the account until your scripts consume them. Country-level targeting is included in the base rate; city, ZIP and ASN filters are paid add-ons on residential plans.

The network covers 90M+ IPs across 195 countries with HTTP/HTTPS and SOCKS5 support. The website publishes a 99.51% success rate and cites a 4.8/5 G2 rating — both vendor-reported figures rather than independently audited ones, so treat them as marketing claims until your own targets confirm them.

| Proxy type | Entry plan | Volume tiers | Per-GB rate | Billing | Buy |
| --- | --- | --- | --- | --- | --- |
| Residential | 5 GB for $5 | 1 TB for $800 | $1/GB → $0.80/GB | Pay-as-you-go, traffic never expires | [ Start with $5 of residential traffic](https://bit.ly/dataimPulse) |
| Datacenter | 10 GB for $5 | 100 GB for $50; 1 TB for $450; custom from $2,250 for 5 TB+ | $0.50/GB → $0.45/GB | Pay-as-you-go, traffic never expires | [ Grab the $0.50/GB datacenter plan](https://bit.ly/dataimPulse) |
| Mobile | 2.5 GB for $5 | 25 GB for $50; 1 TB for $1,600; custom from $8,000 for 5 TB+ | $2/GB → $1.60/GB | Pay-as-you-go, traffic never expires | [ Add mobile IPs to your pool](https://bit.ly/dataimPulse) |
| Premium residential | 1 GB for $5 | 10 GB for $50; custom from $20,000 for 5 TB+ | $5/GB | Pay-as-you-go, dedicated account manager | [ See the premium residential plans](https://dataimpulse.com/premium-residential-proxies/?aff=86938) |

For most Puppeteer work, the $5 / 5 GB residential entry point is the sensible starting move. It's the cheapest per-gigabyte of the four, country targeting is included, and 5 GB is enough to find out whether your target site cooperates before you commit to anything bigger. Premium residential only earns its 5× price if you're hitting something that reliably blocks standard residential IPs — and if it does, the dedicated account manager is the part you're actually paying for.

Worth flagging: volume discounts on mobile and premium residential only kick in at the 1 TB tier. If you're buying 50 GB of mobile traffic, you're paying list price.

## Wiring DataImpulse into a Puppeteer script

The gateway is the same for residential and mobile. Targeting parameters ride on the username, appended after a double underscore.

| Setting | Value | Note |
| --- | --- | --- |
| Host | `gw.dataimpulse.com` | Shared across residential and mobile |
| Port | `823` HTTP(S) rotating, `824` SOCKS5 rotating, `10000–20000` sticky | Protocol and port must match |
| Login / password | From your dashboard | Goes through `page.authenticate()`, never in the flag |
| Targeting | `LOGIN__cr.us;sid.job01;sessttl.600` | Country, session ID and TTL all on the username |

A working rotating script:

js
const puppeteer = require('puppeteer');

(async () => {
  const browser = await puppeteer.launch({
    headless: 'new',
    args: ['--proxy-server=gw.dataimpulse.com:823', '--no-sandbox']
  });

  const page = await browser.newPage();
  await page.authenticate({
    username: 'LOGIN__cr.us',
    password: process.env.PROXY_PASS
  });

  await page.goto('https://ip-api.com/json', { waitUntil: 'domcontentloaded' });
  console.log(await page.$eval('body', (el) => el.innerText));
  await browser.close();
})();


And the sticky version — same credentials, one extra segment:

js
await page.authenticate({
  username: 'LOGIN__cr.us;sid.job01;sessttl.600', // same IP for 10 minutes
  password: process.env.PROXY_PASS
});


`sessttl.600` is 600 seconds. Country codes go in `__cr.us`; add `ci.newyork` for city-level targeting. DataImpulse publishes a full walkthrough of this on their own site if you want the context-level and SOCKS5 variants: 👉 [👉 DataImpulse's Puppeteer setup tutorial](https://dataimpulse.com/tutorials/how-to-set-up-proxies-in-puppeteer/?aff=86938).

Keep credentials in environment variables. A `PROXY_PASS` sitting in a committed file is the kind of thing that gets discovered six months later.

## The cost question nobody answers

Most Puppeteer proxy guides tell you to "use residential proxies" and stop there. The number that matters is cost per *successful* request, not cost per gigabyte, and the two diverge fast when half your requests are coming back as soft blocks.

The practical approach: buy the smallest bucket, run your real workload against your real targets, and count successes rather than attempts. DataImpulse's own pricing guidance makes the same point — start at $5 / 5 GB, measure your own cost per successful request, then scale. With traffic that doesn't expire, a bucket you under-consume this month is still there next month, which is the difference between pay-as-you-go and a subscription that quietly bills you for bandwidth you never used.

Also remember that advanced targeting costs more. On residential plans, routing traffic through city, ZIP or ASN filters is billed above the base per-GB rate. If your script only needs US exit IPs, `__cr.us` is free and keeps your effective rate at $1/GB. Only narrow to city level when the target genuinely requires it.

## Errors you'll actually hit, and what they mean

| Error | Likely cause | Fix |
| --- | --- | --- |
| `ERR_PROXY_CONNECTION_FAILED` | Proxy down or unreachable | Remove from pool, retry with the next one |
| `407 Proxy Authentication Required` | Credentials in the flag, or `authenticate()` called after `goto()` | Move credentials into `page.authenticate()`, call it first |
| `ERR_TUNNEL_CONNECTION_FAILED` | HTTPS CONNECT issues | Check the proxy supports CONNECT; try `proxy-chain` local tunneling |
| `TimeoutError` | Slow proxy or target throttling | Raise timeout, switch to residential |
| `403 Forbidden` | IP or fingerprint flagged | Rotate proxy, enable stealth, randomize user agent |

One thing that isn't in the table: a `200` response doesn't mean you succeeded. Challenge pages and login walls come back with a perfectly normal status code and fully formed HTML. Validate the content, not the status.

## The non-proxy settings that determine whether you get blocked

A residential IP on a script that looks like a robot still gets blocked. Three things do most of the work:

`headless: 'new'` has meaningfully better stealth characteristics than legacy headless mode. Set a realistic user agent and viewport — a blank UA or an odd window size is itself a fingerprint. And add randomized delays between navigations rather than hammering the target at maximum speed; even a residential IP that fires 50 requests per second gets flagged.

## When Puppeteer plus proxies is the wrong tool

Puppeteer is the right call when you need to interact with a page — click, scroll, type into forms, hold a login session. That's what it's for, and a proxy is what stops the target from noticing.

If all you need is data that's already rendered on the page, you're building and maintaining proxy health-check infrastructure to solve a problem you don't have. A managed extraction API returns structured data directly, and the HTML you'd have scraped is just an intermediate format.

There's also a scope limit worth knowing about up front: DataImpulse offers rotating residential, mobile, datacenter and premium residential. It doesn't sell static ISP proxies or a fully managed scraping API. If your project requires stable long-term IPs or banking and government site access, this isn't the provider for it, and no amount of config tweaking will change that.

## FAQ

**Can I set a different proxy for each page in Puppeteer?**
Not with launch arguments. Use context-level proxies via `createIncognitoBrowserContext()`, or intercept requests and route them through an external relay.

**Why am I still getting blocked when I'm using a proxy?**
The proxy only changes your IP. Fingerprint signals — user agent, viewport, timing, TLS characteristics — are unchanged. Rotate more slowly, add delays, and use `headless: 'new'` before assuming the IP is the problem.

**HTTP or SOCKS5?**
HTTP(S) covers most scraping and works with `page.authenticate()`. SOCKS5 matters when you need non-HTTP protocols or when a target restricts HTTP proxies. On DataImpulse, the rotating ports differ: 823 for HTTP(S), 824 for SOCKS5.

**How long can a sticky session last?**
One to 120 minutes at DataImpulse. The default is 30 minutes if you don't specify, and you set it with `sessttl` on the username.

**Do I need a subscription?**
No. DataImpulse is pay-as-you-go per gigabyte with no monthly minimum, and traffic you buy doesn't expire.

## Where to start

If you're setting up Puppeteer with a proxy for the first time, launch a browser with `--proxy-server=gw.dataimpulse.com:823`, authenticate with a country-coded username, and hit an IP echo endpoint before writing any scraping logic. If that returns an IP in the country you asked for, everything downstream is your script's problem and not your proxy's.

Start with the smallest residential bucket you can buy and let your own success rate decide whether you need to spend more: 👉 [👉 set up a DataImpulse account and test a few gigabytes](https://bit.ly/dataimPulse). Five dollars of traffic that never expires is a cheap way to find out whether the target site is going to cooperate — and a much better answer than guessing from a blog post.
