# Magento 2 (v247) Virtual Host Setup Guide

Complete step-by-step guide for configuring Magento 2 with custom domain (m247.local) on Apache web server.

---

## Prerequisites

- Magento 2 successfully installed at: `/var/www/html/m247`
- Apache web server installed
- PHP 8.1+ installed
- MySQL/MariaDB installed
- Root/sudo access to the server

---

## Step 1: Update Magento Environment Configuration

**File**: `/var/www/html/m247/app/etc/env.php`

Add the `website_configuration` section to the configuration array. This enables multi-store URL routing based on domain names.

```php
'website_configuration' => [
    [
        'domain' => 'm247.local',
        'run_code' => 'base',
        'run_type' => 'website'
    ]
]
```

**Location**: Add this configuration at the end of the array, just before the closing `];`

**Example**:
```php
<?php
return [
    'backend' => [
        'frontName' => 'admin'
    ],
    // ... other configurations ...
    'website_configuration' => [
        [
            'domain' => 'm247.local',
            'run_code' => 'base',
            'run_type' => 'website'
        ]
    ]
];
```

---

## Step 2: Update Public Index File

**File**: `/var/www/html/m247/pub/index.php`

Replace lines 28-43 with the following code to enable multi-store domain routing:

```php
$prebootstrap = Bootstrap::create(BP, $_SERVER);
// Setup multi-store urls
$params = $_SERVER;

$objectManager = $prebootstrap->getObjectManager();
// Get website configurations from env.php
$deploymentConfig = $objectManager->get('\Magento\Framework\App\DeploymentConfig');
$websiteConfiguration = $deploymentConfig->get('website_configuration');

foreach ($websiteConfiguration as $website) {
    if ($_SERVER['HTTP_HOST'] === $website['domain']) {
        $params[\Magento\Store\Model\StoreManager::PARAM_RUN_CODE] = $website['run_code'];
        $params[\Magento\Store\Model\StoreManager::PARAM_RUN_TYPE] = $website['run_type'];
    }
}
// End Setup multi-store urls

$bootstrap = Bootstrap::create(BP, $params);
```

**What this does**:
- Reads website configuration from `env.php`
- Matches the current HTTP_HOST with configured domains
- Sets appropriate store/website codes for Magento to load

---

## Step 3: Update Public Get File (Static/Media File Handler)

**File**: `/var/www/html/m247/pub/get.php`

Replace lines 89-121 with the following code:

```php
// Materialize file in application
$params = $_SERVER;

$prebootstrap = \Magento\Framework\App\Bootstrap::create(BP, $params);

// Setup multi-store urls
$objectManager = $prebootstrap->getObjectManager();
// Get website configurations from env.php
$deploymentConfig = $objectManager->get('\Magento\Framework\App\DeploymentConfig');
$websiteConfiguration = $deploymentConfig->get('website_configuration');

foreach ($websiteConfiguration as $website) {
    if ($_SERVER['HTTP_HOST'] === $website['domain']) {
        $params[\Magento\Store\Model\StoreManager::PARAM_RUN_CODE] = $website['run_code'];
        $params[\Magento\Store\Model\StoreManager::PARAM_RUN_TYPE] = $website['run_type'];
    }
}

if (empty($mediaDirectory)) {
    $params[ObjectManagerFactory::INIT_PARAM_DEPLOYMENT_CONFIG] = [];
    $params[Factory::PARAM_CACHE_FORCED_OPTIONS] = ['frontend_options' => ['disable_save' => true]];
}
$bootstrap = \Magento\Framework\App\Bootstrap::create(BP, $params);
// End Setup multi-store urls
```

**What this does**:
- Applies the same multi-store routing logic to static/media file requests
- Ensures proper store context when serving assets

---

## Step 4: Create Virtual Host Configuration File

**File**: `/var/www/html/m247/proxy-le-ssl.conf`

Create this file with the following content:

```apache
<VirtualHost *:80>
    ServerName m247.local
    ServerAlias www.m247.local
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/m247/pub

    <Directory /var/www/html/m247/pub>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Require all granted
    </Directory>

    # Enable mod_rewrite for Magento 2
    <IfModule mod_rewrite.c>
        RewriteEngine On
    </IfModule>

    # Logging
    ErrorLog /var/www/html/m247/var/log/apache_error.log
    CustomLog /var/www/html/m247/var/log/apache_access.log combined

    # Set proper environment variables for Magento
    SetEnv MAGE_MODE "developer"
</VirtualHost>
```

**Important Notes**:
- `DocumentRoot` must point to the `pub` directory
- `AllowOverride All` is required for `.htaccess` to work
- `Require all granted` is Apache 2.4 syntax (use `Allow from all` for Apache 2.2)
- Logs are stored in Magento's `var/log/` directory

---

## Step 5: Update System Hosts File

**File**: `/etc/hosts`

Add the following line to map the domain to localhost:

```bash
127.0.0.1 m247.local
```

**Command to add**:
```bash
sudo nano /etc/hosts
```

Or using command line:
```bash
echo "127.0.0.1 m247.local" | sudo tee -a /etc/hosts
```

**Verify**:
```bash
cat /etc/hosts | grep m247
```

---

## Step 6: Include Virtual Host in Apache Configuration

**File**: `/etc/apache2/sites-available/000-default.conf`

Add the following line inside the file (after other Include statements):

```apache
# VirtualHost for m247 (Magento 2.4.6-p10)
Include /var/www/html/m247/proxy-le-ssl.conf
```

**Important**: Make sure this Include statement appears only ONCE. Check for duplicates.

**Example placement**:
```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html

    # ... other configurations ...
</VirtualHost>

# Include custom virtual hosts
Include /var/www/html/m247/proxy-le-ssl.conf
```

**Edit command**:
```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

---

## Step 7: Configure Magento Base URLs

Run the following commands from the Magento root directory (`/var/www/html/m247`):

```bash
# Set unsecure (HTTP) base URL
bin/magento setup:store-config:set --base-url="http://m247.local/"

# Set secure (HTTPS) base URL - using HTTP for development
bin/magento config:set web/secure/base_url "http://m247.local/"

# Verify configuration
bin/magento config:show web/unsecure/base_url
bin/magento config:show web/secure/base_url
```

**Expected output**:
```
http://m247.local/
http://m247.local/
```

---

## Step 8: Disable HTTPS Enforcement (Development Environment)

Since SSL is not configured in development, disable HTTPS enforcement:

```bash
# Disable HTTPS for frontend
bin/magento config:set web/secure/use_in_frontend 0

# Disable HTTPS for admin panel
bin/magento config:set web/secure/use_in_adminhtml 0

# Verify settings
bin/magento config:show web/secure/use_in_frontend
bin/magento config:show web/secure/use_in_adminhtml
```

**Expected output**:
```
0
0
```

**Note**: If you enabled HTTPS via database queries, these commands will override those settings.

---

## Step 9: Clear Magento Cache

Clear all Magento caches to apply configuration changes:

```bash
# Clean specific cache
bin/magento cache:clean config

# Flush all cache types
bin/magento cache:flush

# Alternative: Clean and flush in one go
bin/magento cache:clean && bin/magento cache:flush
```

**Expected output**:
```
Cleaned cache types:
config
layout
block_html
collections
reflection
db_ddl
compiled_config
eav
customer_notification
config_integration
config_integration_api
full_page
config_webservice
translate
```

---

## Step 10: Enable Required Apache Modules

Ensure required Apache modules are enabled:

```bash
# Enable mod_rewrite (required for Magento)
sudo a2enmod rewrite

# Enable mod_headers
sudo a2enmod headers

# Enable mod_expires
sudo a2enmod expires

# Verify modules are loaded
apache2ctl -M | grep -E "(rewrite|headers|expires)"
```

**Expected output**:
```
 expires_module (shared)
 headers_module (shared)
 rewrite_module (shared)
```

---

## Step 11: Test Apache Configuration

Test the Apache configuration for syntax errors:

```bash
# Test configuration syntax
sudo apache2ctl -t

# Check virtual host configuration
sudo apache2ctl -S | grep m247
```

**Expected output for syntax test**:
```
Syntax OK
```

**Expected output for virtual host check**:
```
port 80 namevhost m247.local (/var/www/html/m247/proxy-le-ssl.conf:1)
         alias www.m247.local
```

---

## Step 12: Set Proper File Permissions

Ensure Magento directories have correct permissions:

```bash
cd /var/www/html/m247

# Get current user's primary group
CURRENT_GROUP=$(id -gn)

# Add current user to www-data group (allows user to access Apache files)
sudo usermod -a -G www-data $USER

# Add www-data to current user's group (allows Apache to access user files)
sudo usermod -a -G $CURRENT_GROUP www-data

# Set ownership to current user and current group
sudo chown -R $USER:$CURRENT_GROUP .

# Set directory permissions
sudo find var generated vendor pub/static pub/media app/etc -type d -exec chmod g+s {} \;

# Set file permissions
sudo chmod u+x bin/magento

# Set writable directories for development
sudo chmod -R 775 var/ generated/ pub/static/ pub/media/

# Give group write permissions
sudo chmod -R g+w var/ generated/ pub/static/ pub/media/
```

**Note**:
- `$USER` refers to your current logged-in user (e.g., `logicrays`)
- `$CURRENT_GROUP` or `$(id -gn)` gets your primary group (e.g., `logicrays`)
- Adding users to each other's groups allows both you and Apache to read/write files
- The `chmod 775` allows owner and group to write, others can only read/execute
- For development, this is safer than `chmod 777`
- For production, use more restrictive permissions (755 for directories, 644 for files)
- **Important**: You may need to log out and log back in for group changes to take effect

---

## Step 13: Restart Apache Web Server

Restart Apache to apply all configuration changes:

```bash
# Method 1: systemctl (Ubuntu/Debian with systemd)
sudo systemctl restart apache2

# Method 2: service command
sudo service apache2 restart

# Method 3: apachectl
sudo apachectl restart

# Verify Apache is running
sudo systemctl status apache2
```

**Expected output**:
```
● apache2.service - The Apache HTTP Server
   Loaded: loaded (/lib/systemd/system/apache2.service; enabled)
   Active: active (running) since ...
```

---

## Step 14: Test the Installation

### Test Frontend
Open your browser and navigate to:
```
http://m247.local/
```

### Test Admin Panel
Navigate to:
```
http://m247.local/admin
```

### Check Logs if Issues Occur

**Apache Error Log**:
```bash
tail -f /var/www/html/m247/var/log/apache_error.log
```

**Magento System Log**:
```bash
tail -f /var/www/html/m247/var/log/system.log
```

**Magento Exception Log**:
```bash
tail -f /var/www/html/m247/var/log/exception.log
```

---

## Troubleshooting

### Issue 1: "404 Not Found" on Homepage

**Solution**:
- Verify `DocumentRoot` points to `/var/www/html/m247/pub` (not `/var/www/html/m247`)
- Check that `.htaccess` file exists in `pub/` directory
- Ensure `AllowOverride All` is set in virtual host configuration
- Restart Apache

### Issue 2: Admin Panel Redirects to HTTPS

**Solution**:
- Run: `bin/magento config:set web/secure/use_in_adminhtml 0`
- Run: `bin/magento config:set web/secure/base_url "http://m247.local/"`
- Clear cache: `bin/magento cache:flush`

### Issue 3: Static Files Not Loading

**Solution**:
- Deploy static content: `bin/magento setup:static-content:deploy -f`
- Set permissions: `chmod -R 777 pub/static/`
- Clear browser cache

### Issue 4: "Allowed memory size exhausted" Error

**Solution**:
- Edit `pub/.htaccess` and increase: `php_value memory_limit 2G`
- Or edit PHP configuration: `/etc/php/8.1/apache2/php.ini`
- Restart Apache after changes

### Issue 5: Database Connection Error

**Solution**:
- Verify database credentials in `app/etc/env.php`
- Test MySQL connection: `mysql -u root -p -h localhost`
- Check that database name is correct: `m247`

### Issue 6: Compilation Errors After Changes

**Solution**:
```bash
# Remove generated files
rm -rf generated/code/* generated/metadata/*

# Recompile DI
bin/magento setup:di:compile

# Deploy static content
bin/magento setup:static-content:deploy -f

# Clear cache
bin/magento cache:flush
```

---

## Quick Reference Commands

### Magento Commands (run from /var/www/html/m247)
```bash
# Check module status
bin/magento module:status

# Run setup upgrade (after module/DB changes)
bin/magento setup:upgrade

# Recompile dependency injection
bin/magento setup:di:compile

# Deploy static content
bin/magento setup:static-content:deploy -f

# Clear cache
bin/magento cache:flush

# Check indexer status
bin/magento indexer:status

# Reindex all
bin/magento indexer:reindex

# Check cron jobs
bin/magento cron:run

# Show current mode
bin/magento deploy:mode:show

# Set developer mode
bin/magento deploy:mode:set developer
```

### Apache Commands
```bash
# Test configuration
sudo apache2ctl -t

# Show virtual hosts
sudo apache2ctl -S

# Restart Apache
sudo systemctl restart apache2

# Check Apache status
sudo systemctl status apache2

# View Apache modules
apache2ctl -M

# Enable a module
sudo a2enmod [module_name]

# Disable a module
sudo a2dismod [module_name]
```

### File Permission Commands
```bash
# Get current user's primary group
CURRENT_GROUP=$(id -gn)

# Add current user to www-data group and vice versa
sudo usermod -a -G www-data $USER
sudo usermod -a -G $CURRENT_GROUP www-data

# Set ownership to current user and current group
sudo chown -R $USER:$CURRENT_GROUP /var/www/html/m247

# Set writable directories for development (775 instead of 777)
sudo chmod -R 775 /var/www/html/m247/var/ /var/www/html/m247/generated/ /var/www/html/m247/pub/static/ /var/www/html/m247/pub/media/

# Give group write permissions
sudo chmod -R g+w /var/www/html/m247/var/ /var/www/html/m247/generated/ /var/www/html/m247/pub/static/ /var/www/html/m247/pub/media/

# Make magento CLI executable
sudo chmod u+x /var/www/html/m247/bin/magento

# Note: Log out and log back in for group changes to take effect
```

---

## Enabling SSL/HTTPS (Optional - Production Setup)

If you want to enable SSL for production, follow these additional steps:

### 1. Generate Self-Signed Certificate (Development)
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/m247.local.key \
  -out /etc/ssl/certs/m247.local.crt
```

### 2. Enable SSL Module
```bash
sudo a2enmod ssl
```

### 3. Create SSL Virtual Host
Add to `/var/www/html/m247/proxy-le-ssl.conf`:

```apache
<VirtualHost *:443>
    ServerName m247.local
    ServerAlias www.m247.local
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/m247/pub

    <Directory /var/www/html/m247/pub>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Require all granted
    </Directory>

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/m247.local.crt
    SSLCertificateKeyFile /etc/ssl/private/m247.local.key

    <IfModule mod_rewrite.c>
        RewriteEngine On
    </IfModule>

    ErrorLog /var/www/html/m247/var/log/apache_error.log
    CustomLog /var/www/html/m247/var/log/apache_access.log combined

    SetEnv MAGE_MODE "developer"
</VirtualHost>
```

### 4. Update Magento Base URLs for HTTPS
```bash
bin/magento setup:store-config:set --base-url-secure="https://m247.local/"
bin/magento config:set web/secure/use_in_frontend 1
bin/magento config:set web/secure/use_in_adminhtml 1
bin/magento cache:flush
```

### 5. Restart Apache
```bash
sudo systemctl restart apache2
```

---

## Configuration Summary

### URLs
- **Frontend**: http://m247.local/
- **Admin Panel**: http://m247.local/admin
- **Static Files**: http://m247.local/static/
- **Media Files**: http://m247.local/media/

### Key Directories
- **Magento Root**: `/var/www/html/m247`
- **Public Directory**: `/var/www/html/m247/pub`
- **Configuration**: `/var/www/html/m247/app/etc/env.php`
- **Logs**: `/var/www/html/m247/var/log/`
- **Virtual Host**: `/var/www/html/m247/proxy-le-ssl.conf`

### Key Files Modified
1. `/var/www/html/m247/app/etc/env.php` - Added website_configuration
2. `/var/www/html/m247/pub/index.php` - Added multi-store routing (lines 28-43)
3. `/var/www/html/m247/pub/get.php` - Added multi-store routing (lines 89-121)
4. `/var/www/html/m247/proxy-le-ssl.conf` - Created virtual host configuration
5. `/etc/hosts` - Added domain mapping
6. `/etc/apache2/sites-available/000-default.conf` - Included virtual host

### Current Settings
- **HTTPS Enforcement**: Disabled (0)
- **Base URL**: http://m247.local/
- **Magento Mode**: developer
- **Multi-Store**: Enabled via domain routing

---

## Notes

- This configuration is for **development environment** only
- For **production**, use proper SSL certificates from Let's Encrypt or commercial CA
- Adjust file permissions for production (avoid `chmod 777`)
- Consider using Redis or Varnish for caching in production
- Regular backups of database and files are recommended
- Keep Magento and all extensions updated for security

---

## Support

For issues:
1. Check Apache error logs: `tail -f /var/www/html/m247/var/log/apache_error.log`
2. Check Magento logs: `tail -f /var/www/html/m247/var/log/system.log`
3. Verify Apache syntax: `sudo apache2ctl -t`
4. Check virtual host configuration: `sudo apache2ctl -S`
5. Verify PHP errors: `tail -f /var/log/apache2/error.log`

---

**Last Updated**: 2025-11-10
**Magento Version**: 2.4.6-p10
**PHP Version**: 8.1.33
**Apache Version**: 2.4.x
**Domain**: m247.local
