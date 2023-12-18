## Grafana/Prometheus 설치 이슈

* 발생현상
  * k8s 설치 스크립트를 이용하여 Grafana 와 Prometheus 관련 Pod Deploy 를 진행함.
  * 이후 Grafana 에 접속하여 미리 설정한 여러 모니터링 정보들을 확인하려고 보였으나 데이터가 모두 NaN 으로 표시되며 보여지지 않음.


* 문제파악하기
  * Grafana 사이트에 제대로 접속되는 것을 보아 Grafana 설정에 문제가 없다고 판단.
  * 또한 Grafana DataSource 설정에 등록된 Prometheus 탭에서 Test 버튼으로 확인한 결과 Prometheus도 정상적으로 연결이 되었음을 확인함.
  * Prometheus 내부적으로 데이터를 가져오는 로직에서 문제가 있다고 판단.
  * kubectl 을 통해 Prometheus-dashboard 를 포트포워딩 하여 확인해봄

* 원인 파악
  * Prometheus 대쉬보드 내부에 'prometheus Error fetching server time' 문구를 확인.
  * 서버에서 timedatectl 를 사용하여 확인하여 보니 'System clock synchronized: no' 로 확인됨.
  * 해당 설정은 Linux 에서 기본 설치되는 'chronyd' 서비스가 정상동작하지 않았던 것.
  * Prometheus 는 기본적으로 시계열 단위로 데이터를 수집하기에 정확한 시간처리가 중요함.
  * chronyd 서비스를 사용하여 NTP (Network Time Protocol) 서버와 통신하여 시계를 동기화가 필요했던것.


* 문제해결
```shell
    timedatectl set-ntp true ## ntp 클라이언트 활성화
    systemctl restart chronyd.service ## chronyd 서비스 재시작
```

