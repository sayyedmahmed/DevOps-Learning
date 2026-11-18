# DNS, Cloudflare, SSL and Wildcard SSL 

## 1. What is DNS?

**DNS (Domain Name System)** works like the internet's phonebook. It
translates a domain name into information such as an IP address so a
browser can find the correct server.

Example: You enter `example.com`; DNS helps your browser find the server
for that domain.

## 2. Common DNS Record Types

  ------------------------------------------------------------------------------
  Record                  What it does            Example
  ----------------------- ----------------------- ------------------------------
  **A**                   Points a domain to an   `example.com → 192.0.2.10`
                          IPv4 address.           

  **AAAA**                Points a domain to an   `example.com → IPv6 address`
                          IPv6 address.           

  **CNAME**               Points a name to        `www → example.com`
                          another domain name.    

  **MX**                  Specifies the mail      `example.com → mail server`
                          servers that receive    
                          email.                  

  **TXT**                 Stores text for         SPF, DKIM, domain verification
                          verification and email  
                          security.               

  **NS**                  Identifies the          Nameserver records
                          authoritative DNS       
                          servers for a domain.   
  ------------------------------------------------------------------------------

## 3. What is Cloudflare?

**Cloudflare** helps manage a domain's DNS, improve website performance,
and protect websites.

Common features:

-   **DNS management:** Add and manage DNS records.
-   **CDN:** Delivers website content from servers closer to visitors.
-   **DDoS protection:** Helps protect websites from traffic-based
    attacks.
-   **SSL/TLS:** Helps secure connections between visitors and websites.
-   **Proxy:** Routes supported website traffic through Cloudflare.

### Cloudflare DNS Proxy

-   **Proxied (orange cloud):** Web traffic passes through Cloudflare.
    Features can include caching and protection.
-   **DNS only (grey cloud):** DNS returns the origin server's address;
    traffic goes directly to that server.

Mail records should generally be set to DNS only.

## 4. What is an SSL/TLS Certificate?

An **SSL certificate** helps secure the connection between a user's
browser and a website.

It: - Enables HTTPS. - Encrypts information sent between the browser and
website. - Helps the browser verify the website's identity.

Modern websites use TLS, the successor to SSL, but people still commonly
say "SSL certificate."

## 5. Normal SSL vs Wildcard SSL

### Normal SSL Certificate

A normal certificate covers the exact domain name(s) listed in the
certificate.

Example: `example.com`

It does not automatically cover `www.example.com` unless that name is
also included.

### Wildcard SSL Certificate

A wildcard certificate uses `*` to cover first-level subdomains.

Example: `*.example.com`

It can cover: - `www.example.com` - `mail.example.com` -
`app.example.com`

It does not automatically cover: - `example.com` (the root domain) -
`dev.app.example.com` (a deeper subdomain)

To secure both the root domain and its first-level subdomains, request a
certificate containing both `example.com` and `*.example.com`.

## 6. Normal SSL vs Wildcard SSL -- Configuration Process

The overall process is similar. The main differences are the certificate
names requested, the validation method, and the names configured on the
web server.

  -----------------------------------------------------------------------
  Step                    Normal SSL              Wildcard SSL
  ----------------------- ----------------------- -----------------------
  **1. Request            Request `example.com`   Request
  certificate**           (and any other required `*.example.com`;
                          names).                 include `example.com`
                                                  too if required.

  **2. Domain             Let's Encrypt can use   Let's Encrypt requires
  validation**            HTTP-01 or DNS-01       DNS-01 validation.
                          validation.             

  **3. DNS changes**      DNS changes may not be  Add the TXT record
                          needed for HTTP-01.     requested by Certbot to
                          DNS-01 requires a TXT   Cloudflare DNS.
                          record.                 

  **4. Certificate        Certbot creates         Same type of files are
  files**                 certificate and key     created, but the
                          files.                  certificate contains
                                                  wildcard coverage.

  **5. Web server         Configure the exact     Configure the root
  config**                domain name(s).         domain and wildcard
                                                  subdomains as required.

  **6. Renewal**          Configure and test      Configure and test
                          automatic renewal.      renewal; automated DNS
                                                  updates are needed for
                                                  unattended DNS-01
                                                  renewal.
  -----------------------------------------------------------------------

### Example: Requesting a Normal Certificate

``` bash
sudo certbot --nginx -d example.com
```

This example uses the Nginx plugin, which can obtain the certificate and
configure Nginx. It assumes the domain and web server are already set up
and reachable.

### Example: Requesting a Wildcard Certificate

``` bash
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  -d "*.example.com" \
  -d "example.com"
```

Certbot displays a DNS TXT record name and value. Add those exact
details to Cloudflare DNS, then complete validation.

**Important:** The manual DNS method may require manual work at renewal
time. For automatic renewal, use a supported Cloudflare DNS plugin or
another automated DNS-01 method.

## 7. Cloudflare DNS Setup for Wildcard Validation

For DNS-01 validation, Certbot asks you to create a TXT record.

Example format (use the exact value Certbot provides):

  Type   Name                Content
  ------ ------------------- ---------------------------
  TXT    `_acme-challenge`   Token provided by Certbot

In Cloudflare:

1.  Open your domain.
2.  Go to **DNS → Records**.
3.  Select **Add record**.
4.  Choose **TXT**.
5.  Enter the name and content provided by Certbot.
6.  Save the record.
7.  Complete Certbot validation.

The TXT record proves that you control the domain. It does not direct
website traffic.

## 8. Configuration Files -- Normal SSL vs Wildcard SSL

**Important:** The Nginx or Apache configuration file is not
fundamentally different for normal and wildcard certificates. In both
cases, you configure the certificate paths and the domain names the
server should handle. The certificate must contain every name used by
the server configuration.

### A. Nginx

Common site configuration location on Ubuntu/Debian:

``` text
/etc/nginx/sites-available/example.com
```

**Normal SSL example:**

``` nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
}
```

**Wildcard SSL example:**

``` nginx
server {
    listen 443 ssl;
    server_name example.com *.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
}
```

**What changed?** - `server_name` lists `example.com` for the normal
example. - The wildcard example lists `example.com` and
`*.example.com`. - The certificate paths are written in the same way.
The actual certificate files must contain the names being served.

### B. Apache

Common site configuration location on Ubuntu/Debian:

``` text
/etc/apache2/sites-available/example.com.conf
```

**Normal SSL example:**

``` apache
<VirtualHost *:443>
    ServerName example.com

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem
</VirtualHost>
```

**Wildcard SSL example:**

``` apache
<VirtualHost *:443>
    ServerName example.com
    ServerAlias *.example.com

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem
</VirtualHost>
```

**What changed?** - The wildcard configuration adds
`ServerAlias *.example.com`. - The certificate paths are configured in
the same way. - The certificate must include both `example.com` and
`*.example.com` if both are served.

### Configuration Comparison at a Glance

  -------------------------------------------------------------------------------------------------
  Item                    Normal SSL                     Wildcard SSL
  ----------------------- ------------------------------ ------------------------------------------
  Certificate request     `-d example.com`               `-d "*.example.com"` (and optionally
                                                         `-d example.com`)

  Validation              HTTP-01 or DNS-01              DNS-01 for Let's Encrypt

  Nginx domain setting    `server_name example.com;`     `server_name example.com *.example.com;`

  Apache domain setting   `ServerName example.com`       `ServerName example.com` plus
                                                         `ServerAlias *.example.com`

  Certificate/key         `ssl_certificate`,             Same directives
  directives              `ssl_certificate_key`          

  Certificate storage     `/etc/letsencrypt/live/.../`   Same directory structure
  -------------------------------------------------------------------------------------------------

## 9. Important Certificate Files

Certbot commonly stores certificate files under:

``` text
/etc/letsencrypt/live/example.com/
```

  -----------------------------------------------------------------------
  File                                Purpose
  ----------------------------------- -----------------------------------
  `fullchain.pem`                     Certificate plus intermediate
                                      certificates; commonly used by the
                                      web server.

  `privkey.pem`                       Private key. Keep it secure and
                                      never share it.

  `cert.pem`                          The domain certificate.

  `chain.pem`                         Intermediate certificate chain.
  -----------------------------------------------------------------------

Renewal configuration is commonly stored under:

``` text
/etc/letsencrypt/renewal/
```

## 10. Apply and Test Configuration Changes

For Nginx:

``` bash
sudo nginx -t
sudo systemctl reload nginx
```

For Apache:

``` bash
sudo apachectl configtest
sudo systemctl reload apache2
```

Check Certbot certificates:

``` bash
sudo certbot certificates
```

Test renewal:

``` bash
sudo certbot renew --dry-run
```

## 11. Key Points to Remember

-   **DNS** translates domain names into DNS records such as IP
    addresses.
-   **Cloudflare** can manage DNS and provide proxy, performance, and
    security features.
-   **SSL/TLS** encrypts the connection between a browser and a website.
-   **Normal SSL** covers the exact domain names listed in the
    certificate.
-   **Wildcard SSL** covers matching first-level subdomains.
-   Let's Encrypt wildcard certificates require **DNS-01 validation**.
-   Nginx and Apache use the same SSL directives for normal and wildcard
    certificates; the domain names and certificate contents differ.
-   A wildcard certificate does not create DNS records or configure
    websites automatically.
-   Protect `privkey.pem`.
-   Configure certificate renewal and test it.
