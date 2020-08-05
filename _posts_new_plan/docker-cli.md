


## 도커 실행중인 전체 컨테이너 로그 경로 
```bash
$ docker ps -qa | xargs docker inspect --format='{{.LogPath}}' | xargs ls -hl
```  

## 특정 컨테이너 로그 경로
```bash
$ docker inspect <컨테이너 아이디> | grep log

$ docker inspect <컨테이너 아이디> --format='{{.LogPath}}'
```