# k8s Trouble-Shooting

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

# 이슈사항

1. Vagrant 를 이용하여 설치후 전체 Pods가 제대로 실행되지 않던 현상. [link](https://github.com/hae-gun/trouble-shooting-k8s/blob/main/issue/issue1.md)
2. 