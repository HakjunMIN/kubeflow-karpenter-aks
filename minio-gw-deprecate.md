# MinIO Gateway Deprecate로 인한 대체 아키첵처

## MinIO에서 사용되는 PVC를 Blobfuse Storage Class를 사용하도록 변경

> [!Note]
> Blobfuse?
> BlobFuse는 Azure 블롭 스토리지용 가상 파일 시스템 드라이버입니다. BlobFuse를 사용하여 Linux 파일 시스템을 통해 기존 Azure 블록 블롭 데이터에 액세스하세요. 

### blobfuse 스토리지 클래스 사용을 위한 aks 기능 설정

```bash
az aks update --resource-group <resource-group> --name <cluster>  --enable-blob-driver
```

### 생성된 sc확인
```bash
k get sc
NAME                     PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
azureblob-fuse-premium   blob.csi.azure.com             Delete          Immediate              true                   2d1h
azureblob-nfs-premium    blob.csi.azure.com             Delete          Immediate              true                   2d1h
azurefile                file.csi.azure.com             Delete          Immediate              true                   14d
azurefile-csi            file.csi.azure.com             Delete          Immediate              true                   14d
azurefile-csi-premium    file.csi.azure.com             Delete          Immediate              true                   14d
azurefile-premium        file.csi.azure.com             Delete          Immediate              true                   14d
default (default)        disk.csi.azure.com             Delete          WaitForFirstConsumer   true                   14d
local-storage            kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  2d18h
managed                  disk.csi.azure.com             Delete          WaitForFirstConsumer   true                   14d
managed-csi              disk.csi.azure.com             Delete          WaitForFirstConsumer   true                   14d
managed-csi-premium      disk.csi.azure.com             Delete          WaitForFirstConsumer   true                   14d
managed-premium          disk.csi.azure.com             Delete          WaitForFirstConsumer   true                   14d
```

### minio pvc 수정

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: azureblob-fuse-premium # Here
  resources:
    requests:
      storage: 20Gi

```

### minio deployment 수정

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  labels:
    app: minio
spec:
  selector:
    matchLabels:
      app: minio
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - args:
        - server  #here
        - /data   #here, 
        env: # 아래 정보도 반드시 필요
        - name: MINIO_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: mlpipeline-minio-artifact
              key: accesskey
        - name: MINIO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: mlpipeline-minio-artifact
              key: secretkey
        image: gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance
        name: minio
        ports:
        - containerPort: 9000
        volumeMounts:
        - mountPath: /data
          name: data
          subPath: minio
        resources:
          requests:
            cpu: 20m
            memory: 100Mi
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: minio-pvc

```

### minio pvc 및 deployment삭제 후 재 적용
```bash
k delete pvc minio-pvc
k delete deploy minio     
cd manifest
kustomize build tls | kubectl apply -f -
```

### Storage Account에서 mlpipeline 디렉토리 추가

* 클러스터가 속해있는 리소스 그룹(일반적으로 `MC_<resource-group>_<cluster-name>-<region>`)에서 `fusecd1XXXXX` 스토리지 어카운트 확인
* 컨테이너에서 `pvc-XXX` 컨테이너가 생성되어 있는지 확인

* 아래 명령어로 mlpipeline디렉토리 생성

```bash
echo "Your text content" > text.txt
az storage blob upload --account-name fusecd1457843dba4c80bb8 --container-name pvc-47dc7e52-cace-468a-b868-55c8e7ac1eca --name minio/mlpipeline/test.txt --file text.txt
```

### 테스트

* Pipeline 구동 후 artifact들이 저장되었는지 확인

```bash
az storage blob list --account-name fusecd1457843dba4c80bb8 --container-name pvc-47dc7e52-cace-468a-b868-55c8e7ac1eca --output table
```