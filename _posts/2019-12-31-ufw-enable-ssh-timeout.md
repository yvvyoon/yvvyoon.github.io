---
title: 'ufw enable 후 ssh 접속 불가 문제'
date: 2019-12-31 16:01:00 -0400
categories: ubuntu
---

Ubuntu의 방화벽인 `ufw`를 활성화하면서 ssh 접속이 끊어질 수 있다는 메시지에 y라고 응답했었다.

```
Command may disrupt existing ssh connections. Proceed with operation (y|n)?
Firewall is active and enabled on system startup
```



당시에는 아무 이상이 없었으나 점심먹고 와서 당연하게 끊어진 EC2를 다시 접속해보니 ssh 타임아웃 에러가 나면서 접속이 되지 않는다. 인스턴스 재부팅도 안 먹고, 인바운드 재설정도 먹지 않는다.

```
ssh: connect to host 52.79.237.12 port 22: Operation timed out
```



구글링해보니 나 말고도 종종 이런 실수를 하는 사람들이 있는 것 같다. 다행히 나보다 먼저 경험한 사람들이 많다. 해결 방법은 아래 단계를 따르면 된다.

---

1. **신규 인스턴스 생성**

2. **기존 인스턴스 중지**

3. **기존 인스턴스에서 볼륨 해제**

    정상적으로 해제되면 상태 탭에 `in-use`에서 `available`로 바뀐다.

    <img width="511" alt="image" src="https://user-images.githubusercontent.com/12066892/71610714-b9e4d980-2bd6-11ea-93b6-f2a901e95c40.png">

4. **해제한 볼륨을 신규 인스턴스에 연결**

    <img width="1137" alt="image" src="https://user-images.githubusercontent.com/12066892/71610806-82c2f800-2bd7-11ea-9608-0cb20175c509.png">

    연결이 잘 되었다면 인스턴스 상세의 블록 디바이스 내역에 새로 추가된 것을 확인할 수 있다.

    <img width="214" alt="image" src="https://user-images.githubusercontent.com/12066892/71610826-aede7900-2bd7-11ea-99a3-a9c127d26939.png">

5. **신규 인스턴스에서 SSH로 접속**

6. **연결된 볼륨 확인**

    ```
    $ sudo lsblk
    
    NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
    loop0     7:0    0  89M  1 loop /snap/core/7713
    loop1     7:1    0  18M  1 loop /snap/amazon-ssm-agent/1480
    xvda    202:0    0   8G  0 disk
    └─xvda1 202:1    0   8G  0 part /
    xvdf    202:80   0   8G  0 disk
    └─xvdf1 202:81   0   8G  0 part
    ```

7. **새로 연결한 볼륨 마운트**

    ```
    $ mkdir mnt
    $ sudo mount /dev/xvdf1 ./mnt
    ```

8. **ufw 설정 파일 수정**

    ```
    $ cd mnt/etc/ufw
    $ vim ufw.conf
    ```

    `ENABLED`의 값을 `no`로 수정하고 빠져나온다.

9. **마운트했던 볼륨을 다시 언마운트**

    언마운트했기 때문에 mnt 디렉토리를 열어보면 아무것도 없다.

    ```
    $ cd
    $ sudo umount ./mnt
    ```

10. **기존 인스턴스에 볼륨 재연결**

    인스턴스 선택 후 디바이스 선택할 때 자동으로 `/dev/sdf`로 잡힐 것이다. 그러나 인스턴스는 루트 디바이스가 무조건 있어야 하기 때문에 `/dev/sda1`으로 설정해야 한다.

11. **기존 인스턴스 SSH 접속 확인**

    인스턴스를 중지했다가 다시 시작했기 때문에 IP는 바뀌고, 정상적으로 접속된다.

    

> https://intellipaat.com/community/8740/locked-myself-out-of-ssh-with-ufw-in-ec2-aws

