```
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cloud.google.com/cluster_autoscaler_unhelpable_since: 2026-02-20T09:36:10+0000
    cloud.google.com/cluster_autoscaler_unhelpable_until: Inf
  creationTimestamp: '2026-02-20T09:36:10Z'
  generateName: loki-chunks-cache-
  generation: 1
  labels:
    app.kubernetes.io/component: memcached-chunks-cache
    app.kubernetes.io/instance: loki
    app.kubernetes.io/name: loki
    apps.kubernetes.io/pod-index: '0'
    controller-revision-hash: loki-chunks-cache-dbf664dd6
    name: memcached-chunks-cache
    statefulset.kubernetes.io/pod-name: loki-chunks-cache-0
  name: loki-chunks-cache-0
  namespace: observability
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: StatefulSet
    name: loki-chunks-cache
    uid: eed07406-e7aa-4d28-a0da-fa74931dbf41
  resourceVersion: '1772093142321391014'
  uid: 10a7084c-31de-4f36-b9f0-b71d15b77859
spec:
  affinity: {}
  containers:
  - args:
    - -m 8192
    - --extended=modern,track_sizes
    - -I 5m
    - -c 16384
    - -v
    - -u 11211
    image: memcached:1.6.23-alpine
    imagePullPolicy: IfNotPresent
    name: memcached
    ports:
    - containerPort: 11211
      name: client
      protocol: TCP
    resources:
      limits:
        memory: 9830Mi
      requests:
        cpu: 500m
        memory: 9830Mi
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      readOnlyRootFilesystem: true
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-vw89b
      readOnly: true
  - args:
    - --memcached.address=localhost:11211
    - --web.listen-address=0.0.0.0:9150
    image: prom/memcached-exporter:v0.14.2
    imagePullPolicy: IfNotPresent
    name: exporter
    ports:
    - containerPort: 9150
      name: http-metrics
      protocol: TCP
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
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-vw89b
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  hostname: loki-chunks-cache-0
  nodeName: gke-basedosdados-dev-basedosdados-dev-f2aab3ed-8e7b
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: loki
  serviceAccountName: loki
  subdomain: loki-chunks-cache
  terminationGracePeriodSeconds: 60
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: kube-api-access-vw89b
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: '2026-02-26T08:05:42Z'
    status: 'True'
    type: PodReadyToStartContainers
  - lastProbeTime: null
    lastTransitionTime: '2026-02-26T08:04:49Z'
    status: 'True'
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: '2026-02-26T08:05:42Z'
    status: 'True'
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: '2026-02-26T08:05:42Z'
    status: 'True'
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: '2026-02-26T08:04:49Z'
    status: 'True'
    type: PodScheduled
  containerStatuses:
  - containerID: containerd://36f4f4508226cb41fa9ffa0dfd60ce0240b436eba156089cb70dd0124646c416
    image: docker.io/prom/memcached-exporter:v0.14.2
    imageID: docker.io/prom/memcached-exporter@sha256:d8a61419b8416c0090f40b6eb90eb152a4dd1680e23faba5513c3ac84b478b8f
    lastState: {}
    name: exporter
    ready: true
    resources: {}
    restartCount: 0
    started: true
    state:
      running:
        startedAt: '2026-02-26T08:05:41Z'
    user:
      linux:
        gid: 65534
        supplementalGroups:
        - 65534
        uid: 65534
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-vw89b
      readOnly: true
      recursiveReadOnly: Disabled
  - allocatedResources:
      cpu: 500m
      memory: 9830Mi
    containerID: containerd://7ce014c1b57d620de644ec85de1da9baffcce7082d8e1a3fd83a7bd44d34c671
    image: docker.io/library/memcached:1.6.23-alpine
    imageID: docker.io/library/memcached@sha256:41d19476c100c4b22a07c635e68d4e94e332eadd3afbd8f6bc61cd754d9b5d06
    lastState: {}
    name: memcached
    ready: true
    resources:
      limits:
        memory: 9830Mi
      requests:
        cpu: 500m
        memory: 9830Mi
    restartCount: 0
    started: true
    state:
      running:
        startedAt: '2026-02-26T08:05:03Z'
    user:
      linux:
        gid: 11211
        supplementalGroups:
        - 11211
        uid: 11211
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-vw89b
      readOnly: true
      recursiveReadOnly: Disabled
  hostIP: 10.128.0.12
  hostIPs:
  - ip: 10.128.0.12
  phase: Running
  podIP: 10.0.1.2
  podIPs:
  - ip: 10.0.1.2
  qosClass: Burstable
  startTime: '2026-02-26T08:04:49Z'

```