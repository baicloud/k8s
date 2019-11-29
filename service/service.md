#### 外部服务service

##### 场景一：使用ip地址和端口

在k8s集群中，应用系统需要使用外部服务作为服务的后端，这时可以通过创建一个无Label Selector的Service来实现

```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

创建的不带标签选择器的service，无法选择后端的Pod，系统不会自动创建endpoint，因此手动创建一个与service同名的Endpoint，用于指向实际的后端访问地址。

```
kind: Endpoints
apiVersion: v1
metadata:
 name: my-service
subsets:
 - addresses:
     - ip: 10.240.0.4
   ports:
     - port: 27017
```

##### 场景二：使用URI地址

service类型为ExternalName

```
apiVersion: v1
kind: Service
metadata:
  name: msyql-db
spec:
  type: ExternalName
  externalName: example.com
```

