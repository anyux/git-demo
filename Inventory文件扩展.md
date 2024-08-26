## Inventory文件扩展

Inventory文件 默认/etc/ansible/hosts

```ini
# Inventory文件 /etc/ansible/hosts
# 主机组使用中括号“[]”来定义，接下来，每一行定义一台组内的主机
[myweb]
www.myweb.com
10.23.1.123
```

```yaml
---
- hosts: myweb
  tasks:
   [...]
```

```bash
ansible myweb -a "free -m"
```

### Inventory文件实战

Inventory文件中的主机数量会从几十台到上百台不等

通常这些主机会按照其所服务的应用类型进行分组,比如 database,webserver,caching组等

```ini
[servercheck-web]
www1.servercheck.in
www2.servercheck.in

[servercheck-web:vars]
ansible_ssh_user=servercheck_svc

[servercheck-db]
db1.servercheck.in

[servercheck-log]
log.servercheck.in

[servercheck-backup]
backup.servercheck.in

[servercheck-nodejs]
atl1.servercheck.in
atl2.servercheck.in
nyc1.servercheck.in
nyc2.servercheck.in
nyc3.servercheck.in
ned1.servercheck.in
ned2.servercheck.in

[servercheck-nodejs:vars]
ansible_ssh_user=servercheck_svc
foo=bar
# 按操作系统类型对主机组进行分组

[centos:children]
servercheck-web
servercheck-db
servercheck-nodejs
servercheck-backup

[ubuntu:children]
servercheck-log
```

```bash
ansible centos -m yum -a "name=bash state=latest"
```

```yaml
---
# 对所有主机进行基础配置
- hosts: all
  sudo: true
  roles:
    - security
    - logging
    - firewall
# 配置web主机
- hosts: servercheck-web
  roles:
    - nginx
    - php
    - servercheck-web
# 配置数据库主机
- hosts: servercheck-db
  roles:
    - pgsql
    - db-tuning
# 配置日志主机
- hosts: servercheck-log
  roles:
    - java
    - elasticsearch
    - logstash
    - kibana
# 配置备份主机
- hosts: servercheck-backup
  roles:
    - backup
# 配置Node.js主机
- hosts: servercheck-nodejs
  roles:
    - servercheck-node
```

### 独立的Inventory文件

```bash
ansible-playbook playbook.yml -i inventories/inventory-dev
```

### Inventory变量

```ini
[www]
# 为主机单独定义变量
www1.example.com ansible_ssh_user=johndoe
www2.example.com
[db]
db1.example.com
db2.example.com
# 为一组主机定义变量，这些变量对组内所有主机生效
[db:vars]
ansible_ssh_port=5222
database_performance_mode=true
```
我们建议不要在静态的Inventory文件中定义过多的变量,因为这样定义的变量不仅识别度低,而且维护起来也比较麻烦,尤其是在一行内为一台主机定义多个变量的时候

host_vars目录

```ini
hostedapachesolr/
    　　 host_vars/
    　　　　 nyc1.hostedapachesolr.com
    　　 inventory/
    　　　　 hosts
    　　 main.yml
```
group_vars目录

```ini
hostedapachesolr/
    　　 group_vars/
    　　　　 solr
    　　 host_vars/
    　　　　 nyc1.hostedapachesolr.com
    　　 inventory/
    　　　　 hosts
    　　 main.yml
```

在文件group_vars/solr中，使用YAML语法为主机组slor定义组变量
```yaml
---
do_something_amazing=true
foo=bar
```

### 动态Inventory

Ansible通过调用第三方脚本来动态地配置Inventory文件

如亚马逊AWS、Cobbler、gitalOcean、Lnode、OpenStack等，提供了现成的脚本供Ansible直接调用，其具体的用法在对应的平台上都有详尽的官方文档说明

Ansible启用动态Inventory的机制是通过调用外部脚本（任何脚本都可以，二进制文件也可以，只要运行结果返回的是JSON串就行）生成指定格式的JSON串。Ansible可以对JSON格式的字符串进行解析，并最终将其转化为Ansible可用的Inventory文件格式。所以，所谓的动态Inventory文件脚本开发，其实就是编写脚本根据具体环境将主机信息及关系（这些数据可以通过抓取数据库，调用外部API或者直接读取文件获得）以JSON格式来表示出来，并将其做为脚本输出结果传给Ansible

需要注意的是，用于生成JSON代码的脚本必须支持两个选项：--list和--host。
·--list：返回所有的主机组信息，每个组都应该包含字典形式的主机列表、子组列表，如果需要的话还应该有组变量，最简单的信息是只包含主机列表。返回的数据格式是JSON格式。
·--host：返回该主机的变量列表，或者是返回一个空的字典，使用JSON格式。

#### 动态Inventory脚本的Python实现

在现实环境中,需要结合自身的业务场景编写代码,既可以通过调用外部API也可以查询数据库来取得所需的主机信息,并将其最终转换为JSON代码,供Ansible使用

```bash
ansible all -i inventory.py -m debug -a "var=host_specific_var"

```Inventory文件扩展.md
