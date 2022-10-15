# Урок 7: Устройство кластера.

[Урок 7: Устройство кластера. Вечерняя школа Kubernetes](https://youtu.be/0rN_tUIZx7Y)

# [Agenda лекции](https://youtu.be/0rN_tUIZx7Y?t=87)

- Из чего состоит кластер
- Что происходит в кластере, когда запускаются все абстракции
    - Есть абстракции и есть некоторый физический уровень, на котором всё происходит: запускается докер, контейнеры и прочая.
    - Чего можно ожидать от k8s и чего ожидать не стоит
    - Если что-то пошло не так, то по каким причинам это могло произойти

# [План](https://youtu.be/0rN_tUIZx7Y?t=146)

Дефолтные стандартные компоненты кластера, без которых кластер не может полноценно работать. И фактически нет выбора других компонентов - чтобы запустить кластер k8s нужно использовать конкретно эти компоненты:

- Etcd
- API server
- Controller-manager
- Scheduler
- Kubelet
- Kube-proxy

Также есть компоненты, без которых кластер работать не будет, но в лекции они не рассматриваются. K8s в целом может запускаться. Вдобавок эти компоненты не являются регламентировано жесткими. 

- Контейнеризация: [Docker](https://youtu.be/TJg7QpqCH70), [Docker Compose и Co](https://youtu.be/Hz7fkXQABNo)
Для контейниризации может использоваться как [Docker](https://www.docker.com/) (который, несмотря на депрекейшн, все еще остается дефолтной опцией в k8s), так и [Podman](https://podman.io/), [Containerd](https://containerd.io/), [Cri-O](https://cri-o.io/) и прочая.
- Сеть: [Сети Кубернетес изнутри](https://youtu.be/6QKAt4TjxF8), [Сеть k8s - сетап кластера](https://youtu.be/JNUD9j9QAnA)
Можно использовать: [Calico](https://docs.projectcalico.org/about/about-calico), [Weave Net](https://www.weave.works/oss/net/), [Cillium](https://cilium.io/), [Flannel](https://github.com/flannel-io/flannel)
[Дока k8s](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [DNS](https://youtu.be/MLp8_qVLkzc)
Сейчас дефолтным является [Core dns](https://coredns.io/). В более старых версия использовалась связка [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) и go'шного контроллера к нему. В принципе каким должен быть сам dns в кластере k8s не регламентировано, главное, чтобы он был совместим с самим кластером.

В лекции будет рассмотрен каждый компонент и будет разобран на практических примерах: что каждый компонент будет делать с запускаемым ReplicaSet в кластере.

# Мастер компонеты

## [Etcd](https://youtu.be/0rN_tUIZx7Y?t=362)

![001_etcd.png](./assets/lesson_7/001_etcd.png)

Хранит всю информацию о кластере. Это БД кластера k8s.

Характеристики etcd:

1. key-value
2. изначально кластерная и хорошо работает в кластере
3. использует [raft](https://raft.github.io/). Обычно кластер etcd представляет собой несколько нод, которых нечетное количество (обычно 3 или 5). Raft обеспечивает кворум (quorum), что в свою очередь обеспечивает надежность хранения данных - чтобы данные считались записанными в кластере:
    1. они ВСЕГДА должны быть записаны большинством реплик == нод кластера
    2. ВСЕГДА у реплик должен быть мастер == "главная" нода
    3. если мастер куда-то пропал, оставшиеся реплики большинством выбирают нового мастера
    
    Более подробно по ссылке выше.
    

[8:03](https://youtu.be/0rN_tUIZx7Y?t=483) Обычно в отказоустойчивом кластере k8s существует 3 разных ноды с etcd. Это то, что мы называем мастернодами etcd (не путать с мастером для достижения кворума). На эти же ноды потом бцдцт ставиться другие компненты, которые занимаются именно управлением кластеров.

В общем и целом можно работать с одной нодой. Dev установки обычно работают с одной.

<aside>
🔥 Самое главное, что нужно знать про etcd: Это БД кластера и вся информация хранится в etcd, извлекается из etcd и больше ее нигде нет

</aside>

[Raft](http://thesecretlivesofdata.com/raft/)

С consul'ом k8s не работает, только с etcd. Поддержки других хранилищь у k8s нет, ему это, впрочем, и не нужно.

## [API server](https://youtu.be/0rN_tUIZx7Y?t=572)

- Центральный компонент k8s
- Единственный, кто общается с etcd
- Работает по REST API - мы с ним взаимодействуем с помощью команды `kubectl` - на самом деле под капотом шлются нормальные http запросы к API Server.
Если хочется посмотреть, как это работает, можно запустить команду `kubectl apply -v=10 -f <имя-файла>` с ключом `-v=10` - это полный=verbose mode для команды `kubectl` и посмотреть полный вывод того, что происходит, как происходит обращение в кластер и так далее.
Пример вывода
    
    ```bash
    ...
    I1107 02:51:23.247699   13959 round_trippers.go:443] GET https://5.188.142.58:6443/api/v1/namespaces/<namespace-name> 200 OK in 63 milliseconds
    I1107 02:51:23.247783   13959 round_trippers.go:449] Response Headers:
    I1107 02:51:23.247836   13959 round_trippers.go:452]     Cache-Control: no-cache, private
    I1107 02:51:23.247866   13959 round_trippers.go:452]     Content-Type: application/json
    I1107 02:51:23.247892   13959 round_trippers.go:452]     X-Kubernetes-Pf-Flowschema-Uid: 211233d5-bef2-4121-9c8f-40d25ccae82e
    I1107 02:51:23.247922   13959 round_trippers.go:452]     X-Kubernetes-Pf-Prioritylevel-Uid: 07b62685-8e2d-4ab3-bbc1-4745e33a5d11
    I1107 02:51:23.247951   13959 round_trippers.go:452]     Content-Length: 469
    I1107 02:51:23.247972   13959 round_trippers.go:452]     Date: Sat, 06 Nov 2021 23:51:23 GMT
    I1107 02:51:23.248024   13959 request.go:968] Response Body: {"kind":"Namespace","apiVersion":"v1","metadata":{"name":"<namespace-name>","selfLink":"/api/v1/namespaces/<namespace-name>","uid":"ad1ecf95-f7c5-467b-b6e1-76676ea62a5a","resourceVersion":"2270162","creationTimestamp":"2021-10-01T00:59:35Z","managedFields":[{"manager":"kubectl-create","operation":"Update","apiVersion":"v1","time":"2021-10-01T00:59:35Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{"f:phase":{}}}}]},"spec":{"finalizers":["kubernetes"]},"status":{"phase":"Active"}}
    I1107 02:51:23.248522   13959 request.go:968] Request Body: {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"apps/v1\",\"kind\":\"Deployment\",\"metadata\":{\"annotations\":{},\"name\":\"my-deployment\",\"namespace\":\"<namespace-name>\"},\"spec\":{\"replicas\":2,\"selector\":{\"matchLabels\":{\"app\":\"my-app\"}},\"strategy\":{\"rollingUpdate\":{\"maxSurge\":1,\"maxUnavailable\":1},\"type\":\"RollingUpdate\"},\"template\":{\"metadata\":{\"labels\":{\"app\":\"my-app\"}},\"spec\":{\"containers\":[{\"image\":\"quay.io/testing-farm/nginx:1.12\",\"livenessProbe\":{\"failureThreshold\":3,\"httpGet\":{\"path\":\"/\",\"port\":80},\"initialDelaySeconds\":10,\"periodSeconds\":10,\"successThreshold\":1,\"timeoutSeconds\":1},\"name\":\"nginx\",\"ports\":[{\"containerPort\":80}],\"readinessProbe\":{\"failureThreshold\":3,\"httpGet\":{\"path\":\"/\",\"port\":80},\"periodSeconds\":10,\"successThreshold\":1,\"timeoutSeconds\":1},\"resources\":{\"limits\":{\"cpu\":\"100m\",\"memory\":\"100Mi\"},\"requests\":{\"cpu\":\"10m\",\"memory\":\"100Mi\"}},\"startupProbe\":{\"failureThreshold\":30,\"httpGet\":{\"path\":\"/\",\"port\":80},\"periodSeconds\":10}}]}}}}\n"},"name":"my-deployment","namespace":"<namespace-name>"},"spec":{"replicas":2,"selector":{"matchLabels":{"app":"my-app"}},"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":1},"type":"RollingUpdate"},"template":{"metadata":{"labels":{"app":"my-app"}},"spec":{"containers":[{"image":"quay.io/testing-farm/nginx:1.12","livenessProbe":{"failureThreshold":3,"httpGet":{"path":"/","port":80},"initialDelaySeconds":10,"periodSeconds":10,"successThreshold":1,"timeoutSeconds":1},"name":"nginx","ports":[{"containerPort":80}],"readinessProbe":{"failureThreshold":3,"httpGet":{"path":"/","port":80},"periodSeconds":10,"successThreshold":1,"timeoutSeconds":1},"resources":{"limits":{"cpu":"100m","memory":"100Mi"},"requests":{"cpu":"10m","memory":"100Mi"}},"startupProbe":{"failureThreshold":30,"httpGet":{"path":"/","port":80},"periodSeconds":10}}]}}}}
    I1107 02:51:23.248817   13959 round_trippers.go:423] curl -k -v -XPOST  -H "User-Agent: kubectl/v1.16.13 (linux/amd64) kubernetes/fd22db4" -H "Accept: application/json" -H "Content-Type: application/json" -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IjdqS2pSd2RFYzUtMVNlUmVTbm11aHFkTDRqSkw0TjVSbWx0MDBVWnpqMFEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJzMDAwOTk4Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InMwMDA5OTgtdG9rZW4tZGxiZGYiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiczAwMDk5OCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImNmNGExYWYxLTkxZTAtNDliNy04YTk4LTE1ZDc0Zjc5YmNkZSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpzMDAwOTk4OnMwMDA5OTgifQ.VT-zl-kRQPbn8V5X_JOH2kZ4u4ElZRbkxOjJoy5pYKnrt51bFo49gzdx9u-7CW75wmxF2Wu5Jf10OXFvTmiO8nSlOyZpe7Rv0__53udW1hLqew9vZ226L-Qv3rV99S8iz1d42Zldk1UJf_Hbn7t_GkACKRde8NQpCwhHuSnUKPp9yqSVtd4RI_BJWoUZs8l0xn6g2SM59m8mxON3sqf15hDCvz87TYiSZ8JYqybPmUgVZM2m80ILSaRX-Pv2vmwotf3PtHuLAnmpheqiTj7T3GI84ISjQ4BhWXYxyIGPXnWnY7u66jiYCzEveJFjFNnAWZsohhdMDthkdtnVc0-rhw" 'https://5.188.142.58:6443/apis/apps/v1/namespaces/<namespace-name>/deployments'
    I1107 02:51:23.337906   13959 round_trippers.go:443] POST https://5.188.142.58:6443/apis/apps/v1/namespaces/<namespace-name>/deployments 201 Created in 89 milliseconds
    I1107 02:51:23.337955   13959 round_trippers.go:449] Response Headers:
    I1107 02:51:23.337991   13959 round_trippers.go:452]     Cache-Control: no-cache, private
    I1107 02:51:23.338032   13959 round_trippers.go:452]     Content-Type: application/json
    I1107 02:51:23.338058   13959 round_trippers.go:452]     X-Kubernetes-Pf-Flowschema-Uid: 211233d5-bef2-4121-9c8f-40d25ccae82e
    I1107 02:51:23.338083   13959 round_trippers.go:452]     X-Kubernetes-Pf-Prioritylevel-Uid: 07b62685-8e2d-4ab3-bbc1-4745e33a5d11
    I1107 02:51:23.338112   13959 round_trippers.go:452]     Date: Sat, 06 Nov 2021 23:51:23 GMT
    I1107 02:51:23.338334   13959 request.go:968] Response Body: {"kind":"Deployment","apiVersion":"apps/v1","metadata":{"name":"my-deployment","namespace":"<namespace-name>","selfLink":"/apis/apps/v1/namespaces/<namespace-name>/deployments/my-deployment","uid":"d7930050-a35e-46b1-8606-2897ca984617","resourceVersion":"92966724","generation":1,"creationTimestamp":"2021-11-06T23:51:23Z","annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"apps/v1\",\"kind\":\"Deployment\",\"metadata\":{\"annotations\":{},\"name\":\"my-deployment\",\"namespace\":\"<namespace-name>\"},\"spec\":{\"replicas\":2,\"selector\":{\"matchLabels\":{\"app\":\"my-app\"}},\"strategy\":{\"rollingUpdate\":{\"maxSurge\":1,\"maxUnavailable\":1},\"type\":\"RollingUpdate\"},\"template\":{\"metadata\":{\"labels\":{\"app\":\"my-app\"}},\"spec\":{\"containers\":[{\"image\":\"quay.io/testing-farm/nginx:1.12\",\"livenessProbe\":{\"failureThreshold\":3,\"httpGet\":{\"path\":\"/\",\"port\":80},\"initialDelaySeconds\":10,\"periodSeconds\":10,\"successThreshold\":1,\"timeoutSeconds\":1},\"name\":\"nginx\",\"ports\":[{\"containerPort\":80}],\"readinessProbe\":{\"failureThreshold\":3,\"httpGet\":{\"path\":\"/\",\"port\":80},\"periodSeconds\":10,\"successThreshold\":1,\"timeoutSeconds\":1},\"resources\":{\"limits\":{\"cpu\":\"100m\",\"memory\":\"100Mi\"},\"requests\":{\"cpu\":\"10m\",\"memory\":\"100Mi\"}},\"startupProbe\":{\"failureThreshold\":30,\"httpGet\":{\"path\":\"/\",\"port\":80},\"periodSeconds\":10}}]}}}}\n"},"managedFields":[{"manager":"kubectl","operation":"Update","apiVersion":"apps/v1","time":"2021-11-06T23:51:23Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}}},"f:spec":{"f:progressDeadlineSeconds":{},"f:replicas":{},"f:revisionHistoryLimit":{},"f:selector":{},"f:strategy":{"f:rollingUpdate":{".":{},"f:maxSurge":{},"f:maxUnavailable":{}},"f:type":{}},"f:template":{"f:metadata":{"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"nginx\"}":{".":{},"f:image":{},"f:imagePullPolicy":{},"f:livenessProbe":{".":{},"f:failureThreshold":{},"f:httpGet":{".":{},"f:path":{},"f:port":{},"f:scheme":{}},"f:initialDelaySeconds":{},"f:periodSeconds":{},"f:successThreshold":{},"f:timeoutSeconds":{}},"f:name":{},"f:ports":{".":{},"k:{\"containerPort\":80,\"protocol\":\"TCP\"}":{".":{},"f:containerPort":{},"f:protocol":{}}},"f:readinessProbe":{".":{},"f:failureThreshold":{},"f:httpGet":{".":{},"f:path":{},"f:port":{},"f:scheme":{}},"f:periodSeconds":{},"f:successThreshold":{},"f:timeoutSeconds":{}},"f:resources":{".":{},"f:limits":{".":{},"f:cpu":{},"f:memory":{}},"f:requests":{".":{},"f:cpu":{},"f:memory":{}}},"f:startupProbe":{".":{},"f:failureThreshold":{},"f:httpGet":{".":{},"f:path":{},"f:port":{},"f:scheme":{}},"f:periodSeconds":{},"f:successThreshold":{},"f:timeoutSeconds":{}},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}}}]},"spec":{"replicas":2,"selector":{"matchLabels":{"app":"my-app"}},"template":{"metadata":{"creationTimestamp":null,"labels":{"app":"my-app"}},"spec":{"containers":[{"name":"nginx","image":"quay.io/testing-farm/nginx:1.12","ports":[{"containerPort":80,"protocol":"TCP"}],"resources":{"limits":{"cpu":"100m","memory":"100Mi"},"requests":{"cpu":"10m","memory":"100Mi"}},"livenessProbe":{"httpGet":{"path":"/","port":80,"scheme":"HTTP"},"initialDelaySeconds":10,"timeoutSeconds":1,"periodSeconds":10,"successThreshold":1,"failureThreshold":3},"readinessProbe":{"httpGet":{"path":"/","port":80,"scheme":"HTTP"},"timeoutSeconds":1,"periodSeconds":10,"successThreshold":1,"failureThreshold":3},"startupProbe":{"httpGet":{"path":"/","port":80,"scheme":"HTTP"},"timeoutSeconds":1,"periodSeconds":10,"successThreshold":1,"failureThreshold":30},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"IfNotPresent"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","securityContext":{},"schedulerName":"default-scheduler"}},"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxUnavailable":1,"maxSurge":1}},"revisionHistoryLimit":10,"progressDeadlineSeconds":600},"status":{}}
    ```
    
    Более того, т.к. это REST, никто не мешает использовать `curl`, писать собственные скрипты или использовать готовые библиотеки для общения с API Servre k8s и делать с ним все, что захочется
    
- Authentication and authorization. Вся авторизация и аутентификация происходит через API Server.
Если, например, у пользователя есть неймспейс, то его доступ к этому неймспейсу разрешается через API Server. Если этот пользователь захочет подключиться к чужому неймспейсу, то получить ошибку "forbidden ... нет прав на взаимодействие ..." - собственно говоря, API Server тот, кто разрешает/запрещает доступ к ресурсам. В дальнейших лекциях будут изучаться политики доступа и механизм [RBAC (Role-Based Access Control)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

API Server не только центральный компонент для пользователя (потому что мы с ним взаимодействуем), это центральный компонент для кластера: все компоненты так или иначе берут информацию из API Server'а, складывают информацию в API Server - обновляя свой статус, обновляя статус контейнера, который запущен в кластере и так далее.

Тут важно понимать, что API Server - это единственный компонет, который общается с etcd. Т.е. API Server сам по себе информацию не хранит - он ее передает в etcd, он ее из etcd читает, т.е. фактически является stateless компонентом, как и на самом деле все остальные компненты в кластере k8s, кроме непосредственно самого etcd.

API Server единственный, кто общается с etcd.

etcd единственный, кто хранит информацию о кластере.

**По поводу количества API Server:**
он не хранит состояния, у него нет понятия мастер-ведомый. Обычно для отказоустойчивых кластеров запускают 3 ноды с мастером, т.е. просто 3 сервера на виртуалке/физических серверах. На каждую ноду устанавливают по etcd и по API Server'е. При этом API Server'а смотрят как бы в кластер etcd, общаются с etcd. 

Тут есть разные подходы:

- либо каждый API Server смотри в свой etcd
- либо API Server'а смотрят во все инстансы etcd и общаются с ним как с кластером

на самом деле это особой роли не играет.

Но API Server'а работают в параллели для отказоустойчивости. Помним, что никто не запрещает запуск только одного API Server'а - просто если он выйдет из строя, кластером управлять будет нельзя.

### Работа API Server на практике

![002_api_server.png](./assets/lesson_7/002_api_server.png)

Для упрощения примера используется создание ReplicaSet, а не Deployement, как это практически в 100% случаях бывает в жизни. Позже будет показано, где и что поменяется в случае Deployment.

Помним, что в описании ReplicaSet есть template, по которому в дальнейшем будут создаваться Pod'ы. В template есть указание, сколько реплик из шаблона Pod'а нужно запустить.

Как это работает

1. Пользователь берет yaml с описанием ReplicaSet, например, replicaset.yaml
2. С помощью команды `kubectl create/apply -f replicaset.yaml` запускается процесс создания ReplicaSet и всех нижележащих компонентов
3. kubectl подключается к API Server (помним, что можно попросить kubectl вывести информацию о взаимодействии с API Server с помощью ключа `-v=10`, подробнее можно прочитать в справке `kubectl options`)
4. передает туда информацию о том, что создается ReplicaSet
5. передается описание из yaml, которое в процессе конвертиться в json
6. API Server в ответ на это складывает информацию о том, что в кластере появился ReplicaSet, который хочет создать пользователь в etcd
7. etcd по достижению кворума отвечает API Server'y, что информация сохранена
8. После чего API Server возвращает ответ kubectl (а тот в свою очередь выводить сообщение пользователю), что ReplicaSet в кластере создался
    
    <aside>
    🔥 При этом до реального запуска приложения в кластере k8s еще довольно далеко.
    На этом моменте никаких приложений в кластере k8s еще нет. Все, что там есть - это запись в etcd о том, что какой-то пользователь создал в каком-то неймспейсе ReplicaSet с таким-то описанием в таком-то формате.
    
    </aside>
    
9. На слайде выше указана группа Others - это, собственно говоря некоторые другие компоненты кластера k8s, которые смотрят в API Server, видят информацию о том, что там происходит и пытаются как-то дальше с этой информацией работать. Про тех, кто входит в группу Others ниже.

## [Controller-manager](https://youtu.be/0rN_tUIZx7Y?t=1003)

Фактически представляет из себя один бинарник, ровно так же как и все остальные компоненты k8s: API Server, etcd и другие.

Есть практики, когда его запускают вне кластера, вне docker'а, вне контенера, просто, как, например, systemd unit. 
Но обычно это такой же контейнер, такой же компонент кластера как и всё остальное. И etcd запускают как часть кластера, т.е. как Pod в кластере, и API менеджер, и остальные.

Тем не менее внутри контейнера для Controller-manager находится один бинарь, который в свою очередь содержит n'ое количество контроллеров.

**Что такое контроллер?**

Контроллер для кластера k8s - это фактически то, что реализует сами абстракции k8s, как-то с ними взаимодействует, создает Pod'ы из ReplicaSet'ов. Создает CronJob'ы, Job'ы, из них что-то генерит. Занимается GarbageCollector. И так далее.

- Набор контроллеров
    - [Node controller](https://youtu.be/0rN_tUIZx7Y?t=1105). Отслеживает информацию о том, что ноды кластера постят о себе информацию. Т.е. он не ходит непосредственно на каждую ноду опрашавиать о ее состоянии, он просто смотри в API Server и видит, что какая-то нода перестала постить свой статус.
        - Все ноды кластера k8s раз в какое-то время пишут информацию о себе в API Server, говоря "я все еще живой, я на месте, я тут".
        - Соответственно, если все ноды, зарегистрированные в кластере пишут о себе эту информацию, значит с кластером все хорошо.
        - Если какая-то нода перестанет о себе писать информацию на какой-то срок (который задан в настройках), то node controller
            - заметит эту ситуцию,
            - пометит в кластере k8s, т.е. фактически передаст в API Server информацию о том, что эта нода недоступна,
            - а далее запустит процесс переноса Pod'ов, которые есть на этой ноде, на другие ноды кластеров
    - [Replicaset controller](https://youtu.be/0rN_tUIZx7Y?t=1191). Когда в кластере создается ReplicaSet, у ReplicaSet есть темплейт Pod'ов, созданием Pod'ов по этому описанию как раз и занимается Replicaset controller.
        - Т.е. контроллер смотрит в API Server и,
            - когда видит, что там появляется описание ReplicaSet с темплейтов для Pod'ов, плюс есть информация о количестве необходимых реплик,
            - то создает в кластере нужные Pod'ы
            - и записывает информацию о них в API Server
    - [Endpoints controller](https://youtu.be/0rN_tUIZx7Y?t=1268). В k8s есть такая сущность как Endopoint'ы, которые автоматически создаются для каждого сервиса.
    - И другие ...
    
    <aside>
    🧐 Когда в кластере k8s что-то из чего-то автоматически создается, этим занимается какой-то свой контроллер внутри Controller-manager.
    В кластере k8s очень много разных объектов и фактически за каждый из них отвечает свой собственный контроллер - для того, чтобы этот объект создавался из того, что описал пользователь.
    
    </aside>
    
- [GarbageCollector](https://youtu.be/0rN_tUIZx7Y?t=1308) - "сборщик мусора". 
Возьмем, к примеру, работу Deployment controller
    - при изменении описания ReplicaSet будут созданы новые объекты ReplicaSet
    - при этом старые ReplicaSet удалены не будут,
    - за количество "старых" ReplicaSet, который будут хранится, отвечает параметр [revisionHistoryLimit](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#revision-history-limit) (по умолчанию = 10).
    - Как только у Deployment'а количество сохраненных ReplicaSet превышает этот параметр, задача Controller-manager найти самый старый ReplicaSet и удалить его из кластера, т.е. он подчитащает тот мусор, который больше уже не нужен.

**[Количество Controller Manager в кластере](https://youtu.be/0rN_tUIZx7Y?t=1629):**

Controller Manager, как и любого другого компонента k8s, в кластере может быть один и более.

Т.к. это мастеровой компонент, т.е. тот компонент, который управляет жизнедеятельностью кластера, точно так же как в той или иной степени etcd и API Server, они обычно стоят на отдельных нодах, т.е. там же где API Server и etcd, но не там же, где запускается наше приложение в кластере.

Для dev в маленьком простом кластере это может быть один Controller Manager, но обычно запускают так же 3 - по одному на каждый из master node.

У Controller Manager есть как раз таки понятие мастер-ведомый и одновременно работу в кластере может выполнять только один Controller Manager.

Для того, чтобы выбирать себе мастера и понимать, кто из них сейчас должен выполнять работу, они используют механизм [Lease](https://medium.com/michaelbi-22303/deep-dive-into-kubernetes-simple-leader-election-3712a8be3a99):

**[Вкратце о Lease](https://youtu.be/0rN_tUIZx7Y?t=1680):**

Когда наши 3 Controller Manager'а стартуют в кластере, каждый из них пытается в API Server записать информацию о том, что он мастер и будет выполнять работу. 

Первый, у кого получилось записать такую информацию и становится мастером. 

Остальные видят, что в кластере уже есть информация о том, что есть мастер controller manager и дальше просто следят за тем, что master регулярно обновляет информацию о себе, т.е. он живой и занимается работой.

Если мастер в течении какого-то времени не обновил информацию о себе, это означает, что его больше нет в кластере и оставшиесь Controller Manager ровно по тому же алгоритму будут пытаться прописать себя мастером. Первый у кого это получилось, становится новым мастером.

Если через какое-то время старый мастер оживает, то он видит, что он больше не мастер и начинает просто следить за состоянием текущего мастера.

### [Q&A](https://youtu.be/0rN_tUIZx7Y?t=1398)

1. **Q:** Откуда node manager знает о содержимом упавшей ноды для переноса, если вся эта информация хранится в etcd
**A:** Все верно, вся информация хранится в etcd. 
    1. Controller manager обращается к API Server'y. 
    Т.е. если controller manager'y нужна какая-та информация, которую он хотел бы знать о ноде или о чем-то еще, оно делает запросы к API Server'y точно так же, как может сделать пользователь с помощью `kubectl`, например, `kubectl get pod ...` На самом деле, запросы идут в апи, но суть та же. 
    2. И API Server лезет в etcd, собирает там всю нужную информацию и отдает обратно.
2. **Q:** Какой процесс на ноде отправляет инфу node-controller'y?
**A:** kubelet отправляет всю информацию в API Server (и больше никому). Все заинтересованные в этой информации, могут запросить ее у API Server'а. Т.е. node-controller спрашивает API Server о статусе нод: что там kubelet писал в последний раз, когда и т.д.

### [Работа Controller-manager на практике](https://youtu.be/0rN_tUIZx7Y?t=1512)

![003_controller_manager.png](./assets/lesson_7/003_controller_manager.png)

Сереньким обозначены этапы, которые уже были пройдены (см. выше в разделе про API Server).

1. Процесс Controller-manager всегда висит запущенный в кластере
2. Смотри в API Server - это называется "подписан на события"
Расмотрим ситуацию на примере Replication Controller.
    1. Replication Controller подписан к API Server'y на событие "Создание нового ReplicaSet"
    2. Как только такое событие происходит, Controllare Manager, а точнее Replica Controller внутри него, получает от API Server данное событие.
    3. Controller Manager берет манифест нужного ReplicaSet у API Server'а
    4. На основании поля templates генерит описание Pod'ов в формате json
    5. Отправляет это описание обратнов API Server.
    6. API Server складывает это в etcd 
    7. и возвращает ответ Contreller Manager'y, что была создана запись о Pod'е.
    
    <aside>
    ‼️ На данном этапе еще никаких Pod'ов физически в кластере k8s нет, есть только записи в etcd с описанием Pod'ов.
    
    </aside>
    

## [Scheduler](https://youtu.be/0rN_tUIZx7Y?t=1768)

Также относится к группе компонентов мастеров, которые занимаются управление кластера k8s.

Назначает Pod'ы на ноды (которые у нас были нагенерены Controller-manager, см. предыдущий раздел), учитывая:

- [QoS (Quolity of Service)](https://youtu.be/0rN_tUIZx7Y?t=1823) политика. В зависиомсти от соотношения ресурсов limit и request scheduler принимает решение, куда назначать данный Pod.
Например, если на каких-то нодах есть нехватка оперативной памяти, то понятно, что туда не будут назначаться Pod'ы. 
Если на каких-то нодах есть много свободных ресурсов, то Pod'ы best effort, у которых не указано ни лимитов, ни реквестов, скорее всего будут назначаться на такие ноды.
- [Affinity / anti-affinity](https://youtu.be/0rN_tUIZx7Y?t=1867). Фактически это механихм в кластере k8s, который позволяет сказать, что "я бы хотел, чтобы вот эти Pod'ы запускались на определенной группе нод". Например на нодах, у которых есть gpu (графические процессоры и т.д.). Или сказать, что в эта группа Pod'ов (например, Pod'ов одного деплоймента) никогда не могли запуститься на одной и тоже же ноде, т.е. чтобы кластер k8s всегда их размазывал по разным нодам.
- [Requested resources](https://youtu.be/0rN_tUIZx7Y?t=1904). K8s старается ваши нагрузки равномерно распределять по нодам, т.е. если на одной ноде занято 2 cpu, а на другой 0, то scheduler это видит, он всю эту информацию может читать из API Server'а (точно так же как и Controller Manager и все остальные в кластере) и он примет решение о том, что кажется этот Pod более приоритетный и его надо отправить на ноду, на которой нет занятых ресурсов.
Если подробнее об этом говорить, то на самом деле все эти механизмы - это механизмы фильтрации:
    - Сначала scheduler убирает из списка всех нод в кластере те, которые точно не подойдут
    - Затем включается механизм scoring'а (от score=балл) Т.е. он каждой из оставшихся ноды (которые подходят по параметрам) на основании всяческих разных политик, выставляет баллы по каждому из параметров, на которые он умеет смотреть.
    - Нода, которая получает больше всего баллов, в итоге становится той нодой, на которуюбудет назначен Pod - на которой он будет запускаться по итогу.
- Priority Class
У нас есть какие-то поды более приоритетные, какие-то менее приоритетные. 
Например, у нас есть сервис, который занимается рассылками и, если он не будет работать полчаса и рассылки придут позже, то никто не умрет.

Пример из реальной жизни лектора:

Одна из политик scoring'а, которая есть в scheduler - это скачанный образ из которого хочет запуститься ваш контейнер.

Т.е. scheduler предпочитает среди прочего ту ноду, на которой уже есть образ из которого хочет запуститься ваш контейнер - Pod.

При прочих равных, если на ноде ваше приложение уже запускалось, то scheduler выберет эту ноду.

В приведенном лектором примере, данный механизм мешался и они принудительно его отключали (делается при помощи конфигов).

<aside>
💡 Фактически что умеет делать Scheduler - это просто назначать поды, основываясь на чем-то

</aside>

### Работа Scheduler на практике

![004_scheduler.png](./assets/lesson_7/004_scheduler.png)

1. Controller manager создал в кластере описание пода 
2. и сложил в формате json в api server
3. scheduler так же как и controller manager подписан на определенные событие в k8s: события, которые его интересуют - это создание новых подов
4. как только scheduler видит в k8s api serever событие создание нового пода, которые еще не был назначен на конкретную ноду
5. scheduler берет описание этого пода
6. понимает, на какую ноду можно назначить этот под
7. и дописывает в описание пода кусок о том, что этот под д.б. запущен на такой-то ноде
8. после чего отправляет обновленное описание обратно в api server.
9. Api server складывает обновленное описание в etcd и говорит scheduler'y "всё хорошо, я записал"

<aside>
‼️ Scheduler ничего не запускает. Не созает под в k8s кластере. Не запускает контенеры и/или приложение.
Он просто немножко проапгрейдил информацию в шаблонах пода - прописал туда информацию о том, что этот под должен быть запущен на такой-то ноде. И сохранил эту информацию в api server'е.

</aside>

Касательно количества Scheduler'ов - ситуация ровно такая же, как и с Controller Manager'ом:
Может работать один, может быть кластер с одним scheduler'ом и это нормально.

Но по факту в кластерах запускаю 3 штуки - по одному на каждой мастер ноде. Точно так же как и Controller manager есть понятие мастера - в каждый конкретный момент времени работу выполняет только один scheduler. Если мастер пропадает, оставшиеся scheduler'ы конкурируют за право быть мастером. Тот, у кого получилось, становится мастером. У кого не получилось, висит и ждет, когда мастер пропадет.

### Q&A

**Q:** Если работает один мастер, а второй проснувшийся ранее бывший мастером - уже не в статусе мастера, то второй работает вхолостую?

**A:** Да. Один работает, остальные ничего не делают и ничего не жрут, просто смотрят в api server и ждут, когда отвалится мастер.

**Q:** Как реализован watch 

**A:** Это реализовано на механизмах http2 - на long polling и т.д.

<aside>
📑 **Etcd, Api Server, Controller manager и scheduler** - это именно те компоненты, которые являются именно мастер компонентами, т.е. стоят на мастер нодах и выполняют работу по менеджменту всего гластера.

</aside>

# [Worker компоненты](https://youtu.be/0rN_tUIZx7Y?t=2290)

Не управляют кластером k8s, управляют своими локальными нодами

## [Kubelet](https://youtu.be/0rN_tUIZx7Y?t=2295)

- Работает на каждой ноде
Правда на 100% - работает абсолютно на всех нодах кластера: как на worker'ах, так и на master нодах. Master-нода ровно такая же нода калстера, как и все остальные. Единсттвенное, что ее отличает от других нод кластера, что она специальным образом обозначена и всем остальным подам запрещено на этих нодах запускать. Хотя это ограничение можно снять.
- Обычно единственный компонент, работающий не в Docker.
Потому что это единственный компонент, который общается с docker'ом или другой системой контейниризации (например, kryo и т.д.). Обменивается информацией о том, какие контейнеры нужно запускать, какие нужно останавливать и т.д. 
Работает просто как процесс на хосте.
- Отдает команды Docker daemon
Т.е. по факту общается с docker daemon'ом посредством api docker, чтобы манипулировать контейнерами.
- Создает POD'ы
- Выплоняет проверки: liveness, readiness and startup probes.
Это означает, что нужно учитываеть, что kublet эти запросы выполняет с той же ноды, на которой запущено приложение (грубо говоря по localhost) и у него есть возможность да него достучаться, даже если сама нода отвалилась от кластера, и kubelet об этом не узнает. Но при этом сам кластер k8s поймет, что нода отвалилась и уберет ее из балансировки.
- Отправляет в Api Server информацию о своей ноде:
    - я жив, мы с моей нодой на месте
    - у меня запущены такие-то контейнеры, в таком-то статусе они находятся
    - сколько ресурсов на ноде осталось, сколько было запрошено, сколько было изначально
    
    Соответственно, если до Controller manager перестанет доходить вся эта информация, Controller manager пометит всю ноду как недоступную и все приложения этой ноды он также пометит как недоступные и уберет их из балансировки, выкинет из сервисов, чтобы сервисы не вели на недоступные поды. И, соответственно, весь трафик перераспределится на все доступные ноды в рамках сервисов.
    

### [Работа kubelet на практике](https://youtu.be/0rN_tUIZx7Y?t=2595)

![005_kubelet.png](./assets/lesson_7/005_kubelet.png)

После того как Scheduler выбрал ноду для пода и сохранил эту информацию в описании пода в игру вступает kubelet.

- kubelet (который висит на каждой ноде) смотри в api server и ждет информацию о том, что на его ноду был назначен под.
- соответствено, kubelet той ноды, на которую Scheduler назначил контейнер, видит, что появилась информаци о поде, назначенном на его ноду, но у него еще не запущенном
- kubelet информирует докер о том, что нужно скачать образ и запустить под контейнер с сетью и сам процесс.
- docker выполняет всю работу по запуску, о которой его попросил kubelet, а kubelet периодически пересылает информацию о статусе котейнера в api server.
- Api server сохраняет всю эту информацию в etcd

<aside>
☝ Именно на этом месте в кластере появляется запущенное приложение.
Т.е. с момента запроса о запуске пода до его фактического запуска проходит целая цепочка взаимодействий компонтов внутри k8s кластера.

</aside>

<aside>
☝ K8s такая система, в которой никто никому не отдает никаких команд.
Компонента подписывается на нужные им события, фактически просто их получает, обрабатывает и складывает информацию.
Kаждый компонет k8s занимается ровно своей частью работы и отвечает за свою часть общей инфраструктуры. Т.е. занимается тем, что приводит свою часть общей инфраструктуры к нужному состоянию и записывает это состояние в api server. Остальные компонеты следят за своей частью инфраструктуры и реагируют на изменения в api server'е.
Т.е. api server является центральной точкой правды.

</aside>

## [Kube-proxy](https://youtu.be/0rN_tUIZx7Y?t=3013)

kube-proxy - компонент, который занимается реализацией некоторых сетевых абстракций. 

Важный момент: kube-proxy не занимается непостредственно реализацией сети - этим занимается отдельный компонент (calico и прочее).

- Смотрит в Kube-API
- Стоит на всех серверах (в том числе и на мастер нодах)
- Управляет сетевыми правилами на нодах
- Фактически реализует Service (ipvs и iptables)

# [To be continued ...](https://youtu.be/0rN_tUIZx7Y?t=3042)