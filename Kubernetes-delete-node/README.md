## Kubernetes集群下节点流程及建议事项

在 Kubernetes 中，删除一个节点是一个相对简单的过程。但在执行此操作之前，请确保该节点上运行的所有工作负载已被适当地迁移或可以被中断。以下是如何删除一个节点的步骤：

1. **首先，将节点标记为不可调度**，这会防止新的 Pods 被调度到这个节点上：

   ```bash
   kubectl cordon NODE_NAME
   ```

   请将 `NODE_NAME` 替换为你打算删除的节点的名称。

2. **驱逐节点上的所有 Pods**。这将尝试把 Pods 移动到其他节点，完成这一步后需要观察集群中是否会新增加无法启动的Pod，应为有些Pod会因为节点调度的关系必须运行在特定节点上（其实就是确认待删除的节点在集群内有可替代性）：

   ```bash
   kubectl drain NODE_NAME --ignore-daemonsets --delete-local-data
   ```

   使用 `--ignore-daemonsets` 选项是因为 DaemonSet 的 Pods 通常会在所有节点上运行，驱逐操作不会对它们产生影响。

   注意: 使用 `--delete-local-data` 会导致删除该节点上的任何 EmptyDir 卷数据。如果你不希望删除 EmptyDir 卷数据，请去掉这个选项，并手动处理这些 Pods。

3. **备份节点的yaml文件(可选)**。如果是重要的环境建议先备份好节点yaml文件，如果删除节点后出现错误，在物理机器还没有移除的情况下是可以将节点重新恢复回来。

   ```bash
   kubectl get node NODE_NAME -o yaml > node.yaml
   ```
4. **从 Kubernetes 集群中删除节点对象**。如果是重要的环境建议先备份好节点yaml文件，如果删除节点后出现错误，在物理机器还没有移除的情况下是可以将节点重新恢复回来。

   ```bash
   kubectl delete node NODE_NAME
   ```

5. **在云提供商或你的基础设施中，停止和删除节点的实际机器**。这一步的实现方式会因为你的环境而异。例如，如果你使用 AWS EC2，你可能需要停止和终止相应的 EC2 实例。推荐不要立刻删除这个机器，可以先停停机，如果集群出现问题也可以通过步骤3备份的yaml文件恢复该节点。如果由于某些 EmptyDir 卷数据出现缺失导致业务无法启动也可以得到恢复。
