# Kubernetes-Job/CronJob

## Job 基本概念

```yaml
apiVersion: batch/v1
kind: Job
metadata:
	name: pi
spec:
	template:
		spec:
			containers:
			- name: pi
			  image: resouer/ubuntu-bc
			  command: ["sh", "-c", "eche 'scale=10000;4*a(1)' | bc -l"]
		 restartPolicy: Never
     backoffLimit: 4
```



