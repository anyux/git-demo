## nginx role
背景,为创建高可用的 lnmp 应用,使用role去写,gpt建议是写单个组件的role,后再合并,构建应用,目前就逐一编写role,就从最简单的nginx组件开始编写

gpt建议通过shell脚本,编写出完整的部署代码,再尝试转换为role,这个和role这种构建方式还有些思维上的差异,所以我就直接使用role的方式编写

gpt建议通过`test_nginx.yaml`文件的方式,调试正在编写的role,或通过`tags`调试指定步骤

### 目录结构

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

# 因个人喜欢以yaml文件结尾,因此批量修改yaml文件后缀
# 重命名 .yml 文件为 .yaml
for file in $(find . -name "*.yml"); do
    mv "$file" "${file%.*}.yaml"
done

```
查看目录结构
```bash
tree ./
./
└── nginx
    ├── defaults
    │ └── main.yaml
    ├── files
    ├── handlers
    │ └── main.yaml
    ├── meta
    │ └── main.yaml
    ├── README.md
    ├── tasks
    │ └── main.yaml
    ├── templates
    ├── tests
    │ ├── inventory
    │ └── test.yaml
    └── vars
        └── main.yaml

```

### 构建二进制文件

这里构建nginx的二进制文件,将文件移动到 `nginx/files/`目录下
```bash
#构建编译环境
apt-get update
apt-get install -y build-essential libpcre3 libpcre3-dev libssl-dev zlib1g zlib1g-dev gcc libxml2 libxml2-dev libxslt-dev libgd-dev libxml++2.6-dev

#下载源码包
wget http://nginx.org/download/nginx-1.20.2.tar.gz
tar -zxvf nginx-1.20.2.tar.gz
cd nginx-1.20.2

#配置nginx编译选项,这里要求nginx是一个静态的二进制文件,使用参数--with-ld-opt='-static
./configure \
--with-cc-opt='-g -O2 -fdebug-prefix-map=/build/nginx-65A62v/nginx-1.20.2=. -fstack-protector-strong -Wformat -Werror=format-security -fPIC -Wdate-time -D_FORTIFY_SOURCE=2' \
--with-ld-opt='-static -Wl,-Bsymbolic-functions -Wl,-z,relro -Wl,-z,now -fPIC' \
--prefix=/usr/local/nginx \
--conf-path=/etc/nginx/nginx.conf \
--http-log-path=/var/log/nginx/access.log \
--error-log-path=/var/log/nginx/error.log \
--lock-path=/var/lock/nginx.lock \
--pid-path=/run/nginx.pid \
--modules-path=/usr/lib/nginx/modules \
--http-client-body-temp-path=/var/lib/nginx/body \
--http-fastcgi-temp-path=/var/lib/nginx/fastcgi \
--http-proxy-temp-path=/var/lib/nginx/proxy \
--http-scgi-temp-path=/var/lib/nginx/scgi \
--http-uwsgi-temp-path=/var/lib/nginx/uwsgi \
--with-debug \
--with-compat \
--with-pcre-jit \
--with-http_ssl_module \
--with-http_stub_status_module \
--with-http_realip_module \
--with-http_auth_request_module \
--with-http_v2_module \
--with-http_dav_module \
--with-http_slice_module \
--with-threads \
--with-http_addition_module \
--with-http_gunzip_module \
--with-http_gzip_static_module \
--with-http_sub_module \
--with-stream \
--with-stream_ssl_module \
--with-mail \
--with-mail_ssl_module \
--with-ld-opt='-static'

    
#编译
make

#安装,安装是为了方便打包操作
make install

#打包,此时nginx的二进制文件已经制作完成
tar zcvf nginx-binary-1.20.tar.gz  /usr/local/nginx

#可手动通过配置检查,这里报错是因为没有这个用户,可以手动修改,后面还会通过ansible templates修改此配置文件,因此保持不动,如果觉得不妥,可通过修改configure --user=username --group=group参数指定用户及用户组
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf 
nginx: [emerg] getgrnam("nogroup") failed

```
### 编写tasks
nginx/tasks/main.yaml
```yaml
---
- name: nginx | unarchive binary file
  unarchive:
    src: "nginx-binary-{{ nginx_version }}.tar.gz"
    dest: "/"
    remote_src: "no"
    mode: "0755"
- name: nginx | nginx template conf file
  template:
    src: nginx.conf.j2
    dest: "{{ nginx_path }}/conf/nginx.conf"
    mode: "0644"
- name: nginx | nginx.service template  file
  template:
    src: nginx.service.j2
    dest: "{{ nginx_service_path }}/nginx.service"
```

### 编写vars 
nginx/vars/main.yaml
```yaml
---
# vars file for nginx
nginx_path: "/usr/local/nginx"
nginx_version: "1.20"
nginx_user: "root"
nginx_group: "root"
worker_processes: "4"
listen_port: "80"
server_name: "localhost"
nginx_service_description: "nginx binary with configure"
nginx_service_path: "/usr/lib/systemd/system/"
```
### 编写templates
nginx/templates/nginx.conf.j2
```Jinja2
user  {{nginx_user | d("nobody")}};
pid {{nginx_path}}/logs/nginx.pid;
worker_processes  {{worker_processes | d(1)}};
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    server {
        listen       {{ listen_port }};
        server_name  {{ server_name }};
        location / {
            root   html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

nginx/templates/nginx.service.j2
```Jinja2
[Unit]
Description={{ nginx_service_description }}
After=network.target

[Service]
Type=forking
PIDFile={{nginx_path}}/logs/nginx.pid
ExecStartPre={{nginx_path}}/sbin/nginx  -t -q -g 'daemon on; master_process on;'
ExecStart={{nginx_path}}/sbin/nginx -g 'daemon on; master_process on;'
ExecReload={{nginx_path}}/sbin/nginx -g 'daemon on; master_process on;' -s reload
#ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile {{nginx_path}}/logs/nginx.pid
ExecStop=/bin/kill -s TERM $MAINPID
KillMode=mixed
TimeoutStopSec=5
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```











