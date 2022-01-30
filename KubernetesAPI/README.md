# Kubernetes API

kubectl CLI툴을 사용하지 않고 직접 쿠버네티스 API를 호출

api주소 먼저 확인

  kubectl cluster-info
 
그 주소로 접속하면

  curl https://172.30.4.91:6443

403 status code발생 403 status는 API 서버로부터 적절한 접근권한이 없을때 발생
쿠버네티스 API서버는 기본적으로 사용자 인증을 진행

일반적으로 서버의 Security를 설정하기 위해 JWT를 사용할 수 있습니다. JWT란 서버가 서명한 JSON object로써, JWT를 가지고 있다는 말은 서버가 인증한 비밀번호 같은 것을 가지고 있다는 것을 의미합니다

JWT를 이용하여 서버에 호출하는 방법은 “Authorization” 헤더에 JWT 값을 넣으면 됩니다.

curl -H "Authorization: Bearer $TOKEN" <서버IP>:<포트>

Kubernetes api를 위한 JWT Token은 ServiceAccount에 있습니다. 
쿠버네티스의 ServiceAccount 리소스는 API 호출을 위한 JWT 토큰값을 저장합니다

쿠버네티스는 default라는 이름의 ServiceAccount를 제공하기 때문에 default ServiceAccount에 있는 Secret의 JWT을 이용합니다.

  TOKEN=$(kubectl get secret $(kubectl get sa default \
      -ojsonpath="{.secrets[0].name}") \
      -ojsonpath="{.data.token}" | base64 -d)
  echo $TOKEN
  
 default ServiceAccount에 연결된 Secret속에 들어있는 data.token값을 가져와서 base64로 디코딩해 TOKEN에 저장합니다.
 
  curl -k -H "Authorization: Bearer $TOKEN" https://172.30.4.91:6443
  
 하면 message 부분에 system:serviceaccount:default:default로 바뀌게 됩니다.
 
 사용자 인증은 받았지만 아직 default에는 API를 호출할 수 있는 권한이 없습니다.


default라는 사용자(ServiceAccount)에 적절한 권한을 부여해 다시 시도 해봅니다.( cluster-admin 권한을 주는것은 운영환경에서는 매우 위험한 일입니다. )

  kubectl create clusterrolebinding default-cluster-admin --clusterrole cluster-admin --serviceaccount default:default
  clusterrolebinding.rbac.authorization.k8s.io/default-cluster-admin created
  

 다시 " curl -k -H "Authorization: Bearer $TOKEN" https://172.30.4.91:6443 "를 실행하면 API를 성공적으로 호출할 수 있습니다.
 
