## nginx role
背景,为创建高可用的 lnmp 应用,使用role去写,gpt建议是写单个组件的role,后再合并,构建应用,目前就逐一编写role,就从最简单的nginx组件开始编写

gpt建议通过shell脚本,编写出完整的部署代码,再尝试转换为role,这个和role这种构建方式还有些思维上的差异,所以我就直接使用role的方式编写

gpt建议通过`test_nginx.yaml`文件的方式,调试正在编写的role,或通过`tags`调试指定步骤

```yaml
# test_nginx.yaml
---
- hosts: localhost
  become: true
  roles:
    - nginx
```
创建目录测试 `role` 目录
```bash
mkdir test-role
cd test-role
```
初始化role目录,避免手动创建目录
```bash
ansible-galaxy init nginx
```
查看目录结构
```bash
tree ./
./
└── nginx
    ├── defaults
    │   └── main.yml
    ├── files
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── README.md
    ├── tasks
    │   └── main.yml
    ├── templates
    ├── tests
    │   ├── inventory
    │   └── test.yml
    └── vars
        └── main.yml

```






