🧩 Ubuntu 24.04 기반 OpenStack Caracal
Controller 1 / Compute 1 단일 구성 구축 가이드
1. 개요
1.1 문서 목적

본 문서는 Ubuntu 24.04.03 LTS 환경에서 OpenStack Caracal을 기반으로
Controller 1대와 Compute 1대를 분리하여 단일 구성 OpenStack 환경을 구축하는 것을 목적으로 한다.
DevStack을 사용하지 않고 공식 가이드를 기반으로 핵심 컴포넌트를 수동 설치하여
OpenStack 아키텍처 이해와 실습 재현성을 확보한다.

1.2 구성 범위

Controller Node: OpenStack 제어 및 관리 영역

Compute Node: 가상 머신 실행 영역

1.3 실습 환경 및 전제조건

OS: Ubuntu 24.04.03 LTS

OpenStack: Caracal (Ubuntu Cloud Archive)

가상화 환경: VMware Pro

네트워크: Bridged

계정: root (교육용 폐쇄 환경)

설치 방식: 공식 가이드 기반 수동 설치

2. 전체 아키텍처
2.1 노드 구성
노드	역할
Controller	API, 인증, 네트워크 제어, 대시보드
Compute	인스턴스 실행
2.2 노드별 주요 컴포넌트

Controller Node

Chrony

MariaDB

RabbitMQ

Memcached

Keystone

Glance

Placement

Nova (API, Scheduler, Conductor)

Neutron (Server, L3, DHCP)

Cinder API

Horizon

Compute Node

Chrony

Nova Compute

Neutron OVS Agent

Cinder Volume (LVM)

3. 네트워크 설계
3.1 VMware 네트워크 구성

VMware Bridged Network 사용

Controller / Compute 동일 L2 네트워크 연결

3.2 OpenStack 네트워크 구조

Provider Network: flat

Tenant Network: VXLAN

L2 Agent: Open vSwitch (OVS)

3.3 IP 주소 계획 (예시)
구분	대역
Management Network	192.168.0.0/24
Provider Network	192.168.0.0/24
Floating IP Pool	192.168.0.200 ~ 192.168.0.220
4. 기본 시스템 설정 (Controller / Compute 공통)
4.1 Hostname 설정
hostnamectl set-hostname controller   # Controller
hostnamectl set-hostname compute      # Compute

4.2 /etc/hosts 설정
192.168.0.10 controller
192.168.0.11 compute

4.3 시간 동기화 (Chrony)
apt update
apt install -y chrony
systemctl enable --now chrony

4.4 OpenStack Repository 설정
add-apt-repository cloud-archive:caracal
apt update && apt -y upgrade

5. 미들웨어 구성 (Controller)
5.1 MariaDB 설치 및 설정
apt install -y mariadb-server python3-pymysql


/etc/mysql/mariadb.conf.d/99-openstack.cnf

[mysqld]
bind-address = 0.0.0.0
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8

systemctl restart mariadb

5.2 RabbitMQ
apt install -y rabbitmq-server
rabbitmqctl add_user openstack RABBIT_PASS
rabbitmqctl set_permissions openstack ".*" ".*" ".*"

5.3 Memcached
apt install -y memcached python3-memcache
sed -i 's/-l 127.0.0.1/-l 192.168.0.10/' /etc/memcached.conf
systemctl restart memcached

6. 인증 서비스 구성 (Keystone)
6.1 DB 생성
mysql -u root
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
FLUSH PRIVILEGES;

6.2 Keystone 설치
apt install -y keystone apache2

6.3 Keystone 설정

/etc/keystone/keystone.conf

[database]
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

[token]
provider = fernet

keystone-manage db_sync
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
keystone-manage bootstrap \
  --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne

7. 이미지 서비스 (Glance)
apt install -y glance


DB 설정 → keystone 연동 → glance-manage db_sync

이미지 업로드 테스트

8. Placement 서비스
apt install -y placement-api


DB 생성 및 설정

Apache 재시작

9. Compute 서비스 (Nova)
9.1 Controller
apt install -y nova-api nova-scheduler nova-conductor nova-novncproxy

9.2 Compute
apt install -y nova-compute

openstack compute service list

10. 네트워크 서비스 (Neutron)
10.1 OVS 설치
apt install -y openvswitch-switch
ovs-vsctl add-br br-ex
ovs-vsctl add-port br-ex ens33

10.2 Neutron 구성
apt install -y neutron-server neutron-openvswitch-agent \
neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent

11. 블록 스토리지 (Cinder)
apt install -y cinder-api cinder-scheduler cinder-volume lvm2


LVM 기반 VG 생성

iSCSI 설정

12. Horizon 대시보드
apt install -y openstack-dashboard


ALLOWED_HOSTS = ['*']

Apache 재시작

13. 구축 검증

Horizon 접속: http://controller/horizon

네트워크 생성

인스턴스 생성

Floating IP 연결

Ping / SSH 확인

볼륨 연결 테스트

14. 트러블슈팅 가이드

systemctl status

/var/log/nova/

/var/log/neutron/

openstack service list

15. 결론

본 문서는 Ubuntu 24.04 환경에서 OpenStack Caracal을 기반으로
Controller와 Compute를 분리한 단일 구성 환경을 구축함으로써
OpenStack 핵심 아키텍처와 네트워크 구조를 이해하고 실습 재현성을 확보하였다.
