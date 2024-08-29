# 외부 망 Vnet 분리 아키텍처

## Application Gateway 생성

## Kubeflow Istio Ingress GW 내부IP변경 (ClusterIP가 아님)

https://github.com/HakjunMIN/kubeflow-karpenter-aks/blob/main/manifests/common/istio-1-17/istio-install/base/patches/service.yaml#L7

```yaml
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
namespace: istio-system
annotations:
    service.beta.kubernetes.io/azure-load-balancer-ipv4: 10.240.0.25 (이건 넣으면 이 ip로 지정되고 안넣으면 서브넷에 있는 IP할당받음)
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"

spec:
  type: LoadBalancer

```

## http로 변경

https://github.com/HakjunMIN/kubeflow-karpenter-aks/blob/main/manifests/tls/kf-istio-resources.yaml

* `tls.httpRedirect:true`부분 삭제
* tls인증서 부분삭제
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: kubeflow-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"

```
* `certificate` 오브젝트 삭제

## 발급 인증서를 Applicatation Gateway에 등록

* 등록 시 pfx파일로 변환해야 함. 키볼트에 등록 후 연계해서 사용가능.
```bash
openssl pkcs12 -export -out ./cert.pfx -in ./kubeflow.andrewmin.net/cert1.pem -inkey ./kubeflow.andrewmin.net/privkey1.pem
```

