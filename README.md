# Домашнее задание к занятию "12.2 Команды для работы с Kubernetes"
Кластер — это сложная система, с которой крайне редко работает один человек. Квалифицированный devops умеет наладить работу всей команды, занимающейся каким-либо сервисом.
После знакомства с кластером вас попросили выдать доступ нескольким разработчикам. Помимо этого требуется служебный аккаунт для просмотра логов.

## Задание 1: Запуск пода из образа в деплойменте  

<details>

  <summary>Описание задачи</summary>  

Для начала следует разобраться с прямым запуском приложений из консоли. Такой подход поможет быстро развернуть инструменты отладки в кластере. Требуется запустить деплоймент на основе образа из hello world уже через deployment. Сразу стоит запустить 2 копии приложения (replicas=2). 

Требования:
 * пример из hello world запущен в качестве deployment
 * количество реплик в deployment установлено в 2
 * наличие deployment можно проверить командой kubectl get deployment
 * наличие подов можно проверить командой kubectl get pods
</details>


### Решение

-  Создаем deployment
```bash
 kubectl create deployment hello-world --image=gcr.io/google-samples/hello-app:1.0 --replicas=2
 deployment.apps/hello-world created
 ```
  
- Смотрим запущенные deployment

```bash
kubectl get deployments -o wide
NAME          READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES                                SELECTOR
hello-world   2/2     2            2           105s   hello-app    gcr.io/google-samples/hello-app:1.0   app=hello-world
```  

- Смотрим запущенные pods

```bash
kubectl get pods -o wide
NAME                          READY   STATUS    RESTARTS   AGE    IP            NODE    NOMINATED NODE   READINESS GATES
hello-world-57fbf88c7-nvtjt   1/1     Running   0          116s   10.233.94.7   node0   <none>           <none>
hello-world-57fbf88c7-sntxf   1/1     Running   0          116s   10.233.94.8   node0   <none>           <none>
```
---

## Задание 2: Просмотр логов для разработки

<details>

  <summary>Описание задачи</summary>  

Разработчикам крайне важно получать обратную связь от штатно работающего приложения и, еще важнее, об ошибках в его работе. 
Требуется создать пользователя и выдать ему доступ на чтение конфигурации и логов подов в app-namespace.

Требования: 
 * создан новый токен доступа для пользователя
 * пользователь прописан в локальный конфиг (~/.kube/config, блок users)
 * пользователь может просматривать логи подов и их конфигурацию (kubectl logs pod <pod_id>, kubectl describe pod <pod_id>)
</details>  

### Решение

Необходимо сгенерировать закрытый ключ и подписать сертификат в kubernetes. Назначить пользователю роль.
Для удобства выполнял и тестировал на master ноде, поэтому подключение к кластеру по localhost. 

- Создаем закрытый ключ:
```bash
openssl genrsa -out read_user.key 2048
```

- Создаем запрос на подпись сертификата:  
```bash
openssl req -new -key read_user.key \
-out read_user.csr \
-subj "/CN=read_user"
```

- Подписываем сертфикат в kubernetes, сроком действия на 30 дней
```bash
openssl x509 -req -in read_user.csr \
-CA /etc/kubernetes/pki/ca.crt \
-CAkey /etc/kubernetes/pki/ca.key \
-CAcreateserial \
-out read_user.crt -days 30

Signature ok
subject=CN = read_user
Getting CA Private Key
```

- Результат в директории:
```bash
ls
read_user.crt  read_user.csr  read_user.key
```

- Создаем пользователя в kubernetes
```
kubectl config set-credentials read_user \
--client-certificate=/home/read_user/read_user.crt \
--client-key=/home/read_user/read_user.key

User "read_user" set.
```

- создаем config для подключения к кластеру (за основу можно взять /etc/kubernetes/admin.conf)
```bash
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <<<сертификат сервера>>>
    server: https://localhost:6443
  name: cluster.local
contexts:
- context:
    cluster: cluster.local
    user: read_user
  name: read_user-context
current-context: read_user-context
kind: Config
preferences: {}
users:
- name: read_user
  user:
    client-certificate: /home/read_user/read_user.crt
    client-key: /home/read_user/read_user.key
```

- Проверяем подключение с использованием созданного конфига
```
 kubectl get pods
Error from server (Forbidden): pods is forbidden: User "read_user" cannot list resource "pods" in API group "" in the namespace "default"

 kubectl get pods -n app-namespace
Error from server (Forbidden): pods is forbidden: User "read_user" cannot list resource "pods" in API group "" in the namespace "app-namespace"
```

- Нет доступа, т.к. не определена авторизация пользователя. Биндим доступный по умолчанию ClusterRole "view" на созданного пользователя "read_user". Для этого создаем yaml файл:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read_user
  namespace: app-namespace
subjects:
- kind: User
  name: read_user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

- Применяем созданный yaml (от пользователя с правами админа клатера)

```
kubectl apply -f ./RoleBinding.yaml 
rolebinding.rbac.authorization.k8s.io/read_user created
```

- Проверяем с config пользователя read_user

```bash
 kubectl get pods
Error from server (Forbidden): pods is forbidden: User "read_user" cannot list resource "pods" in API group "" in the namespace "default"
# Не удается, т.к. пользователю нет прав на namespace default

 kubectl get pods -n app-namespace
NAME                          READY   STATUS    RESTARTS   AGE
hello-world-57fbf88c7-b2jtt   1/1     Running   0          35m
hello-world-57fbf88c7-rr2wp   1/1     Running   0          35m
# С указанием namespace app-namespace успешно (за кадром создал деплоймент в этот неймспейс)

 kubectl create deployment hello-world --image=gcr.io/google-samples/hello-app:1.0 --replicas=2
error: failed to create deployment: deployments.apps is forbidden: User "read_user" cannot create resource "deployments" in API group "apps" in the namespace "default"
# Не удается создать деплоймент в неймспейсе default

 kubectl create deployment hello-world -n app-namespace --image=gcr.io/google-samples/hello-app:1.0 --replicas=2
error: failed to create deployment: deployments.apps is forbidden: User "read_user" cannot create resource "deployments" in API group "apps" in the namespace "app-namespace"
# Не удается создать деплоймент в неймспейсе app-namespace

 kubectl delete deployment -n app-namespace hello-world
Error from server (Forbidden): deployments.apps "hello-world" is forbidden: User "read_user" cannot delete resource "deployments" in API group "apps" in the namespace "app-namespace"
# Удалить также не получается
```

- Итого, пользователю read_user предоставлены права read на неймспейс app-namespace

---

## Задание 3: Изменение количества реплик 

<details>

  <summary>Описание задачи</summary>  

Поработав с приложением, вы получили запрос на увеличение количества реплик приложения для нагрузки. Необходимо изменить запущенный deployment, увеличив количество реплик до 5. Посмотрите статус запущенных подов после увеличения реплик. 

Требования:
 * в deployment из задания 1 изменено количество реплик на 5
 * проверить что все поды перешли в статус running (kubectl get pods)

</details>

### Решение  
  
- Скалируем количество реплик в deployment hello-world до 5:
```
kubectl scale --replicas=5 deployment hello-world
deployment.apps/hello-world
```

- Проверяем:
```
kubectl get pods -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
hello-world-57fbf88c7-48kz4   1/1     Running   0          51s   10.233.94.9    node0   <none>           <none>
hello-world-57fbf88c7-5shqn   1/1     Running   0          19s   10.233.94.10   node0   <none>           <none>
hello-world-57fbf88c7-8ndnz   1/1     Running   0          19s   10.233.94.11   node0   <none>           <none>
hello-world-57fbf88c7-nvtjt   1/1     Running   0          14m   10.233.94.7    node0   <none>           <none>
hello-world-57fbf88c7-sntxf   1/1     Running   0          14m   10.233.94.8    node0   <none>           <none>
```
---
