# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Bulls-Bears is a PHP/MySQL virtual stock trading platform using real NSE (National Stock Exchange) data, built for a college event. Users start with ₹1,00,000 virtual cash and trade NSE-listed stocks in delivery or intraday modes.

## Development Setup

### Requirements
- PHP (legacy codebase uses deprecated `mysql_*` functions; requires PHP 5.x or a compatibility shim)
- MySQL database

### Configuration
Copy `core/config.php.sample` to `core/config.php` and fill in:
```php
define('DB_NAME', '...');
define('DB_USER', '...');
define('DB_PASSWORD', '...');
define('DB_HOST', 'localhost');
define('FB_APP_ID', '...');
define('FB_APP_SECRET', '...');
define('BASE', 'http://yourdomain/');
```
`core/config.php` is `.gitignore`d and must never be committed.

### Database Initialization
Run these once in order to set up the schema and seed initial data:
```bash
php core/database.php      # Creates all tables
php core/setting.php       # Seeds market status as 'closed'
php core/init.php          # Fetches NSE stock symbols and populates stockval table
```

### Serving the App
```bash
php -S localhost:8080      # Built-in PHP dev server from repo root
```

### Running Cron Jobs
```bash
bash cron/cron.sh                    # Runs stock_update.php in a 60s loop (keep running)
php cron/stock_graph.php             # Generate daily candlestick charts (run at EOD)
php cron/stock_graph_monthly.php     # Generate monthly charts
php cron/intracheck.php              # Settle intraday positions at market close
php cron/intrachecksell.php          # Process intraday sell settlements
php cron/amo.php                     # Execute After Market Orders at open
```

## Architecture

### Request Routing
All requests flow through `index.php`. Routing is done via `$_REQUEST['o']`:

| `?o=` value | Handler | Description |
|---|---|---|
| `buy` / `sell` | `module/buy.php`, `module/sell.php` | Delivery trades |
| `intrabuy` / `intrasell` | `module/intrabuy.php`, `module/intrasell.php` | Intraday trades (5× leverage) |
| `listStocks` | `module/userhome.php` | Stock browser |
| `pending` | `module/pending.php` | Limit order management |
| `portfolio` | `user/portfolio.php` | Holdings view |
| `leader` | `module/leader.php` | Leaderboard |
| `admin` | `admin/admin.php` | Admin panel (permission-gated) |
| *(default)* | `module/stockwatch.php` | User watchlist |

### Authentication
Authentication is handled entirely via Facebook OAuth in `user/verify.php`. The Facebook user ID is stored as the primary user identifier in the `user` table. `$_SESSION['uid']` holds the logged-in user ID throughout the app. No local login exists.

### Theme / Templating
There is no template engine. `index.php` loads a module file which outputs an HTML fragment, then wraps it in `theme/userprofile.php` (authenticated layout) or `theme/login.php` (unauthenticated). The pattern is:

```php
// index.php
include('module/something.php');   // sets $content or echoes directly
include('theme/userprofile.php');  // wraps with nav, balance bar, etc.
```

### Database Access
All DB access uses the deprecated `mysql_*` API via the `query_database($sql)` wrapper in `core/bootstrap.php`. This wrapper includes retry logic. There is no ORM or query builder — all SQL is written inline, with `mysql_real_escape_string()` for escaping.

### Key Tables

| Table | Purpose |
|---|---|
| `stockval` | Live stock prices (updated by cron every 60s) |
| `user` | Accounts: balance, rank, college, Facebook ID |
| `stocks_bought` | Delivery holdings per user |
| `intraday` | Open intraday positions |
| `lim` | Pending limit orders |
| `watch` | Per-user watchlists (max 50 stocks) |
| `settings` | Single-row: market open/closed status |
| `notifications` | Trade history |

### Trading Logic
- **Delivery** (`buy.php` / `sell.php`): 0.3% brokerage; positions stored in `stocks_bought`; multiple purchases of the same symbol average the cost basis
- **Intraday** (`intrabuy.php` / `intrasell.php`): 0.1% brokerage; 5× leverage; positions stored in `intraday`; auto-settled at market close by `cron/intracheck.php`
- **Limit orders** (`module/limitbuysell.php`): Stored in `lim` table; executed by `cron/amo.php` when price condition is met
- Market hours: 9:10–15:30 IST; `settings.status` toggles between `'open'` and `'closed'`

### Stock Data Feed
`cron/stock_update.php` fetches JSON from three NSE endpoints (NIFTY 50, Junior NIFTY, Midcap 50) and updates `stockval`. `TickerFeed.php` serves the top 50 stocks to the frontend ticker marquee in a pipe-delimited format consumed by `theme/js/ticker.js`.

### Frontend
Static jQuery-based frontend — no build step. Key files:
- `theme/js/function.js` — AJAX handlers for buy/sell forms and watchlist
- `theme/js/ticker.js` — Polls `TickerFeed.php` and renders the live ticker
- `theme/css/style.css` / `mobile.css` — Desktop and mobile stylesheets

Mobile detection uses `module/Mobile_Detect.php`; `/m/index.php` sets a cookie and redirects to the main app.

## Important Constraints

- **PHP version sensitivity**: The codebase relies on `mysql_*` functions removed in PHP 7.0. Any modernization must migrate to MySQLi or PDO.
- **No test suite**: There are no automated tests; verify changes manually via the browser.
- **NSE API dependency**: `cron/stock_update.php` and `core/init.php` fetch live NSE data; these will fail if NSE changes their API endpoints.
- **Chart images are generated artifacts**: `images/day/` and `images/monthly/` contain generated PNGs — do not treat them as source files.
