# Istio Gateway with OpenShift Route
- Deploy application to data plane
```bash
#Create project (namespace)
oc new-project data-plane --display-name="Data Plane"
#Deploy frontend and backend app
oc create -f app.yaml -n data-plane
#Test with cURL
curl $(oc get route frontend -n data-plane -o jsonpath='{.spec.host}')
#Sample output
Frontend version: v1 => [Backend: http://backend:8080, Response: 200, Body: Backend version:v1, Response:200, Host:backend-v1-6b4dd76bbc-bd4jx, Status:200, Message: Hello, Quarkus]

```

- Create control plane and join 
```bash
#Create namespace for control plane
oc new-project control-plane --display-name="Control Plane"

#Create control plane
oc create -f https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/basic-install.yaml -n control-plane

#Wait couple of minutes for creating control plane's pods. Total number of pods is 12
watch oc get pods -n control-plane

#Join data-plane namespace into control-plane
oc create -f https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/member-roll.yaml -n control-plane
 
```
- Inject sidecar
```bash
#Inject sidecars to frontend and backend deployment
oc patch deployment/frontend-v1 -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n data-plane
oc patch deployment/backend-v1 -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n data-plane
```
- Check network policy
```bash
oc get networkpolicy/istio-expose-route -n data-plan -o yaml
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
oc patch deployment/frontend -p '{"spec":{"template":{"metadata":{"labels":{"maistra.io/expose-route":"true"}}}}}' -n data-plane
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
oc create -f https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/wildcard-gateway.yaml -n control-plane
```
- Create route in control-plane
```bash
oc create -f https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/frontend-route-istio.yaml -n control-plane
```
- Crate virtual service in data-plane
```bash
oc create -f https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/frontend-virtual-service.yaml -n data-plane
```
### Canary Deployment
- Deploy frontend-v2
```bash
#Deploy frontend-v2
oc create -f https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/frontend-v2-deployment.yaml -n data-plane
```
- Destination rule and virtual service
```bash
#Create Destination Rule
oc apply -f https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/frontend-destination-rule.yaml -n data-plane
#Replace virtual service
oc apply https://raw.githubusercontent.com/voraviz/openshift-service-mesh-istio-gateway/main/frontend-virtual-service-canary.yaml -f -n data-plane
```
- Test
```bash
#Test with header User-Agent contains "Firefox"
curl -H "User-Agent:Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:78.0) Gecko/20100101 Firefox/78.0" $(oc get route frontend -n control-plane -o jsonpath='{.spec.host}')
#You will get response from frontend-v2
Frontend version: v2 => [Backend: http://backend:8080, Response: 200, Body: Backend version:v1, Response:200, Host:backend-v1-5c45fb5d76-gg8sc, Status:200, Message: Hello, Quarkus]
#Test again with header User-Agent start without "Firefox"
curl -H "User-Agent:Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36 Edg/79.0.309.71" $(oc get route frontend -n control-plane -o jsonpath='{.spec.host}')
#You will get response from frontend-v1
Frontend version: v1 => [Backend: http://backend:8080, Response: 200, Body: Backend version:v1, Response:200, Host:backend-v1-5c45fb5d76-gg8sc, Status:200, Message: Hello, Quarkus]

```