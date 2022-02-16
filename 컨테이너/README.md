# CLI에 Docker container run 했을때

<img width="668" alt="스크린샷 2022-02-16 오후 5 40 08" src="https://user-images.githubusercontent.com/42399580/154227043-6783c37b-6efe-463d-b4b0-b8e349c8bcaa.png">

REST방식으로(POST /vX.X/containers/creaate HTTP/1.1) 도커데몬(외부에서 API를 받아 도커 엔진의 기능을 수행)에 넘겨주면 도커데몬은 GRPC방식으로 containerd로 넘겨주게 된다. 
그럼 이를 shim은 runc로 넘겨줘 요청을 수행(컨테이너생성)
