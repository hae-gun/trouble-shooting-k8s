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


# 복습현황
---
1. 컨테이너 한방 정리 [link](https://webheck.tistory.com/entry/k8s-%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88-%ED%95%9C%EB%B0%A9-%EC%A0%95%EB%A6%AC)
2. 쿠버네티스 빠르게 설치하기 [link](https://webheck.tistory.com/entry/k8s-%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EB%B9%A0%EB%A5%B4%EA%B2%8C-%EC%84%A4%EC%B9%98%ED%95%98%EA%B8%B0)
3. 실무에서 느껴본 쿠버네티스가 정말 편한 이유 [link](https://webheck.tistory.com/entry/k8s-%EC%8B%A4%EB%AC%B4%EC%97%90%EC%84%9C-%EB%8A%90%EA%BB%B4%EB%B3%B8-%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4%EA%B0%80-%EC%A0%95%EB%A7%90-%ED%8E%B8%ED%95%9C-%EC%9D%B4%EC%9C%A0)
4. Object 그려보며 이해하기 [link](##)
---