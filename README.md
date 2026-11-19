# Cloud Mission — AWS 웹 서버 배포

- 실습일: 2026-11-19
- AWS 계정: AWS Organizations 관리 계정 안의 **IAM 사용자** `cloud-mission-user`로 구축
- 리전: **서울 `ap-northeast-2`**
- 배포 대상: EC2 Ubuntu 24.04 LTS / Nginx
- 외부 접속 검증 방식: **(A) 브라우저에서 HTTP로 접속**

## 배포 주소 및 실제 검증

- 확인 당시 접속 URL: **http://13.124.189.205**
- 확인 결과: 브라우저에 **`Welcome to nginx!`** 기본 페이지 표시
- EC2 내부 검사: `curl -I http://localhost` → **`HTTP/1.1 200 OK`**
- SSH: macOS 터미널에서 `ubuntu@13.124.189.205` 접속 성공 (개인 키 인증)
- 서버 버전: `nginx/1.24.0 (Ubuntu)` (검증 당시 출력)
- **EC2에서 외부 HTTPS로 나가는 통신 (`curl -I https://example.com`)은 아직 결과가 제출되지 않아 미검증**

> URL은 실습 당시 퍼블릭 IP입니다. 인스턴스를 중지 후 재시작하거나 종료하면 바뀌거나 더 이상 접속되지 않을 수 있습니다. `http://`로 접속하세요(HTTPS 미구성).

### 증빙 이미지

![브라우저 외부 접속 성공](docs/evidence/browser-nginx.png)

![EC2 내부 curl 200 OK](docs/evidence/ec2-localhost-200.png)

## 아키텍처

![AWS 아키텍처](docs/architecture.png)

| 구성 요소 | 실제 구성 |
| --- | --- |
| VPC | `cloud-mission-vpc` (`10.0.0.0/16`) |
| Public Subnet | `cloud-mission-public-subnet` (`10.0.1.0/24`, `ap-northeast-2a`) |
| Internet Gateway | `cloud-mission-igw` (`igw-049b12df927b4380b`) |
| Public Route Table | `cloud-mission-public-rt`: `10.0.0.0/16 → local`, `0.0.0.0/0 → IGW` |
| EC2 | `cloud-mission-ec2` (`i-084ad134fc6128065`), `t3.micro`, Ubuntu 24.04 |
| EBS | 8 GiB gp3 |
| IP | Public `13.124.189.205` / Private `10.0.1.78` |
| Security Group | `cloud-mission-sg`: HTTP/80 `0.0.0.0/0`, SSH/22 `121.135.181.44/32` |
| SG 아웃바운드 | 전체 트래픽 `0.0.0.0/0` |
| IAM | `cloud-mission-user` → `cloud-mission-group` → `CloudMissionPolicy` (관리자 권한 아님) |

### 네트워크 동작

외부 브라우저 HTTP 요청은 **퍼블릭 IP → Internet Gateway → EC2 네트워크 인터페이스(보안 그룹에서 TCP/80 허용) → Nginx** 경로로 도달합니다. 서브넷에 연결된 라우팅 테이블은 VPC 내부 대상은 `local`, 인터넷 대상은 Internet Gateway로 전달합니다. SSH/22는 지정된 개인 IP에서만 허용합니다. EC2의 퍼블릭 IPv4는 AWS가 네트워크 인터페이스의 사설 IPv4와 매핑합니다.

## 명령어 및 재현

```bash
# Mac에서 개인 키 권한 제한 (키 파일은 공유·제출하지 않음)
chmod 400 ~/Downloads/cloud-mission-key.pem
ssh -i ~/Downloads/cloud-mission-key.pem ubuntu@13.124.189.205

# 위 SSH 세션의 EC2 Ubuntu 터미널에서
sudo apt update
sudo apt install nginx -y
sudo systemctl enable --now nginx
curl -I http://localhost

# 미검증 항목: 직접 실행 후 결과 확인
curl -I https://example.com
```

## 제출 파일

- `docs/architecture.png` — VPC/서브넷/IGW/EC2/보안 그룹 및 트래픽 흐름
- `docs/evidence/browser-nginx.png` — 외부 브라우저 접속 증빙
- `docs/evidence/ec2-localhost-200.png` — EC2 내부 200 응답 증빙
- `docs/troubleshooting.md` — 실제 IAM 조회 권한 오류 분석과 조치
- `docs/cleanup-checklist.md` — **실습 종료 시 실행**할 정리 체크리스트

## 미완료 / 제출 전 확인

- [x] 외부 브라우저 접속 (A) 성공
- [x] EC2 내부 `localhost` HTTP 200 응답
- [x] SSH 개인 키 접속 성공
- [x] EC2의 `curl -I https://example.com` 실행 결과 캡처
- [x] 종료 시 리소스 삭제 확인 및 정리 체크리스트 갱신
