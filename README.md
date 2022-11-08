# ansible-common

Setups [Percona Server](https://www.percona.com/software/mysql-database/percona-server) 5.7 from Percona APT repository.

## Example playbook

```yaml
---
- hosts: myserver
  user: root
  sudo: False
  roles:
   - sunfoxcz.percona
     percona_version: 5.6 # default is 5.7
     percona_nofile: 20000 # default is 10000
     percona_root_password: xxx # will be generated and saved to ~/.ansible_percona_passwords if not provided
     percona_databases:
      - mydatabase
     percona_users:
      - name: user1
        host: 127.0.0.1
        database: mydatabase
        password: xxx # will be generated if not provided
```

## License

Licensed under MIT license. See [LICENSE](LICENSE.md) for details.
