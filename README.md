# my-k8s-ops · 从零 K8s 云原生运维平台

三节点 kubeadm 集群（阿里云 ECS · Ubuntu 22.04 · containerd · Calico）：自建集群 → WordPress 业务上云 → 备份恢复 / 监控告警 / 日志 / 弹性伸缩 / CI/CD 全链路。与《传统架构项目》构成同一业务的演进叙事：从手工部署到声明式 API。

## 架构演进（传统 → 云原生）

| 传统架构（第一阶段） | 云原生方案（本项目） | 一句话收益 |
|---|---|---|
| hosts 绑 IP + Nginx 虚拟主机 | Ingress 统一入口（ingress-nginx，NodePort 80/443） | 加站点不动入口，域名即路由 |
| NFS 手工挂载共享目录 | PVC + StorageClass（local-path） | 存储声明式供给，申请即得 |
| crontab + 备份脚本 | CronJob 备份 + 恢复 Job | 备份/恢复都是平台对象，可审计可重放 |
| Zabbix 逐台手工安装 | Prometheus + Grafana + Alertmanager 手写清单 | 指标与告警即代码 |
| 登服务器 grep 日志 | Loki + Promtail + LogQL | 按标签秒级检索全集群日志 |
| 手动加 Tomcat 实例 | HPA 自动扩缩（min1/max4，CPU 60%） | 弹性交给控制器，实测 1→4 一步到位 |
| scp war 包停机换发布 | GitHub Actions + ACR + 滚动更新 | 发布不停机，rollout undo 一键回滚 |

## 目录结构

```
ansible/                            # 节点标准化剧本（hosts/内核/免密）
apps/
  mysql/                            # Secret(example)/PVC/Deployment/备份 CronJob/恢复 Job
  wordpress/                        # PVC/Deployment/Service + HPA + PDB
  myweb/                            # CI/CD 演示应用：Dockerfile/index.html/deploy/svc/ingress
  monitoring/                       # node-exporter/Prometheus/Grafana/mysqld-exporter/Alertmanager
  metrics-server.yaml               # metrics-server（daocloud 镜像 + insecure-tls patch）
k8s/                                # ingress 清单与 apiserver nodePort-range patch
docs/                               # 每章应用记录与排错实录
.github/workflows/deploy.yml        # 双 job 流水线：云端构建推 ACR + 自建 Runner 内网部署
```

## 里程碑（commit 时间线 = 项目时间线）

| 章 | 内容 | 实测结果 |
|---|---|---|
| P1 | MySQL 上 K8s | 删 Pod 自愈：3 秒拉起、数据保留 |
| P2 | WordPress 上 K8s | Secret 注入/双探针/PVC，文章落库 |
| P3 | Ingress 正门 | 定位跨节点 DNAT 丢包，80 方案 + controller 钉桩 |
| P4 | 备份 CronJob + 恢复演练 | DROP 库 → **恢复 1 秒** → 站点复活、文章零丢失 |
| P5 | 监控告警全栈（手写清单） | 停库 **80s firing → 2min 邮件** → 恢复 70s 自动闭环 |
| P6 | 日志栈（Helm loki-stack） | LogQL 精确命中目标日志行 |
| P7 | metrics-server + HPA + 压测 + PDB | CPU **502%** → **1→4 一步到位**；压测 **34.58 QPS**；RWO 伪弹性反转；缩容删人实测反例 |
| P8 | CI/CD 全链路 | **云端构建 + 自建 Runner 内网部署，安全组零改动**；v2 发布 49s 全绿 + 浏览器验收 |

## 备份与恢复演练（简历核心数字）

- 备份：CronJob `*/30 * * * *`，`mysqldump --single-transaction --source-data=2` 一致性快照内嵌位点 → 专用 PVC，MD5 + 位点文件，保留 7 份，`concurrencyPolicy: Forbid`。
- 演练：手动 `DROP DATABASE` → 站点 500 → apply 恢复 Job（取最新 dump → 清库 → 灌回 → 计时 → 自检）→ **恢复完成耗时 1 秒**，站点回 200、文章零丢失。

## 证据索引

01 集群三节点 Ready · 02-03 WordPress 上云 · 04 Ingress 80 正门 · 05-06 恢复演练 1 秒 · 07-08 Grafana 看板 · 09-10 告警 FIRING/QQ 邮件 · 11 LogQL 命中 · 12 HPA 502%→4 副本 · 13 压测 34.58 QPS · 14 缩容 events + PDB · 15-16 Actions 全绿（含首跑红叉复盘）· 17 v2 页面验收 · 18 rollout 终端三证

## 安全纪律

真实 Secret 一律 `.gitignore`（见本仓 `.gitignore`），仓库只保留 `*.yaml.example` 全占位符副本；公开仓 push 前三层扫雷：文件名、内容、全历史（本项目曾以 `git filter-repo` 清洗历史并强推）。CI 凭据专用化：流水线七件 Secrets 管理，部署侧走集群内自建 Runner，不新增入站端口。
