# OS

## SELinux

Hadoop Cluster를 구성하는 모든 노드는 SELinux를 영구히 disable 해야함

### SELinux의 일시 Disable

```
# getenforce
Enforcing

# setenforce 0
# getenforce
Permissive
```

### 영구 Disable

```
# vi /etc/selinux/config
SELINUX=disabled

# reboot
```

* Enforcing: 정책이 강제로 적용됨.
* Permissive: 정책 위반을 허용하지만 로그는 남김 (audit log).
* Disabled: 커널 수준에서 완전히 비활성화됨 (재부팅 필요).
