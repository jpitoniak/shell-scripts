# Email Server Utilities

This directory contains some utility scripts I use to maintain my email server.  I use Chasquid and Dovecot for my SMTP and IMAP/POP server, respectively, and these scripts ensure that I get all of the intricacies of adding new domains and user accounts or renewing TLS certificates correct.

These scripts are based off the setup I outline on my blog at the links below.  They assume a working installation of both Chasquid and Dovecot, following the instructions outlined in the first post.  [Lego](https://go-acme.github.io/lego/) is also required for obtaining TLS certificates using the [PowerDNS API](https://doc.powerdns.com/authoritative/http-api/index.html).

- [Setting Up a Mail Server with Chasquid and Dovecot – Part 1 The Servers](https://www.kodiakskorner.com/log/466)
- To Be Released

These scripts expect a config file, named `.emailconfig` to be located in the 'root' user's home directory (`/root`).  This file should contain the following:

```
export PDNS_API_URL=https://dnsapi.yourdoamin.com
export PDNS_API_KEY=YOUR_API_KEY
TLS_EMAIL=you@yorudomain.com
```

- PDNS_API_URL is the URL to access the PowerDNS API for your DNS server, which Lego needs to for Let's Encrypt certificate authorization.
- PDNS_API_KEY is the API key used to authenticate to the API
- TLS_EMAIL is the email address that will be passed to Let's Encrypt in certificate requests.

All of the scripts must be run as root or with `sudo`.  I typically install them to `/root/bin`, but `/usr/local/bin` may be a better place if you'll be using `sudo` to call them from non-privileged accounts.

## add-email-domain

Usage: `add-email-domain domain.tld`

Sets up the server to handle mail for the given domain.  This is the most complicated of the scripts.

- It sets up both servers to handle mail for the domain.
- It creates a DKIM signing key for mail sent from the domain.
- It obtains TLS certificates for mail.domain.tld to handle trusted, secure communications with the servers.

In addition to running the commands, you'll need to manually set up DNS records for the domain, including the MX, SPF, DKIM, etc. records.

## add-email-user

Usage: `add-email-user user@domain.tld`

Adds a new email account.

- It checks that the server is already configured to handle mail for the given domain.
- If so, it prompts for a password and creates the account for the user.

Once created, the user can connect to their account via `mail.domain.tld` using all of the standard mail services (SMTP, POP3, IMAP4) through standard secure ports.

## renew-email-certs

Checks for expiring certificates and renews them.

This script is designed to be run via crontab.  It should be run at least once a week, preferably at an "off" time so as not to overload Let's Encrypt's servers.

Use `sudo crontab -e` to edit root's crontab file and add the following:

```
15 4 * * 4 /root/renew-email-certs
```

This runs it at 4:15 am every Thursday.  It checks each certificate for it's expiration date and renews any that are less than 30 days from expiring.
