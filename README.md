# Ansible Collection - adfinis.dovecot

Colletion to deploy the [Dovecot](https://dovecot.org/) IMAP server.

## Requirements

- This collection is designed to work with Debian 10 or later.
    - The exact version of Debian that will work depends on the version of Dovecot is being deployed (Dovecot is installed from the upstream repositories).
    
## Dependencies

This collection does currently not depend on any additional collection.

## License

See [LICENSE](LICENSE)

## Author Information

The `adfinis.openxchange` collection was written by:

- [Adfinis AG](https://www.adfinis.com/) | [GitHub](https://github.com/adfinis)


## Usage

### The Bare Minimum

In this example, we'll set up a minimal single node Dovecot installation that authenticates through PAM.

First, let's write a small playbook:

```yaml
---

- hosts: dovecot
  roles:
    - adfinis.dovecot.dovecot

```

This is already enough for the bare minimum, using mbox files in /var/mail.
But we should set at least some options as host vars:

```yaml
# We want TLS, but please bring your own certs
dovecot_ssl_cert_filename: /etc/ssl/certs/ssl-cert-snakeoil.pem
dovecot_ssl_key_filename: /etc/ssl/private/ssl-cert-snakeoil.key

# Where Dovecot expects the mail homes.  If you're using an external MDA (e.g.
# Postfix virtual mail without going through dovecot-lda) it should deliver here:
dovecot_mail_home: /var/vmail/%d/%Ln
dovecot_mail_location: maildir:~/Maildir
```

### LDAP Authentication

If you want to authenticate against LDAP rather than PAM, override the list of enabled auth modules:

```yaml
dovecot_auth_enabled ["ldap"]

dovecot_auth_ldap_uri: "ldaps://ldap.example.org:636"
dovecot_auth_ldap_bind_dn: "uid=dovecot,cn=service-accounts,dc=example,dc=org"
dovecot_auth_ldap_bind_pass: "PleaseChooseYourOwnPassword"
dovecot_auth_ldap_search_base: "dc=example,dc=org"
```

The default LDAP filters and attribute lists assume that your users are LDAP `posixAccount`s, but of cours this can be overriden:

```yaml
dovecot_auth_ldap_user_attrs:
  - "mailPrimaryAddress=user"
  - "=uid=dovenull"  # set these if you're using a virtual mail user
  - "=gid=dovenull"
dovecot_auth_ldap_user_filter: "(&(objectClass=posixAccount)(|(mailPrimaryAddress=%Lu)(uid=%Ln)))"

# and repeat the same for passdb:
dovecot_auth_ldap_pass_attrs: ...
dovecot_auth_ldap_pass_filter: ...
```
