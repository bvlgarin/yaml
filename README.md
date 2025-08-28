apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: default
  labels:
    app: web-app
spec:
  # Реплики: 4 для обработки пиковой нагрузки + 1 для отказоустойчивости
  # Всего 5 подов - по одному на каждую ноду в мультизональном кластере
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      # Медленное обновление для минимизации простоя
      maxSurge: 1
      maxUnavailable: 0  # Гарантируем доступность во время обновлений
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      # Распределение подов по разным зонам для отказоустойчивости
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: web-app
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: web-app
      
      # Приоритетность для экономии ресурсов ночью
      priorityClassName: medium-priority
      
      containers:
      - name: web-app
        image: your-web-app:latest
        imagePullPolicy: IfNotPresent
        
        # Ресурсы с запасом для стартовой фазы
        resources:
          requests:
            memory: "256Mi"    # 2x от обычного потребления для буфера при инициализации
            cpu: "500m"        # Высокий CPU для старта, потом автоматически scale down
          limits:
            memory: "512Mi"    # Лимит с запасом
            cpu: "1000m"       # Лимит для пиковых нагрузок при старте
        
        # Health checks для управления жизненным циклом
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15  # Даем время на инициализацию (5-10 сек + запас)
          periodSeconds: 10
          failureThreshold: 3
          timeoutSeconds: 5
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 12  # Чуть раньше чем liveness
          periodSeconds: 5
          failureThreshold: 1
          timeoutSeconds: 3
        
        # Startup probe для долгоинициализирующихся приложений
        startupProbe:
          httpGet:
            path: /health
            port: 8080
          failureThreshold: 30  # 10 сек инициализация * 3 = 30 попыток
          periodSeconds: 1
        
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        
        # Переменные окружения для оптимизации
        env:
        - name: JAVA_OPTS
          value: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
        - name: NODE_ENV
          value: "production"

---
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  labels:
    app: web-app
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  type: ClusterIP
  # Session affinity для стабильности соединений
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 часа

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2  # Минимум ночью для экономии ресурсов
  maxReplicas: 5  # Максимум днем для обработки пика
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60  # Консервативный таргет для стабильности
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Медленное уменьшение для ночного цикла
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60   # Быстрое увеличение для дневного пика
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60

---
# PriorityClass для управления приоритетами
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: medium-priority
value: 5000000
globalDefault: false
description: "Medium priority for web application"