# syntax=docker/dockerfile:1.7
# Hardened WordPress image - DevSecOps lab
# Build:  docker build -t deiangg/devsecops:hardened -f docker/Dockerfile docker
# Base image pins to a patched WordPress + PHP version (closes all 46 CVEs found in baseline 5.7.0).
FROM wordpress:6.8.3-php8.3-apache

LABEL org.opencontainers.image.title="DevSecOps WordPress (Hardened)" \
      org.opencontainers.image.description="Hardened WordPress image: patched core, secure PHP/Apache config, reduced surface." \
      org.opencontainers.image.source="https://github.com/DeianGG/DevSecOps" \
      org.opencontainers.image.licenses="MIT"

# ---- 1. System hardening: remove unnecessary tooling --------------------------
RUN set -eux; \
    apt-get update; \
    apt-get purge -y --auto-remove \
        wget \
        less \
        nano \
        vim-tiny \
        || true; \
    apt-get clean; \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# ---- 2. PHP & Apache security configuration ----------------------------------
COPY hardening/security.ini /usr/local/etc/php/conf.d/zz-security.ini
COPY hardening/security.conf /etc/apache2/conf-available/zz-security-hardening.conf
RUN a2enconf zz-security-hardening && a2enmod headers rewrite

# ---- 3. Remove version-disclosure files & unused themes ----------------------
RUN set -eux; \
    rm -f /usr/src/wordpress/readme.html /usr/src/wordpress/license.txt; \
    rm -f /usr/src/wordpress/wp-content/plugins/hello.php; \
    rm -rf /usr/src/wordpress/wp-content/plugins/akismet; \
    find /usr/src/wordpress/wp-content/themes -mindepth 1 -maxdepth 1 -type d \
        ! -name "twentytwentyfour" -exec rm -rf {} +

# ---- 4. Inject hardened WP constants via WORDPRESS_CONFIG_EXTRA --------------
# These get appended to wp-config.php by the upstream entrypoint at runtime.
ENV WORDPRESS_CONFIG_EXTRA="\
define('DISALLOW_FILE_EDIT', true);\
define('DISALLOW_FILE_MODS', true);\
define('AUTOMATIC_UPDATER_DISABLED', false);\
define('WP_AUTO_UPDATE_CORE', 'minor');\
define('FORCE_SSL_ADMIN', false);\
define('WP_DEBUG', false);\
define('WP_DEBUG_DISPLAY', false);\
define('DISABLE_WP_CRON', false);\
define('WP_POST_REVISIONS', 5);\
define('EMPTY_TRASH_DAYS', 7);\
"

# ---- 5. File permissions ------------------------------------------------------
RUN set -eux; \
    chown -R www-data:www-data /usr/src/wordpress /var/www/html; \
    find /usr/src/wordpress -type d -exec chmod 0755 {} \; ; \
    find /usr/src/wordpress -type f -exec chmod 0644 {} \;

# ---- 6. Switch Apache to unprivileged port 8080 so the whole container ------
#         can run as non-root (no CAP_NET_BIND_SERVICE needed).
RUN set -eux; \
    sed -ri 's!^Listen 80$!Listen 8080!' /etc/apache2/ports.conf; \
    sed -ri 's!:80>!:8080>!' /etc/apache2/sites-enabled/000-default.conf; \
    # Apache runtime directories must be writable by www-data
    mkdir -p /var/run/apache2; \
    chown -R www-data:www-data /var/run/apache2 /var/log/apache2 /var/lock

# ---- 7. Healthcheck -----------------------------------------------------------
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -fsS -o /dev/null http://localhost:8080/wp-login.php || exit 1

# Drop privileges. Apache binds 8080 (unprivileged), entrypoint runs as www-data.
USER www-data
EXPOSE 8080
