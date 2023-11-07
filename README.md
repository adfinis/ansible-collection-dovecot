# Ansible Collection - adfinis.dovecot

![License](https://img.shields.io/github/license/adfinis/ansible-collection-dovecot)
![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/adfinis/ansible-collection-dovecot/ansible-lint.yml)
[![adfinis.dovecot on Ansible Galaxy](https://img.shields.io/badge/collection-adfinis.dovecot-blue)](https://galaxy.ansible.com/ui/repo/published/adfinis/dovecot/)

Colletion to deploy the [Dovecot](https://dovecot.org/) IMAP server.

## Requirements

- This collection is designed to work with Debian 10 or later.
    - The exact version of Debian that will work depends on the version of Dovecot is being deployed (Dovecot is installed from the upstream repositories).
    
## Dependencies

- `ansible.posix`

## License

[GPL-3.0-or-later](LICENSE)

## Author Information

The Ansible collection `adfinis.dovecot` collection was written by:

* Adfinis AG | [Website](https://www.adfinis.com/) | [GitHub](https://github.com/adfinis)


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
dovecot_auth_enabled: ["ldap"]

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

### High Availability Director Cluster

In the following example we use this collection to build a simple two-tier Dovecot cluster with two Director nodes and two backend nodes.  The explanation of how Dovecot Director works is out of scope for this README; please consult the [upstream documentation](https://doc.dovecot.org/admin_manual/director/) instead.

First, let's set up our inventory:

```ini
[dovecot:children]
dovecot_director
dovecot_backend

[dovecot_director]
dovecotd01.example.org
dovecotd02.example.org

[dovecot_backend]
dovecotb01.example.org
dovecotb02.example.org
```

With this inventory, our playbook stays the same:

```yaml
---

- hosts: dovecot
  roles:
    - adfinis.dovecot.dovecot
```

Let's configure our Director servers first.  For this, create a file `group_vars/dovecot_director/dovecot.yml`:

```yaml
dovecot_director_enabled: true

# IP addresses of the director nodes
dovecot_director_servers:
  - "2001:db8::11"
  - "2001:db8::12"

# IP addresses of the backend nodes
dovecot_director_mailservers:
  - "2001:db8::21"
  - "2001:db8::22"
  
# If users can log in with either the full email or only the localpart, only use the localpart for director hashing.
dovecot_director_username_hash: "%Ln"
  
# In this example we perform authentication on the backend.  While not the recommended architecture, it is a bit simpler.
dovecot_auth_enabled:
  - static
dovecot_auth_static_passdb_args:
  - "proxy=y"
  - "nopassword=y"
  - "proxy_mech=%m"
```

That's already the director specific config.  Next up is the backend config in `group_vars/dovecot_backend/dovecot.yml`:

```yaml
# Director nodes should be in login_trusted_networks
dovecot_login_trusted_networks:
  - "2001:db8::11"
  - "2001:db8::12"

# Configure an actual authentication method on the backend, in this case LDAP
dovecot_auth_enabled: ["ldap"]

# Same as the single node setup above.
dovecot_auth_ldap_uri: "ldaps://ldap.example.org:636"
dovecot_auth_ldap_bind_dn: "uid=dovecot,cn=service-accounts,dc=example,dc=org"
dovecot_auth_ldap_bind_pass: "PleaseChooseYourOwnPassword"
dovecot_auth_ldap_search_base: "dc=example,dc=org"
```

Finally, there's some config that does not need to be strictly split between Director and backend nodes.  While we could keep this in the two already existing files, we want to reduce code duplication, so we put it in the common group vars in `group_vars/dovecot/dovecot.yml`:

```yaml
dovecot_mail_home: /var/vmail/%d/%Ln
dovecot_mail_location: maildir:~/Maildir
dovecot_mail_uid: dovenull
dovecot_mail_gid: dovenull
```
