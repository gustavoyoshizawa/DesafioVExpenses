## Desafio VExpenses

## Tarefa 1

### Configuração do Provedor AWS
Define a AWS como o provedor de nuvem e estabelece a região us-east-1 para a criação dos recursos.
```hcl
provider "aws" {
  region = "us-east-1"
}
```

### Variáveis de Projeto e Candidato
Define variáveis para armazenar o nome do projeto e do candidato. Elas são usadas para nomear recursos dinamicamente.
```hcl
variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "Gustavo"
}
```

### Chave SSH
Gera um par de chaves RSA de 2048 bits para acesso SSH seguro à instância EC2.
```hcl
resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}
```

Cria um par de chaves na AWS, associando a chave pública à EC2.
```hcl
resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}
```

### VPC e Subnet
Cria uma VPC (Virtual Private Cloud) com o bloco CIDR 10.0.0.0/16, ativando DNS support e DNS hostnames.
```hcl
resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}
```

Cria uma subnet dentro da VPC com o bloco 10.0.1.0/24 e associada à zona de disponibilidade us-east-1a.
```hcl
resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}
```

### Internet Gateway e Roteamento
Cria um Internet Gateway para permitir que os recursos dentro da VPC se comuniquem com a internet.
```hcl
resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}
```

Cria uma tabela de rotas que direciona todo o tráfego (0.0.0.0/0) para o Internet Gateway.
```hcl
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
```

Associa a subnet à tabela de rotas, garantindo que a instância EC2 possa acessar a internet.
```hcl
resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id
}
```

### Grupos de Segurança
Cria um Security Group para controlar o tráfego da instância EC2.
Entrada: Permite conexões SSH (porta 22) de qualquer IP.
Saída: Permite todo tráfego de saída sem restrições.

### ❌Acesso SSH irrestrito
O código permite acesso SSH de qualquer IP (0.0.0.0/0), o que representa um risco de segurança.

###❌Falta de suporte para HTTP e HTTPS
A configuração do grupo de segurança não permite tráfego HTTP e HTTPS, impedindo o acesso à aplicação via navegador.

```hcl
resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH de qualquer lugar e todo o tráfego de saída"
  vpc_id      = aws_vpc.main_vpc.id

  ingress {
    description      = "Allow SSH from anywhere"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
}
```

### Imagem do Sistema Operacional
Busca a AMI mais recente do Debian 12 com virtualização HVM e publicada pelo dono 679593333241.
```hcl
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
```

### Instância EC2
Configura uma instância EC2 com as seguintes características:
- Tipo: t2.micro (uso gratuito da AWS).
- AMI: Debian 12.
- Subnet: Associada à main_subnet.
- Chave SSH: Usa a chave gerada anteriormente.
- Grupo de Segurança: Usa main_sg.
- IP Público: Associado automaticamente.
- Volume de Disco: 20GB, gp2, excluído ao término da instância.
- User Data: Atualiza o sistema (`apt-get update && upgrade`).

### Outputs
Exibe a chave privada SSH.

### ❌Falta de armazenamento local da chave privada
O código exibe a chave privada, mas sem armazená-la adequadamente no sistema de arquivos.

```hcl
output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}
```

Exibe o endereço IP público da instância EC2 para acesso remoto.
```hcl
output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}
```

