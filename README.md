# Gomplate support for nginx-unprivileged docker image

Add-on script to add `gomplate` template engine support to nginx-unprivileged Docker image

## Example dockerfile
Here's an example Dockerfile that demonstrates how to add Gomplate support to the nginx-unprivileged image:

```
FROM nginxinc/nginx-unprivileged:alpine

USER root

RUN apk add --no-cache gomplate && \
  curl https://raw.githubusercontent.com/cubesystems/nginx-gomplate/master/21-gomplate-on-templates.sh \
  -o /docker-entrypoint.d/21-gomplate-on-templates.sh && \
  chmod a+x /docker-entrypoint.d/21-gomplate-on-templates.sh

USER nginx

# Copy your custom template files with .gomplate prefix (Ex. docker/nginx-templates/default.conf.gomplate to replace default vhost)
COPY docker/nginx-templates/ /etc/nginx/templates/
```

## Sample Helm template for nginx deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  minReadySeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: {{ .Release.Name }}
      tier: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: {{ .Release.Name }}
        tier: nginx
        appVersion: {{ .Values.appVersion | quote }}
    spec:
      containers:
        - name: nginx
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 101 # nginx uid
            runAsGroup: 101 # nginx gid
{{- if .Values.deployments.nginx.envFrom }}
          envFrom: {{- toYaml .Values.deployments.nginx.envFrom | nindent 12 }}
{{- end }}
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /ping
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            successThreshold: 1
          volumeMounts:
            - mountPath: /tmp
              name: tmp
            - mountPath: /etc/nginx/conf.d
              name: nginx-conf
      imagePullSecrets:
        - name: {{ .Values.image.registrySecretName }}
      volumes:
        - name: tmp
          emptyDir: {}
        - name: nginx-conf
          emptyDir: {}

```
