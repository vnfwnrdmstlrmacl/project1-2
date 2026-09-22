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

```mermaid
flowchart LR

    %% ==========================
    %% ON-PREMISE
    %% ==========================
    subgraph ONPREM["On-Premise"]

        VIP_IN["Proxy Internet VIP<br/>10.2.2.254"]

        subgraph PROXY_HA["Proxy HA"]
            P1["Proxy1<br/>10.2.2.20<br/>10.2.3.1"]
            P2["Proxy2<br/>10.2.2.21<br/>10.2.3.10"]
        end

        VIP_DB["DB VIP<br/>10.2.3.254"]

        subgraph DB_HA["Database HA"]
            DBA["DB-A<br/>10.2.3.2"]
            DBS["DB-S<br/>10.2.3.3"]
        end

        subgraph ETCD["etcd Cluster"]
            E1["etcd-1<br/>10.2.3.20"]
            E2["etcd-2<br/>10.2.3.21"]
            E3["etcd-3<br/>10.2.3.22"]
        end

        VIP_IN --> P1
        VIP_IN --> P2

        P1 --> VIP_DB
        P2 --> VIP_DB

        VIP_DB --> DBA
        VIP_DB --> DBS

        DBA --> E1
        DBA --> E2
        DBA --> E3

        DBS --> E1
        DBS --> E2
        DBS --> E3
    end


    %% ==========================
    %% AWS
    %% ==========================
    subgraph AWS["AWS"]

        subgraph REGION["Region A"]

            subgraph VPC["VPC"]

                subgraph PUBLIC["Public Subnet"]
                    EC2["EC2"]
                end

                subgraph PRIVATE["Private Subnet"]
                    DMS["DMS"]
                    RDS["RDS"]
                end

                EC2 --> DMS
                EC2 --> RDS

            end

            S3["S3"]

        end
    end


    %% ==========================
    %% ON-PREMISE → AWS
    %% ==========================
    VIP_IN --> EC2
