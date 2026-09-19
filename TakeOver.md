# TakeOver

**Difficulty:** Easy  
**Room:** [TakeOver](https://tryhackme.com/room/takeover)

## Recon

The room gives us the domain `futurevera.thm`, so the first thing I did was add the target IP to `/etc/hosts`.

```bash
sudo nano /etc/hosts
```

I added:

```text
MACHINE_IP futurevera.thm
```

The website itself didn't give us much, so I moved on to enumerating subdomains.

I started with Gobuster:

```bash
gobuster vhost -u https://futurevera.thm -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -k --append-domain
```

The `-k` option is important here because the target uses HTTPS with a certificate that isn't trusted by our machine. It tells Gobuster to ignore certificate validation errors.

The scan revealed interesting virtual hosts, including:

```text
blog.futurevera.thm
support.futurevera.thm
```

Other writeups also report `portal` and `payroll` appearing depending on the wordlist/scan configuration, so the exact enumeration results can vary between runs. The important subdomains for the intended path are `blog` and `support`.

I added the useful subdomains to `/etc/hosts`:

```text
MACHINE_IP blog.futurevera.thm
MACHINE_IP support.futurevera.thm
```

Now I could access them normally through the browser.

## Finding Something Interesting

The `blog` subdomain didn't give me anything particularly useful, so I moved on to:

```text
https://support.futurevera.thm
```

This immediately gave me a certificate-related error.

At first, this looks like an annoyance, but it's actually useful.

Instead of just ignoring the certificate, I inspected it.

## Inspecting the Certificate

When viewing the certificate details, I checked the **Subject Alternative Name (SAN)** / DNS names.

This revealed another hostname:

```text
secrethelpdesk934752.support.futurevera.thm
```

This was the important discovery.

The hostname wasn't something I would reasonably expect to find just by guessing subdomains, which is why checking certificates is useful during reconnaissance. TLS certificates can contain additional hostnames that the server is configured to handle.

I added the newly discovered hostname to `/etc/hosts`:

```text
MACHINE_IP secrethelpdesk934752.support.futurevera.thm
```

Then I visited:

```text
https://secrethelpdesk934752.support.futurevera.thm
```

## Getting the Flag

At this point, the interesting thing is what the hidden hostname does rather than what the webpage looks like.

Using HTTP instead of HTTPS reveals the redirect:

```bash
curl -i http://secrethelpdesk934752.support.futurevera.thm
```

The server responds with a `302 Found` redirect containing:

```text
Location: http://flag{beea0d6edfcee06a59b83fb50ae81b2f}.s3-website-us-west-3.amazonaws.com/
```

And there it is.

The flag is actually exposed inside the redirect URL.

## Flag

```text
flag{beea0d6edfcee06a59b83fb50ae81b2f}
```

## What I Learned

This room was much less about exploiting a service and much more about **proper enumeration**.

The important chain was:

```text
futurevera.thm
      ↓
Subdomain enumeration
      ↓
support.futurevera.thm
      ↓
Inspect TLS certificate
      ↓
secrethelpdesk934752.support.futurevera.thm
      ↓
Inspect HTTP response
      ↓
Flag in redirect
```

The biggest lesson for me was **don't stop when something looks like an error**.

The certificate error initially looked like something I needed to work around. But after getting past it, the certificate itself contained another hostname. That hostname eventually led directly to the flag.

### Tools Used

|Tool|Purpose|
|---|---|
|`Gobuster`|Virtual-host/subdomain enumeration|
|Browser|Inspecting the website and TLS certificate|
|`/etc/hosts`|Mapping discovered hostnames to the target IP|
|`curl`|Inspecting the HTTP response and redirect|
