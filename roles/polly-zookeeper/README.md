polly-zookeeper
=================


```yaml
- name: Zookeeper cluster setup
  hosts: zookeeper
  sudo: yes
  roles:
    - role: polly-zookeeper
```
分组文件

```inventory
[zookeeper]
192.168.239.131 zid=1
192.168.239.134 zid=2
192.168.239.135 zid=3
```

