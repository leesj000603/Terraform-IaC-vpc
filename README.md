# Terraform-IaC-vpc
Terraform을 이용한 AWS VPC 및 관련 리소스 구성


# 1. IAM 역할 생성

### iam-policy.tf

```bash
# IAM 역할 생성
resource "aws_iam_role" "s3_create_bucket_role" {
  name = "ce22-s3-create-bucket-role"
  
  assume_role_policy = jsonencode({
    "Version": "2012-10-17",
    "Statement": [
      {
        "Action": "sts:AssumeRole",
        "Effect": "Allow",
        "Principal": {
          "Service": "ec2.amazonaws.com"
        }
      }
    ]
  })
}

# IAM 정책 정의 (S3에 대한 모든 권한 부여)
resource "aws_iam_policy" "s3_full_access_policy" {
  name        = "ce22-s3-full-access-policy"
  description = "Full access to S3 resources"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:*"  # 모든 S3 액세스 허용
        ]
        Resource = [
          "*"  # 모든 S3 리소스에 대한 권한
        ]
      }
    ]
  })
}

# IAM 역할에 정책 연결
resource "aws_iam_role_policy_attachment" "attach_s3_policy" {
  role       = aws_iam_role.s3_create_bucket_role.name
  policy_arn = aws_iam_policy.s3_full_access_policy.arn
}
```

---

# 2. vpc 생성

### vpc.tf

```bash
variable "vpc-cidr-block" {
  description = "VPC의 CIDR 블록"
  type        = string
  default     = "10.0.0.0/16" # 원하는 기본 CIDR 블록
}

resource "aws_vpc" "vpc" {
  cidr_block = var.vpc-cidr-block
  
  tags = {
    Name = "ce22-terraform-vpc"
  }
}
```

---

# 3. 서브넷 생성

### web-subnets.tf

```bash
# 첫 번째 서브넷의 CIDR 블록 변수 선언
variable "web-subnet1-cidr" {
  description = "첫 번째 서브넷의 CIDR 블록"
  type        = string
  default     = "10.0.1.0/24" # 서브넷1의 기본 CIDR 블록
}

# 두 번째 서브넷의 CIDR 블록 변수 선언
variable "web-subnet2-cidr" {
  description = "두 번째 서브넷의 CIDR 블록"
  type        = string
  default     = "10.0.2.0/24" # 서브넷2의 기본 CIDR 블록
}

# 첫 번째 서브넷 (web-subnet1) 생성
resource "aws_subnet" "web-subnet1" {
  # 이 서브넷이 속할 VPC의 ID. aws_vpc 리소스에서 생성된 VPC를 참조.
  vpc_id = aws_vpc.vpc.id
  
  # 서브넷의 CIDR 블록. var.web-subnet1-cidr 변수를 사용해 IP 범위 설정.
  cidr_block = var.web-subnet1-cidr
  
  # 이 서브넷이 위치할 가용 영역 (서울 리전 - ap-northeast-2a).
  availability_zone = "ap-northeast-2a"
  
  # 인스턴스가 이 서브넷에 생성될 때 자동으로 퍼블릭 IP 할당 여부. true로 설정.
  map_public_ip_on_launch = true

  # 서브넷의 태그. AWS 콘솔에서 식별을 쉽게 하기 위해 Name 태그 설정.
  tags = {
    Name = "ce22-terraform-subnet1"
  }
}

# 두 번째 서브넷 (web-subnet2) 생성
resource "aws_subnet" "web-subnet2" {
  vpc_id = aws_vpc.vpc.id  # vpc.tf 파일의 resource 자원 참조.
  cidr_block = var.web-subnet2-cidr
  availability_zone = "ap-northeast-2b"
  map_public_ip_on_launch = true
  
  tags = {
    Name = "ce22-terraform-subnet2"
  }
}

```

### app-subnets.tf

```bash
# private 으로 내부망에서만 access용 subnet (back, db포함)

# CIDR 블록 변수 선언
variable "app-subnet1-cidr" {
  description = "CIDR block for the first application subnet"
  type        = string
  default     = "10.0.11.0/24"  # 예시 CIDR 블록
}

variable "app-subnet2-cidr" {
  description = "CIDR block for the second application subnet"
  type        = string
  default     = "10.0.21.0/24"  # 예시 CIDR 블록
}

# 첫 번째 애플리케이션 서브넷 (app-subnet1) 생성
resource "aws_subnet" "app-subnet1" {
  vpc_id                  = aws_vpc.vpc.id  
  cidr_block              = var.app-subnet1-cidr
  
  availability_zone       = "ap-northeast-2a" 
  map_public_ip_on_launch = false # private 서브넷 이므로 false 처리

  tags = {
    Name = "ce22-terraform-app_subnet1"
  }
}

# 두 번째 애플리케이션 서브넷 (app-subnet2) 생성
resource "aws_subnet" "app-subnet2" {
  vpc_id                  = aws_vpc.vpc.id  
  cidr_block              = var.app-subnet2-cidr  
  availability_zone       = "ap-northeast-2b"    
  map_public_ip_on_launch = false # private 서브넷 이므로 false 처리

  tags = {
    Name = "ce22-terraform-app_subnet2"
  }
}

```

### terraform 변동 확인 후 적용

```bash
$terraform plan # 변동 예정 사항 확인
$terraform apply -auto-approve
```

### 서브넷 생성된 모습
![image](https://github.com/user-attachments/assets/4e640de7-c606-46bf-865f-db5b10dd950b)


---

# 4. internet gateway 생성

## internet-gw.tf

```bash
# 인터넷 게이트웨이 생성
resource "aws_internet_gateway" "internet-gw" {
  # 이 인터넷 게이트웨이가 연결될 VPC의 ID. aws_vpc 리소스에서 생성된 VPC를 참조.
  vpc_id = aws_vpc.vpc.id

  # 인터넷 게이트웨이의 태그. AWS 콘솔에서 식별을 쉽게 하기 위해 Name 태그 설정.
  tags = {
    Name = "ce22-terraform-igw"  # 인터넷 게이트웨이 이름
  }
}

```

### terraform 변동 확인 후 적용

```bash
$terraform plan # 변동 예정 사항 확인
$terraform apply -auto-approve
```

### 생성된 모습

![image](https://github.com/user-attachments/assets/0f87abde-93e8-40ad-98d0-a6da4a3caa3c)


---

# 5. elastic ip 생성

### eip.tf

```bash
# Elastic IP 생성 (NAT gateway에 할당하기 위한)
resource "aws_eip" "eip" {
  # 이 Elastic IP가 VPC에 연결될 것임을 지정.
  domain = "vpc"

  # 태그를 추가하여 식별하기 쉽게 설정할 수 있음.
  tags = {
    Name = "ce22-terraform-eip"  # Elastic IP의 이름
  }
}
```

### 생성된 모습

![image](https://github.com/user-attachments/assets/71a5b749-a12e-469f-892e-45d4d3615fe2)


---

# 6. nat gateway 생성

### nat-gw.tf

```bash
# NAT 게이트웨이 생성
resource "aws_nat_gateway" "nat-gw" {
  # NAT 게이트웨이에 연결할 Elastic IP의 ID. aws_eip 리소스에서 생성된 EIP를 참조.
  allocation_id     = aws_eip.eip.id
  
  # NAT 게이트웨이의 연결 유형. public으로 설정하여 퍼블릭 NAT 게이트웨이로 만듦.
  connectivity_type = "public"
  
  # NAT 게이트웨이가 속할 서브넷의 ID. aws_subnet 리소스에서 생성된 서브넷을 참조.
  subnet_id         = aws_subnet.web-subnet1.id
  
  # NAT 게이트웨이의 태그. AWS 콘솔에서 식별을 쉽게 하기 위해 Name 태그 설정.
  tags = {
    Name = "ce22-terraform-ngw"  # NAT 게이트웨이의 이름
  }

  # 이 NAT 게이트웨이는 인터넷 게이트웨이에 의존하므로 depends_on 사용.
  depends_on = [aws_internet_gateway.internet-gw]
}
```

### 생성된 모습

![image](https://github.com/user-attachments/assets/8be8f04b-9cc3-43db-893c-1f9b7347ed21)


### elastic ip가 연결된 모습.

![image](https://github.com/user-attachments/assets/c66dfc28-57bc-4f27-a5d5-967ec47f4254)


---

# 7. public 라우팅 테이블으로 public subnet을 igw에 연결

### public-rt.tf

```bash
# 퍼블릭 라우트 테이블 생성
resource "aws_route_table" "public-route-table" {
  # 라우트 테이블이 속할 VPC의 ID. aws_vpc 리소스에서 생성된 VPC를 참조.
  vpc_id = aws_vpc.vpc.id

  # 기본 라우트 설정. 모든 트래픽을 인터넷 게이트웨이로 라우팅.
  route {
    cidr_block = "0.0.0.0/0"  # 모든 IP 주소에 대한 트래픽
    gateway_id = aws_internet_gateway.internet-gw.id  # 인터넷 게이트웨이를 통해 라우팅
  }

  # 라우트 테이블의 태그. AWS 콘솔에서 식별을 쉽게 하기 위해 Name 태그 설정.
  tags = {
    Name = "ce22-terraform-public-rt"  # 퍼블릭 라우트 테이블의 이름
  }
}

# 첫 번째 퍼블릭 라우트 테이블과 서브넷의 연결
resource "aws_route_table_association" "pub-rt-association-1" {
  # 연결할 서브넷의 ID. aws_subnet 리소스에서 생성된 서브넷을 참조.
  subnet_id      = aws_subnet.web-subnet1.id
  
  # 연결할 라우트 테이블의 ID. aws_route_table 리소스에서 생성된 라우트 테이블을 참조.
  route_table_id = aws_route_table.public-route-table.id
}

# 두 번째 퍼블릭 라우트 테이블과 서브넷의 연결
resource "aws_route_table_association" "pub-rt-association-2" {
  subnet_id      = aws_subnet.web-subnet2.id
  route_table_id = aws_route_table.public-route-table.id
}

```

### public라우팅 테이블 생성, 적용된 모습

![image](https://github.com/user-attachments/assets/18c84c8b-de35-4c43-91af-5ac23544da1c)


---

# 8. private 라우팅 테이블으로 private subnet을 natgw에 연결

### private-rt.tf

```bash
# 프라이빗 라우트 테이블 생성
resource "aws_route_table" "private-route-table" {
  # 라우트 테이블이 속할 VPC의 ID. aws_vpc 리소스에서 생성된 VPC를 참조.
  vpc_id = aws_vpc.vpc.id

  # 기본 라우트 설정. 모든 트래픽을 NAT 게이트웨이로 라우팅.
  route {
    cidr_block = "0.0.0.0/0"  # 모든 IP 주소에 대한 트래픽
    gateway_id = aws_nat_gateway.nat-gw.id  # NAT 게이트웨이를 통해 라우팅
  }

  # 라우트 테이블의 태그. AWS 콘솔에서 식별을 쉽게 하기 위해 Name 태그 설정.
  tags = {
    Name = "ce22-terraform-private-rt"  # 퍼블릭 라우트 테이블의 이름
  }
}

# 첫 번째 프라이빗 라우트 테이블과 서브넷의 연결
resource "aws_route_table_association" "pri-rt-association-1" {
  # 연결할 서브넷의 ID. aws_subnet 리소스에서 생성된 서브넷을 참조.
  subnet_id      = aws_subnet.app-subnet1.id
  
  # 연결할 라우트 테이블의 ID. aws_route_table 리소스에서 생성된 라우트 테이블을 참조.
  route_table_id = aws_route_table.private-route-table.id
}

# 두 번째 프라이빗 라우트 테이블과 서브넷의 연결
resource "aws_route_table_association" "pri-rt-association-2" {
  subnet_id      = aws_subnet.app-subnet2.id
  route_table_id = aws_route_table.private-route-table.id
}
```

### private라우팅 테이블 생성, 적용된 모습

![image](https://github.com/user-attachments/assets/4ab7a3d3-71c8-4728-8b48-91f66808b0de)


---

# 9. 보안그룹 설정

### securitygroup.tf

```bash

```

---

# 10. ec2 생성

### ec2.tf

```bash

```

---
