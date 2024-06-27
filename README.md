# Собственный Kubernetes оператор за час

Практические материалы к вебинару [Собственный Kubernetes оператор за час](https://www.youtube.com/watch?v=tFzM-2pwL8A).
> В данном репозитории лежат файлы уже созданного оператора, а в этом README
> описаны инструкции для повторения пути по созданию оператора с нуля.

## Пререквизиты

- Кластер Kubernetes (можно воспользоваться [minikube](https://minikube.sigs.k8s.io/docs/start/))
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)
- [Operator SDK](https://sdk.operatorframework.io/docs/installation/)

## Подготовка проекта

- Создаем директорию для проекта

```bash
mkdir k8s-operator-in-an-hour
```

- Переходим в созданную директорию

```bash
cd k8s-operator-in-an-hour
```

- Инициализируем проект

```bash
operator-sdk init --domain mcs.mail.ru --plugins ansible
```

- Создаем апи нашего оператора

```bash
operator-sdk create api \
    --group ops \
    --version v1alpha1 \
    --kind Project \
    --generate-role
```

- Вставляем в файл `config/samples/ops_v1alpha1_project.yaml` описание будущего объекта типа Project

```yaml
apiVersion: ops.mcs.mail.ru/v1alpha1
kind: Project
metadata:
  name: test
spec:
  members:
    - p.petrov
    - i.ivanov
  environments:
    - name: prod
      resources:
        requests:
          cpu: 4
          memory: 4Gi
        limits:
          cpu: 4
          memory: 4Gi
```

- В файле `config/crd/bases/ops.mcs.mail.ru_projects.yaml` изменяем

```diff
     plural: projects
     singular: project
-  scope: Namespaced
+  scope: Cluster
   versions:
   - name: v1alpha1
```

- Добавлеям таски в Ansible роль. В файл `roles/project/tasks/main.yml` вставляем

```yaml
---
- name: Create a namespace
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: v1
      kind: Namespace
      metadata:
        name: "{{ ansible_operator_meta.name }}-{{ item.name }}"
        labels:
          app.kubernetes.io/managed-by: "projects-operator"
  loop: "{{ environments }}"

- name: Create a resource quota
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: v1
      kind: ResourceQuota
      metadata:
        namespace: "{{ ansible_operator_meta.name }}-{{ item.name }}"
        name: resource-quota
        labels:
          app.kubernetes.io/managed-by: "projects-operator"
      spec:
        hard:
          limits.cpu: "{{item.resources.limits.cpu}}"
          limits.memory: "{{item.resources.limits.memory}}"
          requests.cpu: "{{item.resources.requests.cpu}}"
          requests.memory: "{{item.resources.requests.memory}}"
  loop: "{{ environments }}"

- name: Create a member role building
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: rbac.authorization.k8s.io/v1
      kind: RoleBinding
      metadata:
        name: "{{ item[1] }}"
        namespace: "{{ ansible_operator_meta.name }}-{{ item[0].name }}"
        labels:
          app.kubernetes.io/managed-by: "projects-operator"
      roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: edit
      subjects:
        - kind: ServiceAccount
          name: "{{ item[1] }}"
          namespace: users
  with_nested:
    - "{{ environments }}"
    - "{{ members }}"
```

- Корректируем RBAC права для оператора. Файл `config/rbac/role.yaml` приводим к такому виду

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: manager-role
rules:
  ##
  ## Base operator rules
  ##
  ##
  - apiGroups:
      - ""
    resources:
      - namespaces
      - resourcequotas
    verbs:
      - create
      - delete
      - get
      - list
      - patch
      - update
      - watch
  - apiGroups:
      - rbac.authorization.k8s.io
    resources:
      - rolebindings
    verbs:
      - create
      - delete
      - get
      - list
      - patch
      - update
      - watch
  - apiGroups: 
      - rbac.authorization.k8s.io
    resources:
      - clusterroles
    verbs:
      - bind
    resourceNames:
      - edit
  ## Rules for ops.mcs.mail.ru/v1alpha1, Kind: Project
  ##
  - apiGroups:
      - ops.mcs.mail.ru
    resources:
      - projects
      - projects/status
      - projects/finalizers
    verbs:
      - create
      - delete
      - get
      - list
      - patch
      - update
      - watch
#+kubebuilder:scaffold:rules
```

- В файл `watches.yaml` дописываем ключ для включения слежения за cluster wide объектами

```yaml
  watchClusterScopedResources: true
```

## Сборка проекта и деплой в кластер

- Собираем образ с оператором
> Не забываем поменять `<your dockerhub username>` на ваше реальное имя пользователя на Docker Hub или адрес вашего внутреннего реджистри

```bash
make docker-build IMG=<your dockerhub username>/k8s-operator-in-an-hour:v0.0.1
```

- Пушим образ в реджистри

```bash
make docker-push IMG=<your dockerhub username>/k8s-operator-in-an-hour:v0.0.1
```

- Деплоим все в кластер

```bash
make deploy IMG=<your dockerhub username>/k8s-operator-in-an-hour:v0.0.1
```

## Создание тестового Project в кластере и проверка работы оператора

- Создаем в кластере объект типа Project из подготовленного ранее файла `config/samples/ops_v1alpha1_project.yaml`

```bash
kubectl apply -f config/samples/ops_v1alpha1_project.yaml
```

- Смотрим что появился нэймспейс `test-prod`, ресурс квоты в нем оостветствуют тому, что мы задавали при созданиие Project и имеются в наличии рольбиндинги для пользователей

```bash
$ kubectl get ns

NAME                 STATUS   AGE
default              Active   12d
kube-node-lease      Active   12d
kube-public          Active   12d
kube-system          Active   12d
test-prod            Active   10m
```

```bash
kubectl get rolebinding -n test-prod
kubectl get resourcequota -n test-prod
```

## Очистка окружения

Для удаления оператора, всех созданных им объектов и crd из кластера выполните команду

```bash
make undeploy IMG=<your dockerhub username>/k8s-operator-in-an-hour:v0.0.1
```
