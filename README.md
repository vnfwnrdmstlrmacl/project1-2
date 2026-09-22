## 1. Project Overview

PostgreSQL 데이터베이스의 단일 장애점(SPOF)을 제거하고
장애 발생 시에도 서비스를 지속할 수 있는 고가용성 환경을 구축했습니다.

Patroni와 etcd를 이용해 PostgreSQL 클러스터의 상태를 관리하고,
HAProxy와 Keepalived를 이용해 클라이언트의 DB 접근 경로를 단일 VIP로 구성했습니다.

또한 Ansible을 활용하여 서버 초기 설정 및 PostgreSQL,
Patroni, HAProxy 등의 설치와 설정 과정을 자동화했습니다.

AWS와 개별 소통하기 위해 Ansible을 활용, Tailscale을 사용하였으며
Terraform을 활용하여 AWS의 VPC, RDS, DMS, Lambda, S3를 구축하였습니다.

S3를 스토리지로 사용해 pg_backrest 환경까지 Ansible로 자동화했습니다.

### 기간
2025.12.31 ~ 2026.2.27

### 인원
1명

### 주요 목표

- PostgreSQL HA 환경 구축
- PostgreSQL 장애 시 자동 Failover
- Proxy 장애 시 VIP Failover
- DB 노드 직접 노출 최소화
- Ansible을 활용한 인프라 자동화
- terraform을 활용한 AWS의 기초 설정

## 2. Architecture

┌──────────────────────────────────┐      ┌──────────────────────────────────┐
│         Onprem-Zone              │      │              AWS                  │
│                                   │      │  ┌──────────────────────────────┐ │
│      ┌─────────────────────┐     │      │  │          Region A             │ │
│      │  Proxy VIP:10.2.2.254│     │      │  │  ┌──────────────────────────┐│ │
│      └──────────┬───────────┘     │      │  │  │           VPC            ││ │
│         ┌────────┴────────┐       │      │  │  │  ┌──────────┐            ││ │
│    ┌────▼────┐      ┌─────▼───┐   │      │  │  │  │  Public  │            ││ │
│    │ Proxy1  │      │ Proxy2  │   │      │  │  │  │┌────────┐│  ┌───────┐ ││ │
│    │10.2.2.20│      │10.2.2.21│───┼──────┼──┼──┼──▶│  EC2   ││  │  S3   │ ││ │
│    └─────────┘      └─────────┘   │      │  │  │  │└───┬────┘│  └───────┘ ││ │
│                                   │      │  │  │  └──────┼───┘            ││ │
│      ┌─────────────────────┐     │      │  │  │  ┌───────▼───┐           ││ │
│      │  DB VIP:10.2.3.254  │     │      │  │  │  │  Private   │           ││ │
│      └──────────┬───────────┘     │      │  │  │  │┌────┐ ┌──┐│           ││ │
│         ┌────────┴────────┐       │      │  │  │  ││DMS │ │RDS│          ││ │
│    ┌────▼────┐      ┌─────▼───┐   │      │  │  │  │└────┘ └──┘│           ││ │
│    │ DB-A    │      │ DB-S    │   │      │  │  │  └───────────┘           ││ │
│    │10.2.3.2 │      │10.2.3.3 │   │      │  │  └──────────────────────────┘│ │
│    └─────────┘      └─────────┘   │      │  └────────────────────────────┘ │
│     └──────────┬───────────┘      │      └──────────────────────────────────┘
│   ┌────────┬───┴────┬────────┐   │
│   │Etcd-1  │Etcd-2  │Etcd-3  │   │
│   │.20     │.21     │.22     │   │
│   └────────┴────────┴────────┘   │
└──────────────────────────────────┘


## 3. Tech Stack

### database
- PostgreSQL
- patroni
- etcd

### Proxy/HA
- HAProxy
- Keepalived
- VIP

### Authomation
- Ansible
- Keepalived
- Terraform

### Infrastructure
- Linux
- VMware
- AWS

### Network
- TCP/IP
- SSH
- Proxy
- Virtual IP
- Tailscale

### Backup
- pgbackrest
- AWS

## 4. Implementation

### 5.1 PostgreSQL

- PostgreSQL Primary / Replica 구성
- Streaming Replication 구성
- PostgreSQL 상태 확인 및 장애 대응 구성
- AWS S3 pgbackrest 백업저장
- AWS DMS로 서비스 

### 5.2 Patroni

- PostgreSQL 클러스터 관리
- Leader 선출
- 장애 발생 시 Replica 승격

### 5.3 etcd

- Patroni DCS 구성
- 클러스터 상태 정보 저장
- etcd Cluster 상태 확인

### 5.4 HAProxy

- PostgreSQL 접근 경로 구성
- Backend PostgreSQL 서버 등록
- 현재 서비스 가능한 DB 노드로 트래픽 전달

### 5.5 Keepalived

- Proxy 서버 이중화
- VIP 구성
- Proxy 장애 시 VIP 이동

### 5.6 Ansible

- 서버 초기 설정
- 패키지 설치
- PostgreSQL 설정
- Patroni 설정
- HAProxy 설정
- Keepalived 설정
- Tailscale 기초 설정
- DMS 기초 설정
- pgbackrest 기초 설정
- Terraform tailscale을 위한 기초 설정
- pgBackrest 위한 기본 설정

### 5.7 Terraform
- VPC
- Tailscale-Bridge(EC2/VPN GW)
- RDS
- 3S (S3+IAM)
- DMS
- DMS-Automation(LAMBDA)
- LAMBDA
- Create Ansible Vault
- Connect On-Prem

## TroubleShooting

### SSH Permission denied
-> ssh 접속을 못함
-> 생각해보니까 ssh 안만듦
-> ssh-keygen / ssh-copy-id로 배포

### HAProxy enable error
->journalctl -u haproxy -n 50 --no-pager로 확인
-> permission denied
1. 일반 사용자 계정? -> root로 들어감
2. SELinux? -> 확인
=> getenforce로 확인 => enforcing
-> setenforce 0 => 해결


### DB-S PostgreSQL install error
->DB-S : ping -c 1 8.8.8.8 error
1. DB-A 확인 => 잘 나감
2. proxy Server iptable 확인 => 이상 없음
3. DB-S : ping -c 1 10.2.3.254 확인 => 이상 없음
4. DB-S, DB-A : ip neigh show 10.2.3.254 확인
=> ens160의 lladdr가 다름
5. proxy Server에서 확인 => DB-A는 ens192/ DB-S는 ens160꺼 잡고 있었음.

=> Ansible 코드 변경 => 해결
- name: Prevent ARP flux (arp_ignore/arp_announce)
  ansible.posix.sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    state: present
    reload: true
  loop:
    - { name: net.ipv4.conf.all.arp_ignore, value: "1" }
    - { name: net.ipv4.conf.all.arp_announce, value: "2" }
    - { name: net.ipv4.conf.default.arp_ignore, value: "1" }
    - { name: net.ipv4.conf.default.arp_announce, value: "2" }


