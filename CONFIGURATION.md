# TradeMed Configuration Guide

Complete reference for all configuration options and settings in TradeMed.

## Table of Contents

1. [Environment Variables](#environment-variables)
2. [Database Configuration](#database-configuration)
3. [Bitcoin/Cryptocurrency Configuration](#bitcoincryptocurrency-configuration)
4. [Security Configuration](#security-configuration)
5. [Site Settings](#site-settings)
6. [Order Management](#order-management)
7. [Multi-Vendor Settings](#multi-vendor-settings)
8. [TOR and Proxy Settings](#tor-and-proxy-settings)
9. [Advanced Settings](#advanced-settings)

---

## Environment Variables

All configuration is managed through environment variables, typically set in a `.env` file.

**Location**: `/home/trademed/trademed/.env`

See `.env.example` for a complete template.

### Loading Environment Variables

The `.env` file is loaded by the systemd service:

```ini
# /etc/systemd/system/trademed.service
[Service]
EnvironmentFile=/home/trademed/trademed/.env
```

After changing `.env`:
```bash
sudo systemctl restart trademed
```

---

## Database Configuration

### DATABASE_URL

**Format**: `******hostname/database_name`

**Example**:
```bash
DATABASE_URL=******localhost/trademed_production
```

**Components**:
- `username`: Database user (e.g., `trademed_user`)
- `password`: Database password
- `hostname`: Database server (usually `localhost`)
- `database_name`: Database name (e.g., `trademed_production`)

### Alternative: database.yml

If not using `DATABASE_URL`, configure in `config/database.yml`:

```yaml
production:
  adapter: postgresql
  encoding: unicode
  database: trademed_production
  username: trademed_user
  password: <%= ENV['DB_PASSWORD'] %>
  host: localhost
  port: 5432
  pool: 5
```

---

## Bitcoin/Cryptocurrency Configuration

### Market Server Configuration (Watch-Only Wallet)

#### MARKET_BITCOIND_URI

Bitcoin RPC connection for the market server (no private keys).

**Format**: `******hostname:port`

**Example**:
```bash
MARKET_BITCOIND_URI=******127.0.0.1:8332
```

**Setup**:
1. Configure `~/.bitcoin/bitcoin.conf`:
```ini
server=1
rpcuser=bitcoinrpc
rpcpassword=your_secure_password
rpcallowip=127.0.0.1
rpcport=8332
```

2. Start bitcoind:
```bash
bitcoind -daemon
```

3. Test connection:
```bash
bitcoin-cli -rpcuser=bitcoinrpc -rpcpassword=your_password getblockchaininfo
```

#### MARKET_LITECOIND_URI

Optional Litecoin support (same format as Bitcoin).

**Example**:
```bash
MARKET_LITECOIND_URI=******127.0.0.1:9332
```

### Payment Server Configuration (Full Wallet)

#### PAYOUT_BITCOIND_URI

Bitcoin RPC for payment server (has private keys).

**Example**:
```bash
PAYOUT_BITCOIND_URI=******127.0.0.1:8332
```

**Security Note**: The payment server should be on a separate, isolated machine.

#### PAYOUT_LITECOIND_URI

Optional Litecoin payment processing.

### Blockchain Settings

#### BLOCKCHAIN_CONFIRMATIONS

Number of confirmations required before marking an order as paid.

**Default**: `3`
**Range**: `0-6` (0 = zero-conf, not recommended)

**Examples**:
```bash
# Conservative (recommended)
BLOCKCHAIN_CONFIRMATIONS=3

# Fast (higher risk of reorg)
BLOCKCHAIN_CONFIRMATIONS=1

# Very conservative
BLOCKCHAIN_CONFIRMATIONS=6
```

**Considerations**:
- Higher = more secure against double-spend attacks
- Lower = faster order processing
- For small orders, 1-2 confirmations may be acceptable
- For large orders, 6+ confirmations recommended

#### LISTTRANSACTIONS_COUNT

Number of transactions to retrieve when checking address balances.

**Default**: `200`
**Recommended**: Set to typical weekly order volume

**Example**:
```bash
# For ~50 orders per week
LISTTRANSACTIONS_COUNT=100

# For high volume (500+ orders/week)
LISTTRANSACTIONS_COUNT=1000
```

#### BITCOIND_ORDER_ADDRESS_LABEL

Label for generated payment addresses (payment server).

**Default**: `""` (empty string)

**Example**:
```bash
BITCOIND_ORDER_ADDRESS_LABEL=trademed_orders
```

**Note**: Bitcoin Core 0.17+ uses labels. Earlier versions use accounts.

#### BITCOIND_WATCH_ADDRESS_LABEL

Label for imported watch addresses (market server).

**Default**: `""` (empty string)

**Example**:
```bash
BITCOIND_WATCH_ADDRESS_LABEL=trademed_watch
```

---

## Security Configuration

### SECRET_KEY_BASE

Rails secret key for session encryption.

**Generate**:
```bash
bundle exec rake secret
```

**Example**:
```bash
SECRET_KEY_BASE=a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6a7b8c9d0e1f2g3h4i5j6k7l8m9n0o1p2q3r4s5t6u7v8w9x0y1z2
```

**Security**:
- Must be at least 64 characters
- Keep secret and never commit to version control
- Use different key for each environment (dev, staging, prod)
- Use different key for market and payment servers

### ADMIN_HOSTNAME

Hostname for admin access (protects admin routes).

**Purpose**: Restricts admin access based on Host header.

**Examples**:
```bash
# Separate admin onion address (most secure)
ADMIN_HOSTNAME=admin7dj4k2l8m9.onion

# Subdomain (if your onion service supports it)
ADMIN_HOSTNAME=admin.yoursite.onion

# Same as public (less secure)
ADMIN_HOSTNAME=yoursite.onion
```

**Best Practice**: Use a separate TOR hidden service for admin access.

### ADMIN_API_KEY

Authentication key for admin API (used by payment server).

**Generate**:
```bash
openssl rand -hex 64
```

**Example**:
```bash
ADMIN_API_KEY=a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6a7b8c9d0e1f2
```

**Usage**:
- Set same value on both market and payment servers
- Payment server sends this in API requests
- Market server validates against this value

### DISPLAYNAME_HASH_SALT

Salt for hashing buyer names in feedback/reviews.

**Purpose**: Prevents linking buyers across multiple orders.

**Generate**:
```bash
openssl rand -hex 32
```

**Example**:
```bash
DISPLAYNAME_HASH_SALT=7f8e9d0c1b2a3940506070809aa0b0c0
```

**Security**: Keep secret to prevent brute-force attacks on buyer identities.

### GPG_KEY_ID

Your GPG key ID for signing payment addresses.

**Find your key ID**:
```bash
gpg --list-keys --keyid-format LONG
```

**Example**:
```bash
GPG_KEY_ID=ABCD1234EFGH5678
```

**Setup**:
1. Generate or import GPG key
2. Export public key for market server:
```bash
gpg --armor --export YOUR_KEY_ID > public_key.asc
```
3. Export private key for payment server:
```bash
gpg --armor --export-secret-keys YOUR_KEY_ID > private_key.asc
```

---

## Site Settings

### SITENAME

Your market's display name.

**Default**: `"No name"`

**Examples**:
```bash
SITENAME=CryptoMarket
SITENAME="My Bitcoin Shop"
```

**Used in**:
- Page titles
- Email-like communications (though email isn't actually used)
- Header/branding

### LOGO_FILENAME

Logo image filename (must exist in `app/assets/images/`).

**Default**: `trademed.png`

**Examples**:
```bash
LOGO_FILENAME=logo.png
LOGO_FILENAME=myshop-logo.jpg
```

**Setup**:
```bash
# Copy your logo
cp /path/to/logo.png app/assets/images/

# Update config
echo "LOGO_FILENAME=logo.png" >> .env

# Recompile assets
RAILS_ENV=production bundle exec rake assets:precompile
```

### CURRENCIES

Space-separated list of currencies for users to choose from.

**Default**: `USD AUD EUR CNY GBP CAD`

**Examples**:
```bash
# US-focused
CURRENCIES=USD

# International
CURRENCIES=USD EUR GBP JPY CAD AUD CNY

# Include more options
CURRENCIES=USD EUR GBP CAD AUD CNY JPY KRW BRL
```

**Available codes**: Any currency supported by your exchange rate provider (BitPay or Blockchain.info).

**Note**: Don't remove currencies after users have selected them.

---

## Order Management

### EXPIRE_UNPAID_ORDER_DURATION

Time before unpaid orders expire (in seconds).

**Default**: `43200` (12 hours)

**Examples**:
```bash
# 6 hours
EXPIRE_UNPAID_ORDER_DURATION=21600

# 12 hours (default)
EXPIRE_UNPAID_ORDER_DURATION=43200

# 24 hours
EXPIRE_UNPAID_ORDER_DURATION=86400

# 1 hour (for testing)
EXPIRE_UNPAID_ORDER_DURATION=3600
```

**Considerations**:
- Too short: Buyers may not have time to complete payment
- Too long: Exchange rate volatility risk
- 12-24 hours is typical for Bitcoin payments

### ORDERS_PER_HOUR_THRESHOLD

Maximum orders a buyer can create per hour (anti-abuse).

**Default**: `5`

**Examples**:
```bash
# Strict rate limiting
ORDERS_PER_HOUR_THRESHOLD=2

# Default
ORDERS_PER_HOUR_THRESHOLD=5

# Lenient (high volume store)
ORDERS_PER_HOUR_THRESHOLD=10
```

**Purpose**: Prevents malicious users from creating excessive orders.

### ORDER_AGE_TO_HIDE_PRICE_ON_REVIEW

Days after which review prices are hidden (vendor privacy).

**Default**: `120` days

**Examples**:
```bash
# Hide prices after 60 days
ORDER_AGE_TO_HIDE_PRICE_ON_REVIEW=60

# Hide prices after 6 months
ORDER_AGE_TO_HIDE_PRICE_ON_REVIEW=180

# Never hide prices (leave blank or omit)
# ORDER_AGE_TO_HIDE_PRICE_ON_REVIEW=
```

**Purpose**: Protects vendor revenue information over time.

---

## Multi-Vendor Settings

### COMMISSION

Commission percentage charged to vendors.

**Default**: `0.00` (single vendor)
**Format**: Decimal (e.g., `0.05` = 5%)

**Examples**:
```bash
# Single vendor (no commission)
COMMISSION=0.00

# 5% commission
COMMISSION=0.05

# 10% commission
COMMISSION=0.10
```

**Calculation**: Commission is deducted from vendor payments when orders are finalized.

### ENABLE_VENDOR_REGISTRATION_FORM

Show vendor registration fields on signup form.

**Default**: `false` (not set)
**Values**: `true` or `false`

**Example**:
```bash
ENABLE_VENDOR_REGISTRATION_FORM=true
```

**Effect**:
- `true`: Registration form includes vendor application fields
- `false`: Users register as buyers only; admin must upgrade to vendor

### ENABLE_MANDATORY_PGP_USER_ACCOUNTS

Require PGP public key for all new accounts.

**Default**: `false` (not set)
**Values**: `true` or `false`

**Example**:
```bash
ENABLE_MANDATORY_PGP_USER_ACCOUNTS=true
```

**Effect**:
- `true`: Registration form requires PGP key
- `false`: PGP key is optional

**Use Case**: Maximum security/anonymity markets.

### ENABLE_SUPPORT_TICKETS

Enable support ticket system (for multi-vendor markets).

**Default**: `false` (not set)
**Values**: `true` or `false`

**Example**:
```bash
ENABLE_SUPPORT_TICKETS=true
```

**Effect**:
- `true`: Support ticket link appears in navigation
- `false`: No ticket system (use messaging instead)

**Best For**: Multi-vendor markets where admin needs to communicate with users.

---

## TOR and Proxy Settings

### Payment Server Only

These settings are only needed on the payment server.

#### TOR_PROXY_HOST

TOR SOCKS proxy hostname for payment server.

**Default**: Not set (direct connection)

**Examples**:
```bash
# Local TOR daemon
TOR_PROXY_HOST=127.0.0.1

# Remote TOR proxy
TOR_PROXY_HOST=192.168.1.100
```

#### TOR_PROXY_PORT

TOR SOCKS proxy port.

**Default**: Not set
**Common**: `9050` (TOR default)

**Example**:
```bash
TOR_PROXY_PORT=9050
```

**Setup**:
```bash
# Install TOR
sudo apt install tor

# Check TOR is running
systemctl status tor

# Test connection
curl --socks5-hostname 127.0.0.1:9050 https://check.torproject.org
```

#### ADMIN_API_URI_BASE

Market server admin API base URL (for payment server to connect).

**Format**: `http://hostname` (no trailing slash)

**Examples**:
```bash
# TOR hidden service
ADMIN_API_URI_BASE=http://admin7dj4k2l8m9.onion

# Direct connection (testing only)
ADMIN_API_URI_BASE=http://192.168.1.100:3000
```

**Usage**: Payment server uses this to upload addresses and sync payout data.

---

## Advanced Settings

### Rails Environment

#### RAILS_ENV

Rails environment mode.

**Values**: `development`, `test`, `production`

**Example**:
```bash
RAILS_ENV=production
```

**Always use `production` for deployed systems.**

#### RACK_ENV

Rack environment (should match RAILS_ENV).

**Example**:
```bash
RACK_ENV=production
```

### Database Pool

Control database connection pooling in `config/database.yml`:

```yaml
production:
  pool: <%= ENV.fetch("DB_POOL") { 5 } %>
```

Then in `.env`:
```bash
DB_POOL=10
```

### Asset Host

Serve assets from CDN (not recommended for TOR):

```bash
ASSET_HOST=https://cdn.example.com
```

### Random HTTP Response Padding

Add random padding to responses (anti-fingerprinting).

**Location**: `config/application.rb`

```ruby
Trademed::Application.config.random_pad_http_response = false  # default
```

**Effect**: Adds random padding to make packet size analysis harder.
**Trade-off**: Slightly slower response times.

---

## Configuration Files

### config/application.rb

Main application configuration. All environment variables are read here.

**Key sections**:
- Line 22-99: Environment variable configuration
- Line 14: Cookie encryption settings
- Line 84: Unpermitted params handling

### config/environments/production.rb

Production-specific settings:

**Important settings**:
```ruby
# Serve static files (if no nginx/apache)
config.public_file_server.enabled = ENV['RAILS_SERVE_STATIC_FILES'].present?

# Force SSL (not for TOR hidden services)
# config.force_ssl = true

# Logging level
config.log_level = :info

# Asset compilation
config.assets.compile = false  # Precompile instead
```

### config/initializers/

Initialization scripts run at startup:

- `bitcoinrpc.rb` - Bitcoin RPC client setup
- `litecoinrpc.rb` - Litecoin RPC client setup
- `session_store.rb` - Session configuration
- `content_security_policy.rb` - CSP headers

---

## Testing Configuration

### Verify Settings

Check all configuration is loaded correctly:

```bash
# Start Rails console
RAILS_ENV=production bundle exec rails console

# Check settings
Rails.configuration.sitename
Rails.configuration.blockchain_confirmations
Rails.configuration.currencies
Rails.configuration.admin_hostname
```

### Test Bitcoin Connection

```bash
# From Rails console
bitcoin = BitcoinRPC.new(Rails.configuration.market_bitcoinrpc_uri)
bitcoin.getblockchaininfo
```

### Test Database Connection

```bash
# From Rails console
ActiveRecord::Base.connection.active?
User.count
Product.count
```

---

## Environment Variable Priority

Settings are loaded in this order (later overrides earlier):

1. Default values in `config/application.rb`
2. Environment variables from shell
3. `.env` file (via systemd EnvironmentFile)
4. Inline environment variables

**Example**:
```bash
# In .env
SITENAME=MyMarket

# Override on command line
SITENAME="Test Market" bundle exec rails console
```

---

## Security Best Practices

1. **Never commit `.env` to version control**
   - Use `.gitignore` to exclude it
   - Keep `.env.example` as a template

2. **Use strong secrets**
   - Generate with `openssl rand -hex 64`
   - Use different secrets for each server

3. **Separate admin access**
   - Use different `ADMIN_HOSTNAME`
   - Consider separate TOR hidden service

4. **Secure Bitcoin RPC**
   - Use strong `rpcpassword`
   - Restrict `rpcallowip` to localhost
   - Use different credentials for market vs payment server

5. **Review logs regularly**
   ```bash
   tail -f /home/trademed/trademed/log/production.log
   ```

6. **Backup configuration**
   ```bash
   cp .env .env.backup.$(date +%Y%m%d)
   ```

---

## Troubleshooting

### Configuration Not Loading

```bash
# Check systemd service has EnvironmentFile
sudo systemctl cat trademed | grep EnvironmentFile

# Check .env file exists and is readable
ls -la /home/trademed/trademed/.env
cat /home/trademed/trademed/.env

# Restart service
sudo systemctl restart trademed

# Check logs
sudo journalctl -u trademed -n 50
```

### Bitcoin Connection Fails

```bash
# Test RPC from command line
bitcoin-cli -rpcuser=USER -rpcpassword=PASS getblockchaininfo

# Check bitcoind is running
ps aux | grep bitcoind

# Check bitcoin logs
tail -f ~/.bitcoin/debug.log

# Verify RPC settings in .env match bitcoin.conf
cat ~/.bitcoin/bitcoin.conf
```

### Database Connection Fails

```bash
# Test database connection
psql -U trademed_user -d trademed_production -h localhost

# Check DATABASE_URL format
echo $DATABASE_URL

# Check PostgreSQL is running
sudo systemctl status postgresql
```

---

## Configuration Checklist

Before going live, verify:

- [ ] `SECRET_KEY_BASE` is set and secure
- [ ] `DATABASE_URL` connects successfully
- [ ] `MARKET_BITCOIND_URI` connects to working bitcoind
- [ ] `ADMIN_HOSTNAME` is configured
- [ ] `ADMIN_API_KEY` is set (if using payment server)
- [ ] `GPG_KEY_ID` is valid and key is imported
- [ ] `SITENAME` and `LOGO_FILENAME` are set
- [ ] `BLOCKCHAIN_CONFIRMATIONS` is appropriate
- [ ] `CURRENCIES` list is finalized
- [ ] TOR hidden service is configured
- [ ] `.env` file is protected (600 permissions)
- [ ] Assets are precompiled
- [ ] Database migrations are run
- [ ] Admin user is created

---

## Quick Reference

### Essential Settings

```bash
# Identity
SITENAME=YourMarket
LOGO_FILENAME=logo.png

# Security
SECRET_KEY_BASE=<generated>
ADMIN_HOSTNAME=admin.onion
ADMIN_API_KEY=<generated>
GPG_KEY_ID=ABCD1234

# Database
DATABASE_URL=******localhost/trademed_production

# Bitcoin
MARKET_BITCOIND_URI=******127.0.0.1:8332
BLOCKCHAIN_CONFIRMATIONS=3

# Site
CURRENCIES=USD EUR GBP
COMMISSION=0.00
```

### After Configuration Changes

```bash
# Restart application
sudo systemctl restart trademed

# Check logs
sudo journalctl -u trademed -f

# Test in browser
curl http://localhost:3000
```

---

## Resources

- Environment Variables in Rails: https://guides.rubyonrails.org/configuring.html
- Bitcoin RPC API: https://developer.bitcoin.org/reference/rpc/
- PostgreSQL Connection Strings: https://www.postgresql.org/docs/current/libpq-connect.html

---

## Support

For configuration questions:
- Review this guide first
- Check logs for specific errors
- Verify all required settings are present
- Email: tordoctor@tutanota.com
