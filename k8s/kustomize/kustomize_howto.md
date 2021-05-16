# kustomize 사용법 정리

## 기본 사용법

target directory에는 resource인 yaml 파일들과 kustomization.yaml 파일이 있어야 함
```shell
$ ls kustomize_example

kustomization.yaml deployment.yaml service.yaml
```

patch, compose 등의 작업 결과물을 string으로 뱉을 수도 있고, 바로 생성할 수도 있음
```shell
$ kubectl kustomize ./kustomize_example

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  ...
```

```shell
$ kubectl apply -k ./kustomize_sample

Deployment my-nginx created
```


## 쓰임의 종류

#### 1. cross-cutting field 수정 <a name="sec1"></a>

모든 resource에 같은 namespace를 써넣는다는지 등 단순 반복작업을 할 수 있음

#### 2. compose <a name="sec2"></a>

여러 resource를 하나의 파일로 구성

아래 예제는 [1번](#sec1)과 [2번](#sec2)을 보여줌
```shell
$ cat kustomize_example/kustomization.yaml

namespace: testing
namePrefix: test-
nameSuffix: "-001"
commonLabels:
  type: testing
commonAnnotations:
  editor: shellwedance
resources:
  - deployment.yaml
  - service.yaml
  
$ kubectl apply -k ./kustomize_example
service/test-my-nginx-001 created
deployment.apps/test-nginx-deployment-001 created
```

#### 3. patch <a name="sec3"></a>

kubectl patch와 같이 특정 필드값을 바꿀 수 있음

- patchesStrategicMerge 필드를 이용하여 Deployment의 replica 수, 메모리 request 등 수정 가능
  - 모든 필드를 patch하진 못함
- patchJson6902 필드를 이용하여 JSON 경로의 필드를 수정 가능
  - operation으로 replace 뿐만 아니라 add, remove, move, copy, test 등이 존재함

```shell
cat kustomize_example/kustomization.yaml

resources:
  - deployment.yaml

# Deployment의 replica 수를 바꾸는 예제
patchesStrategicMerge:
  - patch-replica.yaml

# Pod 메모리 resource를 수정하는 예제
patchesJson6902:
  - target:
      kind: Deployment
      name: my-nginx
      group: apps
      version: v1
    path: patch-resource.yaml


# patchesStrategicMerge를 통한 특정 필드값 변경
$ cat kustomize_example/patch-replica.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3

# patchesJson6902를 통해 JSON 경로의 필드값 변경
$ cat kustomize_example/patch-resource.yaml

- op: add
  path: /spec/template/spec/containers/0/resources/limits/memory
  value: 512Mi
```

#### 4. base/overlay <a name="sec4"></a>

[1번](#sec1), [2번](#sec2), [3번](#sec3)과 같이 resource를 일일이 지정하지 않고, base directory를 지정 가능
- overlay에서는 base를 여러 개 지정할 수도 있음

```shell
$ ls kustomize_example/

# base로 쓰이는 base 디렉토리와 overlay로 쓰이는 dev 디렉토리
base/ dev/ 


$ ls kustomize_example/base

kustomization.yaml deployment.yaml service.yaml

$ cat kustomize_example/base/kustomization.yaml

resources:
  - deployment.yaml
  - service.yaml
 

$ ls kustomize_example/dev/

# 적용 내용은 3번과 동일
kustomization.yaml patch-replica.yaml patch-resource.yaml


$ cat kustomize_example/dev/kustomization.yaml

bases:
  - ../base

namespace: dev-my-nginx

patchesStrategicMerge:
  - patch-replica.yaml

patchesJson6902:
  - target:
      kind: Deployment
      name: my-nginx
      group: apps
      version: v1
    path: patch-resource.yaml


# 사용법
$ kubectl apply -k ./kustomize_example/dev
```

## Reference

https://coding-start.tistory.com/388

https://github.com/levi-yo/kubernetes-sample/tree/master/kube-kustomize
