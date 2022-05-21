### [#2799](https://github.com/kubernetes/enhancements/issues/2799) Reduction of Secret-based Service Account Tokens

**Stage:** Graduating to Beta
**Feature group:** auth
**Feature gate:** `LegacyServiceAccountTokenNoAutoGeneration` **Default value:** `true`

Now that the [TokenRequest API](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/) has been stable since [Kubernetes 1.22](https://sysdig.com/blog/kubernetes-1-22-whats-new/#542), it is time to do some cleaning and promote the use of this API over the old tokens.

Up until now, Kubernetes automatically created a service account `Secret` when creating a `Pod`. That token `Secret` contained the credentials for accessing the API.

Now, API credentials are obtained directly through the TokenRequest API, and are mounted into Pods using a projected volume. Also, these tokens will be automatically invalidated when their associated Pod is deleted. You can still create token secrets manually if you need it.

The features to track and clean up existing tokens will be added in future versions of Kubernetes.



k8s1.24创建serviceaccount后不会自动创建secret来保存token，需要手动创建或者用TokenRequest api获取token

```
apiVersion: v1
kind: Secret
metadata:
  name: dash-admin-secret
  annotations:
    kubernetes.io/service-account.name: dash-admin
type: kubernetes.io/service-account-token
```

安装dashboard 添加账号密码登录

https://github.com/kubernetes/dashboard


