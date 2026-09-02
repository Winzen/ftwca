```
apiVersion: apps/v1

kind: StatefulSet

metadata:

  annotations:

    meta.helm.sh/release-name: loki

    meta.helm.sh/release-namespace: observability

  creationTimestamp: '2024-06-04T00:05:14Z'

  generation: 1

  labels:

    app.kubernetes.io/component: single-binary

    app.kubernetes.io/instance: loki

    app.kubernetes.io/managed-by: Helm

    app.kubernetes.io/name: loki

    app.kubernetes.io/part-of: memberlist

    app.kubernetes.io/version: 3.0.0

    helm.sh/chart: loki-6.6.2

  name: loki

  namespace: observability

  resourceVersion: '1772093312692303004'

  uid: 141c5822-41b7-45d2-bcc9-c125e1634a16

spec:

  persistentVolumeClaimRetentionPolicy:

    whenDeleted: Delete

    whenScaled: Delete

  podManagementPolicy: Parallel

  replicas: 1

  revisionHistoryLimit: 10

  selector:

    matchLabels:

      app.kubernetes.io/component: single-binary

      app.kubernetes.io/instance: loki

      app.kubernetes.io/name: loki

  serviceName: loki-headless

  template:

    metadata:

      annotations:

        checksum/config: 7c009abf8b3c8fbfb191b8c0d96d91bb1525f808cabb67c178e31baa98417225

      creationTimestamp: null

      labels:

        app.kubernetes.io/component: single-binary

        app.kubernetes.io/instance: loki

        app.kubernetes.io/name: loki

        app.kubernetes.io/part-of: memberlist

    spec:

      affinity:

        podAntiAffinity:

          requiredDuringSchedulingIgnoredDuringExecution:

          - labelSelector:

              matchLabels:

                app.kubernetes.io/component: single-binary

            topologyKey: kubernetes.io/hostname

      automountServiceAccountToken: true

      containers:

      - args:

        - -config.file=/etc/loki/config/config.yaml

        - -target=all

        image: docker.io/grafana/loki:3.0.0

        imagePullPolicy: IfNotPresent

        name: loki

        ports:

        - containerPort: 3100

          name: http-metrics

          protocol: TCP

        - containerPort: 9095

          name: grpc

          protocol: TCP

        - containerPort: 7946

          name: http-memberlist

          protocol: TCP

        readinessProbe:

          failureThreshold: 3

          httpGet:

            path: /ready

            port: http-metrics

            scheme: HTTP

          initialDelaySeconds: 30

          periodSeconds: 10

          successThreshold: 1

          timeoutSeconds: 1

        resources: {}

        securityContext:

          allowPrivilegeEscalation: false

          capabilities:

            drop:

            - ALL

          readOnlyRootFilesystem: true

        terminationMessagePath: /dev/termination-log

        terminationMessagePolicy: File

        volumeMounts:

        - mountPath: /tmp

          name: tmp

        - mountPath: /etc/loki/config

          name: config

        - mountPath: /etc/loki/runtime-config

          name: runtime-config

        - mountPath: /var/loki

          name: storage

      dnsPolicy: ClusterFirst

      enableServiceLinks: true

      restartPolicy: Always

      schedulerName: default-scheduler

      securityContext:

        fsGroup: 10001

        runAsGroup: 10001

        runAsNonRoot: true

        runAsUser: 10001

      serviceAccount: loki

      serviceAccountName: loki

      terminationGracePeriodSeconds: 30

      volumes:

      - emptyDir: {}

        name: tmp

      - configMap:

          defaultMode: 420

          items:

          - key: config.yaml

            path: config.yaml

          name: loki

        name: config

      - configMap:

          defaultMode: 420

          name: loki-runtime

        name: runtime-config

  updateStrategy:

    rollingUpdate:

      partition: 0

    type: RollingUpdate

  volumeClaimTemplates:

  - apiVersion: v1

    kind: PersistentVolumeClaim

    metadata:

      creationTimestamp: null

      name: storage

    spec:

      accessModes:

      - ReadWriteOnce

      resources:

        requests:

          storage: 100Gi

      volumeMode: Filesystem

    status:

      phase: Pending

status:

  availableReplicas: 0

  collisionCount: 0

  currentReplicas: 1

  currentRevision: loki-79b68b5d44

  observedGeneration: 1

  replicas: 1

  updateRevision: loki-79b68b5d44

  updatedReplicas: 1
```
