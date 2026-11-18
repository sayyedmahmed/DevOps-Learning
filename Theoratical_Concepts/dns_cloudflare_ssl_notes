# DNS, Cloudflare, SSL and Wildcard SSL -- Quick Revision Notes

## 1. What is DNS?

**DNS (Domain Name System)** works like the internet's phonebook. It
translates a domain name into an IP address so a browser can find the
correct server.

Example:

-   You enter `example.com` in your browser.
-   DNS finds the IP address associated with `example.com`.
-   Your browser connects to that server.

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

  **MX**                  Specifies which mail    `example.com → mail server`
                          servers receive email   
                          for a domain.           

  **TXT**                 Stores text used for    SPF, DKIM, domain verification
                          verification and email  
                          security.               

  **NS**                  Identifies the          Nameserver records
                          authoritative DNS       
                          servers for a domain.   
  ------------------------------------------------------------------------------

## 3. What is Cloudflare?

**Cloudflare** is a service that helps manage a domain's DNS, improve
website performance, and protect websites.

Common Cloudflare features:

-   **DNS management:** Add and manage DNS records.
-   **CDN:** Delivers website content from servers closer to visitors.
-   **DDoS protection:** Helps protect websites from traffic-based
    attacks.
-   **SSL/TLS:** Helps secure connections between visitors and websites.
-   **Proxy:** Can route website traffic through Cloudflare for
    supported services.

### Cloudflare DNS Proxy

Cloudflare DNS records can have two common proxy settings:

-   **Proxied (orange cloud):** Web traffic passes through Cloudflare.
    Cloudflare can provide features such as caching and protection.
-   **DNS only (grey cloud):** DNS returns the origin server's address;
    traffic goes directly to that server.

Not every DNS record should be proxied. For example, mail records should
generally be DNS only.

## 4. What is an SSL/TLS Certificate?

An **SSL certificate** is used to secure the connection between a user's
browser and a website.

It:

-   Enables HTTPS.
-   Encrypts information sent between the browser and website.
-   Helps the browser verify the website's identity.

**Note:** SSL is the older name. Modern websites use TLS, but people
still commonly say "SSL certificate."

Example:

-   Without HTTPS: `http://example.com`
-   With HTTPS: `https://example.com`

## 5. Normal SSL vs Wildcard SSL

### Normal SSL Certificate

A normal certificate covers the specific domain name(s) listed in the
certificate.

Example:

-   `example.com`

It does not automatically cover `www.example.com` unless that name is
also included in the certificate.

### Wildcard SSL Certificate

A wildcard certificate uses `*` to cover multiple first-level subdomains
of a domain.

Example certificate name:

-   `*.example.com`

It can cover:

-   `www.example.com`
-   `mail.example.com`
-   `app.example.com`

A wildcard certificate does **not** automatically cover:

-   `example.com` (the root domain)
-   `dev.app.example.com` (a deeper subdomain)

To secure both the root domain and its subdomains, the certificate
should include both `example.com` and `*.example.com`.

### Quick Comparison

  -----------------------------------------------------------------------
  Normal SSL                          Wildcard SSL
  ----------------------------------- -----------------------------------
  Covers the specific name(s) listed  Covers first-level subdomains
  in the certificate.                 matching `*.example.com`.

  Additional names may need to be     One certificate can cover many
  added to the certificate.           matching subdomains.

  Can be validated using HTTP-01 or   Let's Encrypt wildcard certificates
  DNS-01 with Let's Encrypt.          require DNS-01 validation.
  -----------------------------------------------------------------------

**Simple analogy:**

-   Normal SSL: A security lock for one named office.
-   Wildcard SSL: A security arrangement for the branch offices under
    one main domain.

## 6. How to Obtain a Wildcard SSL Certificate

A common free option is **Let's Encrypt**, with **Certbot** used to
request and manage certificates.

Typical process:

1.  Install Certbot on the server.
2.  Request a certificate for `*.example.com` and, if needed,
    `example.com`.
3.  Prove domain ownership using DNS-01 validation.
4.  Add the TXT record requested by Certbot to your DNS provider (for
    example, Cloudflare).
5.  Wait for DNS to update, then complete validation.
6.  Install the certificate on the web server.
7.  Configure automatic renewal and test it.

### Example Certbot Command

``` bash
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  -d "*.example.com" \
  -d "example.com"
```

Certbot will display a TXT record name and value. Add the exact details
it provides to Cloudflare DNS.

**Important:** This manual method may require you to repeat the DNS step
when renewing. For automatic renewal, use a supported DNS plugin or
another automated DNS-01 method.

## 7. Cloudflare DNS Setup for Wildcard Validation

When Certbot requests DNS-01 validation, it asks you to create a TXT
record.

Example format (use the exact value Certbot gives you):

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
7.  Complete the Certbot validation.

The TXT record proves that you control the domain. It does not direct
website traffic.

## 8. Web Server Configuration

After obtaining the certificate, configure your web server to use the
certificate and private key.

### Nginx

Common site configuration location on Ubuntu/Debian:

``` text
/etc/nginx/sites-available/example.com
```

Example:

``` nginx
server {
    listen 443 ssl;
    server_name example.com *.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

Test and reload Nginx:

``` bash
sudo nginx -t
sudo systemctl reload nginx
```

### Apache

Common site configuration location on Ubuntu/Debian:

``` text
/etc/apache2/sites-available/example.com.conf
```

Example:

``` apache
<VirtualHost *:443>
    ServerName example.com
    ServerAlias *.example.com

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem
</VirtualHost>
```

Test and reload Apache:

``` bash
sudo apachectl configtest
sudo systemctl reload apache2
```

The certificate must contain the domain names configured in the web
server. DNS records and web server virtual-host configuration must also
direct each subdomain to the correct service.

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

  `privkey.pem`                       Private key; keep it secure and
                                      never share it.

  `cert.pem`                          The domain certificate.

  `chain.pem`                         Intermediate certificate chain.
  -----------------------------------------------------------------------

Renewal configuration is commonly stored under:

``` text
/etc/letsencrypt/renewal/
```

## 10. Useful Commands

Check Nginx configuration:

``` bash
sudo nginx -t
```

Reload Nginx:

``` bash
sudo systemctl reload nginx
```

Check Apache configuration:

``` bash
sudo apachectl configtest
```

Reload Apache:

``` bash
sudo systemctl reload apache2
```

Check Certbot certificates:

``` bash
sudo certbot certificates
```

Test certificate renewal:

``` bash
sudo certbot renew --dry-run
```

## 11. Key Points to Remember

-   **DNS** translates domain names into records such as IP addresses.
-   **Cloudflare** can manage DNS and provide proxy, performance, and
    security features.
-   **SSL/TLS** encrypts the connection between a browser and a website.
-   **Normal SSL** covers the exact domain names listed in the
    certificate.
-   **Wildcard SSL** covers matching first-level subdomains.
-   **DNS-01 validation** is required to obtain a Let's Encrypt wildcard
    certificate.
-   A wildcard certificate does not create DNS records or configure
    websites automatically.
-   Protect the private key (`privkey.pem`).
-   Configure certificate renewal and test it.
