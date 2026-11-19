# 트러블슈팅 보고서 — IAM 최소 권한으로 VPC/EC2 실습

- 날짜: 2026-11-19
- 환경: 서울 리전(`ap-northeast-2`), `cloud-mission-user`, 그룹 `cloud-mission-group`, 고객 관리형 정책 `CloudMissionPolicy`
- 범위: 실제 콘솔 스크린샷과 사용자 제공 정책 내용으로 확인된 사건만 기록함

## 사례 1. VPC 상세 화면에서 403 액세스 거부

| 항목 | 기록 |
| --- | --- |
| **증상** | IAM 사용자로 VPC 페이지를 열면 `You are not authorized to perform this operation`이 나타남. 첫 번째는 `ec2:DescribeNetworkAcls` 부족 메시지. VPC 생성 자체는 성공했지만 상세 정보 일부를 불러오지 못함. |
| **원인 가설** | 최소 권한 IAM 정책에 VPC 콘솔의 부가 조회 API가 포함되지 않음. VPC 생성 권한과 상세 화면 조회 권한은 다름. |
| **검증** | 오류 메시지에 명시된 `ec2:DescribeNetworkAcls`와 정책 `CloudMissionRead`의 `Action` 목록을 대조함. 초기 목록에 이 작업이 없었음. |
| **조치** | `DescribeNetworkAcls`, `DescribeDhcpOptions`, `DescribeTags`, `DescribeNatGateways`, `DescribeFlowLogs`를 조회 작업에 추가하고 저장함. 재접속 후 더 이상 `DescribeNetworkAcls` 오류가 나타나지 않고 VPC 상세 정보가 표시됨. |
| **결과** | VPC `cloud-mission-vpc`가 생성되어 `Available` 상태임을 확인. 이후 별개의 `ec2:DescribeEgressOnlyInternetGateways` 접근 오류가 발생함. |
| **재발 방지** | 리소스 생성 권한뿐 아니라 콘솔에서 실제로 호출하는 조회 작업도 검토. 오류의 API 작업 이름과 현재 정책을 함께 기록. |

## 사례 2. 연속적인 EC2/VPC 콘솔 조회 오류

| 항목 | 기록 |
| --- | --- |
| **증상** | VPC 상세 화면에서 `ec2:DescribeEgressOnlyInternetGateways`, 서브넷 생성 화면에서 `ec2:DescribeCarrierGateways` 부족 메시지가 연속 발생. |
| **원인 가설** | 사용하지 않는 IPv6/통신사 네트워크 기능도 AWS 콘솔에서 부가 정보를 조회하므로 초기의 개별 `Describe...` 권한 목록만으로는 부족함. |
| **검증** | 각각 오류 문구에 해당 API 이름이 정확히 나타남. 사용자 제공 IAM JSON의 기존 조회 권한 목록에는 해당 작업들이 누락되어 있었음. |
| **조치** | 조회용 정책을 `"Action": ["ec2:Describe*"]`, `"Resource": "*"`, `"aws:RequestedRegion": "ap-northeast-2"`로 변경. 생성·변경·삭제용 `CloudMissionManage` 문은 유지함. |
| **결과** | 이후 서브넷 `cloud-mission-public-subnet` 생성 성공, IGW 연결, 라우팅 테이블 구성 및 EC2 배포까지 성공함. |
| **재발 방지** | 실습 중 콘솔 조회 권한은 분리 관리하고, 배포 후 실제 사용된 `Describe` API를 CloudTrail 등으로 검토하여 더 좁은 목록으로 축소할 계획. `Describe*`는 편의상 폭넓게 조회를 허용하므로 리소스 수준의 엄격한 최소 권한과 동일하지 않음. |

## 사례 3. EC2 생성 화면의 보안 그룹 조회 경고

- **증상:** EC2 시작 화면에 `ec2:GetSecurityGroupsForVpc` 권한 부족 경고가 표시됨.
- **가설:** `ec2:Describe*`에 포함되지 않는 별도 `Get...` API의 조회 권한이 필요함.
- **검증:** 경고 문구에서 정확한 API 이름 확인.
- **조치 안내:** `CloudMissionRead`의 Action에 `ec2:GetSecurityGroupsForVpc`를 추가하도록 안내함.
- **확인 가능한 결과:** 후속 캡처에서는 경고가 계속 보였으나, 최종적으로 EC2 인스턴스 시작에 **성공**하고 선택한 `cloud-mission-sg`로 웹 서버 접근이 가능했음. **이 개별 권한이 실제 반영되었는지는 제공된 자료만으로 확인되지 않음.**
- **재발 방지:** 경고와 실제 리소스 생성 차단을 구분하고, 정책 저장·전파 및 그룹 연결 상태를 확인함.

## 참고: 의도된 액세스 거부와 실제 장애 구분

- `compute-optimizer:GetEnrollmentStatus` 액세스 거부: EC2 상세 화면에 나타났지만 이번 과제에는 Compute Optimizer 권한이 필요하지 않아 추가하지 않음.
- Billing/Health/Security Hub 화면의 액세스 거부: 제한된 IAM 사용자에서는 예상 가능한 현상이며, 웹 서비스 접근 실패를 의미하지 않음.
- `AdministratorAccess`는 부여하지 않음.
- 읽기 권한 `ec2:Describe*` 및 관리 작업 `Resource: "*"` 때문에 현 정책은 **서비스·리전 수준 제한**이지, 리소스 ARN/태그 수준의 완전한 최소 권한 구현은 아님. 제출 시 이 한계를 명시하고 필요 시 정책을 더 세분화해야 함.

## 연결된 검증 증빙

- [외부 웹 서버 접속](evidence/browser-nginx.png)
- [내부 200 OK 응답](evidence/ec2-localhost-200.png)
