# TradeMed Customization Guide

This guide covers all the ways you can customize TradeMed to match your brand and requirements.

## Table of Contents

1. [Visual Customization](#visual-customization)
2. [Configuration Options](#configuration-options)
3. [Web Server Configuration](#web-server-configuration)
4. [Application Settings](#application-settings)
5. [Email and Notifications](#email-and-notifications)
6. [Advanced Customization](#advanced-customization)

---

## Visual Customization

### 1. Site Logo

**Location**: `app/assets/images/`

Replace the default logo with your own:

```bash
# Copy your logo (recommended: 260px width, PNG format)
cp /path/to/your/logo.png app/assets/images/logo.png

# Update .env configuration
echo "LOGO_FILENAME=logo.png" >> .env

# Recompile assets
RAILS_ENV=production bundle exec rake assets:precompile
```

**Supported formats**: PNG, JPG, GIF
**Recommended dimensions**: Width ~260px, proportional height

### 2. Background Banner Image

**Location**: `app/assets/stylesheets/application_layout.css.scss`

Add a background banner across the top of the page:

```scss
.backgroundimg {
  background-image: url('your-banner-image.jpg');
  background-size: cover;
  background-position: center;
  min-height: 150px;
}
```

Steps:
1. Place your banner image in `app/assets/images/`
2. Edit `app/assets/stylesheets/application_layout.css.scss`
3. Add the CSS above (around line 70-80)
4. Recompile assets: `RAILS_ENV=production bundle exec rake assets:precompile`

### 3. Color Scheme

**Location**: `app/assets/stylesheets/application_layout.css.scss`

Customize the color scheme by overriding Bootstrap variables:

```scss
// Add at the top of application_layout.css.scss, before @import "bootstrap"

// Primary brand colors
$brand-primary: #337ab7;  // Your primary color
$brand-success: #5cb85c;  // Success messages
$brand-info: #5bc0de;     // Info messages
$brand-warning: #f0ad4e;  // Warning messages
$brand-danger: #d9534f;   // Error messages

// Navigation colors
$navbar-default-bg: #f8f8f8;
$navbar-default-link-color: #777;
$navbar-default-link-hover-color: #333;

// Body background
$body-bg: #ffffff;
$text-color: #333333;
```

After changes:
```bash
RAILS_ENV=production bundle exec rake assets:precompile
sudo systemctl restart trademed
```

### 4. Typography

**Location**: `app/assets/stylesheets/application_layout.css.scss`

Customize fonts:

```scss
// Add custom font imports at the top
@import url('https://fonts.googleapis.com/css2?family=Roboto:wght@300;400;700&display=swap');

// Override Bootstrap font family
$font-family-sans-serif: 'Roboto', 'Helvetica Neue', Helvetica, Arial, sans-serif;
$font-size-base: 14px;
$line-height-base: 1.6;

// Heading sizes
$font-size-h1: 36px;
$font-size-h2: 30px;
$font-size-h3: 24px;
```

### 5. Navigation Layout

**Location**: `app/views/layouts/application.html.erb`

The main layout file controls the overall page structure:

- **Line 13-25**: Header with logo and login buttons
- **Line 27-34**: Left sidebar navigation
- **Line 36**: Main content area

To modify the layout:
```bash
# Edit the layout file
nano app/views/layouts/application.html.erb

# Common modifications:
# - Change grid columns (col-md-3, col-md-9) for sidebar width
# - Reposition logo placement
# - Modify header structure
```

### 6. Footer Customization

**Location**: `app/views/layouts/application.html.erb` (bottom of file)

Add a custom footer:

```erb
<!-- Add before the closing </div> tag at the end -->
<div class="row">
  <div class="col-md-12 footer">
    <hr>
    <p class="text-center text-muted">
      &copy; <%= Time.now.year %> <%= Rails.configuration.sitename %>
      | <a href="/about">About</a>
      | <a href="/terms">Terms</a>
    </p>
  </div>
</div>
```

Then style it in `application_layout.css.scss`:
```scss
.footer {
  margin-top: 50px;
  padding: 20px 0;
  border-top: 1px solid #e5e5e5;
}
```

---

## Configuration Options

### Environment Variables

All configuration is done via environment variables in the `.env` file.

**Location**: `/home/trademed/trademed/.env`

See [.env.example](.env.example) for a complete template with all options.

### Key Configuration Settings

#### Site Branding

```bash
# Site name displayed in title and throughout the site
SITENAME=YourMarketName

# Logo filename (must be in app/assets/images/)
LOGO_FILENAME=logo.png
```

#### Currency Settings

```bash
# Space-separated list of currencies users can choose from
CURRENCIES=USD EUR GBP CAD AUD CNY JPY

# Default: USD AUD EUR CNY GBP CAD
```

#### Order Settings

```bash
# Hours before unpaid orders expire (default: 12 hours)
EXPIRE_UNPAID_ORDER_DURATION=43200  # in seconds

# Max orders per hour per buyer (rate limiting)
ORDERS_PER_HOUR_THRESHOLD=5

# Days after which to hide prices in reviews (for vendor privacy)
ORDER_AGE_TO_HIDE_PRICE_ON_REVIEW=120

# Number of blockchain confirmations required
BLOCKCHAIN_CONFIRMATIONS=3
```

#### Multi-Vendor Settings

```bash
# Commission percentage (0.00 for single vendor)
COMMISSION=0.00

# Enable vendor registration form
ENABLE_VENDOR_REGISTRATION_FORM=false

# Require PGP for all user accounts
ENABLE_MANDATORY_PGP_USER_ACCOUNTS=false

# Enable support ticket system
ENABLE_SUPPORT_TICKETS=false
```

#### Security Settings

```bash
# Salt for hashing display names in feedback
DISPLAYNAME_HASH_SALT=random_string_here

# Admin hostname (can be different from public site)
ADMIN_HOSTNAME=admin.youronion.onion
```

---

## Web Server Configuration

### Nginx Configuration (Recommended)

#### Basic Configuration

**Location**: `/etc/nginx/sites-available/trademed`

```nginx
upstream trademed_app {
    server 127.0.0.1:3000 fail_timeout=0;
}

server {
    listen 80;
    server_name localhost;
    
    # Document root - points to Rails public directory
    root /home/trademed/trademed/public;
    
    # Maximum upload size (for product images)
    client_max_body_size 10M;
    
    # Serve static files directly
    location ~ ^/(assets|system)/ {
        gzip_static on;
        expires max;
        add_header Cache-Control public;
        add_header ETag "";
        break;
    }
    
    # Serve system files (uploaded images)
    location /system/ {
        alias /home/trademed/trademed/public/system/;
        expires max;
        add_header Cache-Control public;
    }
    
    # Pass everything else to the Rails app
    location / {
        # Check for static files first
        try_files $uri @trademed_app;
    }
    
    location @trademed_app {
        proxy_pass http://trademed_app;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Increase timeouts for long requests
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        proxy_redirect off;
    }
    
    # Error pages
    error_page 500 502 503 504 /500.html;
    error_page 404 /404.html;
    error_page 422 /422.html;
    
    location = /500.html {
        root /home/trademed/trademed/public;
    }
}
```

#### SSL/TLS Configuration (if not using TOR)

```nginx
server {
    listen 443 ssl http2;
    server_name yourdomain.com;
    
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    
    # SSL settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # ... rest of configuration same as above
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}
```

#### Security Headers

Add these headers to your nginx configuration:

```nginx
# Add inside the server {} block

# Prevent clickjacking
add_header X-Frame-Options "SAMEORIGIN" always;

# Prevent MIME type sniffing
add_header X-Content-Type-Options "nosniff" always;

# XSS Protection
add_header X-XSS-Protection "1; mode=block" always;

# Content Security Policy (important for TradeMed's no-JS design)
add_header Content-Security-Policy "default-src 'self'; script-src 'none'; frame-ancestors 'none'; img-src 'self' data:; style-src 'self' 'unsafe-inline';" always;

# Referrer Policy
add_header Referrer-Policy "no-referrer" always;

# Permissions Policy
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

#### TOR Hidden Service with Nginx

**TOR Configuration**: `/etc/tor/torrc`

```
HiddenServiceDir /var/lib/tor/trademed/
HiddenServicePort 80 127.0.0.1:8080
```

**Nginx Configuration**: Listen on port 8080 for TOR

```nginx
server {
    listen 127.0.0.1:8080;
    server_name your_onion_address.onion;
    
    # ... rest of configuration as above
}
```

#### Admin-Only Hidden Service

For added security, create a separate hidden service for admin access:

**TOR Configuration**: `/etc/tor/torrc`

```
# Public market access
HiddenServiceDir /var/lib/tor/trademed_public/
HiddenServicePort 80 127.0.0.1:8080

# Admin-only access
HiddenServiceDir /var/lib/tor/trademed_admin/
HiddenServicePort 80 127.0.0.1:8081
```

**Nginx Configuration**: Separate server blocks

```nginx
# Public access
server {
    listen 127.0.0.1:8080;
    server_name public_onion.onion;
    
    # Block admin routes
    location ~ ^/admin {
        return 404;
    }
    
    # ... rest of public configuration
}

# Admin access
server {
    listen 127.0.0.1:8081;
    server_name admin_onion.onion;
    
    # Only allow admin routes
    location ~ ^/admin {
        proxy_pass http://trademed_app;
        # ... proxy settings
    }
    
    # Block public routes
    location / {
        return 404;
    }
}
```

### Apache Configuration (Alternative)

**Location**: `/etc/apache2/sites-available/trademed.conf`

```apache
<VirtualHost *:80>
    ServerName localhost
    
    # Document root - points to Rails public directory
    DocumentRoot /home/trademed/trademed/public
    
    <Directory /home/trademed/trademed/public>
        Require all granted
        Options -MultiViews
        AllowOverride None
    </Directory>
    
    # Passenger configuration (if using Passenger)
    PassengerRuby /home/trademed/.rbenv/shims/ruby
    PassengerAppEnv production
    PassengerFriendlyErrorPages off
    
    # OR Proxy configuration (if using separate app server)
    ProxyPreserveHost On
    ProxyPass /system/ !
    ProxyPass /assets/ !
    ProxyPass / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/
    
    # Static file serving
    <LocationMatch "^/(assets|system)/">
        Header set Cache-Control "public, max-age=31536000"
    </LocationMatch>
    
    # Security headers
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-XSS-Protection "1; mode=block"
    Header always set Content-Security-Policy "default-src 'self'; script-src 'none'; frame-ancestors 'none';"
    
    # Logging
    ErrorLog ${APACHE_LOG_DIR}/trademed_error.log
    CustomLog ${APACHE_LOG_DIR}/trademed_access.log combined
</VirtualHost>
```

Enable required modules:
```bash
sudo a2enmod proxy proxy_http headers rewrite ssl
sudo a2ensite trademed
sudo systemctl reload apache2
```

---

## Application Settings

### Database Configuration

**Location**: `/home/trademed/trademed/config/database.yml`

Or via environment variable in `.env`:

```bash
DATABASE_URL=******localhost/trademed_production
```

### Session Configuration

**Location**: `/home/trademed/trademed/config/initializers/session_store.rb`

```ruby
Rails.application.config.session_store :cookie_store, 
  key: '_trademed_session',
  expire_after: 24.hours  # Adjust session lifetime
```

### Asset Pipeline

**Location**: `/home/trademed/trademed/config/initializers/assets.rb`

Add custom JavaScript or CSS files:

```ruby
# Precompile additional assets
Rails.application.config.assets.precompile += %w( custom.css custom.js )
```

### Content Security Policy

**Location**: `/home/trademed/trademed/config/initializers/content_security_policy.rb`

Modify CSP if needed (though TradeMed intentionally disables JavaScript):

```ruby
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self
  policy.script_src  :none  # No JavaScript by design
  policy.style_src   :self, :unsafe_inline
  policy.img_src     :self, :data
  policy.font_src    :self
end
```

---

## Email and Notifications

**Note**: TradeMed intentionally does NOT use email for anonymity. All communication is done through the internal messaging system.

### Internal Messaging Customization

**Location**: `app/views/messages/`

Customize message templates and layouts:
- `conversations.html.erb` - Message inbox
- `show_conversation.html.erb` - Message thread view

---

## Advanced Customization

### Custom Routes

**Location**: `/home/runner/work/trademed/trademed/config/routes.rb`

Add custom pages or modify existing routes:

```ruby
# Add custom static pages
get '/about', to: 'pages#about'
get '/terms', to: 'pages#terms'
get '/privacy', to: 'pages#privacy'

# Custom product categories URL structure
resources :products, path: 'shop'
```

### Custom Controllers

**Location**: `app/controllers/pages_controller.rb`

Create a pages controller for static content:

```ruby
class PagesController < ApplicationController
  def about
    # renders app/views/pages/about.html.erb
  end
  
  def terms
    # renders app/views/pages/terms.html.erb
  end
  
  def privacy
    # renders app/views/pages/privacy.html.erb
  end
end
```

### Product Display Customization

**Location**: `app/views/products/`

- `index.html.erb` - Product listing page
- `show.html.erb` - Individual product page

Common customizations:
```erb
<!-- Modify product grid layout -->
<div class="row">
  <% @products.each do |product| %>
    <div class="col-md-4">  <!-- Change col-md-4 to col-md-3 for 4 columns -->
      <!-- product card -->
    </div>
  <% end %>
</div>
```

### Order Process Customization

**Location**: `app/views/orders/`

Customize the checkout flow:
- `new.html.erb` - Order creation form
- `show.html.erb` - Order details and payment info

### Vendor Dashboard

**Location**: `app/views/vendor/`

Customize vendor management interface:
- `products/` - Product management
- `orders/` - Order management
- `feedbacks/` - Feedback management

### Admin Panel Customization

**Location**: `app/views/admin/`

Customize admin interface:
- `entry/index.html.erb` - Admin dashboard
- `users/` - User management
- `orders/` - Order management
- `btc_address/` - Bitcoin address management

### Custom Validators

**Location**: `app/models/`

Add custom validation logic to models:

```ruby
# app/models/product.rb
validate :custom_price_validation

private

def custom_price_validation
  if unitprices.any? { |up| up.price_btc < 0.0001 }
    errors.add(:base, "Minimum price is 0.0001 BTC")
  end
end
```

### Custom Helpers

**Location**: `app/helpers/application_helper.rb`

Add custom helper methods:

```ruby
module ApplicationHelper
  def format_btc(amount)
    "₿ #{sprintf('%.8f', amount)}"
  end
  
  def product_image_or_placeholder(product)
    if product.image1.present?
      image_tag(product.image1.url(:medium))
    else
      image_tag('placeholder.jpg', alt: 'No image')
    end
  end
end
```

---

## File and Directory Paths Reference

### Important Paths

```
/home/trademed/trademed/                      # Application root
├── app/
│   ├── assets/
│   │   ├── images/                           # Logo and images
│   │   ├── stylesheets/                      # CSS/SCSS files
│   │   │   ├── application_layout.css.scss   # Main layout styles
│   │   │   └── admin.css                     # Admin styles
│   │   └── javascripts/                      # JavaScript (minimal usage)
│   ├── controllers/                          # Application logic
│   ├── models/                               # Database models
│   ├── views/
│   │   ├── layouts/
│   │   │   └── application.html.erb          # Main layout template
│   │   ├── products/                         # Product views
│   │   ├── orders/                           # Order views
│   │   ├── messages/                         # Messaging views
│   │   ├── vendor/                           # Vendor dashboard
│   │   └── admin/                            # Admin panel
│   └── helpers/                              # View helpers
├── config/
│   ├── application.rb                        # Main config (env vars)
│   ├── routes.rb                             # URL routing
│   ├── database.yml                          # Database config
│   ├── environments/
│   │   └── production.rb                     # Production settings
│   └── initializers/                         # Initialization code
├── public/
│   ├── assets/                               # Compiled assets (don't edit)
│   ├── system/                               # Uploaded files
│   ├── 404.html                              # Error pages
│   └── 500.html
├── log/
│   └── production.log                        # Application logs
└── .env                                      # Environment variables
```

### Public Assets (After Compilation)

```
/home/trademed/trademed/public/
├── assets/                    # Compiled CSS, JS, images
│   └── [fingerprinted files]
└── system/                    # User uploads (product images)
    └── [hash-based paths]
```

### Nginx/Apache Serving Paths

When configuring your web server, point to:

- **Document Root**: `/home/trademed/trademed/public`
- **Static Assets**: `/home/trademed/trademed/public/assets`
- **Uploaded Files**: `/home/trademed/trademed/public/system`
- **Error Pages**: `/home/trademed/trademed/public/404.html`, etc.

---

## Applying Changes

After making customization changes:

### 1. Asset Changes (CSS, Images, etc.)

```bash
# Recompile assets
cd /home/trademed/trademed
RAILS_ENV=production bundle exec rake assets:precompile

# Clear old assets (optional)
rm -rf public/assets/*
RAILS_ENV=production bundle exec rake assets:precompile

# Restart application
sudo systemctl restart trademed
```

### 2. View/Template Changes

```bash
# No compilation needed, just restart
sudo systemctl restart trademed
```

### 3. Configuration Changes

```bash
# After editing .env or config files
sudo systemctl restart trademed

# Verify changes took effect
sudo journalctl -u trademed -n 50
```

### 4. Web Server Configuration Changes

```bash
# For Nginx
sudo nginx -t                    # Test configuration
sudo systemctl reload nginx      # Reload without downtime

# For Apache
sudo apache2ctl configtest       # Test configuration
sudo systemctl reload apache2    # Reload without downtime
```

---

## Troubleshooting

### Assets Not Loading

```bash
# Check if assets were compiled
ls -la public/assets/

# Recompile assets
RAILS_ENV=production bundle exec rake assets:clobber
RAILS_ENV=production bundle exec rake assets:precompile

# Check web server can access files
sudo -u www-data ls /home/trademed/trademed/public/assets/
```

### Configuration Not Taking Effect

```bash
# Verify .env file is loaded
sudo systemctl status trademed
sudo journalctl -u trademed -n 50

# Ensure EnvironmentFile is set in systemd service
sudo nano /etc/systemd/system/trademed.service
# Should have: EnvironmentFile=/home/trademed/trademed/.env

# Reload systemd and restart
sudo systemctl daemon-reload
sudo systemctl restart trademed
```

### Web Server Not Serving Files

```bash
# Check permissions
ls -la /home/trademed/trademed/public/

# Fix permissions if needed
sudo chown -R trademed:www-data /home/trademed/trademed/public/
sudo chmod -R 755 /home/trademed/trademed/public/

# Check web server error logs
sudo tail -f /var/log/nginx/error.log
# or
sudo tail -f /var/log/apache2/error.log
```

---

## Best Practices

1. **Always test in development first** before applying to production
2. **Backup your database** before major customizations
3. **Keep a copy of original files** before modifying
4. **Use version control** for custom changes
5. **Document your customizations** for future reference
6. **Test after each change** to identify issues quickly
7. **Monitor logs** after deploying changes
8. **Keep assets optimized** (compress images, minify CSS)

---

## Resources

- Rails Asset Pipeline: https://guides.rubyonrails.org/asset_pipeline.html
- Bootstrap 3 Documentation: https://getbootstrap.com/docs/3.3/
- Nginx Documentation: https://nginx.org/en/docs/
- Apache Documentation: https://httpd.apache.org/docs/
- TOR Hidden Service Setup: https://community.torproject.org/onion-services/

---

## Support

For questions about customization:
- Review this guide first
- Check application logs for errors
- Test changes in development environment
- Email: tordoctor@tutanota.com
