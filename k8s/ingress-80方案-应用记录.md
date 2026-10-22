# Ingress 80 方案应用记录(2026-10-22,P3 收官)

## 背景
- 服务器重启后公网 IP 全部变更(master .106→8.155.137.114 / node1 .131.183→8.155.137.246 / node2 .170 不变)。
- 家宽→阿里云方向实测:**只有 ingress controller pod 所在节点(k8s-node2)的 NodePort 能通**;
  其他节点收到 SYN 后要跨节点 DNAT 转发给 pod,这一方向的流量被丢(31124 间歇性超时的真因)。
- 结论:与其和 31124 的高位端口抽风搏斗,不如改用安全组老资格的 80/443。

## 三处变更(均已应用并验证)
1. kube-apiserver 扩 NodePort 范围(kubeadm 默认 30000-32767,不含 80):
   `/etc/kubernetes/manifests/kube-apiserver.yaml` 在 `--service-cluster-ip-range` 后插入
   `- --service-node-port-range=1-32767`
   (静态 pod 自动重启约 40~70 秒;原文件备份 `/root/kube-apiserver.yaml.bak-20261022`)
2. ingress-nginx svc 改 nodePort:
   `kubectl -n ingress-nginx patch svc ingress-nginx-controller -p "$(cat k8s/ingress-svc-nodeport80.json)"`
3. ingress controller 钉死 k8s-node2(防 pod 漂移后公网入口失联):
   ```
   kubectl -n ingress-nginx patch deploy ingress-nginx-controller --type merge \
     -p '{"spec":{"template":{"spec":{"nodeSelector":{"kubernetes.io/hostname":"k8s-node2"}}}}}'
   ```

## 连带修正
- WordPress siteurl/home 去端口(mysql 在 magedu 命名空间):
  ```
  kubectl -n magedu exec deploy/mysql -- mysql -uwpuser -p$WP_PASS wordpress -e \
  "update wp_options set option_value='http://wordpress.magedu.com' where option_name in ('siteurl','home');"
  ```
- Windows hosts:`wordpress.magedu.com → 8.155.135.170(k8s-node2)`

## 验证
- master 本机:`curl -H "Host: wordpress.magedu.com" http://localhost:80/` → 200
- 家宽浏览器 http://wordpress.magedu.com → 首页正常(工作区 `证据图片/第二阶段/04-P3-wordpress正门-经ingress-80端口.png`)
- 注意:NodePort 是 iptables DNAT,节点上 `ss -tlnp` 看不到 80 监听属正常

## 教训(面试素材)
- "跨节点 DNAT 的公网流量被丢"——NodePort 从公网通不通,取决于 ingress pod 落在哪个节点,
  而不是安全组;hosts 只指 pod 所在节点,或把 controller 改 DaemonSet。
- kubeadm 集群改 apiserver 参数要改静态 pod manifest,保存即自动生效;`kubeadm upgrade` 可能还原,需复查。
