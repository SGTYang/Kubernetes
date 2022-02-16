# CLI에 Docker container run을 하면 REST방식으로(POST /vX.X/containers/creaate HTTP/1.1) 도커데몬(외부에서 API를 받아 도커 엔진의 기능을 수행)에 넘겨주면 도커데몬은 GRPC방식으로 containerd로 넘겨주게 된다. 그럼 이를 shim은 runc로 넘겨줘 요청을 수행(컨테이너생성)
