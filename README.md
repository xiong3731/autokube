# Autokube

使用 Ansible 自动化部署与管理 Kubernetes 集群的项目模板。

- 通过分角色的方式组织安装与配置（如 `containerd`、`cni`、`kube_components`、`api_lb`、`cert` 等）
- 可按需启用或跳过特定角色，支持分阶段部署与干跑校验
- 约定清晰的目录结构，便于扩展、新增自定义任务与变量

## 目录结构

项目关键目录与文件如下（节选）：

```
ansible.cfg                 # Ansible 全局配置
inventory/staging/          # 示例环境的清单与变量
  ├─ hosts.yaml             # 主机与分组定义
  └─ group_vars/            # 组级变量（按需新增）
playbooks/
  ├─ main.yaml              # 主流程 Playbook（正式部署）
  └─ test.yaml              # 试运行/校验 Playbook（可选）
roles/
  ├─ api_lb/                # API 负载均衡相关角色
  ├─ bootstrap/             # 基础初始化与系统准备
  ├─ cert/                  # 证书签发与分发
  ├─ cni/                   # 容器网络（如 Calico/Flannel 等）
  ├─ common/                # 通用工具与配置
  ├─ containerd/            # 容器运行时（containerd）
  ├─ docker/                # 容器运行时（docker，可选）
  ├─ kube_components/       # Kubernetes 组件（kubelet、kubeadm、kubectl 等）
  └─ test_cluster/          # 测试/演示用角色
```

## 环境要求

- 控制端：macOS 或 Linux，建议 Python 3.8+，Ansible 2.10+（或更高）
- 目标节点：Ubuntu 24.04 LTS，可通过 SSH 访问，具备 `sudo` 权限，系统时间同步
- 已配置好密钥登录或可交互密码登录（推荐密钥）

## 快速开始

1) 安装 Ansible（任选其一）：

- macOS：`brew install ansible`
- 通用：`pip install ansible`

2) 配置清单文件 `inventory/staging/hosts.yaml`（示例）：

```yaml
all:
  children:
    masters:
      hosts:
        master1:
          ansible_host: 10.0.0.10
    workers:
      hosts:
        worker1:
          ansible_host: 10.0.0.11
    # 供 playbooks 使用的聚合组：包含 masters 与 workers
    cluster:
      children:
        masters:
        workers:
    # 主控节点别名组：与 masters 重叠，用于某些 play
    master:
      hosts:
        master1:
          ansible_host: 10.0.0.10
    # 引导/初始化阶段主控节点组
    master_bootstrap:
      hosts:
        master1:
          ansible_host: 10.0.0.10
    lb:
      hosts:
        lb1:
          ansible_host: 10.0.0.20
```

3) 设置组变量（可选）：在 `inventory/staging/group_vars/` 下创建对应组名的 `yaml` 文件，覆盖默认变量。例如：

```yaml
# inventory/staging/group_vars/masters.yaml
kubernetes_version: "1.29.0"
container_runtime: "containerd"
```

4) 连通性测试：

- `ansible -i inventory/staging/hosts.yaml all -m ping`

5) 干跑（不改动远端，仅检查）：

- `ansible-playbook -i inventory/staging/hosts.yaml playbooks/test.yaml --check`

6) 正式部署：

- `ansible-playbook -i inventory/staging/hosts.yaml playbooks/main.yaml`

## Playbooks 说明

本仓库包含两个主要 Playbook：

- `playbooks/main.yaml`：完整集群部署流程，按以下顺序执行：
  - `common tasks`（hosts: `cluster`）：运行 `roles/common`，进行通用系统准备与工具安装。
  - `install runtime`（hosts: `cluster`）：根据变量 `enable_docker` 选择安装 `docker` 或 `containerd`。
  - `kube_components`（hosts: `cluster`）：调用 `roles/kube_components` 的 `download_and_install.yaml`，下载并安装 `kubeadm`、`kubectl`、`kubelet` 等组件。
  - `init certificates`（hosts: `cluster`）：运行 `roles/cert`，初始化与分发所需证书。
  - `etcd`（hosts: `master`）：执行 `roles/kube_components/tasks/etcd.yaml`，部署或配置 `etcd`。
  - `api_lb`（hosts: `master`）：执行 `roles/api_lb/tasks/nginx.yaml`，为 API Server 配置 Nginx 负载均衡。
  - `kube-apiserver`（hosts: `master`）：执行 `roles/kube_components/tasks/kube_apiserver.yaml`，部署 API Server。
  - `kube-controller-manager`（hosts: `master`）：执行 `roles/kube_components/tasks/kube_controller_manager.yaml`，部署 Controller Manager。
  - `kube-scheduler`（hosts: `master`）：执行 `roles/kube_components/tasks/kube_scheduler.yaml`，部署 Scheduler。
  - `TLS Bootstrapping`（hosts: `master_bootstrap`）：预留 TLS 引导阶段（目前为占位，按需补充）。
  - `kube-proxy`（hosts: `cluster`）：执行 `roles/kube_components/tasks/kube_proxy.yaml`，部署 kube-proxy。
  - `kubelet`（hosts: `cluster`）：执行 `roles/kube_components/tasks/kubelet.yaml`，部署 kubelet 并加入节点。
  - `cni`（hosts: `master_bootstrap`）：当 `cni == "calico"` 时执行 `roles/cni/tasks/calico.yaml` 安装 Calico。
  - `test cluster`（hosts: `master_bootstrap`）：运行 `roles/test_cluster` 进行基本验证。

- `playbooks/test.yaml`：仅安装 CNI（Calico）用于试跑与验证（hosts: `master_bootstrap`）。

## 变量与定制

- 每个角色的默认变量位于 `roles/<role>/defaults/main.yaml`；如需自定义，优先在 `inventory/staging/group_vars/` 或 `host_vars/` 中覆盖。
- 也可通过命令行传入：`ansible-playbook ... -e "kubernetes_version=1.29.0 container_runtime=containerd"`
- 常见可定制项：
  - `kubernetes_version`：K8s 版本
  - `container_runtime`：`containerd` 或 `docker`
  - `cni_type`：如 `calico`、`flannel` 等（具体取决于 `roles/cni` 的实现）

## 按标签运行

可仅执行某些角色或步骤（示例标签，按项目实际角色实现为准）：

- 仅安装容器运行时：
  - `ansible-playbook -i inventory/staging/hosts.yaml playbooks/main.yaml --tags containerd`
- 仅部署 Kubernetes 组件：
  - `ansible-playbook -i inventory/staging/hosts.yaml playbooks/main.yaml --tags kube_components`
- 仅部署 CNI：
  - `ansible-playbook -i inventory/staging/hosts.yaml playbooks/main.yaml --tags cni`

## 常见问题与排查

- 使用 `--syntax-check` 与干跑：
  - `ansible-playbook -i inventory/staging/hosts.yaml playbooks/main.yaml --syntax-check`
  - `ansible-playbook -i inventory/staging/hosts.yaml playbooks/main.yaml --check`
- SSH/权限问题：确保控制端能免密连接并且远端具备 `sudo` 权限（`ansible_become: true`）。
- 版本不匹配：检查各角色 `defaults/main.yaml` 与组变量的版本是否一致。
- 幂等性：大多数任务设计为幂等，若发现重复变更，请开启更详细日志并定位对应角色任务。

## 开发与扩展

- 新增角色：在 `roles/` 下创建目录与 `tasks/main.yaml`、`defaults/main.yaml` 等文件。
- 共享变量：优先在 `group_vars`/`host_vars` 管理，避免在 Playbook 内硬编码。
- 代码质量：建议使用 `ansible-lint` 与 `yamllint` 进行基础检查。

## 免责声明

- 本项目为示例/模板性质，具体生产环境需根据自身要求进行审查与加固（安全、备份、监控、日志等）。
- 许可证暂未指定（如需开源，请补充 LICENSE 文件）。

---

如需我根据你的实际节点拓扑和目标版本生成更完善的 `inventory` 与变量示例，请告诉我节点 IP、角色分布与期望的 Kubernetes/CNI 版本。