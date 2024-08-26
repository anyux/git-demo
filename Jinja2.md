[TOC]
## Jinja2

<p style="background-color: rgba(0, 128, 0, 1); color: rgba(255, 255, 255, 1)">掌握了Jinja才是深入Ansible-playbook的开始
</p>

### Jinja2 For循环

变量的提取使用 `{{variable}}`
`{%statement execution%}` 括起来的内容为Jinja2命令执行语句

```Jinja2
{% for item in all_items %}
    {{ item }}
    {% endfor %}
```

```python
#导入模块
from jinja2 import Template

#设置模板内容
template_content = ''' 
{% for id in range(201,211) %} 
192.168.37.{{ id }} web{{ "%02d"|format(id-200) }}.magedu
{% endfor %} 
''' 
# 创建模板对象 
template = Template(template_content)

# 渲染模板 
output = template.render()

# 打印模板
print(output)
```

### Jinja2 If 条件

```Jinja2
{% if my_conditional %}
    　　 ...
    {% endif %}
```

编排目录结构

> <pre>
> mysqlconf.yaml
> roles/mysqlconf/
>         ├── templates
>         │ └── mycnf.j2
> </pre>

```bash
mkdir -p roles/mysqlconf/templates
```

只定义了Templates而没有定义Tasks,Ansible也支持这样的方式,只是mysqlconf这个role的功能不全而已,但不影响其正常使用

我们本次的Tasks调度配置在mysqlconf.yaml文件中

mysqlconf.yaml

```yaml
- name : Mysql conf template
  hosts : ubuntu
  vars:
    PORT: 1331
  tasks:
   - template: 
      src: roles/mysqlconf/templates/mycnf.j2 
      dest: /etc/mycnf.conf.yaml
```

roles/mysqlconf/templates/mycnf.j2
```Jinja2
{% if PORT %}
    bind-address=0.0.0.0:{{ PORT }}
{% else %}
    bind-address=0.0.0.0:3306
{% endif %}
```

```bash
ansible-playbook mysqlconf.yaml 

PLAY [Mysql conf template] ******

TASK [Gathering Facts] ******
ok: [192.168.255.110]

TASK [template] ******
changed: [192.168.255.110]

PLAY RECAP ******
192.168.255.110            : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Jinja多值合并

```Jinja2
{% for node in groups["db"] %}
    {{ node | join("") }}:5672
    {% if not loop.last %}
    {% endif %}
{% endfor %}
```
这段代码涉及Jinja和Ansible的内置变量
`groups`是Ansible的内置变量

| 变量名称               | 描述                                      |
|--------------------|-----------------------------------------|
| hostvars           | 主机变量名                                   |
| inventory_hostname | 当前Ansible可识别的hosts                      |
| group_names        | 当前主机的所属组                                |
| groups             | 字典数组,数组名,包括: {"all": [..], 'web': [..]} |

列表转换字符
```jinjia2
{{ list | join(" ") }}
```
在 Jinja2 中,`loop` 是在 `for` 循环内部自动定义的变量它提供了一些关于当前循环的信息,比如：

- `loop.index`: 当前迭代的索引,从 1 开始
- `loop.index0`: 当前迭代的索引,从 0 开始
- `loop.revindex`: 逆向索引,从 1 开始（从后往前数的索引）
- `loop.revindex0`: 逆向索引,从 0 开始
- `loop.first`: 如果当前迭代是第一次迭代,则为 `True`
- `loop.last`: 如果当前迭代是最后一次迭代,则为 `True`
- `loop.length`: 循环对象的长度
- `loop.cycle`: 用于在多个值之间循环的助手函数


```python
# 导入必要的模块
# DataLoader 加载 Ansible 所需的数据文件
from ansible.parsing.dataloader import DataLoader
# InventoryManager 管理和解析 Ansible 的主机清单
from ansible.inventory.manager import InventoryManager
# VariableManager 管理 Ansible 环境中的所有变量，包括全局变量、主机变量、组变量
from ansible.vars.manager import VariableManager

# 加载 Ansible 数据
loader = DataLoader()
inventory = InventoryManager(loader=loader, sources=['/etc/ansible/hosts'])
variable_manager = VariableManager(loader=loader, inventory=inventory)

# 获取并打印 groups 变量
groups = inventory.get_groups_dict()

#导入模块
from jinja2 import Template

#设置模板内容
template_content ='''
{% for node in groups["all"] %}
    {{ node | join("") }}:5672
    {% if not loop.last %}
    {% endif %}
{% endfor %}
'''
# 创建模板对象 
template = Template(template_content)

# 定义要传递给模板的变量
context = {
    'groups': groups,
}
# 渲染模板 
output = template.render(context)

# 打印模板
print(output)
```

```ini
roles/join/
     ├── templates
     │ └── list.j2
join.yaml
```

roles/join/templates/list.j2

```jinjia2
{% for node in groups["all"] %}
{{ node | join("") }}:5672
{% if not loop.last %}
{% endif %}
{% endfor %}
```

join.yaml

```yaml
---
- hosts : all
  gather_facts : no
  vars:
    PORT: 1331
  tasks:
   - template: 
      src: roles/join/templates/list.j2 
      dest: /data/list.txt
  roles:
    - { role: join }
```

```bash
ansible-playbook join.yaml 

PLAY [all] *****

TASK [template] *****
changed: [192.168.255.110]
changed: [192.168.255.101]

PLAY RECAP *****
192.168.255.101            : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.255.110            : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

### Jinja2 default 设定

default()默认值的设定有助于程序的健壮性和简洁性

原来的示例如下
```Jinja2
{% if PORT %}
    bind-address=0.0.0.0:{{ PORT }}
{% else %}
    bind-address=0.0.0.0:3306
{% endif %}
```

roles/mysqlconf/templates/mycnf.j2
```Jinja2
bind-address=0.0.0.0:{{ PORT | defult(3306) }}
```

### Ansible结合Jinja2生成Nginx配置

编排目录

```bash
mkdir -p roles/nginxconf/{tasks,templates,vars}/

```

```ini
nginxconf.yaml
    roles/nginxconf/
    ├── tasks
    │ ├── file.yaml
    │ └── main.yaml
    ├── templates
    │ └── nginx.conf.j2
    └── vars
    　　 └── main.yaml
```

编辑file.yaml,定义nginxconf role的一个功能集(一个文件一个功能集)

roles/nginxconf/tasks/file.yaml
```yaml
---
  - name: nginx.conf.j2 tempalte transfer example
    template: 
      src: nginx.conf.j2 
      dest: /etc/nginx/nginx.conf.template
```
roles/nginxconf/tasks/main.yaml
```yaml
---
  - import_tasks: file.yaml
```
roles/nginxconf/templates/nginx.conf.j2
```Jinja2
{% if nginx_use_proxy %}
    {% for proxy in nginx_proxies %}
    upstream {{ proxy.name }} {
    　　 # server 127.0.0.1:{{ proxy.port }};
    　　 server {{ ansible_default_ipv4.address }}:{{ proxy.port }};
    }
    {% endfor %}
    {% endif %}
    server {
    　　 listen 80;
    　　 server_name {{ nginx_server_name }};
    　　 access_log off;
    　　 error_log /dev/null crit;
    　　 rewrite ^ https:// $server_name$request_uri? permanent;
    }
    server {
    　　 listen 443 ssl;
    　　 server_name {{ nginx_server_name }};
    　　 ssl_certificate /etc/nginx/ssl/{{ nginx_ssl_cert_name }};
    　　 ssl_certificate_key /etc/nginx/ssl/{{ nginx_ssl_cert_key }};
    　　 root {{ nginx_web_root }};
    　　 index index.html index.html;
    {% if nginx_use_auth %}
    　　 auth_basic "Restricted";
    　　 auth_basic_user_file /etc/nginx/{{project_name}}.htpasswd;
    {% endif %}
    {% if nginx_use_proxy %}
    {% for proxy in nginx_proxies %}
    　　 location {{ proxy.location }} {
    　　　　 proxy_set_header X-Real-IP $remote_addr;
    　　　　 proxy_set_header X-Forwarded-Proto http;
    　　　　 proxy_set_header X-Url-Scheme $scheme;
    　　　　 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    　　　　 proxy_set_header Host $http_host;
    　　　　 proxy_set_header X-NginX-Proxy true;
    　　　　 proxy_redirect off;
    　　　　 proxy_pass http://{{ proxy.name }};
    　　　　 break;
    　　 }
    {% endfor %}
    {% endif %}
    {% if nginx_server_static %}
    　　 location / {
    　　　　 try_files $uri $uri/ =404;
    　　 }
    {% endif %}
    }
```

roles/nginxconf/vars/main.yaml
如果yaml看不懂,可以通过[在线转换](https://www.bejson.com/json/json2yaml/)
主要是对于yaml文件列表,字典的转换,不太灵活
这个文件主要是对 `nginx_proxies` 变量,通过循环`.`来读取变量 
```yaml
---
  nginx_server_name: www.magedu.com
  nginx_web_root: /opt/magedu/
  nginx_use_proxy: true
  nginx_ssl_cert_name: ifa.crt
  nginx_ssl_cert_key: ifa.key
  nginx_use_auth: true
  project_name: suspicious
  nginx_server_static: true
  nginx_proxies:
   - name: suspicious
     location: /
     port: 2368
   - name: suspicious-api
     location: /api
     port: 3000
```
转换后
```json
{
  "nginx_server_name": "www.magedu.com",
  "nginx_web_root": "/opt/magedu/",
  "nginx_use_proxy": true,
  "nginx_ssl_cert_name": "ifa.crt",
  "nginx_ssl_cert_key": "ifa.key",
  "nginx_use_auth": true,
  "project_name": "suspicious",
  "nginx_server_static": true,
  "nginx_proxies": [
    {
      "name": "suspicious",
      "location": "/",
      "port": 2368
    },
    {
      "name": "suspicious-api",
      "location": "/api",
      "port": 3000
    }
  ]
}

```
nginxconf.yaml

```yaml
---
- name: Nginx Proxy Server's Conf Dynamic Create
  hosts: centos
  gather_facts: true
  roles:
   - { role: nginxconf }
     
- name: Nginx WebServer's Conf Dynamic Create
  hosts: ubuntu
  vars:
    nginx_use_proxy: false
    nginx_use_auth: false
    nginx_server_static: false
  gather_facts: no
  roles:
   - { role: nginxconf }
```

```bash
ansible-playbook nginxconf.yaml
```

```bash
ansible-playbook nginxconf.yaml 

PLAY [Nginx Proxy Server's Conf Dynamic Create] ******

TASK [Gathering Facts] ******
ok: [192.168.255.101]

TASK [nginxconf : nginx.conf.j2 tempalte transfer example] ******
changed: [192.168.255.101]

PLAY [Nginx WebServer's Conf Dynamic Create] ******

TASK [nginxconf : nginx.conf.j2 tempalte transfer example] ******
changed: [192.168.255.110]

PLAY RECAP ******
192.168.255.101            : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
192.168.255.110            : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```

### Ansible结合Jinja2生成Apache多主机配置

通过Ansible的Jinja模板,生成如下的Apache多主机配置

```txt
NameVirtualHost *:80
    <VirtualHost *:80>
    　　 ServerName apache.magedu.com
    　　 DocumentRoot /data/magedu/
    　　 <Directory "/data/magedu/">
    　　　　 AllowOverride All
    　　　　 Options -Indexes FollowSymLinks
    　　　　 Order allow,deny
    　　　　 Allow from all
    　　 </Directory>
    </VirtualHost>
    <VirtualHost *:80>
    　　 ServerName apache.magedu.otherdomain.com
    　　 DocumentRoot /data/otherdomain/
    　　 ServerAdmin stanley@magedu.com
    　　 <Directory "/data/otherdomain/">
    　　　　 AllowOverride All
    　　　　 Options -Indexes FollowSymLinks
    　　　　 Order allow,deny
    　　　　 Allow from all
    　　 </Directory>
    </VirtualHost>
```

编排目录
```ini
roles/apacheconf/
├── tasks
│ ├── file.yml
│ └── main.yml
├── templates
│ └── apache.conf.j2
└── vars
  └── main.yml
```
### Ansible Galaxy
Galaxy 可以下载Ansible官方Roles和collections,允许从ansible官方下载,创建,分享,管理

集合可以包含多个角色（Roles）、模块（Modules）、插件（Plugins）、剧本（Playbooks）、和文档（Documentation）等

集合的优势
模块化：集合将 Ansible 的功能打包成独立的单元，便于管理和复用。
易于分享：开发者可以将集合发布到 Ansible Galaxy 或其他平台，方便他人下载和使用。
版本控制：集合有自己的版本控制，可以确保在不同项目中使用特定版本的集合，避免因更新导致的兼容性问题。
自包含：集合包含了所有必要的依赖，减少了对外部资源的依赖。

Ansible-galaxy命令用法
默认下载的Roles存放于`/etc/ansible/roles`目录下,可在`/etc/ansible/ansible.cfg`自定义存放目录

```bash
ansible-galaxy init my_role
```
当前目录下创建一个名为 my_role 的目录，并生成角色的标准目录结构，包括 tasks、handlers、files、templates、vars、defaults、meta 等目录和文件

```bash
ansible-galaxy install geerlingguy.mysql
Starting galaxy role install process
- downloading role 'mysql', owned by geerlingguy
- downloading role from https://github.com/geerlingguy/ansible-role-mysql/archive/4.3.4.tar.gz
- extracting geerlingguy.mysql to /root/.ansible/roles/geerlingguy.mysql
- geerlingguy.mysql (4.3.4) was installed successfully

```
这个命令会将 geerlingguy.mysql 角色下载到本地的 roles 目录中，通常是 `~/.ansible/roles`

```bash
ansible-galaxy install -r requirements.yml
```
如果你有一个 requirements.yml 文件，其中列出了多个角色或集合，你可以使用以下命令来批量安装

删除本地已安装的角色或集合
```bash
ansible-galaxy remove geerlingguy.mysql

```

命令用于列出本地安装的所有角色或集合

```bash
ansible-galaxy list
```
在 Ansible Galaxy 上搜索角色或集合
```bash
ansible-galaxy search mysql
```
发布自己的角色或集合

```bash
ansible-galaxy import username reponame
```
username 是你的 Galaxy 或 GitHub 用户名，reponame 是你的角色或集合所在的仓库名称。


```bash
ansible-galaxy role
```
你可以结合其他命令来创建、删除、列表或导出角色

集合是 Ansible 2.9 之后引入的概念，它们是打包的模块、插件和角色的集合
创建一个新集合

```bash
ansible-galaxy collection init my_namespace.my_collection
```
安装一个集合
```bash
ansible-galaxy collection install community.general
```
配置你的 Ansible Galaxy 环境，比如设置代理或其他配置参数

```bash
ansible-galaxy setup --proxy http://proxy.example.com:8080
```

将本地创建的集合发布到 Ansible Galaxy
```bash
ansible-galaxy collection publish path/to/my_collection-1.0.0.tar.gz
```

[reoles平台](https://galaxy.ansible.com/ui/)


