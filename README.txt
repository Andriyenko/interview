# Interview

Опишите deployment для веб-приложения в kubernetes в виде yaml-манифеста. Оставляйте в коде комментарии по принятым решениям. Есть следующие вводные:
* у нас мультизональный кластер (три зоны), в котором пять нод
* приложение требует около 5-10 секунд для инициализации
* по результатам нагрузочного теста известно, что 4 пода справляются с пиковой нагрузкой
* на первые запросы приложению требуется значительно больше ресурсов CPU, в дальнейшем потребление ровное в районе 0.1 CPU. По памяти всегда “ровно” в районе 128M memory
* приложение имеет дневной цикл по нагрузке – ночью запросов на порядки меньше, пик – днём
* хотим максимально отказоустойчивый deployment
* хотим минимального потребления ресурсов от этого deployment’а


apiVersion: v1
kind: Namespace
metadata:
  name: nginx
  labels:
    name:  nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
# Для распределния веб-приложения по всем нодам воспользовался affinity, с приоритетом 100 рекомендую не ложить все на одну и ту же ноду
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: nginx
              topologyKey: kubernetes.io/hostname
      terminationGracePeriodSeconds: 40 # для livenessProb, nginx посылает сигнал SIGTERM, период ожидания составляет 40 сек для корретного завершения приложения, пока не пошлется сигнал SIGKILL
      containers:
      - name: nginx
        image: nginx
# проверка на запуск, дал еще 2 сек для запуска приложения
        startupProbe:
          httpGet:
            path: /
            port: 80
          timeoutSeconds: 12
          failureThreshold: 1
# по умолчанию livenessProbe лучше совсем не использовать, либо сделать простую tcp проверку
        livenessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 60 
          periodSeconds: 120
        resources:
          requests:
            cpu: 100m
            memory: 128Mi # Ограничим контейнеры по памяти, так как requests == limits получим самый большой приоритет - Guaranteed
          limits:
            cpu: 300m # лимит в 300 был указан для того, чтобы работало hpa. После сбора информации с vpa, требуется поправить. 
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  labels:
    app: nginx
  namespace: nginx
spec:
  ports:
  - port: 80
    targetPort: 80
    name: nginx-svc
  selector:
    name: nginx-svc
  type: LoadBalancer
---
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: nginx
  updatePolicy:
    updateMode: "Off"   # Для сбора статистики использования ресурсов и после внесение изменений реквестов/лимитов в деплойменте
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        controlledResources: ["cpu", "memory"]
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx
  namespace: nginx
spec:
  minAvailable: 80% # В задании указано, что 4 ноды справляются с нагрузками, указываем минус 20% (5 нода)
  selector:
    matchLabels:
      app:  nginx
---
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 3 # почему минимум 3 поды: 1 пода может упасть, останется 1 пода и может не выдержать нагрузку, пока поднимется вторая пода, для этого используем 3 минимум
  maxReplicas: 10
  metrics:
   - type: Resource
     resource:
       name: cpu
       target:
         type: Utilization
         averageUtilization: 70
  behavior:
    scaleDown: # уменьшаем кол-во подов по минимальной политики
      stabilizationWindowSeconds: 60 # таймаут, чтобы не было флаппа
      policies:
      - type: Percent
        value: 5
        periodSeconds: 60
      - type: Pods
        value: 5
        periodSeconds: 300
      selectPolicy: Min
    scaleUp: # Увеличиваем кол-во под до 100% за 10 сек
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 10
