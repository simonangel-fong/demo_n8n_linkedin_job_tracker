
```sh

```

- Node: HTTP Request node: list pods in demo namespace**

Method:  GET
URL:     https://kubernetes.default.svc/api/v1/namespaces/demo/pods
Headers: Authorization = Bearer {{ $json.token }}
         Content-Type  = application/json
SSL:     disable certificate verification (self-signed cert in minikube)