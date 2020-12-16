---
layout: post
title: Redis Install
---

요즘에 인프라 환경 셋팅에 관심이 생겨 Redis 셋팅을 해보기로 하였습니다. 실제로 진행은 AWS E2에 terraform 으로 하였지만 내용을 풀어서 설명하도록 하겠습니다. 

먼저 설치는 위에서 언급한대로 AWS의 EC2에 yum을 통해 설치를 진행하였고 3개의 master 노드와 3개의 slave로 구성하였습니다. (클러스터를 하기 위해서는 적어도 master가 3개 필요합니다.)

AWS 리눅스에는 기본적으로 epel 레파지토리가 설치되어 있습니다. 단, 활성화가 필요합니다. 

아래 커멘트를 통해 활성화를 진행 합니다. 

```yaml
sudo amazon-linux-extras install epel -y
```

다음으로 remi 레파지토리를 설치하고 레디스 설치도 진행하도록 합니다. 

```yaml
sudo yum -y install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
sudo yum -y --enablerepo=remi install redis
```

설치를 진행하고나면 외부에서 접근을 위해 설정을 수정해야 합니다. 
기본적으로 yum으로 설치 시 설정 파일은 /etc/redis.conf가 됩니다. 

해당 파일 내용 중 "bind 127.0.0.1" 로 로컬 접속만 허용되어 있는데, 외부 접속을 위해 0.0.0.0 으로 변경합니다. 
아래처럼 말이죠

```yaml
bind 0.0.0.0
```

그렇게 총 6대의 EC2에 Redis를 설치하고나면 클러스터를 생성해야 합니다. 

처음 언급한대로 master3, slave3으로 진행할 것인데 **아래와 같은 설정은 할 필요 없습니다.** 아니 하면 클러스터가 생성되지 않습니다. 


```yaml
replicaof <마스터IP> <마스터PORT>
```

위의 설정은 단순 master, slave를 설정하면서 slave 쪽에 하는 설정인데 위와 같이 하지 않아도 cluster 생성 명령어의 서버 순서에 따라 master, slave가 정해집니다. 

```yaml
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

클러스터 생성 전 위의 설정을 모든 노드의 /etc/redis.conf 에 추가합니다. 

그리고 Redis를 재기동합니다. 

```yaml
sudo systemctl restart redis
```

여기까지 진행 했다면 Redis는 클러스터 생성이 가능한 상태가 됩니다. 

아래 커맨드를 통해 클러스터를 생성합니다. 아래 커맨드 기준 처음 3개의 노드가 master가 되고 나머지 3개의 노드는 slave가 됩니다. 그리고 --cluster-replicas 1의 설정에 의해 각각 마스터에는 slave가 하나씩 추가됩니다. 

```yaml
redis-cli --cluster create xxx.xxx.xxx.xx:6379 xxx.xxx.xxx.xx:6379 xxx.xxx.xxx.xx:6379 xxx.xxx.xxx.xx:6379 xxx.xxx.xxx.xx:6379 xxx.xxx.xxx.xx:6379 --cluster-replicas 1
```

추가로 Redis의 경우 기본적으로 6379 포트를 사용하지만 클러스터를 생성하게 되면 기본 포트에 10000을 더한 포트를 추가로 사용하게 됩니다. 위의 경우에는 16379 포트도 같이 오픈이 되어야 합니다. 

위에서는 일일이 커맨드로 EC2를 생성 후 yum을 통해 설치하였지만, AWS를 사용한다면 설정까지 진행한 상태로 AMI 이미지를 생성해두고 손쉽게 배포할 수 있을 것 입니다. 

저는 처음 진행하는 설치였고 terraform으로 인스턴스만 생성하고 user_data를 통해 기본 설치까지만 진행 후 설정을 진행하였습니다. 


아래는 AWS 환경 셋팅 및 인스턴스를 만드는데 사용한 terraform 파일 입니다.

```yaml
resource "aws_key_pair" "study_key" {
  key_name = "study-key"
  public_key = file("~/.ssh/study-key.pub")
}

resource "aws_default_route_table" "study_route_table" {
  default_route_table_id = aws_vpc.study_vpc.default_route_table_id
}

resource "aws_vpc" "study_vpc" {
  cidr_block  = "10.10.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support = true
  instance_tenancy = "default"

  tags = {
    Name = "study_vpc"
  }
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.study_vpc.id

  tags = {
    Name = "main"
  }
}

resource "aws_route_table" "r" {
  vpc_id = aws_vpc.study_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }

  tags = {
    Name = "main"
  }
}

resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.public_subnet1.id
  route_table_id = aws_route_table.r.id
}

resource "aws_route_table_association" "b" {
  subnet_id      = aws_subnet.public_subnet2.id
  route_table_id = aws_route_table.r.id
}

resource "aws_security_group_rule" "allow_all" {
  type              = "egress"
  to_port           = 0
  protocol          = "-1"
  from_port         = 0
  cidr_blocks = ["0.0.0.0/0"]
  security_group_id = aws_security_group.default_security_group.id
}

resource "aws_security_group" "default_security_group" {
  name = "default_security_group"
  vpc_id = aws_vpc.study_vpc.id

  ingress {
    from_port = "22"
    to_port = "22"
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_subnet" "public_subnet1" {
  vpc_id = aws_vpc.study_vpc.id
  cidr_block = "10.10.1.0/24"
  map_public_ip_on_launch = true
  tags = {
    Name = "public_subnet1"
  }
}

resource "aws_subnet" "public_subnet2" {
  vpc_id = aws_vpc.study_vpc.id
  cidr_block = "10.10.2.0/24"
  map_public_ip_on_launch = true
  tags = {
    Name = "public_subnet2"
  }
}

resource "aws_subnet" "private_subnet1" {
  vpc_id = aws_vpc.study_vpc.id
  cidr_block = "10.10.3.0/24"
  tags = {
    Name = "private_subnet2"
  }
}

resource "aws_subnet" "private_subnet2" {
  vpc_id = aws_vpc.study_vpc.id
  cidr_block = "10.10.4.0/24"
  tags = {
    Name = "private_subnet2"
  }
}

resource "aws_instance" "redis_master1" {
  ami           = "ami-03461b78fdba0ff9d"
  instance_type = "t3.micro"
  vpc_security_group_ids = [aws_security_group.default_security_group.id]
  subnet_id = aws_subnet.public_subnet1.id
  key_name = aws_key_pair.study_key.key_name
  user_data = <<-EOF
        #! /bin/bash
        sudo yum -y update
		sudo amazon-linux-extras install epel -y
        sudo yum -y install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
        sudo yum -y --enablerepo=remi install redis
  EOF

  tags = {
    Name = "redis_master1"
  }
} 

resource "aws_instance" "redis_slave1" {
  ami           = "ami-03461b78fdba0ff9d"
  instance_type = "t3.micro"
  vpc_security_group_ids = [aws_security_group.default_security_group.id]
  subnet_id = aws_subnet.public_subnet2.id
  key_name = aws_key_pair.study_key.key_name
  user_data = <<-EOF
        #! /bin/bash
        sudo yum -y update
		sudo amazon-linux-extras install epel -y
        sudo yum -y install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
        sudo yum -y --enablerepo=remi install redis
  EOF

  tags = {
    Name = "redis_slave1"
  }
}

resource "aws_instance" "redis_master2" {
  ami           = "ami-03461b78fdba0ff9d"
  instance_type = "t3.micro"
  vpc_security_group_ids = [aws_security_group.default_security_group.id]
  subnet_id = aws_subnet.public_subnet1.id
  key_name = aws_key_pair.study_key.key_name
  user_data = <<-EOF
        #! /bin/bash
        sudo yum -y update
		sudo amazon-linux-extras install epel -y
        sudo yum -y install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
        sudo yum -y --enablerepo=remi install redis
  EOF

  tags = {
    Name = "redis_master2"
  }
}

resource "aws_instance" "redis_slave2" {
  ami           = "ami-03461b78fdba0ff9d"
  instance_type = "t3.micro"
  vpc_security_group_ids = [aws_security_group.default_security_group.id]
  subnet_id = aws_subnet.public_subnet2.id
  key_name = aws_key_pair.study_key.key_name
  user_data = <<-EOF
        #! /bin/bash
        sudo yum -y update
		sudo amazon-linux-extras install epel -y
        sudo yum -y install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
        sudo yum -y --enablerepo=remi install redis
  EOF

  tags = {
    Name = "redis_slave2"
  }
}

```



