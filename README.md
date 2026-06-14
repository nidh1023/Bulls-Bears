# Bulls-Bears

A virtual stock trading platform built for college events, powered by live NSE (National Stock Exchange) data. Users start with ₹1,00,000 virtual cash and trade real NSE-listed stocks across three modes: delivery, intraday (5× leverage), and limit orders.

> Originally built for **Conjura '12**, sponsored by Geojit BNP Paribas.

---

## Tech Stack

- **Backend:** PHP (legacy `mysql_*` API — requires PHP 5.x)
- **Database:** MySQL
- **Frontend:** jQuery, vanilla JS, CSS
- **Auth:** Facebook OAuth
- **Data:** NSE JSON endpoints (NIFTY 50, Junior NIFTY, Midcap 50)

---

## Prerequisites

- PHP 5.x (the codebase uses `mysql_*` functions removed in PHP 7.0)
- MySQL 5.x
- A Facebook App (for OAuth login)

---

## Setup

### 1. Configure the app

```bash
cp core/config.php.sample core/config.php
```

Edit `core/config.php`:

```php
define('DB_NAME',       'your_db_name');
define('DB_USER',       'your_db_user');
define('DB_PASSWORD',   'your_db_password');
define('DB_HOST',       'localhost');
define('FB_APP_ID',     'your_facebook_app_id');
define('FB_APP_SECRET', 'your_facebook_app_secret');
define('BASE',          'http://localhost:8080/');
```

### 2. Initialize the database

Run these scripts **once**, in order:

```bash
php core/database.php   # Create all tables
php core/setting.php    # Seed market status as 'closed'
php core/init.php       # Fetch NSE stock list and populate stockval table
```

### 3. Start the dev server

```bash
php -S localhost:8080
```

Visit `http://localhost:8080` — you'll see the Facebook login page.

---

## Running Cron Jobs

These background jobs keep stock data live and settle positions. Run them separately from the web server.

| Script | When to run | Purpose |
|---|---|---|
| `bash cron/cron.sh` | Continuously (during market hours) | Fetches live NSE prices every 60s |
| `php cron/stock_graph.php` | End of day | Generate daily candlestick chart PNGs |
| `php cron/stock_graph_monthly.php` | End of month | Generate monthly charts |
| `php cron/intracheck.php` | At market close (15:30 IST) | Settle open intraday buy positions |
| `php cron/intrachecksell.php` | At market close (15:30 IST) | Settle open intraday sell positions |
| `php cron/amo.php` | At market open (9:10 IST) | Execute After Market Orders |

---

## Project Structure

```
Bulls-Bears/
├── core/           # DB connection, schema, config
├── module/         # Feature handlers (buy, sell, leaderboard, etc.)
├── user/           # Auth (Facebook OAuth) and portfolio
├── admin/          # Admin dashboard
├── cron/           # Background jobs for stock updates & settlement
├── theme/          # HTML templates, CSS, JS
├── page/           # Static info pages (rules, winners)
├── images/         # Generated stock charts (day/ and monthly/)
├── index.php       # Application entry point & router
├── TickerFeed.php  # Live ticker data endpoint
└── logout.php      # Session termination
```

---

## How Routing Works

All requests go through `index.php`. The `?o=` query parameter selects the module:

| URL | Module | Action |
|---|---|---|
| `/?o=buy` | `module/buy.php` | Buy stock (delivery) |
| `/?o=sell` | `module/sell.php` | Sell stock (delivery) |
| `/?o=intrabuy` | `module/intrabuy.php` | Intraday buy (5× leverage) |
| `/?o=intrasell` | `module/intrasell.php` | Intraday sell |
| `/?o=listStocks` | `module/userhome.php` | Browse all stocks |
| `/?o=portfolio` | `user/portfolio.php` | View holdings |
| `/?o=pending` | `module/pending.php` | Manage limit orders |
| `/?o=leader` | `module/leader.php` | Leaderboard |
| `/?o=admin` | `admin/admin.php` | Admin panel |
| `/?o=feedback` | `module/feedback.php` | Feedback form |
| *(default)* | `module/stockwatch.php` | User watchlist |

---

## Trading Mechanics

| Mode | Brokerage | Leverage | Settled |
|---|---|---|---|
| Delivery | 0.3% | 1× | Manually (sell order) |
| Intraday | 0.1% | 5× | Auto at 15:30 IST by cron |
| Limit order | — | 1× | When price condition is met |

Market hours: **9:10 – 15:30 IST**. Market status (`open`/`closed`) is stored in the `settings` table and updated by the stock update cron.

---

## Database Overview

| Table | Description |
|---|---|
| `stockval` | Live prices, change %, day range for all stocks |
| `user` | User accounts — balance, Facebook ID, college, rank |
| `stocks_bought` | Delivery holdings per user |
| `intraday` | Open intraday positions |
| `lim` | Pending limit orders |
| `watch` | Per-user watchlists (max 50 stocks) |
| `notifications` | Trade history |
| `settings` | Market open/closed status |

---

## Known Limitations

- **PHP 7+ incompatible** — `mysql_*` functions were removed in PHP 7.0. Running on PHP 7+ requires migrating to MySQLi or PDO.
- **No automated tests** — verify changes manually in the browser.
- **NSE API** — `core/init.php` and `cron/stock_update.php` depend on NSE's public JSON endpoints, which may change without notice.
- **Facebook OAuth** — uses an older OAuth flow; may need updating if Facebook deprecates the API version.
