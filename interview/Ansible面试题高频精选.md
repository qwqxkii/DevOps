# Ansible 面试题高频精选

> 适用方向：Linux 运维、系统运维、DevOps、SRE 初中级岗位  
> 排序依据：三份资料的重复程度、运维面试中的实际常见程度，以及问题对候选人实操能力的区分度。  
> 频率为经验分级，不是严格统计数据。建议先掌握第一部分，再准备第二、第三部分。

## 一、必问级：★★★★★

### 1. 什么是 Ansible？它的工作原理和主要优势是什么？

Ansible 是一个 IT 自动化工具，常用于批量配置管理、软件部署、任务编排和应用发布。

基本流程：控制节点读取 Inventory 和 Playbook，通过 SSH 连接 Linux/Unix 受管节点，将模块及参数发送到远端执行，取得结果后清理临时内容。Windows 主机通常使用 WinRM。

主要优势：

- Agentless：受管端通常无需常驻 Agent，降低部署和维护成本。
- Playbook 使用 YAML，可读性较好。
- 模块丰富，可覆盖软件包、文件、服务、用户、云资源等管理场景。
- 大多数内置模块具备幂等性，Playbook 可以安全重复执行。
- 支持变量、模板、Role、条件和循环，便于复用。

注意：Agentless 不等于受管端完全没有依赖。Linux 主机通常仍需 SSH 和可用的 Python 解释器；`raw` 模块是常见例外。

### 2. 什么是幂等性？为什么它很重要？

幂等性是指一个自动化任务执行一次或重复执行多次，最终系统都处于相同的期望状态。如果目标已经符合要求，任务应返回 `ok`，而不是重复修改。

例如：

```yaml
- name: Ensure nginx is installed
  ansible.builtin.package:
    name: nginx
    state: present
```

无论执行多少次，都只是保证 nginx 已安装。幂等性使 Playbook 能够安全重跑，减少重复创建、重复追加、反复重启等副作用，是配置一致性和故障恢复的基础。

常见破坏幂等性的写法是直接使用 `shell` 执行追加命令，例如 `echo xxx >> file`。应优先使用 `lineinfile`、`template`、`user`、`package`、`service` 等状态型模块，必要时配合 `creates`、`removes`、`changed_when`。

### 3. Playbook 是什么？主要由哪些部分组成？

Playbook 是用 YAML 编写的自动化任务定义文件。它由一个或多个 Play 组成，每个 Play 把一组主机与要执行的任务关联起来。

常见关键字：

- `hosts`：目标主机或主机组。
- `tasks`：按顺序执行的任务。
- `become`：是否进行权限提升。
- `vars`：变量。
- `handlers`：被 `notify` 触发的特殊任务。
- `roles`：引用可复用 Role。
- `when`、`loop`、`tags`：条件、循环和标签。

```yaml
---
- name: Deploy nginx
  hosts: web
  become: true
  tasks:
    - name: Install nginx
      ansible.builtin.package:
        name: nginx
        state: present
```

### 4. Inventory 是什么？如何组织主机和变量？

Inventory 用来定义受管主机、主机分组、连接参数及相关变量，可使用 INI、YAML 或动态 Inventory 插件。

```ini
[web]
web01 ansible_host=192.168.1.11
web02 ansible_host=192.168.1.12

[db]
db01 ansible_host=192.168.1.21

[all:vars]
ansible_user=ops
```

生产环境通常将公共变量放在 `group_vars/`，单机差异放在 `host_vars/`，避免把大量变量直接堆在 Inventory 主文件中。

常用检查命令：

```bash
ansible-inventory -i inventory.ini --graph
ansible-inventory -i inventory.ini --host web01
ansible all -i inventory.ini -m ping
```

### 5. 常用 Ansible 模块有哪些？为什么不建议滥用 shell/command？

常用模块包括：

- 软件包：`package`、`dnf`、`apt`
- 文件：`copy`、`template`、`file`、`lineinfile`、`unarchive`
- 服务：`systemd_service`、`service`
- 用户与权限：`user`、`group`、`authorized_key`
- 命令：`command`、`shell`
- 网络与下载：`uri`、`get_url`、`wait_for`
- 排错：`ping`、`setup`、`debug`

应优先使用专用模块，因为它们通常会检查当前状态，具有更好的幂等性、返回值和错误处理。`command` 不经过 Shell，不能直接使用管道、重定向等 Shell 特性；确实需要这些特性时才使用 `shell`，并通过 `creates`、`removes` 或条件判断控制重复执行。

### 6. Ansible 变量可以在哪里定义？优先级如何理解？

变量可定义在 Role defaults、Inventory、`group_vars`、`host_vars`、Play 的 `vars`、Role vars、任务变量、`set_fact`、`register` 结果以及命令行 `-e` 中。

面试时不必死背完整的二十多级优先级，但应记住：

- Role 的 `defaults/main.yml` 优先级很低，适合放可被覆盖的默认值。
- `group_vars`、`host_vars` 适合维护环境和主机差异。
- Role 的 `vars/main.yml` 优先级较高，不适合放希望调用者轻易覆盖的默认值。
- 命令行 Extra Vars（`-e`）优先级最高。

发生变量覆盖时，可使用 `ansible-inventory --host <主机>`、`debug: var=变量名` 和 `-vvv` 排查变量实际值及来源。

### 7. Role 是什么？标准目录结构有哪些？

Role 是对任务、变量、模板、文件和 Handler 的标准化封装，用于提高 Playbook 的复用性和可维护性。

```text
roles/nginx/
├── defaults/main.yml
├── vars/main.yml
├── tasks/main.yml
├── handlers/main.yml
├── templates/
├── files/
├── meta/main.yml
└── tests/
```

其中 `tasks/main.yml` 是主要任务入口；`handlers/main.yml` 定义 Handler；`templates/` 存放 Jinja2 模板；`files/` 存放直接分发的静态文件；`defaults/` 放低优先级默认变量。

### 8. Facts 是什么？如何查看和使用？

Facts 是 Ansible 从受管主机收集的系统信息，例如操作系统、IP 地址、CPU、内存和网络接口。Play 默认通过 `setup` 模块收集 Facts。

```bash
ansible web01 -m ansible.builtin.setup
ansible web01 -m ansible.builtin.setup -a 'filter=ansible_distribution*'
```

```yaml
- name: Install package on RedHat family
  ansible.builtin.dnf:
    name: nginx
    state: present
  when: ansible_facts['os_family'] == 'RedHat'
```

如果 Playbook 不需要 Facts，可设置 `gather_facts: false` 缩短执行时间；大规模环境也可配置 Fact Cache。

### 9. Handler 和 notify 是什么？什么时候使用？

Handler 是由任务通过 `notify` 通知后才执行的特殊任务，常用于配置文件发生变化后重启或重载服务。

```yaml
tasks:
  - name: Render nginx config
    ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Reload nginx

handlers:
  - name: Reload nginx
    ansible.builtin.service:
      name: nginx
      state: reloaded
```

默认情况下 Handler 在 Play 的相应阶段末尾执行；即使被多次通知，同一个 Handler 通常也只执行一次。如必须立即执行，可使用 `meta: flush_handlers`。

### 10. 如何使用 Ansible Vault 管理密码和密钥？

Ansible Vault 用于加密变量文件或单个变量，防止数据库密码、API Key 等敏感信息以明文进入代码仓库。

```bash
ansible-vault create secrets.yml
ansible-vault edit secrets.yml
ansible-vault encrypt_string 'P@ssw0rd' --name db_password
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file /secure/path/vault-pass
```

Vault 保护的是静态存储数据。Vault 密码文件本身不能提交到 Git，CI/CD 中应由凭据系统或专用 Secret 管理系统注入，并避免通过日志输出敏感变量，可对相关任务设置 `no_log: true`。

### 11. Ad-Hoc 命令和 Playbook 有什么区别？

Ad-Hoc 是通过 `ansible` 命令临时执行一次性任务，适合连通性检查、临时查询或紧急批处理；Playbook 是可保存、可审查、可重复运行的自动化流程。

```bash
ansible all -m ping
ansible web -b -m package -a 'name=nginx state=present'
ansible all -m command -a 'uptime'
```

注意：Ad-Hoc Command 不是“剧本”。需要长期维护、多人协作或多步骤编排时，应写 Playbook 并纳入版本控制。

### 12. 你实际写过哪些 Playbook？如何讲项目经验？

不要只回答“写过 nginx、Zabbix”。建议按以下结构回答：

1. 场景和规模：管理多少台、什么系统、什么环境。
2. 目标：解决什么手工操作或一致性问题。
3. 设计：Inventory、Role、变量、模板、Vault 如何组织。
4. 安全性：如何测试、灰度、回滚和保护密钥。
5. 结果：节省多少时间、失败率或配置漂移是否下降。

示例：我用 Ansible 为约 100 台主机批量部署 Zabbix Agent。按系统版本在 `group_vars` 中维护包名和仓库地址，通过 Jinja2 生成配置文件，用 Handler 在配置变更时重启服务。上线前先跑 `--syntax-check` 和测试组，再通过 `serial` 分批发布；敏感 PSK 用 Vault 管理。

## 二、高频级：★★★★☆

### 13. Jinja2 模板有什么作用？template 和 copy 有什么区别？

Jinja2 用于根据变量、条件、循环和过滤器生成动态配置文件。

- `copy`：复制内容基本固定的文件。
- `template`：渲染 `.j2` 模板，不同主机可生成不同配置。

```jinja2
worker_processes {{ ansible_facts['processor_vcpus'] | default(2) }};
{% for backend in backend_servers %}
server {{ backend.host }}:{{ backend.port }};
{% endfor %}
```

生产中可在更新配置后用 `validate` 参数先校验语法，例如用 `nginx -t` 检查待发布配置，再触发 Handler。

### 14. when、loop、register 分别怎么用？

- `when`：条件执行任务。
- `loop`：遍历列表，重复执行任务。
- `register`：保存任务返回结果，供后续条件判断或调试。

```yaml
- name: Check nginx config
  ansible.builtin.command: nginx -t
  register: nginx_check
  changed_when: false

- name: Show result
  ansible.builtin.debug:
    var: nginx_check.stderr
  when: nginx_check.rc == 0
```

### 15. become 是什么？与 remote_user 有什么区别？

`remote_user` 或 `ansible_user` 决定使用哪个账号建立 SSH 连接；`become: true` 表示连接后再通过 sudo 等方式提升权限，`become_user` 指定切换到哪个用户，默认通常是 root。

```yaml
- hosts: web
  remote_user: ops
  become: true
  become_user: root
```

出现 `Permission denied` 时，应分别检查 SSH 用户和密钥、文件权限、sudoers 配置、是否需要 `become`，而不是只盯着网络。

### 16. 如何调试执行失败的 Playbook？

建议按层排查：

1. `ansible-inventory --graph` 检查 Inventory 和目标范围。
2. 手工 `ssh`，再用 `ansible <host> -m ping -vvv` 检查连接、认证和 Python。
3. `ansible-playbook site.yml --syntax-check` 检查语法。
4. 使用 `--list-hosts`、`--list-tasks`、`--list-tags` 确认执行对象。
5. 使用 `-vvv` 查看模块参数和返回值，但注意日志可能包含敏感信息。
6. 用 `debug` 查看变量，用 `--start-at-task`、`--step`、`--limit` 缩小范围。
7. 到目标机检查服务日志、权限、磁盘、SELinux、防火墙和依赖。

注意：`--check` 是模拟执行，不是所有模块都能完全准确支持，不能把它当成绝对可靠的测试结果。

### 17. --syntax-check、--check、--diff 分别有什么作用？

- `--syntax-check`：检查 YAML 和 Playbook 基本语法，不执行任务。
- `--check`：Check Mode，预测可能发生的变更，尽量不真正修改系统。
- `--diff`：显示模板或文件修改前后的差异，常与 `--check` 组合使用。

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check --diff --limit test
```

### 18. 如何实现滚动更新，尽量减少停机？

使用 `serial` 控制每批处理的主机数，再结合负载均衡摘除、健康检查、失败阈值和 Handler。

```yaml
- hosts: web
  serial: 2
  max_fail_percentage: 20
  tasks:
    - name: Deploy application
      ansible.builtin.unarchive:
        src: app.tar.gz
        dest: /opt/app
      notify: Restart app

    - name: Wait for service
      ansible.builtin.wait_for:
        port: 8080
        timeout: 60
```

真实生产流程通常是：从负载均衡摘除当前批次 → 部署 → 启动并健康检查 → 加回负载均衡 → 处理下一批。

### 19. delegate_to 和 run_once 有什么区别？

- `delegate_to`：把当前任务委托给另一台主机执行。
- `run_once`：该任务在当前批次中只执行一次，而不是每台目标主机都执行。

二者可以组合，例如只在控制节点执行一次数据库迁移：

```yaml
- name: Run database migration once
  ansible.builtin.command: /opt/app/migrate.sh
  run_once: true
  delegate_to: db_admin_host
```

`local_action` 可理解为委托到控制节点的简写，但目前更推荐明确写 `delegate_to: localhost`。

### 20. 如何提高 Ansible 管理大量主机时的执行速度？

常见方法：

- 根据控制节点和网络承载能力合理增大 `forks`。
- 不需要主机信息时设置 `gather_facts: false`，或启用 Fact Cache。
- 启用 SSH ControlMaster/ControlPersist 和 `pipelining`，减少连接开销。
- 使用 `strategy: free` 让快主机不必一直等待慢主机，但要确认任务无跨主机顺序依赖。
- 缩小执行范围，合理使用 `--limit` 和 `tags`。
- 避免低效循环、重复下载和不必要的任务，优先使用批量能力强的模块。
- 大文件同步可评估 `synchronize`，同时关注控制节点 CPU、文件句柄和网络瓶颈。

`pipelining` 可能与部分 sudo 配置冲突，上线前应验证，不能只为速度盲目开启。

## 三、常问进阶级：★★★☆☆

### 21. 静态 Inventory 和动态 Inventory 有什么区别？

静态 Inventory 由人工维护 INI/YAML 主机列表，适合规模较小、主机变化少的环境。动态 Inventory 通过插件从云平台、虚拟化平台或 CMDB 获取实时主机数据，适合实例经常创建和销毁的环境。

企业环境优先使用官方或对应 Collection 提供的 Inventory Plugin，而不是维护脆弱的自定义脚本；同时应利用缓存、标签分组并控制查询权限。

### 22. include_tasks 和 import_tasks 有什么区别？

- `import_tasks` 是静态导入，在 Playbook 解析阶段展开，适合固定任务结构，便于静态检查和列出任务。
- `include_tasks` 是运行时动态包含，可根据变量选择文件，也可以配合循环。

简单记忆：结构固定用 `import_tasks`，需要运行时决定或循环包含用 `include_tasks`。

### 23. 如何处理任务失败、定义失败条件或实现恢复？

常用机制包括：

- `ignore_errors`：忽略失败继续执行，应谨慎使用。
- `failed_when`：自定义什么情况算失败。
- `changed_when`：自定义什么情况算发生变更。
- `block`、`rescue`、`always`：实现类似异常捕获和清理逻辑。
- `any_errors_fatal`、`max_fail_percentage`：控制多主机执行的失败策略。

```yaml
- block:
    - name: Deploy new version
      ansible.builtin.command: /opt/deploy.sh
  rescue:
    - name: Roll back
      ansible.builtin.command: /opt/rollback.sh
  always:
    - name: Record deployment result
      ansible.builtin.debug:
        msg: Deployment finished
```

### 24. tags 有什么用途？如何只执行部分任务？

Tags 用于给任务或 Role 分类，运行时通过 `--tags` 只执行指定部分，或用 `--skip-tags` 跳过某些任务。

```yaml
- name: Install nginx
  ansible.builtin.package:
    name: nginx
    state: present
  tags: [nginx, install]
```

```bash
ansible-playbook site.yml --tags nginx
ansible-playbook site.yml --skip-tags restart
```

### 25. Ansible 如何接入 CI/CD？

典型流程是：代码提交 → YAML/`ansible-lint` 检查 → 测试环境运行 → 人工审批或变更窗口 → 通过 `--limit`、`serial` 灰度生产 → 健康检查 → 保留日志和制品 → 失败时回滚。

关键点：

- Playbook、Role 和 Inventory 进入 Git 管理并进行代码评审。
- 不把 Vault 密码或 SSH 私钥写进仓库和流水线日志。
- 固定 Ansible Core、Collection 和 Role 版本，避免环境漂移。
- 使用专用自动化账号和最小权限。
- 生产发布设置并发、失败阈值、超时和审计。

### 26. Ansible Tower、Automation Controller 和 AWX 是什么？

AWX 是社区上游项目；Red Hat Ansible Automation Platform 中对应的企业控制组件称为 Automation Controller，历史上常被称为 Ansible Tower。

相对纯 CLI，它们提供 Web UI、API、RBAC、凭据管理、作业模板、调度、集中日志、通知和工作流，适合团队协作与审计。它们不是 Ansible 执行原理的替代品，而是企业级控制和治理层。

## 四、建议删除或降低优先级的题目

以下内容不是完全没用，但对一般 Linux 运维面试的投入产出比较低，除非 JD 明确涉及再准备：

- 自定义 Module、Callback、Lookup Plugin 的开发与分发细节。
- Ansible Galaxy 与 Collections 的完整发布流程。
- 特定云厂商动态 Inventory 插件的参数细节。
- 用 Ansible 实现“不可变基础设施”的理论比较。
- Ansible 与 Kubernetes 的宽泛选型讨论。
- 已逐渐过时的 `accelerate` 模式。
- 冷门 Strategy Plugin、Callback Plugin 的内部实现。

## 五、面试前速记清单

面试前至少确认自己能够脱稿说明：

- Ansible 从控制节点到受管节点的执行过程。
- Agentless、SSH、Python、模块和幂等性的关系。
- Inventory、Playbook、Task、Module、Role 各自负责什么。
- 变量常见定义位置及 `-e` 最高、Role defaults 最低。
- `template + notify + handler` 的完整配置发布流程。
- `when`、`loop`、`register`、`become` 的基本用法。
- Vault 如何使用，Vault 密码本身如何保管。
- `-vvv`、`--syntax-check`、`--check --diff`、`--limit` 如何排错。
- `serial + delegate_to + wait_for` 如何完成滚动更新。
- 至少准备一个自己真实做过或能完整讲清楚的 Playbook 项目。

## 六、资料来源

- [博客园：ansible 是什么？怎么玩？给出常问的面试题和答案](https://www.cnblogs.com/peteremperor/p/19703281)
- [LabEx：Ansible 面试题及答案](https://labex.io/zh/tutorials/ansible-ansible-interview-questions-and-answers-593672)
- [行癫代码库：企业级 Ansible 常见面试题](https://gitea.xingdiancloud.com/diandian/ansible/src/commit/f72a1b83f307fca9cb89cca425cf3f2cc03bf058/ansible-main/%E4%BC%81%E4%B8%9A%E7%BA%A7Ansible%E5%B8%B8%E8%A7%81%E9%9D%A2%E8%AF%95%E9%A2%98.md)

> 注：本文对原资料进行了交叉去重、纠错和改写，并依据通用 Linux 运维面试经验调整顺序；“出现频率”属于经验判断，而非来自公开样本的精确统计。
