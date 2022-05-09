
# 서비스 어카운트 생성
kubectl create serviceaccount pod-pod
# 롤 생성
kubectl create role creating-pod --verb=create --resource=pod
# 롤 바인딩 해주기
kubectl create rolebinding creating-pod-bind --role=creating-pod --serviceaccount=default:pod-pod

# pod 하나 생성

cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pod-in-pod
spec:
  serviceAccountName: pod-pod
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
EOF


# 생성한 pod에서 다른 pod 생성
kubectl exec -ti boot -c nginx -- bash -c '''
curl -s \
  --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  -H "Content-Type: application/json" \
  -d "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"name\":\"nginx\"},\"spec\":{\"containers\":[{\"name\":\"nginx\",\"image\":\"nginx\",\"ports\":[{\"containerPort\":80}]}]}}" \
  https://kubernetes.default.svc/api/v1/namespaces/$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)/pods
'''
