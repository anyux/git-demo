[TOC]
## Ansible插件
### Ansible插件使用场景

Ansible核心功能及自身500多个功能模块已能满足企业绝大多数场景所需

- 连接插件: 当需要连接到特定类型的系统时，如网络设备、Docker 容器或云服务，连接插件能够定制连接方式
- 遍历插件: 指定新的遍历方式
- 过滤器插件: 常用于将复杂的数据结构转换为所需的格式，或对变量进行复杂计算
- 回调插件: 在任务执行过程中实时地输出日志或将执行结果存储到第三方系统
- 自定义模块: 当 Ansible 内置模块无法满足特定需求时，可以编写自定义模块来执行特定任务
- 策略插件: 策略插件可以用来定制 Ansible 的行为，如限制并发任务数或设置主机失败时的处理策略
- 库插件可以扩展 Jinja2 模板的功能，允许在模板中调用自定义的 Python 函数或方法。适用于在模板渲染时需要复杂的数据处理

### Ansible插件类型

[ansible插件网址](https://docs.ansible.com/ansible/latest/plugins/plugins.html)

+   [Action plugins动作插件](https://docs.ansible.com/ansible/latest/plugins/action.html)
+   [Become plugins成为插件](https://docs.ansible.com/ansible/latest/plugins/become.html)
+   [Cache plugins缓存插件](https://docs.ansible.com/ansible/latest/plugins/cache.html)
+   [Callback plugins回调插件](https://docs.ansible.com/ansible/latest/plugins/callback.html)
+   [Cliconf pluginsCLICONF插件](https://docs.ansible.com/ansible/latest/plugins/cliconf.html)
+   [Connection plugins连接插件](https://docs.ansible.com/ansible/latest/plugins/connection.html)
+   [Docs fragments文档片段](https://docs.ansible.com/ansible/latest/plugins/docs_fragment.html)
+   [Filter plugins过滤器插件](https://docs.ansible.com/ansible/latest/plugins/filter.html)
+   [Httpapi pluginsHttpAPI 插件](https://docs.ansible.com/ansible/latest/plugins/httpapi.html)
+   [Inventory plugins库存插件](https://docs.ansible.com/ansible/latest/plugins/inventory.html)
+   [Lookup plugins查找插件](https://docs.ansible.com/ansible/latest/plugins/lookup.html)
+   [Modules模块](https://docs.ansible.com/ansible/latest/plugins/module.html)
+   [Module utilities模块实用程序](https://docs.ansible.com/ansible/latest/plugins/module_util.html)
+   [Netconf plugins网络会议插件](https://docs.ansible.com/ansible/latest/plugins/netconf.html)
+   [Shell plugins外壳插件](https://docs.ansible.com/ansible/latest/plugins/shell.html)
+   [Strategy plugins策略插件](https://docs.ansible.com/ansible/latest/plugins/strategy.html)
+   [Terminal plugins终端插件](https://docs.ansible.com/ansible/latest/plugins/terminal.html)
+   [Test plugins测试插件](https://docs.ansible.com/ansible/latest/plugins/test.html)
+   [Vars plugins变量插件](https://docs.ansible.com/ansible/latest/plugins/vars.html)

#### Connection类型插件
最常用的是 paramiko SSH、ssh和本地连接类型
ansible 默认使用smart连接类型

>smart连接会根据目标主机是否支持ControlPersist功能来智能选择是否使用SSH连接  
> 
> 如果不支持，‌则可能回退到使用Paramiko等其他连接方式  
> 
>ControlPersist是OpenSSH提供的一个功能，‌它允许在SSH连接关闭后继续保持控制连接的活动状态一段时间‌。  
> 
> ‌这个功能可以显著提高SSH连接的效率，‌特别是在需要频繁连接到同一台远程主机时，‌因为它避免了每次连接都需要重新进行身份验证的开销。  
> 
> ‌通过配置ControlPersist，‌用户可以在后续的SSH会话中重用已有的控制连接，‌从而加快连接速度并减少资源消耗
 

[paramiko模块](https://www.paramiko.org/)
安装paramiko模块
```bash
pip3 install paramiko

pip3 show paramiko
Name: paramiko
Version: 3.4.1
Summary: SSH2 protocol library
Home-page: https://paramiko.org
Author: Jeff Forcier
Author-email: jeff@bitprophet.org
License: LGPL
Location: /usr/local/lib/python3.8/dist-packages
Requires: cryptography, pynacl, bcrypt
Required-by: 

```
`--connection`或`-c`直接指定连接类型
```bash
ansible all -m ping -c ssh
ansible all -m ping --connection paramiko_ssh
```

`-e`传递参数,`ansible_connection=paramkiko_ssh`
```bash
ansible all -m ping  -e ansible_connection=paramiko_ssh
```
在指定 Inventory 中的参数
```bash
ansible-playbook -i "remote_host ansible_host=192.168.255.110 ansible_user=root ansible_connection=paramiko_ssh" paramiko_ssh_example.yml
```

```bash
export ANSIBLE_CONNECTION=paramiko_ssh
ansible all -m ping 

```
#### Lookup类型插件

新的Lookup插件放在ansible.cfg指定的同级目录下即可生效

#### vars类型插件
从名称看可以得知是变量类型插件，这些变量并非来自Inventory、Playbook、命令终端，而是通过host_vars、group_vars产生的，需要留意的是这些变量也可以通过Inventory产生，言外之意是Vars类型的插件绝大多数时候用不到

#### Filter类型插件
Filter类型插件其实是Jinja2模板引擎的Filter，Jinja2的常用Filter实现有to_yaml、to_json，官网实现的Filter Plugins代码全合并在core.py脚本中，新Filter插件需在该脚本的基础上编写

#### Callback类型插件

该插件允许程序捕获响应的事件，并进行一些自定义响应，如前面提到的插入日志到数据库、发送邮件等功能。该类功能对运维的日常工作还是有些帮助的，因为我们可以通过统计数据库对平时的发布频次、成功/失败率、代码质量进行大数据处理和可预见性分析

官方也提供了很多Callback类型插件，如log_plays捕获Playbook事件日志写入文件，并在Playbook执行完毕时发送邮件给相关人员；syslog_json将Ansible日志以JSON格式输出等。同时Ansible还支持通过命名的方式按序执行插件，如希望插件第一个执行，则命名为1_first.py，如希望最后一个执行，则命名为z_last.py即可。下面我们学习如何编写自己的插件



