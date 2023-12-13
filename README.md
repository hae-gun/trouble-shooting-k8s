# K8S Trouble-Shooting

## k8s cmd 키워드 정리

```shell
# 컨테이너 목록 확인
$ kubelet get pods -A

# 노드 확인
$ kubelet get node

# 컨테이너 환경 로그 확인
$ kubelet logs <name> -n <namesapce>

# 컨테이너 상태확인
$ kubelet describe pod <name> -n <namesapce>
 
```