# Baseline Performance Test Setup

# Configure an App

## deploy client
```bash
kubectl apply -k client/base
```

Wait for the client deployment to be ready:
```bash
kubectl wait --for=condition=available deployment/sleep -n client
```

## deploy httpbin
```bash
kubectl apply -k httpbin/base
```

Wait for the httpbin deployment to be ready:
```bash
kubectl wait --for=condition=available deployment/httpbin -n httpbin
```

## curl the httpbin service from the sleep pod
```bash
kubectl exec deploy/sleep -n client -c sleep -- curl -s httpbin.httpbin.svc.cluster.local:8000/get
```

## remove httpbin
```bash
kubectl delete -k httpbin/base
```

# Set up the Performance Test

## deploy 50 namespace tiered-app
```bash
NUM=1 # 1, 5, 20, 30, or 50
kubectl apply -k tiered-app/${NUM}-namespace-app/base
```

## curl the tiered-app from the sleep pod
```bash
kubectl exec deploy/sleep -n client -- curl -s http://tier-1-app-a.ns-1.svc.cluster.local:8080
```

Label a node that is not running any workload pods:
```bash
NODE=ambient-control-plane
kubectl label node/${NODE} node=loadgen
```

If the node is a k8s control plane, remove the `NoSchedule` taint:
```bash
kubectl taint nodes ${NODE} node-role.kubernetes.io/control-plane:NoSchedule-
```

If you are running in a kind cluster, install the k8s metrics server:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

If you installed the metrics server, patch the metrics server to disable certificate validation:
```bash
kubectl patch -n kube-system deployment metrics-server --type=json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

## deploy 50 vegeta loadgenerators
```bash
kubectl apply -k loadgenerators/${NUM}-loadgenerators
```

## watch logs of vegeta loadgenerator
```bash
kubectl logs -l app=vegeta -f -n ns-1
```

## watch top pods
```bash
watch kubectl top pods -n ns-1
watch kubectl top pods -n kube-system --sort-by cpu
```

## collect logs
In the `experiment-data/tail-logs.sh` script change the following variables
```
# Define the range of namespaces
start_namespace=1
end_namespace=${NUM}

# Define the output file
output_file="450rps-10m-50-app-baseline-data-run-1.md"
```

Run the script to collect logs:
```
cd experiment-data && ./tail-logs.sh && cd ..
```

## example exec into vegeta to run your own test (optional)
```bash
kubectl --namespace ns-1 exec -it deploy/vegeta -c vegeta -- /bin/sh
```

test run:
```bash
echo "GET http://tier-1-app-a.ns-1.svc.cluster.local:8080" | vegeta attack -dns-ttl=0 -rate 500/1s -duration=2s | tee results.bin | vegeta report -type=text
```

## uninstall
```bash
kubectl delete -k loadgenerators/${NUM}-loadgenerators
kubectl delete -k tiered-app/${NUM}-namespace-app/base
kubectl delete -k client/base
```