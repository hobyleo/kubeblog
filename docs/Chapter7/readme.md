apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: test-container
    image: registry.cn-beijing.aliyuncs.com/qingfeng666/nginx:latest
    volumeMounts:
    - mountPath: /test-nginx
      name: myhostpath
  volumes:
  - name: myhostpath
    hostPath:
      path: /tmp/nginx
      type: DirectoryOrCreate