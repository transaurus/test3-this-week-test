# 使用 Ingress 公開 Chaos Dashboard

有時您可能需要讓外部使用者存取 Chaos Dashboard，同時將其置於現有監控儀表板的子路徑下。

以下展示如何在路徑 `/chaos-mesh` 下公開 Chaos Dashboard 的範例：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-chaos-dashboard-under-subpath
  namespace: chaos-mesh
  annotations:
    nginx.ingress.kubernetes.io/use-regex: 'true'
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    nginx.ingress.kubernetes.io/configuration-snippet: |
      sub_filter '<head>' '<head> <base href="/chaos-mesh/">';
spec:
  rules:
    - http:
        paths:
          - path: /chaos-mesh/?(.*)
            pathType: Prefix
            backend:
              service:
                name: chaos-dashboard
                port:
                  number: 2333
```

您也可在 https://github.com/chaos-mesh/chaos-mesh/blob/master/examples/dashboard/ingress-subpath.yaml 找到此範例。