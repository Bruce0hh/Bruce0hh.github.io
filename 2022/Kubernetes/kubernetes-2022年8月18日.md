[TOC]

# Kubernetes-RBAC

> 在 Kubernetes 项目中，负责完成授权工作的机制，就是RBAC：基于角色的访问控制（Role-Based Access Control）。

## 基本概念

1. **Role**：角色，它其实是一组规则，定义了一组对Kubernetes API对象的操作权限。
2. **Subject**：被作用者，可以是“人”，也可以是“机器”，也可以是你在Kubernetes里定义的“用户”。
3. **RoleBinding**：定义了“被作用者”和“角色”的绑定关系。

## Role

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: mynamespace
  name: example-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

- `namespace`是Kubernetes项目里的一个逻辑管理单位。不同Namespace的API对象，在通过kubectl命令进行操作的时候，是互相隔离开的。
- `rules`字段，就是Role对象所定义的权限规则。上述规则的含义就是：允许“被作用者”，对mynamespace下面的Pod对象，进行GET、WATCH和LIST操作。

## RoleBinding

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: ServiceAccount
  name: example-sa
  namespace: mynamespace
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

- 通过`roleRef`字段，Rolebinding对象就可以直接通过名字，来引用我们前面定义的Role对象（example-role），从而定义了Subject和Role之间的绑定关系。
- 当你希望某一个Role作用于所有的Namespace的时候，可以使用`ClusterRole`和`ClusterRoleBinding`这两个组合，它们的作用域范围不受Namespace的限制。

## ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: mynamespace
  name: example-sa
```

Kubernetes会为一个ServiceAccount自动创建并分配一个Secret对象。这个Secret，就是这个ServiceAccount对应的、用来跟APIServer进行交互的授权文件，我们一般称它为：Token。Token文件的内容一般是证书或者密码，它以一个Secret对象的方式保存在Etcd当中。

如果一个Pod没有声明serviceAccountName，Kubernetes会自动在它的Namespace下创建一个名叫default的默认ServiceAccount，然后分配给这个Pod。

一个ServiceAccount，在Kubernetes里对应的“用户”的名字是：`system:serviceaccount:<Namespace名字>:<ServiceAccount名字>`，而它对应的内置“用户组”的名字，就是：`system:serviceaccounts:<Namespace名字>`。

## 总结

- 所谓角色（Role），其实就是一组权限规则列表。而我们分配这些权限的方式，就是通过创建RoleBinding对象，将被作用者（subject）和权限列表进行绑定。

- 尽管权限的被作用者可以有很多种（比如，User、Group等），但在我们平常的使用中，最普遍的用法还是ServiceAccount。
- Role + RoleBinding + ServiceAccount的权限分配方式是经常使用的组合。