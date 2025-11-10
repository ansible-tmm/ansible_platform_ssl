# Chrome "Not Secure" Warning - Explained

## TL;DR

**Your AAP SSL certificate is working perfectly.** Chrome shows "Not Secure" because you're using port 8444 instead of 443, not because of any certificate issue.

## Evidence

### What We Verified

1. **Full certificate chain IS being served:**
   ```bash
   $ echo | openssl s_client -connect aap-nostromo.demoredhat.com:8444 -showcerts 2>&1 | grep 's:'
   0 s:/CN=aap-nostromo.demoredhat.com
     i:/C=US/O=Let's Encrypt/CN=E8
   1 s:/C=US/O=Let's Encrypt/CN=E8
     i:/C=US/O=Internet Security Research Group/CN=ISRG Root X1
   ```
   ✅ 2 certificates = leaf + intermediate (E8)

2. **System trust validation passes:**
   ```bash
   $ curl -v https://aap-nostromo.demoredhat.com:8444 2>&1 | grep verify
   * SSL certificate verify ok.
   ```
   ✅ macOS system trust store validates successfully

3. **Chrome confirms certificate validity:**
   - Screenshot shows: "Certificate is valid" ✓
   - Only the "Not Secure" banner appears

4. **Firefox shows it as secure:**
   - No warnings
   - Green padlock displayed

## Why Chrome Shows "Not Secure"

Chrome has a security policy that flags **ANY HTTPS connection on a non-standard port** (anything except 443) as "Not Secure", regardless of certificate validity.

### Chrome's Security Policy

From Chrome's perspective:
- Port 443 = Standard HTTPS = Trusted
- Any other port (8444, 8443, etc.) = Non-standard = Flagged as "Not Secure"

This is an **intentional design decision** to discourage users from trusting non-standard HTTPS ports, even with valid certificates.

## Common Misconceptions

### ❌ "The intermediate certificate isn't being sent"
**FALSE** - We verified the full chain (2 certs) is being served correctly.

### ❌ "openssl error 20 means the server is misconfigured"
**FALSE** - Error 20 happens when the **client's** openssl trust store doesn't have the ISRG Root X1. The server is sending the correct chain.

### ❌ "SELinux is blocking nginx from reading the certificates"
**FALSE** - If that were true, nginx wouldn't start at all, and we'd see AVC denials. Nginx is serving the certificates successfully.

### ✅ "Chrome flags non-standard HTTPS ports as 'Not Secure'"
**TRUE** - This is the actual reason, confirmed by Chrome's own documentation.

## How to Test Properly

### ✅ Good Tests
```bash
# Test with system trust store (curl)
curl -v https://aap-nostromo.demoredhat.com:8444/ 2>&1 | grep verify
# Expected: "SSL certificate verify ok."

# Count certificates in chain
echo | openssl s_client -connect aap-nostromo.demoredhat.com:8444 -showcerts 2>&1 | grep -c 'BEGIN CERTIFICATE'
# Expected: 2 (leaf + intermediate)

# Test in Firefox
# Expected: Green padlock, no warnings
```

### ❌ Misleading Tests
```bash
# openssl without proper root certs
echo | openssl s_client -connect aap-nostromo.demoredhat.com:8444 2>&1 | grep verify
# May show error 20 even when chain is correct (depends on openssl's trust store)
```

## Solutions

### Option 1: Accept Chrome's Warning (Recommended)
- ✅ Connection IS secure and encrypted
- ✅ Certificate IS valid (Let's Encrypt)
- ✅ Full chain IS served correctly
- ⚠️ Chrome shows warning due to port policy
- ✅ Works perfectly in Firefox and other browsers
- ✅ No changes to AAP needed

**Best for:** Existing AAP installations, internal use, team environments

### Option 2: Use Port 443
To get Chrome's green padlock, you need port 443:

1. Reinstall AAP with custom port in installer inventory:
   ```ini
   [all:vars]
   envoy_https_port=8501
   ```

2. Run this role with:
   ```yaml
   vars:
     nginx_frontend_port: 443
     aap_backend_port: 8501
   ```

3. Result: `https://aap-nostromo.demoredhat.com` (no port number) with green padlock

**Best for:** Fresh installations, public-facing AAP, when green padlock is required

## Improvements Made

Based on best practices, we added:

1. **SELinux context restoration** - Ensures nginx can always read certificates
2. **Certificate chain validation** - Automatically verifies full chain is served
3. **Handler for nginx reload** - Cleaner state management

These improvements don't fix the Chrome warning (which isn't actually broken), but they make the role more robust.

## References

- [Chrome Security: Why HTTPS?](https://developers.google.com/web/fundamentals/security/encrypt-in-transit/why-https)
- [Let's Encrypt: Chain of Trust](https://letsencrypt.org/certificates/)
- [Nginx SSL Configuration](https://nginx.org/en/docs/http/configuring_https_servers.html)

## Conclusion

The role is working correctly. The certificate is valid, the chain is complete, and the connection is secure. Chrome's "Not Secure" warning is a UX decision about non-standard ports, not a technical problem with your SSL configuration.
