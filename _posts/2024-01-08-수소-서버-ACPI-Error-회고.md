---
title: "수소 서버 ACPI Error 회고"
date: 2024-01-08 23:38:00 +09:00
categories: [회고, infra, ACPI]
author: wlgns12370
tags: [회고, infra, ACPI]
---

GET-P 24년도 서비스 개편을 위해 기존의 코드를 레거시하고 새 Spring Project를 만들다 발생한 일입니다. 

여느때와 다름없이 수소 서버 code-server를 사용해 작업하던중, Spring Project Initialize가 안되었습니다. 원인을 찾기 위해서 트러블 슈팅을 하던 중, 메모리 용량이 초과 되었다는 사실을 알게 되었습니다.

## 메모리 용량 초과

---

- 디스크 조회 명령어

```bash
df -h // disk filesystem -human-readable
```

위 명령어를 사용하면, 디스크의 용량을 사람이 읽기 쉬운 형식으로 간단하게 조회할 수 있습니다.

```bash
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           1.6G  2.0M  1.6G   1% /run
/dev/sda2       228G  216G   0G  100% /
tmpfs           7.8G     0  7.8G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/sda1       1.1G  6.1M  1.1G   1% /boot/efi
tmpfs           1.6G  4.0K  1.6G   1% /run/user/1000
```

서버의 두 번째 파티션(`/dev/sda2`)에서 Use가 100%를 차지하고 있었습니다. 이후 로그 백업 파일이 많이 생성된 것을 파악했습니다.

![Untitled](https://github.com/Principes-Artis-Mechanicae/Principes-Artis-Mechanicae.github.io/assets/30788586/4e712f1e-b4c3-4abb-857a-52ed5f52365c)

로그 파일들의 총용량은 `150GB` 였고, 가장 큰 용량을 차지하고 있었던 `kern.log` 파일을 열어 보았습니다.

- kern.log 파일 조회

![캡처](https://github.com/Principes-Artis-Mechanicae/Principes-Artis-Mechanicae.github.io/assets/30788586/8a7bba2d-fe0d-414c-9758-73019825fae9)

결과는 `ACPI Error`가 발생하여 `1초`에 `100건` 정도의 로그가 계속 발생하고 있었습니다. 이후 에러를 해석해보니 **BIOS 및 메인보드 펌웨어 업데이트 이슈**라는 것을 알게 되었습니다. 하지만 **2월**까지 하드웨어 업데이트, 성능 개량을 할 수 없었습니다.

> 💡 **ACPI(Adavanced Configuration and Power Interface) : 미국 인텔과 마이크로소프트가 공동으로 프로젝트하여 만든 인터페이스로, 운영 체제와 하드웨어 간의 상호 작용을 관리하는 역할을 하는 국제 표준입니다.**
> 

- /etc/logrotate.d/~

```bash
{
        rotate 1
        weekly
        missingok
        notifempty
        compress
        delaycompress
        sharedscripts
        postrotate
                /usr/lib/rsyslog/rsyslog-rotate
        endscript
}
```

임시방편으로 **`logrotate` 순환점을 1로 세팅**하여 **log의 용량을 낮추었습니다.** 메인 배포 서버가 아닌 작업용 서버이기 때문에, 임시방편으로 해결할 수 있었지만, 배포 서버에 관제 및 지속적인 상태 관리의 필요성을 크게 깨닳은 경험이었습니다.