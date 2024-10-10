---
title: cloudnative-pg
permalink: /docs/home-ops/cnpg/
---

Time to setup a postgres cluster like a BOSS. CLOUD NATIVE

First we set up the chart and secret but now the tricky part, the cluster (version 16)

`cluster16.yaml`
```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/postgresql.cnpg.io/cluster_v1.json
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres16
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:16.4-27
  primaryUpdateStrategy: unsupervised
  storage:
    size: 100Gi
    storageClass: ceph-rbd
  superuserSecret:
    name: cloudnative-pg-secret
  enableSuperuserAccess: true
  postgresql:
    parameters:
      max_connections: "400"
      shared_buffers: 256MB
  nodeMaintenanceWindow:
    inProgress: false
    reusePVC: true
  # TODO we need to dig in here
  #resources:
  #  requests:
  #    cpu: 500m
  #  limits:
  #    memory: 4Gi
  monitoring:
    enablePodMonitor: true
  backup:
    retentionPolicy: 30d
    barmanObjectStore: &barmanObjectStore
      data:
        compression: bzip2
      wal:
        compression: bzip2
        maxParallel: 8
      destinationPath: s3://s3-cnpg/
      endpointURL: https://s3.amazonaws.com
      s3BucketRegion: us-east-2
      # Note: serverName version needs to be incremented
      # when recovering from an existing cnpg cluster
      serverName: &currentCluster postgres16-v1
      s3Credentials:
        accessKeyId:
          name: cloudnative-pg-secret
          key: aws-access-key-id
        secretAccessKey:
          name: cloudnative-pg-secret
          key: aws-secret-access-key
  # Note: previousCluster needs to be set to the name of the previous
  # cluster when recovering from an existing cnpg cluster
  bootstrap:
    recovery:
      source: &previousCluster postgres16-v0
  # Note: externalClusters is needed when recovering from an existing cnpg cluster
  externalClusters:
    - name: *previousCluster
      barmanObjectStore:
        <<: *barmanObjectStore
        serverName: *previousCluster
```

s3:s3.amazonaws.com/s3-volsync

thaynes@HaynesHyperion:~$ k -n databases logs postgres16-1-full-recovery-9kvfg
Defaulted container "full-recovery" out of: full-recovery, bootstrap-controller (init), k8tz (init)
{"level":"info","ts":"2024-10-01T23:21:58-04:00","msg":"barman-cloud-check-wal-archive checking the first wal","logging_pod":"postgres16-1-full-recovery"}
{"level":"info","ts":"2024-10-01T23:21:58-04:00","logger":"barman-cloud-check-wal-archive","msg":"2024-10-01 23:21:58,839 [19] ERROR: Barman cloud WAL archive check exception: An error occurred (301) when calling the HeadBucket operation: Moved Permanently","pipe":"stderr","logging_pod":"postgres16-1-full-recovery"}
{"level":"error","ts":"2024-10-01T23:21:58-04:00","msg":"Error invoking barman-cloud-check-wal-archive","logging_pod":"postgres16-1-full-recovery","currentPrimary":"","targetPrimary":"postgres16-1","options":["--endpoint-url","https://s3.amazonaws.com","--cloud-provider","aws-s3","s3://s3-cnpg/","postgres16-v1"],

"exitCode":-1,"error":"exit status 4","stacktrace":"github.com/cloudnative-pg/cloudnative-pg/pkg/management/log.(*logger).Error\n\tpkg/management/log/log.go:125\ngithub.com/cloudnative-pg/cloudnative-pg/pkg/management/barman/archiver.(*WALArchiver).CheckWalArchiveDestination\n\tpkg/management/barman/archiver/archiver.go:257\ngithub.com/cloudnative-pg/cloudnative-pg/pkg/management/postgres.(*InitInfo).checkBackupDestination\n\tpkg/management/postgres/restore.go:950\ngithub.com/cloudnative-pg/cloudnative-pg/pkg/management/postgres.InitInfo.Restore\n\tpkg/management/postgres/restore.go:254\ngithub.com/cloudnative-pg/cloudnative-pg/internal/cmd/manager/instance/restore.restoreSubCommand\n\tinternal/cmd/manager/instance/restore/cmd.go:90\ngithub.com/cloudnative-pg/cloudnative-pg/internal/cmd/manager/instance/restore.NewCmd.func2\n\tinternal/cmd/manager/instance/restore/cmd.go:63\ngithub.com/spf13/cobra.(*Command).execute\n\tpkg/mod/github.com/spf13/cobra@v1.8.1/command.go:985\ngithub.com/spf13/cobra.(*Command).ExecuteC\n\tpkg/mod/github.com/spf13/cobra@v1.8.1/command.go:1117\ngithub.com/spf13/cobra.(*Command).Execute\n\tpkg/mod/github.com/spf13/cobra@v1.8.1/command.go:1041\nmain.main\n\tcmd/manager/main.go:66\nruntime.main\n\t/opt/hostedtoolcache/go/1.22.6/x64/src/runtime/proc.go:271"}

{"level":"error","ts":"2024-10-01T23:21:58-04:00","msg":"Error while restoring a backup","logging_pod":"postgres16-1-full-recovery","error":"unexpected failure invoking barman-cloud-wal-archive: exit status 4","stacktrace":"github.com/cloudnative-pg/cloudnative-pg/pkg/management/log.(*logger).Error\n\tpkg/management/log/log.go:125\ngithub.com/cloudnative-pg/cloudnative-pg/pkg/management/log.Error\n\tpkg/management/log/log.go:163\ngithub.com/cloudnative-pg/cloudnative-pg/internal/cmd/manager/instance/restore.restoreSubCommand\n\tinternal/cmd/manager/instance/restore/cmd.go:92\ngithub.com/cloudnative-pg/cloudnative-pg/internal/cmd/manager/instance/restore.NewCmd.func2\n\tinternal/cmd/manager/instance/restore/cmd.go:63\ngithub.com/spf13/cobra.(*Command).execute\n\tpkg/mod/github.com/spf13/cobra@v1.8.1/command.go:985\ngithub.com/spf13/cobra.(*Command).ExecuteC\n\tpkg/mod/github.com/spf13/cobra@v1.8.1/command.go:1117\ngithub.com/spf13/cobra.(*Command).Execute\n\tpkg/mod/github.com/spf13/cobra@v1.8.1/command.go:1041\nmain.main\n\tcmd/manager/main.go:66\nruntime.main\n\t/opt/hostedtoolcache/go/1.22.6/x64/src/runtime/proc.go:271"}
thaynes@HaynesHyperion:~$


```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/kustomize.toolkit.fluxcd.io/kustomization_v1beta2.json
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: &app cloudnative-pg
  namespace: flux-system
spec:
  targetNamespace: databases
  commonMetadata:
    labels:
      app.kubernetes.io/name: *app
  dependsOn:
    - name: external-secrets-stores
  interval: 15m
  retryInterval: 1m  
  timeout: 2m
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./kubernetes/main/apps/databases/cloudnative-pg/app
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2beta1
      kind: HelmRelease
      name: cloudnative-pg
      namespace: databases
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/kustomize.toolkit.fluxcd.io/kustomization_v1beta2.json
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: &app cloudnative-pg-cluster
  namespace: flux-system
spec:
  targetNamespace: databases
  commonMetadata:
    labels:
      app.kubernetes.io/name: *app
  dependsOn:
    - name: cloudnative-pg
  interval: 15m
  retryInterval: 1m  
  timeout: 2m
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./kubernetes/main/apps/databases/cloudnative-pg/cluster
```

This black magic!

```yaml
        initContainers:
          init-db:
            image:
              repository: ghcr.io/onedr0p/postgres-init
              tag: 16
```

https://github.com/onedr0p/containers/pkgs/container/postgres-init

```
318e-4db5-a282-26b70a1789ea","error":"while updating database owner password: wrong username 'thaynes' in secret, expected 'postgres'","stacktrace":"sigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).reconcileHandler\n\tpkg/mod/sigs.k8s.io/controller-runtime@v0.18.4/pkg/internal/controller/controller.go:324\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).processNextWorkItem\n\tpkg/mod/sigs.k8s.io/controller-runtime@v0.18.4/pkg/internal/controller/controller.go:261\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).Start.func2.2\n\tpkg/mod/sigs.k8s.io/controller-runtime@v0.18.4/pkg/internal/controller/controller.go:222"}
{"level":"error","ts":"2024-10-02T22:57:16-04:00","msg":"Reconciler error","controller":"cluster","controllerGroup":"postgresql.cnpg.io","controllerKind":"Cluster","Cluster":{"name":"postgres16","namespace":"databases"},"namespace":"databases","name":"postgres16","reconcileID":"293dd293-b1d1-4a2b-9a73-54833c04edc1","error":"while updating database owner password: wrong username 'thaynes' in secret, expected 'postgres'","stacktrace":"sigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).reconcileHandler\n\tpkg/mod/sigs.k8s.io/controller-runtime@v0.18.4/pkg/internal/controller/controller.go:324\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).processNextWorkItem\n\tpkg/mod/sigs.k8s.io/controller-runtime@v0.18.4/pkg/internal/controller/controller.go:261\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).Start.func2.2\n\tpkg/mod/sigs.k8s.io/controller-runtime@v0.18.4/pkg/internal/controller/controller.go:222"}
{"level":"info","ts":"2024-10-02T22:57:17-04:00","logger":"barman-cloud-check-wal-archive","msg":"2024-10-02 22:57:17,271 [58] ERROR: Barman cloud WAL archive check exception: An error occurred (301) when calling the HeadBucket operation: Moved Permanently","pipe":"stderr","logging_pod":
```

WOAH GRAFANA https://cloudnative-pg.io/documentation/1.22/quickstart/