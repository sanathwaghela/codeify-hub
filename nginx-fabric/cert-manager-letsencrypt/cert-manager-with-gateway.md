### Helm value 
this should be enable to use cert manager for http01  challenge by lets encrypt

```yaml
config:
  apiVersion: controller.config.cert-manager.io/v1alpha1
  kind: ControllerConfiguration
  enableGatewayAPI: true
```

or by command 

```bash
helm upgrade --install cert-manager oci://quay.io/jetstack/charts/cert-manager --namespace cert-manager \
  --set config.apiVersion="controller.config.cert-manager.io/v1alpha1" \
  --set config.kind="ControllerConfiguration" \
  --set config.enableGatewayAPI=true
```

This is important because some of the cert-manager components only perform the Gateway API check on startup. You can restart cert-manager with the following command:

```bash
kubectl rollout restart deployment cert-manager -n cert-manager
```

docs http01 challenge :- https://cert-manager.io/docs/configuration/acme/http01/

docs gateway to use cert manager cert :- https://cert-manager.io/docs/usage/gateway/

currently http01 challange is creating problem with me here is alternative we will use dns01 challenge


Important wildcard domain cert only issued by dns01 not possible with http verification

