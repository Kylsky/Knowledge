## 1.部署文件

deployment方式部署集群需要一个yml文件，格式如下：

```yml
#api版本，默认写下面的内容
apiVersion: extensions/v1beta1
#类型
kind: Deployment
#元数据，说明当前部署文件的名称
metadata: 
  name: tomcat-deploy
#说明
spec:
  #集群数
  replicas: 2 
  #模板
  template: 
    metadata:
      labels:
        #pod名称
        app: tomcat-cluster
    spec:
      volumes: 
      - name: web-app
        hostPath:
          path: /mnt
      containers:
      - name: tomcat-cluster
        image: tomcat:latest
        resources:
          requests:
            cpu: 0.5
            memory: 200Mi
          limits:
            cpu: 1
            memory: 512Mi
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: web-app
          mountPath: /usr/local/tomcat/webapps

```

