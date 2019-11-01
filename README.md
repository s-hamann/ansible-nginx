Nginx
=====

This role sets up nginx as a web server.
PHP support can be enabled using PHP-FPM.

Deploying web content is outside the scope of this role.

Requirements
------------

If TLS encryption (i.e. HTTPS) is desired, the target system needs to have a suitable X.509 certificate.
This roles does not handle deploying certificates.

Role Variables
--------------

* `nginx_extra_groups`  
  A list of groups that the web server system user is added to.
  This allows granting access to additional resources, such as the private key file.
  All groups need to exist on the target system; this role does not create them.
  Empty by default.
* `nginx_extra_packages`  
  A list of packages that should be installed in addition to those deemed necessary by this role.
  This can be used, for example, to ensure that certain PHP extensions are present.
  Empty by default.
* `nginx_extra_use_flags`  
  A dictionary with additional use flags.
  The following keys are valid:
    * `nginx`  
    * `php`  
  Each key holds a list of USE flags to enable or disable for the respective software in addition to those deemed necessary by this role.
  This can be used, for example, to ensure that certain PHP extensions are present.
  Empty by default.
  Only used on Gentoo.
* `nginx_php_version`  
  The PHP version slot to use, i.e. the major and minor version only.
  If not set, the latest version in portage that is not masked (by keyword, package mask, ...) is used.
  Only used on Gentoo.
* `nginx_writable_paths`  
  If the target system uses systemd, the file system is mostly read-only to nginx (and PHP).
  A list of paths that should be writable can be given in `nginx_writable_paths`.
  This typically includes upload paths.
  Directories containing log files (defined by the appropriate variables) are automatically made writable.
  By default, no paths are writable.
* `nginx_inaccessible_paths`  
  If the target system uses systemd, this option takes a list of paths, that should not be accessible at all for nginx (and PHP).
  Regardless of this option, home directories are made inaccessible and the directories holding TLS private keys are made inaccessible for PHP.
  Optional.
* `nginx_config_template`  
  An alternative jinja2 template to generate `nginx.conf`.
  The template may extend the default `nginx.conf.j2`, overriding only certain blocks.
  Optional.
* `nginx_worker_processes`  
  The number of worker processes nginx uses to serve requests.
  Defaults to the number virtual CPUs detected by Ansible.
* `nginx_tls13_only`  
  If set to `true`, nginx is configured to support only TLSv1.3.
  The target system needs to have OpenSSL 1.1 or newer for this to work.
  If set to `false` (the default) TLSv1.2 is supported in addition to TLSv1.3.
* `nginx_ciphers`  
  A string describing the enabled cipher suites as understood by OpenSSL.
  The default contains only strong AEAD ciphers with PFS.
* `nginx_max_body_size`  
  Maximum allowable body size in MB.
  In particular, this limits the maximum size of file uploads.
  Default is `1`, which is usually more than enough for sites that do not accept file uploads.
* `nginx_keepalive_timeout`  
  The timeout (in seconds) for HTTP keep-alive connections.
  Defaults to `65` seconds.
* `nginx_gzip_static`  
  Enable support for pre-compressed static responses.
  This setting does not globally enable `gzip_static` in nginx, but does it in a location for certain compressible file types.
  A script to generate or update the statically compressed files in all defined web roots is installed as `/usr/local/sbin/update_gzip_static.sh`.
  The handler `nginx_update_gzip_static` may be notified to run this script.
  `gzip_static` can  be enabled for custom locations as well, if this is desired.
  Defaults to `true`.
* `nginx_gzip_dynamic`  
  Enable dynamic response compression.
  Only responses with certain content types and a minimal length are compresses in order to avoid a negative impact on performance.
  Defaults to `true`.
* `nginx_access_log`  
  The global access log file.
  This can be overwritten per vhost.
  Defaults to `false`, which disables logging on the global level.
* `nginx_access_log_options`  
  Options to the `access_log` directive, such as `buffer` (cf. the [nginx documentation](https://nginx.org/en/docs/http/ngx_http_log_module.html#access_log)).
  Optional.
* `nginx_log_format`  
  A custom access log format specification.
  Refer to the [nginx documentation](https://nginx.org/en/docs/http/ngx_http_log_module.html#log_format) on `log_format` for information on what to put here.
  Optional.
* `nginx_error_log`  
  The global log file for errors.
  This can be overwritten per vhost.
  If not set, logs go to the systemd journal or syslog, as appropriate.
* `nginx_ocsp_stapling`  
  Enable or disable OCSP Stapling for all TLS-enabled vhosts.
  Defaults to `true`.
* `nginx_security_headers`  
  This role sets up a number of security relevant HTTP headers.
  `nginx_security_headers` is a dictionary that allows tuning their values from the defaults or adding new headers.
  Keys are header names and values are header values.
  Optional.
* `nginx_caching_policy`  
  A dictionary describing the caching policy for each content type.
  The keys are content types, such as `text/html`.
  Regular expressions are also allowed when prefixed with `~` (case-sensitive matching) or `~*` (case-insensitive matching).
  The special value `default` is for all responses that are not matched by any other key.
  Values are caching times as accepted by the `expires` directive (cf. the [nginx documentation](https://nginx.org/en/docs/http/ngx_http_headers_module.html#expires)).
* `nginx_extra_options`  
  A dictionary with additional options.
  The following keys are valid:
    * `nginx` (for the main part in `nginx.conf`)  
    * `nginx_http` (for the `http` block in `nginx.conf`)  
  Each key in turn holds a list of options for the respective configuration file or file section.
* `nginx_vhosts`  
  A list of dictionaries describing vhost configurations.
  Each vhost is written to a separate configuration file on the target system.
  For each list entry, the following keys are valid:
    * `filename`  
      The name of the file to hold the vhost configuration.
      Mandatory.
    * `template`  
      Path to the jinja2 template file for this vhost.
      See `nginx_config_template`.
      Optional.
    * `server_name`  
      A list of host names for this vhost.
      The first vhost for a `ip` and `port` combination is the default one and accepts all names not set on any other vhosts.
      Optional.
    * `ip`  
      The IP address this vhost listens to.
      This can be used to make nginx reachable on a specific network interface only.
      Defaults to `*`, i.e. all interfaces.
    * `port`  
      The TCP port on which this vhost is reachable.
      Defaults to `443` when `use_tls` is set and `80` otherwise.
    * `unencrypted_port`  
      When `use_tls` is set, this port is configured to accept unencrypted HTTP connection.
      All connections are redirected to HTTPS.
      Optional.
    * `use_tls`  
      Whether to enable TLS (i.e. HTTPS) for this vhost.
      Defaults to `false`.
    * `tls_cert`  
      Path to a PEM-encoded X.509 certificate for this vhost.
      The file needs to exist and be readable by the nginx user.
      Mandatory if `use_tls` is set, ignored otherwise.
    * `tls_cert_key`  
      Path to the PEM-encoded private key file for the certificate.
      The file needs to exist and be readable by the nginx user.
      Mandatory if `use_tls` is set, ignored otherwise.
    * `use_php`  
      Whether to enable PHP for this vhost.
      Defaults to `false`.
    * `root`  
      Path to the web root directory for this vhost.
      Mandatory, except in some special configurations.
    * `index`  
      Index file names, separated by spaces.
      Defaults to `index.php index.html index.htm` (where `index.php` is omitted if `use_php` is not set).
    * `security_headers`  
      A dictionary containing vhost-specific overrides of `nginx_security_headers`.
      Optional.
    * `disabled_security_headers`  
      A list of header names that should *not* be set for this vhost.
      Optional.
    * `error_page`  
      A list of error pages for specific status codes.
      Refer to `error_page` in the [nginx documentation](https://nginx.org/en/docs/http/ngx_http_core_module.html#error_page) for information on the format.
      Optional.
    * `access_log`  
      Sets the access log for this vhost.
      This can be overwritten in locations.
      If not set, the global access log file (cf. `nginx_access_log`) is used.
    * `error_log`  
      Sets the error log for this vhost.
      If not set, the global error log file (cf. `nginx_error_log`) is used.
    * `static_assets`  
      A regular expression that matches all URLs of static assets.
      The default value should match most common file types.
    * `log_static_assets`  
      If set to `false`, access to static assets (cf. `static_assets`) is not logged.
      Defaults to `true`.
    * `extra_options`  
      A list of additional configuration options for this vhost.
      Optional.
    * `php_extra_options`  
      A list of additional configuration options for the location block that handles PHP URLs.
      Only used when `use_php` is `true`.
      Optional.
    * `blocked_locations`  
      A list of paths relative to the web root directory that should not be served via HTTP (a 404 is returned).
      The web server still has access to these files and may include them server-side.
      This is useful, for instance, for configuration files and packaging artifacts.
      The paths are interpreted as prefixes, i.e. all files and directories with these prefixes match.
      Locations beginning with `=` override this and can make blocked path available.
    * `locations`  
      A dictionary of location configurations.
      Keys are location matching patterns as understood by nginx (cf. the `location` directive in the [nginx documentation](https://nginx.org/en/docs/http/ngx_http_core_module.html#location)).
      In short, they are matched against the URL relative to the web root directory and may be a prefix string (possibly prefixed with `=` or `^~`) or a regular expression (prefixed with `~` or `~*`).
      Each location key in turn holds a dictionary containing the configuration for this location.
      The following keys are valid for locations:
      * `return`  
        This can be used to always return a specific status code.
        See `return` in the [nginx documentation](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#return) for further information.
      * `gzip_dynamic`  
        Specifically enable or disable dynamic response compression for this location. Boolean.
        See also: `nginx_gzip_dynamic`.
      * `gzip_static`  
        Specifically enable or disable support for pre-compressed responses. Boolean.
        See also: `nginx_gzip_static`.
      * `access_log`  
        Sets the access log for this location.
        If not set, the vhost's access log file is used.
      * `extra_options`  
        A list of additional configuration optional options for this location.

The following options are specific to PHP and are only used if at least one vhost has `use_php` set.
* `nginx_php_process_manager`  
  What process manager to use for PHP's resource pool.
  One of `static`, `dynamic` or `ondemand`.
  The default is `dynamic`.
* `nginx_php_max_children`  
  The maximum number of PHP processes to run simultaneously.
  This is the limit for the number of requests that can be served simultaneously by PHP.
  The default is `5`, but should usually be changed.
* `nginx_php_min_spare_children`, `nginx_php_max_spare_children`  
  If `nginx_php_process_manager` is `dynamic`, PHP keeps a number of spare processes to quickly handle new connections.
  These two settings put a limit on the number of spare processes.
  When using a different process manager, these settings have no effect.
  The defaults are `1` and `3`, respectively, but should usually be changed.
* `nginx_php_process_idle_timeout`  
  If `nginx_php_process_manager` is `ondemand`, keeps idle processes alive to serve further requests.
  This sets the time (in seconds) after which idle processes are killed to free up resources.
  This setting is ignored when using a different process manager.
  The default is `10` seconds.
* `nginx_php_max_requests`  
  Respawn PHP processes after handling this number of requests.
  The default value `0` disables this.
* `nginx_php_timeout`  
  The maximum time (in seconds) for PHP requests to be processed.
  Defaults to `30` seconds.
* `nginx_php_status_path`  
  URL-path to the PHP status page, which displays information about the FPM pool.
  The status page will be made available on every vhost that uses PHP and does not define a location with the same path.
  By default, this is not set, i.e. there is no status page.
* `nginx_php_ping_path`  
  URL-path to the PHP ping page.
  This page returns status code `200` and the body `pong` if PHP-FPM is working.
  The ping page will be made available on every vhost that uses PHP and does not define a location with the same path.
  By default, this is not set, i.e. there is no ping page.
* `nginx_php_monitor_ips`  
  A list of IP addresses that can access the ping and status pages.
  If this is not set, access is not restricted.
* `nginx_php_timezone`  
  The system's time zone.
  Can also be set using the variable `timezone`.
* `nginx_php_error_log`  
  The log file for PHP errors.
  Defaults to `syslog`.
* `nginx_php_enabled_functions`  
  This role disables a number of functions in PHP that are a common source of security issues or allow system access.
  To prevent breaking applications, `nginx_php_enabled_functions` can hold a list of functions that should explicitly not be disabled.
  Empty by default.
* `nginx_php_extensions`  
  A list of PHP extensions to load, such as `curl` or `gd`.
  Extensions that are not part of the PHP distribution are automatically installed using the target system's package management.
  This may not work in all cases.
  Empty by default.
* `nginx_php_zend_extensions`  
  A list of PHP Zend extensions to load, such as `opcache`.
  Extensions that are not part of the PHP distribution are automatically installed using the target system's package management.
  This may not work in all cases.
  Empty by default.
* `nginx_php_extra_options`  
  Additional configuration options for PHP.
  This variable is a dictionary with the following keys:
    * `php-fpm` (for the `global` section of `php-fpm.conf`)  
    * `php-fpm_pool` (for the pool section of `php-fpm.conf`)  
    * any other key is added to `php.ini` as a section named like the key  
  The values are in turn dictionaries where keys are PHP configuration options for the appropriate section and values are the corresponding configuration values.

Dependencies
------------

This role does not set up TLS certificates and therefore depends on a role that generates and deploys them, if HTTPS support is desired.

Example Configuration
---------------------

The following is short example for some of the configuration options this role provides:

```yaml
nginx_writable_paths:
  - '/var/www/example/uploads'
nginx_inaccessible_paths:
  - '/mnt/backups'
nginx_max_body_size: 16
nginx_security_headers:
  Content-Security-Policy: "default-src 'none'"
  X-Frame-Options: 'DENY'
nginx_caching_policy:
  default: off
  text/css: 7d
  ~image/: 1h
nginx_vhosts:
  - filename: example.conf
    use_tls: true
    unencrypted_port: 80
    tls_cert: "/etc/ssl/private/{{ inventory_hostname }}.pem"
    tls_cert_key: "/etc/ssl/private/{{ inventory_hostname }}.key"
    use_php: true
    root: '/var/www/example/'
    access_log: '/var/log/nginx/example/access.log'
    extra_options:
      - 'rewrite ^(/download/.*)/media/(.*)\..*$ $1/mp3/$2.mp3 last'
    blocked_locations:
      - '/config'
      - '/backups'
      - '\.zip$'
    locations:
      /doc:
        access_log: false
        gzip_static: true
        extra_options:
          - 'index index.html'
      ~\.log$:
        return: 403
nginx_php_enabled_functions:
  - 'phpinfo'
nginx_php_extensions:
  - 'curl'
  - 'gd'
nginx_php_extra_options:
  php-fpm:
    emergency_restart_threshold: 3
  php-fpm_pool:
    request_terminate_timeout: 10
  PHP:
    precision: 16
    variables_order: 'GPCS'
    allow_url_include: true
  bcmath:
    bcmath.scale: 16
  Session:
    session.name: 'SessionID'
```

License
-------

MIT
