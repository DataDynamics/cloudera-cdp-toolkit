# L4 Switch

## L4 Switch 연결 대상 서비스

* HUE
* Impala Coordinator
  * Impala Shell
  * JDBC/ODBC
* Cloudera Manager Server
  * User에서 UI에 대한 로드 밸런싱
  * CM Server 이중화에 따른 CM Agent에 대한 로드 밸런싱
* Ranger Admin
* Resource Manager
* NiFi

## L4 Load Balancing Rule

## HAProxy

* SW 방식의 Load Balancer
* Large Traffic 처리시 Performance Issue 있으므로 Large Traffic에서는 HW L4 Switch 사용 권장

### 빌드 및 RPM 설치

```
# dnf install -y systemd-devel pcre-devel

# git clone https://github.com/DataDynamics/HAProxy-3-RPM-builder
# cd HAProxy-3-RPM-builder
# make
...

# cd rpmbuild/RPMS/x86_64
# rpm -Uvh haproxy-3.1.7-1.el9.x86_64.rpm
```

### 기본 설정 파일

기본 `/etc/haproxy.cfg` 파일이며 이 파일은 서비스 시작시 에러가 발생함

```
global
    #log /dev/log    local0
    log 127.0.0.1 local0
    #log /dev/log    local1 notice
    log 127.0.0.1 local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/haproxy.sock mode 600 level admin
    pidfile /var/run/haproxy.pid
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    
listen stats
  bind :9000
  mode http
  stats enable
  stats realm Haproxy\ Statistics  # Title text for popup window
  stats uri /haproxy_stats

  # Default SSL material locations
  ca-base /etc/ssl/certs
  crt-base /etc/ssl/private

  # Default ciphers to use on SSL-enabled listening sockets.
  # For more information, see ciphers(1SSL).
  ssl-default-bind-ciphers kEECDH+aRSA+AES:kRSA+AES:+AES256:RC4-SHA:!kEDH:!LOW:!EXP:!MD5:!aNULL:!eNULL

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http
```

기본 설정 파일에서는 `/var/log/messages` 파일에 다음과 같이 에러 로그가 기록됨

```
Jun  3 23:24:00 localhost systemd[1]: Starting HAProxy Load Balancer...
Jun  3 23:24:00 localhost systemd[1]: haproxy.service: Control process exited, code=exited, status=1/FAILURE
Jun  3 23:24:00 localhost systemd[1]: haproxy.service: Failed with result 'exit-code'.
Jun  3 23:24:00 localhost systemd[1]: Failed to start HAProxy Load Balancer.
```

### 설정 파일 검증

```
# haproxy -c -f /etc/haproxy/haproxy.cfg
```

### systemd 서비스 파일

`/usr/lib/systemd/system/haproxy.service` 파일의 기본 구조는 다음과 같음

```
[Unit]
Description=HAProxy Load Balancer
Documentation=man:haproxy(1)
After=syslog.target network.target

[Service]
EnvironmentFile=-/etc/sysconfig/haproxy
EnvironmentFile=-/etc/sysconfig/haproxy
Environment="CONFIG=/etc/haproxy/haproxy.cfg" "PIDFILE=/run/haproxy.pid" "EXTRAOPTS=-S /run/haproxy-master.sock"
ExecStartPre=/usr/sbin/haproxy -f $CONFIG -c -q $EXTRAOPTS
ExecStart=/usr/sbin/haproxy -Ws -f $CONFIG -p $PIDFILE $EXTRAOPTS
ExecReload=/usr/sbin/haproxy -f $CONFIG -c -q $EXTRAOPTS
ExecReload=/bin/kill -USR2 $MAINPID
KillMode=mixed
Restart=always
SuccessExitStatus=143
Type=notify

# The following lines leverage SystemD's sandboxing options to provide
# defense in depth protection at the expense of restricting some flexibility
# in your setup (e.g. placement of your configuration files) or possibly
# reduced performance. See systemd.service(5) and systemd.exec(5) for further
# information.

# NoNewPrivileges=true
# ProtectHome=true
# If you want to use 'ProtectSystem=strict' you should whitelist the PIDFILE,
# any state files and any other files written using 'ReadWritePaths' or
# 'RuntimeDirectory'.
# ProtectSystem=true
# ProtectKernelTunables=true
# ProtectKernelModules=true
# ProtectControlGroups=true
# If your SystemD version supports them, you can add: @reboot, @swap, @sync
# SystemCallFilter=~@cpu-emulation @keyring @module @obsolete @raw-io

[Install]
WantedBy=multi-user.target
```

### Statistics Web Admin 활성화

기본 `/etc/haproxy.cfg` 파일에 다음을 추가하고 HAProxy를 재시작하도록 하며 http://<haproxy-ip>:8404/stats 접속을 통해 HAProxy의 로드 밸런싱 처리 현황을 확인할 수 있음

```
listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 10s
    stats realm HAProxy\ Statistics
    stats auth admin:admin123     # 사용자/비밀번호
    stats admin if TRUE           # Enable web admin
    timeout connect 5000
    timeout client  5000
    timeout server  5000
```
