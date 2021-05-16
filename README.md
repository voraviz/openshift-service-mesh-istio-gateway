# Istio Gateway with OpenShift Route
- Deploy application to data plane
- Create project
  ```bash
  oc new-project data-plane --display-name="Data Plane"
  ```
- Deploy frontend and backend app
  ```bash
  oc create -f app.yaml -n data-plane
  watch -d oc get pods -n data-plane
  ```
- Test with cURL
  ```bash
  curl $(oc get route frontend -n data-plane -o jsonpath='{.spec.host}')
  ```
  Output
  ```bash
  Frontend version: v1 => [Backend: http://backend:8080, Response: 200, Body: Backend version:v1, Response:200, Host:backend-v1-6b4dd76bbc-bd4jx, Status:200, Message: Hello, Quarkus]
  ```

- Create control plane and join data plane to control plane
  - Create namespace for control plane
    ```bash
    oc new-project control-plane --display-name="Control Plane"
    ```
  - Create control plane
    ```bash
    oc create -f https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/smcp.yaml -n control-plane
    ```
  - Wait couple of minutes for creating control plane
    ```bash
    watch -d oc get smcp -n control-plane
    ```
  - Join data-plane namespace into control-plane
    ```bash
    oc create -f https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/member-roll.yaml -n control-plane
    ```
  - Check Service Mesh Member Roll status
    ```bash
    oc describe smmr/default -n control-plane
    ```
- Inject sidecar to both frontend and backend
  ```bash
  oc patch deployment/frontend-v1 -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n data-plane
  oc patch deployment/backend-v1 -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n data-plane
  ```
- Check that sidecar is injected
  ```bash
  oc get pods -n data-plane
  ```
- Check network policy
  ```bash
  oc get networkpolicy/istio-expose-route-basic-install -n data-plane -o yaml
  ```
- Network policy istio-expose-route
  ```yaml
  spec:
    ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            network.openshift.io/policy-group: ingress
    podSelector:
      matchLabels:
        maistra.io/expose-route: "true"
    policyTypes:
    - Ingress
  ```
- Patch frontend with label
  ```bash
  oc patch deployment/frontend-v1 -p '{"spec":{"template":{"metadata":{"labels":{"maistra.io/expose-route":"true"}}}}}' -n data-plane
  ```
- Test
  ```bash
  curl $(oc get route frontend -n data-plane -o jsonpath='{.spec.host}')
  # Sample output
  Frontend version: v1 => [Backend: http://backend:8080, Response: 200, Body: Backend version:v1, Response:200, Host:backend-v1-5c45fb5d76-gg8sc, Status:200, Message: Hello, Quarkus]
  ```
## Istio Ingress Gateway

**Remark: replace SUBDOMAIN with your cluster's subdomain.**

- Create gateway in control-plane
```bash
SUBDOMAIN=$(oc whoami --show-console  | awk -F'apps.' '{print $2}')
curl -s https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/wildcard-gateway.yaml  | sed 's/SUBDOMAIN/'$SUBDOMAIN'/' | oc create -n control-plane -f -
```
<!-- oc create -f https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/wildcard-gateway.yaml -n control-plane -->

- Create route in control-plane
```bash
SUBDOMAIN=$(oc whoami --show-console  | awk -F'apps.' '{print $2}')
curl -s https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/frontend-route-istio.yaml | sed 's/SUBDOMAIN/'$SUBDOMAIN'/' | oc create -n control-plane -f -
```
- Crate virtual service in data-plane
```bash
SUBDOMAIN=$(oc whoami --show-console  | awk -F'apps.' '{print $2}')
curl -s https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/frontend-virtual-service.yaml | sed 's/SUBDOMAIN/'$SUBDOMAIN'/' | oc create -n data-plane -f -
```
- Test gateway
```bash
curl $(oc get route/frontend -n control-plane -o jsonpath='{.spec.host}')
```
### Canary Deployment
Deploy frontend v2 and configure canary deployment to route only request from Firefox to v2
- Deploy frontend-v2
```bash
oc create -f https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/frontend-v2-deployment.yaml -n data-plane
```
- Create Destination for frontend v1 and v2
```bash
oc apply -f https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/frontend-destination-rule.yaml -n data-plane
```
- Update frontend virtual service with canary deployment configuration
```bash
curl -s https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/frontend-virtual-service-canary.yaml |sed 's/SUBDOMAIN/'$SUBDOMAIN'/'|oc apply -n data-plane -f -
```
- Test
  - Set User-Agent to Firefox
    ```bash
    #Test with header User-Agent contains "Firefox"
    curl -H "User-Agent:Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:78.0) Gecko/20100101 Firefox/78.0" $(oc get route frontend -n control-plane -o jsonpath='{.spec.host}')
    #You will get response from frontend-v2
    Frontend version: v2 => [Backend: http://backend:8080, Response: 200, Body: Backend version:v1, Response:200, Host:backend-v1-5c45fb5d76-gg8sc, Status:200, Message: Hello, Quarkus]
    ```
  - Set User-Agent to Chrome
    ```bash
    curl -H "User-Agent:Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36 Edg/79.0.309.71" $(oc get route frontend -n control-plane -o jsonpath='{.spec.host}')
    #You will get response from frontend-v1
    Frontend version: v1 => [Backend: http://backend:8080, Response: 200, Body: Backend version:v1, Response:200, Host:backend-v1-5c45fb5d76-gg8sc, Status:200, Message: Hello, Quarkus]
    ```
# Secure with TLS
## Service Mesh v2
- Create cretificate and secret key
  ```bash
  bin/create-certificate.sh
  ```
- Create secrets
  ```bash
  oc create secret generic frontend-credential \
  --from-file=tls.key=certs/frontend.key \
  --from-file=tls.crt=certs/frontend.crt \
  -n control-plane
  ```
- Update gateway with TLS
  ```bash
  curl -s https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/wildcard-gateway-tls.yaml|sed 's/SUBDOMAIN/'$SUBDOMAIN'/' | oc apply -n control-plane -f -
  ```
- Update frontend router to passthrough
  ```bash
   curl -s https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/frontend-route-istio-passthrough.yaml |sed 's/SUBDOMAIN/'$SUBDOMAIN'/' | oc apply -n control-plane -f -
  ```
- Test
  ```bash
  curl -vk https://$(oc get route frontend -n control-plane -o jsonpath='{.spec.host}')
  ```