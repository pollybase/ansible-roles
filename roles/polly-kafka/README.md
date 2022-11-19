polly-kafka
=========


```yaml
- hosts: servers
  become: true
  roles:
    - role: polly-jre
    - role: polly-zookeeper
    - role: polly-kafka
```

