### Create-a-Java-app-helmchart

### Technologies used:

Java, helm

### Project description:

1-create a java-app chart

### Instructions:

###### Step 1: Create a chart structure

```
helm create java-app
```

###### Step 2: Create db-config.yaml, db-secret.yaml and java-app-ingress.yaml template engine

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Values.configName }}
data:
  {{- range $key, $value := .Values.configData}}
  {{ $key }}: {{ $value }}
  {{- end}}

```

```
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Values.secretName }}
type: Opaque
data:
  {{- range $key, $value := .Values.secretData}}
  {{ $key }}: {{ $value | b64enc }}
  {{- end}}
```

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Values.appName }}-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: {{ .Values.ingress.hostName }}
    http:
      paths:
      - backend:
          service:
            name: {{ .Values.appName }}
            port:
              number: {{ .Values.servicePort }}
        pathType: {{ .Values.ingress.pathType }}
        path: {{ .Values.ingress.path }}
```

###### Step 3: Create a java-app deployment and service template engine

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.appName }}-deployment
  labels:
    app: {{ .Values.appName }}
spec:
  replicas: {{ .Values.appReplicas }}
  selector:
    matchLabels:
      app: {{ .Values.appName }}
  template:
    metadata:
      labels:
        app: {{ .Values.appName }}
    spec:
      imagePullSecrets:
      - name: {{ .Values.registrySecret }}
      containers:
      - name: {{ .Values.appContainerName }}
        image: "{{ .Values.appImage }}:{{ .Values.appVersion }}"
        ports:
        - containerPort: {{ .Values.containerPort }}
        env:
        {{- range $key, $value := .Values.regularData }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end}}

        {{- range $key, $value := .Values.secretData}}
        - name: {{ $key }}
          valueFrom:
            secretKeyRef:
              # in loop, we lose global context, but can access global context with $
              # $ is 1 variable that is always global and will always point to the root context
              # so $.Value instead of .Values
              name: {{ $.Values.secretName }}
              key: {{ $key }}
        {{- end}}

        {{- range $key, $value := .Values.configData}}
        - name: {{ $key }}
          valueFrom:
            configMapKeyRef:
              name: {{ $.Values.configName }}
              key: {{ $key }}
        {{- end}}
```

```
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.appName }}-service
spec:
  selector:
    app: {{ .Values.appName }}
  ports:
  - protocol: TCP
    port: {{ .Values.servicePort }}
    targetPort: {{ .Values.containerPort }}
```

###### Step 4: Create a default values.yaml for java-helmchart

```
appName: my-app
appReplicas: 1
registrySecret: my-registry-key
appContainerName: myapp
appImage: myimage
appVersion: versiontag
containerPort: 80

servicePort: 80

configName: my-config
configData: {}

secretName: my-secret
secretData: {}

regularData: {}

ingress:
hostName: my-java-app.com
pathType: Exact
path: /
```

###### Step 5: Create a values-override.yaml file

```
appName: java-app
appReplicas: 3
registrySecret: my-registry-key
appContainerName: javamysqlapp
appImage: nanajanashia/demo-app
appVersion: java-mysql-app
containerPort: 8080

servicePort: 8080

configName: db-config
configData:
  DB_SERVER: my-release-mysql-primary

secretName: db-secret
secretData:
  DB_USER: my-user
  DB_PWD: my-pass
  DB_NAME: my-app-db
  MYSQL_ROOT_PASSWORD: secret-root-pass

regularData: {}
 # MY_ENV: my-value

ingress:
  hostName: my-java-app.com # set this value to Linode nodebalancer address for LKE
  pathType: Exact
  path: /
```

###### Step 6: Test

```
helm install my-java-app-helmchart java-app -f java-app/values-override.yaml --dry-run --debug
```

###### Step 7: Install the java-app-helmchart

```
helm install my-java-app-helmchart java-app -f java-app/values-override.yaml
```
