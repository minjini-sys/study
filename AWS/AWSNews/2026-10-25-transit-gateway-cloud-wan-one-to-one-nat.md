# AWS Transit Gateway와 Cloud WAN의 관리형 1:1 NAT

- 정리 날짜: 2026-10-25
- 원문 게시일: 2026-10-23
- 공식 출처: [AWS What's New - AWS Transit Gateway and AWS Cloud WAN now support 1:1 NAT](https://aws.amazon.com/about-aws/whats-new/2026/10/aws-transit-gateway-cloud-wan-1-1-nat/)

## 핵심 내용

AWS Transit Gateway와 AWS Cloud WAN이 attachment에서 직접 사용할 수 있는 managed one-to-one network address translation, 즉 **1:1 NAT**를 지원한다. 이 기능은 Regional Private NAT Gateway와 통합되어 제공된다.

서로 같은 private CIDR을 사용하는 network는 원래 destination address만으로 어느 network에 packet을 전달할지 구분하기 어렵다. 이번 기능을 사용하면 한쪽 network의 source CIDR을 충돌하지 않는 별도 private IP range로 변환해 두 network를 renumbering하지 않고 연결할 수 있다.

translation rule은 Transit Gateway 또는 Cloud WAN attachment에 정의한다. AWS가 양방향 address translation을 자동 적용하므로 별도의 NAT appliance를 배포하거나 return traffic용 변환 규칙을 직접 작성할 필요가 없다.

지원되는 연결 대상에는 Amazon VPC, AWS Site-to-Site VPN, AWS Direct Connect와 Transit Gateway Connect를 통해 연결된 SD-WAN appliance가 포함된다.

## 왜 중요한가

기업 network에서는 여러 조직이 독립적으로 같은 RFC 1918 address range를 사용하는 일이 흔하다. 예를 들어 인수한 회사와 기존 회사가 모두 `10.0.0.0/16`을 사용하면 route table만 연결해서는 두 network가 서로 통신할 수 없다.

기존 해결 방법은 한쪽 network의 IP를 전부 변경하거나 EC2 기반 NAT appliance를 배치하는 것이었다. renumbering은 application configuration, firewall rule, DNS와 운영 문서를 함께 바꿔야 하므로 비용과 시간이 크다. NAT appliance는 high availability, scaling, patching, failover와 asymmetric routing을 직접 관리해야 한다.

managed 1:1 NAT는 이러한 translation을 AWS network attachment에 포함한다. merger and acquisition, partner connectivity, multicloud, duplicate VPC address space와 오래된 data center 연결을 더 빠르게 구성할 수 있다.

## 1:1 NAT의 개념

1:1 NAT는 하나의 source IP를 하나의 translated IP에 고정적으로 대응시키는 방식이다. 여러 private host가 하나의 public IP를 공유하는 일반적인 internet egress NAT와 목적이 다르다.

예를 들어 두 VPC가 모두 `10.0.0.0/16`을 사용한다고 가정한다.

- VPC A 실제 CIDR: `10.0.0.0/16`
- VPC B 실제 CIDR: `10.0.0.0/16`
- VPC B가 VPC A에 보이는 translated CIDR: `100.64.0.0/16`

VPC A는 VPC B의 resource에 접근할 때 `100.64.0.0/16` destination을 사용한다. attachment의 NAT 기능이 translated address를 VPC B의 실제 `10.0.0.0/16` address로 바꾼다. return traffic도 AWS가 반대 방향으로 자동 변환한다.

translation은 IP address 문제를 해결하지만 DNS name, application allow list와 certificate identity를 자동으로 바꾸지는 않는다. application이 translated address 또는 이를 가리키는 DNS record를 사용하도록 별도로 설계해야 한다.

## 동작 방식

1. Transit Gateway 또는 Cloud WAN에 VPC, VPN, Direct Connect 또는 Connect attachment를 연결한다.
2. 충돌하는 source CIDR과 충돌하지 않는 translated private range를 정한다.
3. 해당 attachment에 address translation rule을 구성한다.
4. source network에서 나오는 packet의 address가 translated range로 변환된다.
5. 다른 network는 translated CIDR을 destination으로 route한다.
6. Transit Gateway 또는 Cloud WAN이 packet을 올바른 attachment로 전달한다.
7. Regional Private NAT Gateway가 destination 측에서 실제 private address로 변환한다.
8. return traffic에는 반대 방향 translation이 자동 적용된다.
9. CloudWatch metric과 EventBridge event로 translated traffic 및 상태를 관찰한다.

별도의 NAT instance나 third-party virtual appliance가 packet path에 필요하지 않으며 AWS가 translation capability를 운영하고 확장한다.

## 서비스 및 아키텍처 관점

### Transit Gateway 사용 시

Transit Gateway는 Region 내 여러 VPC와 on-premises network를 hub-and-spoke 형태로 연결한다. 각 attachment의 실제 CIDR이 겹치면 특정 attachment에 translated CIDR을 지정하고 Transit Gateway route table에는 translated range를 기준으로 route를 구성한다.

environment, business unit 또는 trust zone별로 Transit Gateway route table을 분리하면 1:1 NAT와 network segmentation을 함께 적용할 수 있다. propagation만 의존하지 말고 translated prefix가 어느 attachment로 전달되는지 명확히 검토해야 한다.

### Cloud WAN 사용 시

AWS Cloud WAN은 여러 Region과 site의 network를 core network policy로 중앙 관리한다. 1:1 NAT를 attachment에 적용하면 overlapping CIDR이 있는 branch, acquired company, partner 또는 multicloud segment를 기존 global network에 포함할 수 있다.

Cloud WAN segment와 attachment policy는 통신 가능한 network 범위를 정하고 NAT rule은 address ambiguity를 해결한다. NAT가 연결 허용을 의미하지 않으므로 segment isolation과 route policy를 별도로 유지해야 한다.

### 연결 유형별 활용

- VPC attachment: 독립적으로 만든 VPC의 중복 CIDR 연결
- Site-to-Site VPN: partner 또는 branch network와의 private connectivity
- Direct Connect: data center와 AWS 사이의 전용 연결
- Transit Gateway Connect: SD-WAN appliance를 통한 branch 및 multicloud 연결
- M&A: 인수 기업 network를 즉시 renumbering하지 않고 단계적으로 통합

## 보안 관점

- NAT는 address translation 기능이지 firewall이 아니다. security group, network ACL, AWS Network Firewall과 route segmentation을 별도로 적용한다.
- translated CIDR이 기존 VPC, on-premises, VPN partner 또는 future expansion range와 겹치지 않는지 IPAM으로 검증한다.
- attachment와 translation rule을 생성하거나 수정할 IAM principal을 최소화한다.
- source IP가 translated address로 보이므로 log 분석과 incident response 문서에 실제 CIDR과 translated CIDR의 mapping을 기록한다.
- CloudTrail로 Transit Gateway, Cloud WAN과 NAT configuration 변경을 감사한다.
- third-party 연결에서는 상대 조직이 광고하거나 허용할 translated prefix를 계약 및 firewall rule과 일치시킨다.
- DNS response가 실제 중복 주소를 반환하지 않도록 Route 53 Resolver rule, private hosted zone 또는 DNS forwarding을 설계한다.
- east-west traffic을 허용하기 전에 필요한 port와 protocol만 열고 default-deny 원칙을 유지한다.
- translation rule 변경은 여러 network의 reachability에 영향을 줄 수 있으므로 승인과 change window를 둔다.

## 운영 관점

도입 전에는 전체 address inventory가 필요하다. AWS VPC IP Address Manager(IPAM), route table, Direct Connect route, VPN configuration과 SD-WAN controller 정보를 모아 실제 CIDR과 translated CIDR을 하나의 source of truth로 관리하는 것이 좋다.

운영 시 다음 항목을 점검한다.

- forward와 return path가 동일한 Transit Gateway 또는 Cloud WAN attachment를 통과하는가?
- translated prefix가 필요한 route table과 BGP session에 올바르게 전달되는가?
- translated range의 크기가 source CIDR과 1:1 mapping에 적합한가?
- security rule과 application allow list가 translated address를 기준으로 작성되었는가?
- DNS가 client가 도달 가능한 translated address를 반환하는가?
- CloudWatch metric에서 drop, volume과 예상치 못한 traffic 변화가 나타나는가?
- EventBridge event가 network operations alert와 연결되어 있는가?
- 장애 시 translation rule과 route를 되돌릴 rollback 절차가 있는가?
- NAT 처리 비용과 Transit Gateway, Cloud WAN, VPN 또는 Direct Connect data processing 비용을 함께 추적하는가?

연결 테스트는 단순 ping만으로 끝내지 않는다. TCP session, application protocol, DNS resolution, return path, MTU, latency와 failover를 함께 검증해야 한다.

이 기능은 Transit Gateway와 Cloud WAN이 제공되는 모든 commercial AWS Region에서 Regional Private NAT Gateway를 통해 사용할 수 있다. 비용은 Amazon VPC pricing을 기준으로 확인해야 한다.

## 도입 전 확인할 질문

1. 중복되는 실제 CIDR과 owner를 모두 확인했는가?
2. translated CIDR이 현재와 미래의 network range에 충돌하지 않는가?
3. 어느 attachment에 translation을 적용할지 명확한가?
4. route table과 BGP advertisement는 실제 CIDR이 아니라 translated CIDR을 올바르게 사용하고 있는가?
5. application과 DNS가 translated address로 통신할 수 있는가?
6. firewall과 allow list에서 실제 주소와 translated 주소를 혼동하지 않는가?
7. log와 monitoring에서 원본 host를 추적할 mapping이 준비되어 있는가?
8. 장기적으로 renumbering할지 1:1 NAT를 계속 운영할지 결정했는가?

## 시험 관점 정리

- AWS Transit Gateway는 여러 VPC와 on-premises network를 중앙 hub로 연결하는 Regional network transit service다.
- AWS Cloud WAN은 여러 Region과 site의 global network를 policy 기반으로 관리한다.
- CIDR은 IP address range를 prefix length와 함께 표현하는 방식이다.
- overlapping CIDR은 서로 다른 network가 같은 address range를 사용할 때 발생한다.
- 1:1 NAT는 하나의 실제 IP를 하나의 translated IP와 고정적으로 mapping한다.
- Private NAT Gateway는 private network 간 address translation에 사용할 수 있다.
- Site-to-Site VPN은 IPsec tunnel로 on-premises와 AWS를 연결한다.
- Direct Connect는 on-premises와 AWS 사이에 전용 network connection을 제공한다.
- Transit Gateway Connect는 GRE와 BGP를 이용해 SD-WAN appliance를 Transit Gateway에 연결한다.
- NAT는 route와 security control을 대신하지 않는다.
- route table은 destination prefix에 따라 packet의 next hop을 결정한다.
- asymmetric routing은 forward와 return traffic이 서로 다른 path를 사용해 stateful device에서 문제가 생기는 상황이다.

## 같이 보면 좋은 학습 포인트

- RFC 1918 private address range
- CIDR subnetting과 route longest prefix match
- Transit Gateway route table과 propagation
- AWS Cloud WAN segment와 core network policy
- Regional Private NAT Gateway
- source NAT와 destination NAT
- 1:1 NAT와 many-to-one NAT 차이
- Site-to-Site VPN과 Direct Connect
- BGP route advertisement
- Route 53 Resolver hybrid DNS
- AWS VPC IP Address Manager
- CloudWatch network metric과 EventBridge automation

## 한 줄 정리

AWS Transit Gateway와 Cloud WAN의 관리형 1:1 NAT는 중복 CIDR을 가진 VPC, data center와 partner network를 renumbering이나 자체 NAT appliance 없이 연결하게 해주지만 translated prefix의 route, DNS, 보안 정책과 address inventory를 일관되게 관리해야 한다.
