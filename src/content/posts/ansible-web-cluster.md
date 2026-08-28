---
title: Ansible 实战：自动化部署动态 Web 集群
published: 2026-08-28
description: 使用 Ansible 在多台 CentOS 主机上统一安装 Nginx 和 Curl，渲染包含节点信息的动态首页，并自动完成服务连通性测试
image: " "
tags: [Ansible,Nginx,CentOS,Web集群,自动化运维]
category: 实战项目
draft: false
---

## 实验目标

这次实验使用 Ansible 部署一个简单的 Web 集群，要求每台节点都能独立完成以下工作：

1. 统一安装 Nginx 和 Curl；
2. 动态生成网站首页；
3. 访问不同节点时显示该节点自己的主机名、IP 和系统版本；
4. 部署完成后自动测试网站，并在终端打印测试结果。

实验主机使用 CentOS Stream 9：

```text
NAME="CentOS Stream"
VERSION="9"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="9"
PLATFORM_ID="platform:el9"
PRETTY_NAME="CentOS Stream 9"
```

## 准备项目目录

先创建项目目录，并在其中准备 Ansible 配置和主机清单：

```bash
mkdir -p ~/ansible-web
cd ~/ansible-web
```

### inventory.ini

```ini
[web]
192.168.8.28
192.168.8.38

[web:vars]
ansible_user=root
ansible_ssh_common_args=-o StrictHostKeyChecking=no
```

这里使用 `web` 作为主机组名称，后面的 Playbook 也必须使用同一个组名。`ansible_user=root` 指定远程登录账号，`ansible_ssh_common_args` 用于关闭第一次连接时的主机密钥确认提示。

### ansible.cfg

在项目目录创建 `ansible.cfg`：

```ini
[defaults]
inventory = /home/qqq/ansible-web/inventory.ini
host_key_checking = False
```

配置完成后，执行 Ansible 命令时可以省略 `-i inventory.ini`。

测试多台主机是否连通：

```bash
ansible all -m ping
```

成功时会看到：

```text
192.168.8.38 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
192.168.8.28 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

## 创建动态首页模板

创建 `index.html.j2`。Jinja2 模板中的变量会在 Playbook 执行时替换为每台目标主机自己的信息：

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>欢迎来到 {{ project_name }}</title>
    <style>
        body { font-family: Arial; text-align: center; margin-top: 50px; }
    </style>
</head>
<body>
    <h1>自动化部署大成功！</h1>
    <h3>当前为您提供服务的节点信息如下：</h3>
    <hr>
    <p><b>项目代号：</b>{{ project_name }}</p>
    <p><b>主机名称：</b>{{ ansible_hostname }}</p>
    <p><b>节点 IP：</b>{{ ansible_default_ipv4.address }}</p>
    <p><b>操作系统：</b>{{ ansible_distribution }} {{ ansible_distribution_major_version }}</p>
    <p><b>部署时间：</b>{{ ansible_date_time.date }} {{ ansible_date_time.time }}</p>
</body>
</html>
```

`ansible_hostname`、`ansible_default_ipv4.address`、`ansible_distribution` 和 `ansible_date_time` 都来自 Ansible 自动收集的 Facts。因此同一个模板发送到不同主机后，网页内容会自动显示对应节点的数据。

## 编写 Playbook

创建 `web.yml`：

```yaml
---
- name: 部署 Web 集群
  hosts: web

  vars:
    project_name: "UwU"

  tasks:
    - name: 安装 Nginx 和 Curl
      dnf:
        name:
          - nginx
          - curl
        state: present

    - name: 发送动态网站首页
      template:
        src: ./index.html.j2
        dest: /usr/share/nginx/html/index.html
        backup: true

    - name: 启动 Web 服务并设置开机自启
      service:
        name: nginx
        state: started
        enabled: true

    - name: 检查 Web 服务端口
      shell: ss -tulnp | grep :80
      register: web_port
      changed_when: false

    - name: 打印端口信息
      debug:
        msg: "{{ web_port.stdout_lines }}"

    - name: 使用 Curl 测试网页
      command: curl http://127.0.0.1
      register: web_response
      changed_when: false

    - name: 打印网页返回内容
      debug:
        msg: "{{ web_response.stdout_lines }}"
```

这个 Playbook 的任务顺序是：安装依赖、发送模板、启动 Nginx、检查 80 端口、使用 Curl 访问本机网页，并打印返回内容。

其中，`register` 用来保存命令执行结果，`debug` 负责输出结果；`changed_when: false` 表示检查类命令不会修改主机状态，因此不会被误报为变更任务。

## 执行部署

在项目目录执行：

```bash
ansible-playbook web.yml
```

执行过程会先收集 Facts，然后在两台主机上依次执行任务。成功时，每台主机都应显示 `failed=0`：

```text
PLAY RECAP
192.168.8.28 : ok=7 changed=2 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
192.168.8.38 : ok=7 changed=2 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

## 验证动态内容

Playbook 最后的 Curl 任务会打印渲染后的 HTML。两台机器使用的是同一个模板，但主机信息不同：

```text
<p><b>项目代号：</b>UwU</p>
<p><b>主机名称：</b>localhost</p>
<p><b>节点 IP：</b>192.168.8.28</p>
<p><b>操作系统：</b>CentOS 9</p>
```

另一台节点会显示自己的地址，例如：

```text
<p><b>节点 IP：</b>192.168.8.38</p>
```

也可以直接从控制节点访问两台服务器的地址，确认每个节点返回了自己的页面信息：

```bash
curl http://192.168.8.28
curl http://192.168.8.38
```

## 实验结果

这次实验把 Ansible 的几个常用能力组合起来：

- 使用 inventory 管理多台 Web 主机；
- 使用变量设置项目名称；
- 使用 `dnf` 安装统一的软件包；
- 使用 `template` 渲染动态 HTML；
- 使用 Facts 获取主机名、IP 和操作系统信息；
- 使用 `service` 管理 Nginx 服务和开机自启；
- 使用 `register` 与 `debug` 保存并打印测试结果。

完成后，多台主机拥有统一的 Nginx 配置和页面结构，同时每台主机都会展示自己的节点信息。这种方式可以作为后续配置负载均衡、批量部署应用和排查集群问题的基础。
