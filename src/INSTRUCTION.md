# Інструкції з розгортання Django ToDo App у Kubernetes

Цей документ містить покрокові інструкції щодо розгортання Django ToDo List веб-додатку в кластері Kubernetes, а також докладні пояснення щодо вибору конфігурацій для Deployment, Horizontal Pod Autoscaler (HPA) та Service. Всі Kubernetes-ресурси будуть знаходитись у просторі імен `mateapp`.

## Передумови

Перед початком переконайтеся, що у вас встановлені та налаштовані наступні інструменти:

* **Docker**: Для збірки та керування Docker-образами.
* **kubectl**: Інструмент командного рядка для взаємодії з Kubernetes кластером.
* **Кластер Kubernetes**: Активний і доступний кластер (наприклад, Minikube, Docker Desktop Kubernetes або будь-який хмарний кластер).
* **Акаунт Docker Hub** (або доступ до іншого Container Registry): Для зберігання Docker-образу.
* **Форкнутий репозиторій** з кодом додатку.

## 1. Структура Проєкту

Згідно з початковими умовами та вашим уточненням, всі необхідні Kubernetes маніфести та цей файл інструкцій (`INSTRUCTION.md`) будуть розміщені в директорії `src`, поруч із `Dockerfile`, `manage.py` та `requirements.txt`.

Приклад структури директорій:
````
devops_todolist_kubernetes_task_5_working_with_deployments/
├── .infrastructure/
├── src/
│   ├── accounts/
│   ├── api/
│   │   └── views.py
│   ├── lists/
│   ├── todolist/
│   ├── Dockerfile
│   ├── manage.py
│   ├── requirements.txt
│   ├── deployment.yml         # Розгортання додатку
│   ├── hpa.yml                # Автомасштабування
│   ├── service.yml            # Доступ до додатку
│   └── INSTRUCTION.md         # Цей файл
├── .gitignore
├── LICENSE
└── README.md
````
## 2. Підготовка Docker-образу

Перед розгортанням додатку в Kubernetes, необхідно зібрати Docker-образ вашого додатку та завантажити його до Container Registry (наприклад, Docker Hub), щоб Kubernetes міг його витягнути.

* **Крок 2.1: Перейдіть до робочої директорії**

    Відкрийте термінал і перейдіть до директорії `src`, де знаходиться ваш `Dockerfile`:

    ```bash
    cd ~/devops_todolist_kubernetes_task_5_working_with_deployments/src
    ```

* **Крок 2.2: Зберіть Docker-образ**

    Виконайте наступну команду, замінивши `your-username` на ваш **логін Docker Hub**. Ця команда збере Docker-образ з тегом `latest`.

    ```bash
    docker build -t your-username/todo-app:latest .
    ```

* **Крок 2.3: Увійдіть до Docker Hub**

    Якщо ви ще не увійшли до свого акаунту Docker Hub, виконайте:

    ```bash
    docker login
    ```
    Введіть ваш логін та пароль, коли буде запропоновано.

* **Крок 2.4: Завантажте образ до Docker Hub**

    Завантажте щойно створений образ до вашого репозиторію в Docker Hub:

    ```bash
    docker push your-username/todo-app:latest
    ```

## 3. Створення Kubernetes Маніфестів

Створіть наступні три файли у директорії `src`.

### 3.1. `deployment.yml`

Цей файл описує Deployment — ресурс Kubernetes, який керує розгортанням та життєвим циклом ваших подів.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-app-deployment
  namespace: mateapp
  labels:
    app: todo-app
spec:
  replicas: 2 # Забезпечуємо 2 поди в idle-стані
  selector:
    matchLabels:
      app: todo-app
  strategy:
    type: RollingUpdate # Використовуємо стратегію RollingUpdate
    rollingUpdate:
      maxSurge: 25% # Дозволяємо на 25% більше підів під час оновлення
      maxUnavailable: 25% # Дозволяємо, щоб 25% підів були недоступними під час оновлення
  template:
    metadata:
      labels:
        app: todo-app
    spec:
      containers:
      - name: todo-app-container
        image: leoleiden/todo-app:latest
        ports:
        - containerPort: 8080 # Порт, який прослуховує Django-додаток з Dockerfile
        resources:
          requests: # Запитувані ресурси
            cpu: "100m" # 0.1 CPU core (100 millicores)
            memory: "128Mi" # 128 Megabytes
          limits: # Максимально дозволені ресурси
            cpu: "200m" # 0.2 CPU cores
            memory: "128Mi" # 128 Megabytes
        livenessProbe: # Перевірка життєздатності контейнера
          httpGet:
            path: /api/health # Шлях до health check endpoint
            port: 8080
          initialDelaySeconds: 15 # Затримка перед першою перевіркою
          periodSeconds: 20 # Частота перевірок
          failureThreshold: 3 # Кількість невдалих перевірок до перезапуску
        readinessProbe: # Перевірка готовності контейнера до прийому трафіку
          httpGet:
            path: /api/ready # Шлях до readiness check endpoint
            port: 8080
          initialDelaySeconds: 30 # Даємо Django час на запуск і міграції
          periodSeconds: 20 # Частота перевірок
          failureThreshold: 3 # Кількість невдалих перевірок до позначення як NotReady
```
### 3.2. `hpa.yml`
```yaml
apiVersion: autoscaling/v2beta2 # Використовуємо v2beta2 для Kubernetes 1.19
kind: HorizontalPodAutoscaler
metadata:
  name: todo-app-hpa
  namespace: mateapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: todo-app-deployment # Посилання на наш Deployment
  minReplicas: 2 # Мінімальна кількість подів
  maxReplicas: 5 # Максимальна кількість подів
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70 # Масштабувати при 70% середнього використання CPU
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70 # Масштабувати при 70% середнього використання пам'яті
```
### 3.3. `service.yml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: todo-app-service
  namespace: mateapp
spec:
  selector:
    app: todo-app # Повинен відповідати labels в Deployment
  type: NodePort # Використовуємо NodePort для простоти доступу
  ports:
    - protocol: TCP
      port: 80 # Порт, на який буде слухати Service всередині кластера
      targetPort: 8080 # Порт контейнера, який виставлений у Dockerfile
      nodePort: 32493 # Конкретний NodePort, отриманий з kubectl get svc
```
## 4. Розгортання Додатку в Kubernetes

Після створення всіх YAML-файлів (`deployment.yml`, `hpa.yml`, `service.yml`), виконайте наступні кроки для розгортання вашого додатку в Kubernetes кластері. Переконайтеся, що ви знаходитесь у директорії `src`, де розташовані ваші `.yml` файли.

* **Крок 4.1: Створіть Namespace `mateapp`**

    Спочатку створіть простір імен (`namespace`), в якому будуть розміщені ваші Kubernetes-ресурси.

    ```bash
    kubectl create namespace mateapp
    ```
    * *Примітка*: Якщо ви отримаєте повідомлення `Error from server (AlreadyExists): namespaces "mateapp" already exists`, це означає, що простір імен вже існує, і це нормально. Це не є помилкою, що перешкоджає подальшій роботі.

* **Крок 4.2: Застосуйте Deployment**

    Розгорніть ваш Django ToDo додаток, створивши Deployment:

    ```bash
    kubectl apply -f deployment.yml -n mateapp
    ```

* **Крок 4.3: Застосуйте Horizontal Pod Autoscaler (HPA)**

    Створіть HPA, який відповідатиме за автоматичне масштабування вашого додатку:

    ```bash
    kubectl apply -f hpa.yml -n mateapp
    ```

* **Крок 4.4: Застосуйте Service**

    Створіть Service, який забезпечить мережевий доступ до подів вашого додатку:

    ```bash
    kubectl apply -f service.yml -n mateapp
    ```

## 5. Перевірка Стану Розгортання

Після застосування всіх маніфестів важливо перевірити статус розгорнутих ресурсів, щоб переконатися, що все працює належним чином.

* **Перевірте Pods**:
    Перегляньте статус ваших подів. Ви повинні побачити 2 поди у стані `Running`.

    ```bash
    kubectl get pods -n mateapp
    ```
    Приклад виводу:
    ```
    NAME                                     READY   STATUS    RESTARTS   AGE
    todo-app-deployment-848b8d4c64-lbvzf     1/1     Running   0          3m43s
    todo-app-deployment-848b8d4c64-mdzzw     1/1     Running   0          3m43s
    ```

* **Перевірте Deployment**:
    Перегляньте статус вашого Deployment. Вивід повинен показати `2/2` в колонці `READY`, що підтверджує, що бажана кількість реплік працює.

    ```bash
    kubectl get deploy -n mateapp
    ```
    Приклад виводу:
    ```
    NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
    todo-app-deployment   2/2     2            2           5m9s
    ```

* **Перевірте HPA**:
    Перегляньте статус Horizontal Pod Autoscaler. HPA повинен показувати `MINPODS: 2`, `MAXPODS: 5`, `REPLICAS: 2`. Колонка `TARGETS` може спочатку показувати `<unknown>/<target>%` доки HPA не збере достатньо метрик з Pods.

    ```bash
    kubectl get hpa -n mateapp
    ```
    Приклад виводу:
    ```
    NAME           REFERENCE                       TARGETS                       MINPODS   MAXPODS   REPLICAS   AGE
    todo-app-hpa   Deployment/todo-app-deployment  <unknown>/60%, <unknown>/60%  2         5         2          2m3s
    ```

  * **Перевірте Service**:
      Перегляньте деталі вашого Service. Вивід покаже тип сервісу (`NodePort`), внутрішню IP-адресу кластера (`CLUSTER-IP`), зовнішню IP-адресу (може бути `localhost` або `<pending>` для локальних кластерів) та мапування портів (`PORT(S)`).

      ```bash
      kubectl get svc -n mateapp
      ```
      Приклад виводу:
      ```
      NAME               TYPE            CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
      todo-app-service   LoadBalancer    10.101.222.183   localhost        80:32493/TCP   2m2s
      ```
---

## 6. Explanations of choosing resource requests and limits, choosing HPA configuration, choosing strategy configuration, and accessing the application after deployment.
### 6.1 Justification for RollingUpdate Strategy (and its maxSurge, maxUnavailable parameters)
RollingUpdate Strategy:

Reasoning: RollingUpdate is the standard and recommended update strategy in Kubernetes for web applications that require high availability. It allows for the gradual replacement of old Pod versions with new ones, ensuring that the application remains accessible throughout the update process. This is achieved by creating new Pods and progressively terminating old ones as the new Pods become "ready".

maxSurge and maxUnavailable Parameters (25% values):

Reasoning: For this task, the default values of maxSurge: 25% and maxUnavailable: 25% have been chosen. With an initial replica count of 2, 25% of 2 is 0.5, which rounds up to 1. Therefore, maxSurge will be 1, and maxUnavailable will also be 1.

Practical Implication of maxSurge: 1: During an update, one additional Pod can be created. This means the cluster will temporarily contain 3 Pods (2 old + 1 new), ensuring a smooth transition without significantly increasing resource strain. This also guarantees that new Pods are launched and become "ready" before old Pods are terminated.

Practical Implication of maxUnavailable: 1: Allows one Pod to be unavailable during the update. With 2 replicas, this means at least 1 Pod will always remain available to handle requests, ensuring basic service continuity.

Why These Numbers: For a ToDo application, which is likely not extremely mission-critical but needs to maintain availability, the default 25% values represent an optimal compromise. They balance update speed with minimal downtime, without requiring excessive over-provisioning of resources

### 6.2 Justification for Resource Requests and Limits (CPU and Memory)
Setting resource requests (requests) and limits (limits) for containers is crucial for the stability and efficiency of applications running in Kubernetes.

Requests:

CPU: 100m (0.1 CPU core): This is sufficient for a Django application in an idle state, as it typically isn't CPU-intensive. This is the minimum guaranteed amount of resources Kubernetes will allocate.

Memory: 128Mi (128 Megabytes): A reasonable starting value that provides enough buffer for Python's overhead and small bursts in memory usage. Estimates suggest Django applications consume between 60MB and 130MB.

Why These Numbers: These values are critically important for the correct functioning of the Horizontal Pod Autoscaler, as the HPA calculates utilization percentages based on the requested resources.

Limits:

CPU: 500m (0.5 CPU core): Allows the Pod to use more resources when processing requests (providing a "burst" capacity), avoiding throttling, while still controlling its maximum consumption to prevent node overload.

Memory: 128Mi (128 Megabytes): It is recommended to set memory requests and limits to be equal. This ensures predictable behavior and minimizes the risk of the Pod being prematurely terminated due to out-of-memory (OOMKilled) issues, as memory cannot be overcommitted. The Pod will only be killed if the overall node runs out of memory, which ensures predictable performance.

Why These Numbers: These values are a starting point, and in real production environments, they should be fine-tuned based on monitoring the application's actual resource usage under various loads.

### 6.3 Justification for HPA Configuration
The Horizontal Pod Autoscaler (HPA) automatically scales the number of Deployment replicas based on observed metrics.

apiVersion: autoscaling/v2beta2:

Reasoning: Your Kubernetes cluster is running on version 1.19.7. The autoscaling/v2beta2 API version is appropriate for this cluster version and supports scaling based on multiple metrics (CPU and memory), which aligns with the task requirements.

minReplicas: 2:

Reasoning: This is the minimum number of Pods the HPA will maintain. This matches the task requirement of 2 Pods in an "idle state" and ensures basic fault tolerance and initial capacity.

maxReplicas: 5:

Reasoning: This is the maximum number of Pods the HPA can scale the Deployment up to. Limiting the maximum number prevents uncontrolled resource growth in response to unforeseen traffic spikes or misconfiguration, helping to control costs and cluster resources.

Scaling by CPU and Memory (averageUtilization: 70%):

Reasoning: The HPA will aim to maintain an average CPU and memory utilization of 70% across all Pods in the Deployment. This value is a common "sweet spot." It is high enough to avoid over-scaling for minor load fluctuations but low enough to react to significant traffic increases before resources are fully exhausted, providing a sufficient performance buffer. If we set a higher percentage, Pods might become overloaded before scaling occurs. If lower, scaling would happen more frequently, potentially increasing costs. For a typical Django application that might experience traffic bursts, 70% is a reasonable starting point. 

### 6.4 Justification for Application Access after Deployment (Kubernetes Service)
A Kubernetes Service provides a stable network endpoint for dynamic Pods.

Choice of Service Type (LoadBalancer or NodePort):

For Production Environments: LoadBalancer is the most recommended Service type. It automatically provisions an external load balancer in a cloud provider, providing a public IP address and distributing traffic. This ensures high availability, scalability, and even traffic distribution.

For Local Testing / Demonstrations: NodePort is a simpler option. It opens a static port (from the 30000-32767 range) on each node in the cluster. The application becomes accessible externally via Node_IP:NodePort.

Why LoadBalancer/NodePort Was Chosen: The task requires external access to the application. LoadBalancer is ideal if there's cloud provider integration. If not (as with Minikube/Docker Desktop), NodePort is a convenient alternative.

## 7. How to Access the Application:
For a LoadBalancer Service: Obtain the Service's external IP address using kubectl get svc <service-name> -n <namespace> -o jsonpath='{.status.loadBalancer.ingress.ip}'. The application will be accessible at http://<EXTERNAL-IP>.

For a NodePort Service:

Find the assigned NodePort (e.g., 32493) from the output of kubectl get svc <service-name> -n <namespace>.

Get the IP address of a cluster node (e.g., minikube ip for Minikube, or localhost for Docker Desktop).

The application will be accessible at http://<Kubernetes_Node_IP>:<NODE-PORT>.