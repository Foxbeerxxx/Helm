# Домашнее задание к занятию "`Helm`" - `Татаринцев Алексей`

---

### Задание 1



1. `Структура`

```
myapp/
  Chart.yaml
  values.yaml
  values-dev.yaml
  values-prod.yaml
  templates/
    namespace.yaml
    configmap.yaml
    svc-frontend.yaml
    deploy-frontend.yaml
    svc-backend.yaml
    deploy-backend.yaml
    sts-postgres.yaml
    svc-postgres.yaml
    ingress.yaml

```

2. `Chart.yaml`

```
apiVersion: v2
name: myapp
description: Multi-component app
type: application
version: 0.1.0
appVersion: "1.0.0"
```

3. `values.yaml (общие)`
```
namespace: myapp

imagePullPolicy: IfNotPresent

frontend:
  image:
    repository: registry.example.com/myapp/frontend
    tag: "1.0.0"
  replicas: 2
  port: 80
  env: []
  resources: {}

backend:
  image:
    repository: registry.example.com/myapp/backend
    tag: "1.0.0"
  replicas: 2
  port: 8080
  env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: postgres-secret
          key: url
  resources: {}

postgres:
  image:
    repository: postgres
    tag: "16.3"
  storage:
    size: 10Gi
  port: 5432
  auth:
    user: myapp
    password: mysecret
    db: myapp
  resources: {}

ingress:
  enabled: false
  className: ""
  host: myapp.local

```

4. `values-dev.yaml`

```
frontend:
  image:
    tag: "1.0.1-dev"
  replicas: 1

backend:
  image:
    tag: "1.0.1-dev"
  replicas: 1

ingress:
  enabled: true
  className: "nginx"
  host: dev.myapp.local

```

5. `values-prod.yaml`
```
frontend:
  image:
    tag: "1.0.3"
  replicas: 3
  resources:
    requests: {cpu: "200m", memory: "256Mi"}
    limits:   {cpu: "500m", memory: "512Mi"}

backend:
  image:
    tag: "1.0.3"
  replicas: 3
  resources:
    requests: {cpu: "300m", memory: "384Mi"}
    limits:   {cpu: "800m", memory: "768Mi"}

postgres:
  storage:
    size: 50Gi

ingress:
  enabled: true
  className: "nginx"
  host: myapp.example.com

```
6. ` templates/namespace.yaml `

```
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.namespace }}
```
7. `templates/configmap.yaml `

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: {{ .Values.namespace }}
data:
  APP_NAME: "myapp"

```

8. `templates/svc-frontend.yaml`

```
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: {{ .Values.namespace }}
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: {{ .Values.frontend.port }}

```

9. ` templates/deploy-frontend.yaml `

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: {{ .Values.namespace }}
spec:
  replicas: {{ .Values.frontend.replicas }}
  selector:
    matchLabels: { app: frontend }
  template:
    metadata:
      labels: { app: frontend }
    spec:
      containers:
      - name: frontend
        image: "{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}"
        imagePullPolicy: {{ .Values.imagePullPolicy }}
        ports:
        - containerPort: {{ .Values.frontend.port }}
        envFrom:
        - configMapRef: { name: myapp-config }
        env:
{{- toYaml .Values.frontend.env | nindent 8 }}
        resources:
{{- toYaml .Values.frontend.resources | nindent 10 }}

```
10. ` templates/svc-backend.yaml `

```
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: {{ .Values.namespace }}
spec:
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: {{ .Values.backend.port }}

```

11. ` templates/deploy-backend.yaml `

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: {{ .Values.namespace }}
spec:
  replicas: {{ .Values.backend.replicas }}
  selector:
    matchLabels: { app: backend }
  template:
    metadata:
      labels: { app: backend }
    spec:
      containers:
      - name: backend
        image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
        imagePullPolicy: {{ .Values.imagePullPolicy }}
        ports:
        - containerPort: {{ .Values.backend.port }}
        envFrom:
        - configMapRef: { name: myapp-config }
        env:
{{- toYaml .Values.backend.env | nindent 8 }}
        resources:
{{- toYaml .Values.backend.resources | nindent 10 }}

```
12. ` templates/svc-postgres.yaml `

```
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: {{ .Values.namespace }}
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432

```
13. ` templates/sts-postgres.yaml `

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: {{ .Values.namespace }}
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels: { app: postgres }
  template:
    metadata:
      labels: { app: postgres }
    spec:
      containers:
      - name: postgres
        image: "{{ .Values.postgres.image.repository }}:{{ .Values.postgres.image.tag }}"
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          value: "{{ .Values.postgres.auth.user }}"
        - name: POSTGRES_PASSWORD
          value: "{{ .Values.postgres.auth.password }}"
        - name: POSTGRES_DB
          value: "{{ .Values.postgres.auth.db }}"
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: {{ .Values.postgres.storage.size }}

```
14. ` templates/ingress.yaml `
```
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: {{ .Values.namespace }}
  annotations:
    kubernetes.io/ingress.class: {{ .Values.ingress.className | quote }}
spec:
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
{{- end }}

```
15. `Установка и проверка.  `

```
helm install myapp ./myapp -n myapp --create-namespace -f myapp/values-dev.yaml
kubectl -n myapp get pods,svc,ingress
```
![1](https://github.com/Foxbeerxxx/Helm/blob/main/img/img1.png)



16. `Добавляю в host и проверяю через Ingress `

```
echo "127.0.0.1 dev.myapp.local" | sudo tee -a /etc/hosts
curl -i -H "Host: dev.myapp.local" http://127.0.0.1/
```
![2](https://github.com/Foxbeerxxx/Helm/blob/main/img/img2.png)


16. `Версии`

```
в Chart.yaml
Меняю appVersion: "1.25-alpine"

правлю image

image: "{{ .Values.frontend.image.repository }}:{{ default (default .Chart.AppVersion .Values.global.imageTag) .Values.frontend.image.tag }}"

Обновляю версию 
helm -n myapp upgrade myapp ./myapp --reuse-values
```

![2](https://github.com/Foxbeerxxx/Helm/blob/main/img/img2.png)

---

### Задание 2

`Приведите ответ в свободной форме........`

1. `Заполните здесь этапы выполнения, если требуется ....`
2. `Заполните здесь этапы выполнения, если требуется ....`
3. `Заполните здесь этапы выполнения, если требуется ....`
4. `Заполните здесь этапы выполнения, если требуется ....`
5. `Заполните здесь этапы выполнения, если требуется ....`
6. 

```
Поле для вставки кода...
....
....
....
....
```

`При необходимости прикрепитe сюда скриншоты
![Название скриншота 2](ссылка на скриншот 2)`


---

### Задание 3

`Приведите ответ в свободной форме........`

1. `Заполните здесь этапы выполнения, если требуется ....`
2. `Заполните здесь этапы выполнения, если требуется ....`
3. `Заполните здесь этапы выполнения, если требуется ....`
4. `Заполните здесь этапы выполнения, если требуется ....`
5. `Заполните здесь этапы выполнения, если требуется ....`
6. 

```
Поле для вставки кода...
....
....
....
....
```

`При необходимости прикрепитe сюда скриншоты
![Название скриншота](ссылка на скриншот)`

### Задание 4

`Приведите ответ в свободной форме........`

1. `Заполните здесь этапы выполнения, если требуется ....`
2. `Заполните здесь этапы выполнения, если требуется ....`
3. `Заполните здесь этапы выполнения, если требуется ....`
4. `Заполните здесь этапы выполнения, если требуется ....`
5. `Заполните здесь этапы выполнения, если требуется ....`
6. 

```
Поле для вставки кода...
....
....
....
....
```

`При необходимости прикрепитe сюда скриншоты
![Название скриншота](ссылка на скриншот)`
