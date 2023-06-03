## Kubernetes实现cronjob定期执行shell脚本
### Kubernetes cronjob定期执行shell脚本的作用
很多现有教程只说明如何通过 cronjob 定期执行一个命令，但是在实际生产使用 cronjob 过程中，尤其是需要执行大量 shell 命令时往往不能满足需求。通过执行一个 configmap 关联的shell脚本，既可以灵活调整shell脚本中的命令也能满足 cronjob 运行更复杂的 shell 命令。

在 Kubernetes 中定期执行 Shell 脚本的 CronJob 有很多用途，例如：
 - 定期维护：CronJob 可以被设置为定期执行某些维护任务，例如清理旧的日志文件，更新数据库，维护索引等。
 - 数据备份和恢复：你可以创建一个CronJob，定期对数据库或者文件系统进行备份，如果需要，也可以执行恢复操作。
 - 定期数据处理：如果你的系统需要定期处理数据（例如每天晚上），你可以创建一个 CronJob 来处理这些任务。
 - 监控和报告：CronJob 可以定期执行监控任务，比如检查系统的健康状态，或者生成报告。
 - 自动扩容和缩容：你可以创建 CronJob，根据预测的系统负载，在需要的时候自动增加或减少运行的 Pod 的数量。
 - 定时任务：CronJob 也可以用于执行任何你想在特定时间运行的任务。
### 1. 实现流程
#### 1.1 定义configmap
```bash
cat <<EOF >  script-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: script-configmap  
data: #shell 脚本的命令，可以根据实际情况创建更复杂的命令
  myscript.sh: |
	#!/bin/bash
	echo "Hello, Kubernetes!"
EOF
```
#### 1.2 定义cronjob
```bash
cat <<EOF >  my-cronjob.yaml
apiVersion: batch/v1  #kubernetes 1.21版本后请使用batch/v1，1.21前使用batch/v1beta1
kind: CronJob
metadata:
  name: my-cronjob
spec:
  schedule: "*/1 * * * *" #每分钟执行一次
  concurrencyPolicy: Allow #并发策略，允许并发运行
  jobTemplate:
	spec:
  	template:
    	spec:
      	volumes:
        	- name: script-volume
          	configMap:
            	name: script-configmap
      	containers:
        	- name: my-container
          	image: centos
          	volumeMounts:
            	- name: script-volume
              	mountPath: /scripts
          	command: ["/bin/bash", "/scripts/myscript.sh"]
      	restartPolicy: OnFailure
EOF
```
有关cronjob常配置的api参数：
 - spec.schedule：调度，必需字段，指定任务运行周期，格式同 Cron
 - spec.jobTemplate：Job 模板，必需字段，指定需要运行的任务，格式同 Job
 - spec.startingDeadlineSeconds ：启动 Job 的期限（秒级别），该字段是可选的。如果因为任何原因而错过了被调度的时间，那么错过执行时间的 Job 将被认为是失败的。如果没有指定，则没有期限
 - spec.concurrencyPolicy：并发策略，该字段也是可选的。它指定了如何处理被 Cron Job 创建的 Job 的并发执行。只允许指定下面策略中的一种：
a.Allow（默认）：允许并发运行 Job
b.Forbid：禁止并发运行，如果前一个还没有完成，则直接跳过下一个
c.Replace：取消当前正在运行的 Job，用一个新的来替换

> 注意，当前策略只能应用于同一个 Cron Job 创建的 Job。如果存在多个 Cron Job，它们创建的 Job 之间总是允许并发运行。
 - spec.suspend ：挂起，该字段也是可选的。如果设置为 true，后续所有执行都会被挂起。它对已经开始执行的 Job 不起作用。默认值为 false
 - spec.successfulJobsHistoryLimit 和 .spec.failedJobsHistoryLimit ：历史限制，是可选的字段。它们指定了可以保留多少完成和失败的 Job
#### 1.3 查看cronjob是否正确运行
```bash
#查看cronjob状态
kubectl get cronjob
NAME     	SCHEDULE  	SUSPEND   ACTIVE   LAST SCHEDULE   AGE
my-cronjob   */5 * * * *   False 	2    	19s         	23m

#查看job状态
kubectl get jobs
NAME              	COMPLETIONS   DURATION   AGE
my-cronjob-28096067   1/1       	34s    	3m38s
my-cronjob-28096068   1/1       	34s    	2m38s
my-cronjob-28096069   1/1       	34s    	98s

#查看pod状态
kubectl get po
NAME                    	READY   STATUS  	RESTARTS   AGE
my-cronjob-28096067-qxtqg   0/1 	Completed   0      	5m54s
my-cronjob-28096068-7scqd   0/1 	Completed   0      	4m54s
my-cronjob-28096069-gtlzg   0/1 	Completed   0      	3m54s
```
