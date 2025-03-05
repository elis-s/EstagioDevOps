Documentação Terraform - Projeto VExpenses
Objetivo
Este projeto demonstra a criação de uma infraestrutura básica na AWS utilizando Terraform, com o objetivo de provisionar uma instância EC2, VPC, Subnet, Internet Gateway e regras de segurança. A infraestrutura é configurada para rodar o servidor Nginx automaticamente na instância EC2, além de aplicar melhorias de segurança para acesso SSH.

1. Análise Técnica do Código Terraform
1.1 Provider AWS
O código inicia com a configuração do provider AWS, onde é definida a região da AWS para o provisionamento dos recursos.

provider "aws" {
  region = "us-east-1"
}
1.2 Variáveis
São definidas duas variáveis para facilitar a personalização do código:

projeto: Nome do projeto (default: "VExpenses")
candidato: Nome do candidato (default: "SeuNome")

variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
}
1.3 Chave SSH
Uma chave privada RSA é gerada para acesso SSH à instância EC2. A chave pública é usada para criar o par de chaves na AWS, garantindo o acesso seguro à instância.


resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}
1.4 VPC
Criação de uma VPC com CIDR 10.0.0.0/16, permitindo segmentação e controle de tráfego de rede.


resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}
1.5 Subnet
Criação de uma subnet pública dentro da VPC, com CIDR 10.0.1.0/24, associada à zona de disponibilidade us-east-1a.


resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}
1.6 Internet Gateway (IGW)
Configuração de um Internet Gateway para permitir o tráfego de entrada e saída da VPC para a internet.


resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id
  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}
1.7 Tabela de Roteamento e Associações
Criação de uma tabela de rotas para definir o tráfego de saída para qualquer destino (0.0.0.0/0) via o Internet Gateway, além de associar a tabela à subnet criada.


resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }
  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table"
  }
}

resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id
  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table_association"
  }
}
1.8 Security Group
O Security Group define as regras de acesso. Inicialmente, ele permite SSH na porta 22 de qualquer IP, o que pode representar um risco de segurança. O tráfego HTTP (porta 80) também é permitido para permitir o acesso ao Nginx.


resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH e HTTP"
  vpc_id      = aws_vpc.main_vpc.id

  ingress {
    description      = "Allow SSH from specific IP"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["<seu-ip-publico>/32"]
  }

  ingress {
    description      = "Allow HTTP from anywhere"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}
1.9 Instância EC2
Criação de uma instância EC2 do tipo t2.micro, utilizando a imagem Debian 12, configurada com 20 GB de disco e a chave SSH gerada.


resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]
  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              apt-get install -y nginx
              systemctl start nginx
              systemctl enable nginx
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}
1.10 Outputs
Ao final, o código gera dois outputs:

private_key: A chave privada RSA para acesso à instância EC2.
ec2_public_ip: O IP público da instância EC2.

output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}
2. Modificações e Melhorias Implementadas
2.1 Restringir Acesso SSH
A principal melhoria de segurança foi restringir o acesso SSH à instância EC2 para um IP específico, em vez de permitir o acesso de qualquer IP.


ingress {
  description      = "Allow SSH from specific IP"
  from_port        = 22
  to_port          = 22
  protocol         = "tcp"
  cidr_blocks      = ["<SEU_IP_PUBLICO>/32"]
}
Justificativa: Isso reduz o risco de ataques de força bruta e aumenta a segurança, permitindo o acesso SSH apenas de locais confiáveis.

2.2 Automação da Instalação do Nginx
A instalação do Nginx foi automatizada com o uso do user_data, que roda um script bash para instalar o Nginx assim que a instância EC2 for criada.


user_data = <<-EOF
            #!/bin/bash
            apt-get update -y
            apt-get upgrade -y
            apt-get install -y nginx
            systemctl start nginx
            systemctl enable nginx
            EOF
Justificativa: A automação facilita a replicação e garante consistência em ambientes de produção, eliminando a necessidade de intervenção manual.

2.3 Regras de Segurança para Nginx
Uma regra foi adicionada para permitir tráfego HTTP (porta 80) de qualquer IP, permitindo que a instância EC2 possa fornecer serviços web.


ingress {
  description      = "Allow HTTP from anywhere"
  from_port        = 80
  to_port          = 80
  protocol         = "tcp"
  cidr_blocks      = ["0.0.0.0/0"]
}
Justificativa: O servidor Nginx deve ser acessível via HTTP, e essa regra permite que os usuários externos possam acessar o conteúdo web servido pela instância.

Arquivo Terraform Modificado
O arquivo main.tf modificado, contendo todas as alterações acima, está disponível abaixo:

provider "aws" {
  region = "us-east-1"
}

variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
}

resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}

resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}

resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}

resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}

resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table"
  }
}

resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table_association"
  }
}

resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH e HTTP"
  vpc_id      = aws_vpc.main_vpc.id

  # Regras de entrada
  ingress {
    description      = "Allow SSH from specific IP"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["<seu-ip-publico>/32"]
  }

  ingress {
    description      = "Allow HTTP from anywhere"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  # Regras de saída
  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}

data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}

resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              apt-get install -y nginx
              systemctl start nginx
              systemctl enable nginx
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}

output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}


Instruções de Uso:

Pré-requisitos
Antes de executar o código Terraform, certifique-se de ter os seguintes pré-requisitos instalados:

Terraform: A versão recomendada para este projeto é a 1.0 ou superior.

Para instalar o Terraform, siga as instruções oficiais: https://learn.hashicorp.com/tutorials/terraform/install-cli
AWS CLI: O Terraform interage com a AWS através da CLI.
	
O primeiro passo é inicializar o Terraform para baixar os plugins necessários e configurar o ambiente de trabalho. Execute o seguinte comando no diretório onde está o arquivo main.tf:
	terraform init
Esse comando inicializa o diretório de trabalho do Terraform e baixa todos os provedores e módulos necessários.

3. Verifique o Plano de Execução
Antes de aplicar qualquer modificação, é importante revisar o que será criado, alterado ou destruído na AWS. Para isso, use o comando terraform plan:
	terraform plan
Esse comando mostrará o que o Terraform irá criar, modificar ou destruir na infraestrutura da AWS com base no código no arquivo main.tf.

4. Aplique a Configuração
Para criar a infraestrutura conforme o código, utilize o comando terraform apply. Esse comando criará todos os recursos necessários na AWS (VPC, subnets, EC2, etc.):
	terraform apply
O Terraform pedirá uma confirmação antes de aplicar as mudanças. Digite yes para continuar.
Durante o processo de aplicação, a AWS provisionará os recursos descritos no arquivo Terraform, incluindo uma instância EC2 com Nginx instalado automaticamente.

5. Verifique a Infraestrutura Criada
Após a aplicação ser concluída, você pode verificar os recursos criados na AWS Console:

EC2: A instância EC2 será criada na subnet pública e terá o Nginx instalado e rodando.
VPC e Subnet: A infraestrutura de rede será configurada conforme o código.
Security Groups: Um grupo de segurança será criado permitindo acesso SSH de um IP específico (no caso, você configurou para um IP público) e acesso HTTP (porta 80).
Para obter o IP público da instância EC2, execute o seguinte comando:
	terraform output ec2_public_ip
Esse comando retornará o endereço IP público da instância EC2.

6. Acesse a Instância EC2
Com o IP público da instância EC2 em mãos, você pode acessar a instância via SSH usando a chave privada gerada no processo (seu arquivo .pem da chave SSH). Use o seguinte comando para acessar a instância:
	ssh -i /caminho/para/sua-chave.pem ubuntu@<ec2_public_ip>

7. Destrua a Infraestrutura
Quando terminar de testar a infraestrutura, você pode destruir todos os recursos criados para não incorrer em custos adicionais. Use o comando abaixo para destruir todos os recursos provisionados:
	terraform destroy
O Terraform pedirá uma confirmação. Digite yes para confirmar e destruir a infraestrutura.

