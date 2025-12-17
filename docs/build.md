# 🧩 Ubuntu 24.04 기반 OpenStack Caracal Controller 1 / Compute 1 단일 구성 구축 가이드

## 목차
- [1. 개요](#1-개요)
- [2. 전체 아키텍처](#2-전체-아키텍처)
- [3. 네트워크 설계](#3-네트워크-설계)
- [4. 기본 시스템 설정](#4-기본-시스템-설정-controller--compute-공통)
- [5. 미들웨어 구성](#5-미들웨어-구성-controller)
- [6. 인증 서비스 구성](#6-인증-서비스-구성-keystone)
- [7. 이미지 서비스](#7-이미지-서비스-glance)
- [8. Placement 서비스](#8-placement-서비스)
- [9. Compute 서비스](#9-compute-서비스-nova)
- [10. 네트워크 서비스](#10-네트워크-서비스-neutron)
- [11. 블록 스토리지](#11-블록-스토리지-cinder)
- [12. Horizon 대시보드](#12-horizon-대시보드)
- [13. 구축 검증](#13-구축-검증)
- [14. 트러블슈팅 가이드](#14-트러블슈팅-가이드)
- [15. 결론](#15-결론)

---

## 1. 개요

### 1.1 문서 목적

본 문서는 Ubuntu 24.04.03 LTS 환경에서 OpenStack Caracal을 기반으로 **Controller 1대**와 **Compute 1대**를 분리하여 단일 구성 OpenStack 환경을 구축하는 것을 목적으로 한다.

DevStack을 사용하지 않고 공식 가이드를 기반으로 핵심 컴포넌트를 수동 설치하여 OpenStack 아키텍처 이해와 실습 재현성을 확보한다.

### 1.2 구성 범위

| 노드 | 역할 |
|------|------|
| **Controller Node** | OpenStack 제어 및 관리 영역 |
| **Compute Node** | 가상 머신 실행 영역 |

### 1.3 실습 환경 및 전제조건

- **OS**: Ubuntu 24.04.03 LTS
- **OpenStack**: Caracal (Ubuntu Cloud Archive)
- **가상화 환경**: VMware Pro
- **네트워크**: Bridged
- **계정**: root (교육용 폐쇄 환경)
- **설치 방식**: 공식 가이드 기반 수동 설치

---

## 2. 전체 아키텍처

### 2.1 노드 구성

| 노드 | 역할 |
|------|------|
| **Controller** | API, 인증, 네트워크 제어, 대시보드 |
| **Compute** | 인스턴스 실행 |

### 2.2 노드별 주요 컴포넌트

#### Controller Node
- Chrony
- MariaDB
- RabbitMQ
- Memcached
- Keystone
- Glance
- Placement
- Nova (API, Scheduler, Conductor)
- Neutron (Server, L3, DHCP)
- Cinder API
- Horizon

#### Compute Node
- Chrony
- Nova Compute
- Neutron OVS Agent
- Cinder Volume (LVM)

---

## 3. 네트워크 설계

### 3.1 VMware 네트워크 구성

- VMware Bridged Network 사용
- Controller / Compute 동일 L2 네트워크 연결

### 3.2 OpenStack 네트워크 구조

- **Provider Network**: flat
- **Tenant Network**: VXLAN
- **L2 Agent**: Open vSwitch (OVS)

### 3.3 IP 주소 계획 (예시)

| 구분 | 대역 |
|------|------|
| Management Network | 192.168.0.0/24 |
| Provider Network | 192.168.0.0/24 |
| Floating IP Pool | 192.168.0.200 ~ 192.168.0.220 |

| 노드 | IP 주소 |
|------|---------|
| Controller | 192.168.0.10 |
| Compute | 192.168.0.11 |

---

## 4. 기본 시스템 설정 (Controller / Compute 공통)

### 4.1 Hostname 설정

**Controller 노드:**
```bash
hostnamectl set-hostname controller
```

**Compute 노드:**
```bash
hostnamectl set-hostname compute
```

### 4.2 /etc/hosts 설정

```bash
cat >> /etc/hosts <<EOF
192.168.0.10 controller
192.168.0.11 compute
EOF
```

### 4.3 시간 동기화 (Chrony)

```bash
apt update
apt install -y chrony
systemctl enable --now chrony
```


---

## 5. 미들웨어 구성 (Controller)

### 5.1 MariaDB 설치 및 설정

```bash
apt install -y mariadb-server python3-pymysql
```

**설정 파일 생성:**
```bash
cat > /etc/mysql/mariadb.conf.d/99-openstack.cnf <<EOF
[mysqld]
bind-address = 0.0.0.0
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
EOF
```

**서비스 재시작:**
```bash
systemctl restart mariadb
```

### 5.2 RabbitMQ

```bash
apt install -y rabbitmq-server
rabbitmqctl add_user openstack 1234
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

> **주의**: `RABBIT_PASS`를 실제 사용할 비밀번호로 변경하세요.

### 5.3 Memcached

```bash
apt install -y memcached python3-memcache
sed -i 's/-l 127.0.0.1/-l 192.168.0.10/' /etc/memcached.conf
systemctl restart memcached
```

---

## 6. 인증 서비스 구성 (Keystone)

### 6.1 DB 생성

```bash
mysql -u root <<EOF
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY '1234';
FLUSH PRIVILEGES;
EOF
```

> **주의**: `KEYSTONE_DBPASS`를 실제 사용할 비밀번호로 변경하세요.

### 6.2 Keystone 설치

```bash
apt install -y keystone apache2
```

### 6.3 Keystone 설정

**/etc/keystone/keystone.conf 편집:**
```ini
[database]
connection = mysql+pymysql://keystone:1234@controller/keystone

[token]
provider = fernet
```

**데이터베이스 초기화 및 설정:**
```bash
keystone-manage db_sync
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

keystone-manage bootstrap \
  --bootstrap-password 1234 \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

**Apache 재시작:**
```bash
systemctl restart apache2
```

### 6.4 환경 변수 설정

```bash
cat > ~/admin-openrc <<EOF
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=1234
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF

source ~/admin-openrc
>> 확인
openstack token issue
```

---

## 7. 이미지 서비스 (Glance)

### 7.1 DB 생성

```bash
mysql -u root <<EOF
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY '1234';
FLUSH PRIVILEGES;
EOF
```

### 7.2 Glance 사용자 및 서비스 생성

```bash
source ~/admin-openrc

openstack user create --domain default --password 1234 glance
openstack role add --project admin --user glance admin
openstack service create --name glance --description "OpenStack Image" image

openstack endpoint create --region RegionOne image public http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292
openstack endpoint create --region RegionOne image admin http://controller:9292

```

### 7.3 Glance 설치 및 설정

```bash
apt install -y glance
```

**/etc/glance/glance-api.conf 편집:**
```ini
[database]
connection = mysql+pymysql://glance:1234@controller/glance

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = admin        
username = glance
password = 1234

[paste_deploy]
flavor = keystone

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

```

**데이터베이스 동기화:**
```bash
glance-manage db_sync
systemctl restart glance-api
```

### 7.4 이미지 업로드 테스트

```bash
wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img

openstack image create "cirros" \
  --file cirros-0.6.2-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public

openstack image list
```

---

## 8. Placement 서비스

### 8.1 DB 생성

```bash
mysql -u root <<EOF
CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY '1234';
FLUSH PRIVILEGES;
EOF
```

### 8.2 Placement 사용자 및 서비스 생성

```bash
source ~/admin-openrc

openstack user create --domain default --password 1234 placement
openstack role add --project admin --user placement admin
openstack service create --name placement --description "Placement API" placement

openstack endpoint create --region RegionOne placement public http://controller:8778
openstack endpoint create --region RegionOne placement internal http://controller:8778
openstack endpoint create --region RegionOne placement admin http://controller:8778
```

### 8.3 Placement 설치 및 설정

```bash
apt install -y placement-api
```

**/etc/placement/placement.conf 편집:**
```ini
[placement_database]
connection = mysql+pymysql://placement:1234@controller/placement

[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = admin
username = placement
password = PLACEMENT_PASS
```

**데이터베이스 동기화:**
```bash
placement-manage db sync
systemctl restart apache2
```

---

## 9. Compute 서비스 (Nova)

### 9.1 Controller Node 설정

#### 9.1.1 DB 생성

```bash
mysql -u root <<EOF
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY '1234';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY '1234';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY '1234';
FLUSH PRIVILEGES;
EOF
```

#### 9.1.2 Nova 사용자 및 서비스 생성

```bash
source ~/admin-openrc

openstack user create --domain default --password 1234 nova
openstack role add --project admin --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute

openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
```

#### 9.1.3 Nova 설치

```bash
apt install -y nova-api nova-conductor nova-scheduler nova-novncproxy
```

#### 9.1.4 Nova 설정

**/etc/nova/nova.conf 편집:**
```ini
[DEFAULT]
log_dir = /var/log/nova
lock_path = /var/lock/nova
state_path = /var/lib/nova
transport_url = rabbit://openstack:1234@controller:5672/
my_ip = 192.168.77.129

[api]
auth_strategy = keystone

[api_database]
connection = mysql+pymysql://nova:1234@controller/nova_api

[database]
connection = mysql+pymysql://nova:1234@controller/nova

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = admin
username = nova
password = 1234

[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]
api_servers = http://controller:9292

[placement]
region_name = RegionOne
project_domain_name = Default
project_name = admin
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = 1234

[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = admin
username = neutron
password = 1234
service_metadata_proxy = true
metadata_proxy_shared_secret = 1234
```

#### 9.1.5 데이터베이스 동기화

```bash
nova-manage api_db sync
>> 
3 RLock(s) were not greened, to fix this error make sure you run eventlet.monkey_patch() before importing any other modules.
root@controller:~# mysql -u root -e "USE nova_api; SHOW TABLES;" | head
Tables_in_nova_api
aggregate_hosts
aggregate_metadata
aggregates
alembic_version
allocations
build_requests
cell_mappings
consumers
flavor_extra_specs

>>
nova-manage cell_v2 map_cell0
nova-manage cell_v2 create_cell --name=cell1 --verbose
nova-manage db sync
nova-manage cell_v2 list_cells

systemctl restart nova-api nova-scheduler nova-conductor nova-novncproxy
```

### 9.2 Compute Node 설정

#### 9.2.1 Nova Compute 설치

```bash
apt install -y nova-compute
```

#### 9.2.2 Nova 설정

**/etc/nova/nova.conf 편집:**
```ini
[DEFAULT]
log_dir = /var/log/nova
lock_path = /var/lock/nova
state_path = /var/lib/nova
transport_url = rabbit://openstack:1234@controller:5672/
my_ip = 192.168.77.130

[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = admin
username = nova
password = 1234

[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html

[glance]
api_servers = http://controller:9292

[placement]
region_name = RegionOne
project_domain_name = Default
project_name = admin
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = 1234

[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = admin
username = neutron
password = 1234

[libvirt]
virt_type = kvm

```

**서비스 재시작:**
```bash
systemctl restart nova-compute
```

### 9.3 검증

**Controller 노드에서:**
```bash
source ~/admin-openrc
openstack compute service list
>>
(트러블슈팅) PermissionError: [Errno 13] Permission denied: '/var/log/nova/nova-manage.log'
즉, nova 사용자로 nova-manage를 실행했는데, /var/log/nova/에 로그 파일을 쓸 권한이 없음
[컨트롤러]
chown -R nova:nova /var/log/nova
chmod 750 /var/log/nova
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
(원인)
nova.conf에서:
log_dir = /var/log/nova
이렇게 지정돼 있는데, /var/log/nova가 nova:nova 소유가 아니면 nova-manage가 로그를 못 써서

[확인]
root@controller:~# openstack compute service list
+-----------+-----------+-----------+----------+---------+-------+-------------+
| ID        | Binary    | Host      | Zone     | Status  | State | Updated At  |
+-----------+-----------+-----------+----------+---------+-------+-------------+
| 41161ec8- | nova-     | controlle | internal | enabled | up    | 2025-12-    |
| 5850-     | conductor | r         |          |         |       | 17T11:23:52 |
| 49f3-     |           |           |          |         |       | .000000     |
| 8cc9-     |           |           |          |         |       |             |
| 76c3eda6b |           |           |          |         |       |             |
| 6ae       |           |           |          |         |       |             |
| 02222068- | nova-     | controlle | internal | enabled | up    | 2025-12-    |
| 0d18-     | scheduler | r         |          |         |       | 17T11:23:53 |
| 42da-     |           |           |          |         |       | .000000     |
| 8a3d-     |           |           |          |         |       |             |
| 0dd9d1444 |           |           |          |         |       |             |
| a02       |           |           |          |         |       |             |
| 7217a25a- | nova-     | compute   | nova     | enabled | up    | 2025-12-    |
| 4dd9-     | compute   |           |          |         |       | 17T11:23:54 |
| 48f7-     |           |           |          |         |       | .000000     |
| 8a72-     |           |           |          |         |       |             |
| bd4f5fed0 |           |           |          |         |       |             |
| e5b       |           |           |          |         |       |             |
+-----------+-----------+-----------+----------+---------+-------+-------------+
root@controller:~# 

>>
openstack catalog list
openstack image list
```

---
# 10. 네트워크 서비스 (Neutron – Open vSwitch + VXLAN)

## 구성 목표

- Controller 1 + Compute 1
- VM은 사설 IP(VXLAN) 사용
- VM ↔ VM ping 통신
- Host-Only 네트워크를 VXLAN 터널망으로 사용

---

## 10.1 Controller Node 설정

### 10.1.1 DB 생성

```bash
mysql -u root <<EOF
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY '1234';
FLUSH PRIVILEGES;
EOF
```

### 10.1.2 Neutron 사용자 및 서비스 생성

```bash
source ~/admin-openrc

openstack user create --domain default --password 1234 neutron
openstack role add --project admin --user neutron admin

openstack service create \
  --name neutron \
  --description "OpenStack Networking" network

openstack endpoint create --region RegionOne network public http://controller:9696
openstack endpoint create --region RegionOne network internal http://controller:9696
openstack endpoint create --region RegionOne network admin http://controller:9696
```

### 10.1.3 Neutron 설치

**Controller 노드:**

```bash
apt install -y neutron-server neutron-openvswitch-agent \
  neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
```

> ❌ `neutron-linuxbridge-agent` 사용 안 함

### 10.1.4 Neutron 설정 (Controller)

#### `/etc/neutron/neutron.conf`

```ini
[database]
connection = mysql+pymysql://neutron:1234@controller/neutron

[DEFAULT]
core_plugin = ml2
service_plugins = router
transport_url = rabbit://openstack:1234@controller:5672/
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = admin
username = neutron
password = 1234

[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = admin
username = nova
password = 1234
```

#### `/etc/neutron/plugins/ml2/ml2_conf.ini` (Controller / Compute 공통)

```ini
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch
extension_drivers = port_security

[ml2_type_vxlan]
vni_ranges = 1:1000

[securitygroup]
enable_ipset = true
```

> ✔️ Linuxbridge 관련 설정 제거 완료

#### `/etc/neutron/plugins/ml2/openvswitch_agent.ini` (Controller)

```ini
[ovs]
local_ip = 192.168.56.10      # Controller Host-Only IP
bridge_mappings = provider:br-ex

[agent]
tunnel_types = vxlan

[vxlan]
l2_population = true

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

#### `/etc/neutron/metadata_agent.ini`

```ini
[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = 1234
```

### 10.1.5 OVS 설정 (Controller)

```bash
apt install -y openvswitch-switch
systemctl enable --now openvswitch-switch
```

> **참고:** External 네트워크를 나중에 사용할 경우를 대비해 `br-ex`는 유지

### 10.1.6 DB 동기화 및 서비스 재시작

```bash
neutron-db-manage \
  --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head

systemctl restart neutron-server neutron-openvswitch-agent \
  neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent nova-api
```

---

## 10.2 Compute Node 설정

### 10.2.1 Neutron 설치

```bash
apt install -y neutron-openvswitch-agent openvswitch-switch
systemctl enable --now openvswitch-switch
```

### 10.2.2 Neutron 설정 (Compute)

#### `/etc/neutron/neutron.conf`

```ini
[DEFAULT]
transport_url = rabbit://openstack:1234@controller:5672
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = admin
username = neutron
password = 1234
```

#### `/etc/neutron/plugins/ml2/openvswitch_agent.ini`

```ini
[ovs]
local_ip = 192.168.56.11      # Compute Host-Only IP
bridge_mappings = provider:br-ex

[agent]
tunnel_types = vxlan

[vxlan]
l2_population = true

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

### 10.2.3 서비스 재시작

```bash
systemctl restart neutron-openvswitch-agent nova-compute
```

---

## 10.3 검증

Controller 노드에서:

```bash
openstack network agent list
```

## 11. 블록 스토리지 (Cinder)

### 11.1 Controller Node 설정

#### 11.1.1 DB 생성

```bash
mysql -u root <<EOF
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
FLUSH PRIVILEGES;
EOF
```

#### 11.1.2 Cinder 사용자 및 서비스 생성

```bash
source ~/admin-openrc

openstack user create --domain default --password CINDER_PASS cinder
openstack role add --project service --user cinder admin

openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3

openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s
```

#### 11.1.3 Cinder 설치

```bash
apt install -y cinder-api cinder-scheduler
```

#### 11.1.4 Cinder 설정

**/etc/cinder/cinder.conf 편집:**
```ini
[database]
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder

[DEFAULT]
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone
my_ip = 192.168.0.10

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = cinder
password = CINDER_PASS

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

**데이터베이스 동기화:**
```bash
cinder-manage db sync

systemctl restart cinder-scheduler apache2
```

### 11.2 Compute Node 설정 (Storage Node)

#### 11.2.1 LVM 설정

```bash
apt install -y lvm2 thin-provisioning-tools

# 물리 볼륨 생성 (예: /dev/sdb)
pvcreate /dev/sdb

# 볼륨 그룹 생성
vgcreate cinder-volumes /dev/sdb
```

**/etc/lvm/lvm.conf 편집:**
```
devices {
    filter = [ "a/sdb/", "r/.*/"]
}
```

#### 11.2.2 Cinder Volume 설치

```bash
apt install -y cinder-volume tgt
```

#### 11.2.3 Cinder 설정

**/etc/cinder/cinder.conf 편집:**
```ini
[database]
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder

[DEFAULT]
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone
my_ip = 192.168.0.11
enabled_backends = lvm
glance_api_servers = http://controller:9292

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = cinder
password = CINDER_PASS

[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = tgtadm

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

**서비스 재시작:**
```bash
systemctl restart tgt cinder-volume
```

### 11.3 검증

```bash
openstack volume service list
```

---

## 12. Horizon 대시보드

### 12.1 설치

```bash
apt install -y openstack-dashboard
```

### 12.2 설정

**/etc/openstack-dashboard/local_settings.py 편집:**
```python
OPENSTACK_HOST = "controller"
ALLOWED_HOSTS = ['*']

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': 'controller:11211',
    }
}

OPENSTACK_KEYSTONE_URL = "http://%s:5000/identity/v3" % OPENSTACK_HOST
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

TIME_ZONE = "Asia/Seoul"
```

### 12.3 Apache 재시작

```bash
systemctl reload apache2
```

### 12.4 접속

브라우저에서 `http://controller/horizon` 또는 `http://192.168.0.10/horizon`으로 접속

- **Domain**: Default
- **Username**: admin
- **Password**: ADMIN_PASS

---

## 13. 구축 검증

### 13.1 Horizon 접속

브라우저에서 `http://controller/horizon` 접속 후 로그인

### 13.2 네트워크 생성

#### Provider Network 생성
```bash
source ~/admin-openrc

openstack network create --share --external \
  --provider-physical-network provider \
  --provider-network-type flat provider

openstack subnet create --network provider \
  --allocation-pool start=192.168.0.200,end=192.168.0.220 \
  --dns-nameserver 8.8.8.8 --gateway 192.168.0.1 \
  --subnet-range 192.168.0.0/24 provider-subnet
```

#### Self-Service Network 생성
```bash
openstack network create selfservice

openstack subnet create --network selfservice \
  --dns-nameserver 8.8.8.8 --gateway 172.16.1.1 \
  --subnet-range 172.16.1.0/24 selfservice-subnet

openstack router create router
openstack router add subnet router selfservice-subnet
openstack router set router --external-gateway provider
```

### 13.3 인스턴스 생성

#### Flavor 생성
```bash
openstack flavor create --id 0 --vcpus 1 --ram 512 --disk 1 m1.nano
```

#### Security Group 규칙 추가
```bash
openstack security group rule create --proto icmp default
openstack security group rule create --proto tcp --dst-port 22 default
```

#### SSH Key Pair 생성
```bash
ssh-keygen -q -N "" -f ~/.ssh/id_rsa
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
```

#### 인스턴스 생성
```bash
openstack server create --flavor m1.nano --image cirros \
  --nic net-id=$(openstack network list | grep selfservice | awk '{print $2}') \
  --security-group default --key-name mykey test-instance

openstack server list
```

### 13.4 Floating IP 연결

```bash
openstack floating ip create provider
openstack server add floating ip test-instance <FLOATING_IP>
```

### 13.5 Ping / SSH 확인

```bash
ping <FLOATING_IP>
ssh cirros@<FLOATING_IP>
```

### 13.6 볼륨 연결 테스트

```bash
openstack volume create --size 1 test-volume
openstack server add volume test-instance test-volume
```

---

## 14. 트러블슈팅 가이드

### 14.1 서비스 상태 확인

```bash
systemctl status <service_name>
```

### 14.2 로그 확인

```bash
tail -f /var/log/nova/nova-compute.log
tail -f /var/log/neutron/neutron-server.log
tail -f /var/log/apache2/error.log
```

### 14.3 OpenStack 서비스 확인

```bash
openstack compute service list
openstack network agent list
openstack volume service list
```

### 14.4 일반적인 문제

#### 문제 1: Nova Compute가 Controller에 등록되지 않음
**해결책**: `/etc/nova/nova.conf`의 `transport_url`, `my_ip` 확인

#### 문제 2: 인스턴스가 네트워크 연결 안 됨
**해결책**: Neutron 에이전트 상태 확인, OVS 브리지 확인

#### 문제 3: Horizon 접속 불가
**해결책**: Apache 로그 확인, `ALLOWED_HOSTS` 설정 확인

---

## 15. 결론

본 문서는 Ubuntu 24.04 환경에서 OpenStack Caracal을 기반으로 Controller와 Compute를 분리한 단일 구성 환경을 구축함으로써 OpenStack 핵심 아키텍처와 네트워크 구조를 이해하고 실습 재현성을 확보하였다.

### 핵심 성과
- DevStack 없이 공식 가이드 기반 수동 설치 완료
- Controller/Compute 분리 아키텍처 구현
- Provider Network 및 Self-Service Network 구성
- 완전한 클라우드 인프라 구축 및 검증

---

## 참고 자료

- [OpenStack Official Documentation](https://docs.openstack.org/)
- [OpenStack Caracal Release Notes](https://releases.openstack.org/caracal/)
- [Ubuntu Cloud Archive](https://wiki.ubuntu.com/OpenStack/CloudArchive)

---

## 라이선스

이 문서는 교육 목적으로 작성되었습니다.

---

**작성일**: 2025
**버전**: 1.0
