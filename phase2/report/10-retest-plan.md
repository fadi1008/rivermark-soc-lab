# Retest Plan

## F-02 retest — Exposed SSH key
1. `sudo -u www-data find /var/www -name 'id_*' 2>/dev/null` → expect no output.
2. Attempt SSH from the container's www-data as `rivermark@<target>` using any leftover key material → expect `Permission denied (publickey)`.
3. From the SOC side, confirm the old key fingerprint no longer appears in accepted authentications.

## F-04 retest — Sudo Vim
1. `sudo -l -U rivermark` → expect no NOPASSWD rule.
2. `sudo vim -c ':!/bin/sh'` → expect failure ("a password is required").