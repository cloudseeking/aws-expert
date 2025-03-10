# EC2 Bootstrap

O processo de bootstrap de instâncias EC2 é fundamental para configurar servidores automaticamente durante a inicialização. Este documento descreve as melhores práticas e métodos para implementar o bootstrap eficiente em instâncias EC2 usando Terraform.

## O que é Bootstrap de EC2?
Bootstrap é o processo de inicialização e configuração automática de uma instância EC2 quando ela é lançada. Isso inclui instalação de software, configuração de serviços e execução de scripts personalizados.

## Métodos de Bootstrap

### 1. User Data

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    echo "<h1>Servidor configurado via Terraform</h1>" > /var/www/html/index.html
  EOF
  
  tags = {
    Name = "WebServer"
  }
}
```

### 2. Cloud-Init

```hcl
resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  user_data = <<-EOF
    #cloud-config
    package_update: true
    packages:
      - nginx
      - docker
    runcmd:
      - systemctl start nginx
      - systemctl enable nginx
      - docker pull nginx:latest
  EOF
  
  user_data_replace_on_change = true
}
```

### 3. Arquivos de Template

```hcl
data "template_file" "init" {
  template = file("${path.module}/scripts/init.sh.tpl")
  
  vars = {
    region = var.aws_region
    env    = var.environment
  }
}

resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  user_data = data.template_file.init.rendered
}
```

## Melhores Práticas

1. **Idempotência**: Scripts de bootstrap devem ser idempotentes (podem ser executados várias vezes sem efeitos colaterais)
2. **Modularização**: Divida scripts complexos em componentes reutilizáveis
3. **Logs**: Implemente logging adequado para facilitar a depuração
4. **Tratamento de Erros**: Adicione verificações de erro e mecanismos de recuperação
5. **Segurança**: Evite armazenar credenciais em scripts de bootstrap; use AWS Secrets Manager ou Parameter Store

## Ferramentas Avançadas

- **AWS Systems Manager (SSM)**: Para gerenciamento de configuração e execução de comandos
- **Ansible**: Para orquestração de configuração mais complexa
- **Packer**: Para criar AMIs pré-configuradas, reduzindo o tempo de bootstrap

## Exemplo Completo com SSM

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = var.instance_type
  iam_instance_profile = aws_iam_instance_profile.ssm_profile.name
  
  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y amazon-ssm-agent
    systemctl start amazon-ssm-agent
    systemctl enable amazon-ssm-agent
  EOF
  
  tags = {
    Name = "AppServer"
    Environment = var.environment
  }
}

resource "aws_ssm_document" "app_config" {
  name          = "app-configuration"
  document_type = "Command"
  
  content = jsonencode({
    schemaVersion = "2.2"
    description   = "Configure application server"
    mainSteps = [
      {
        action = "aws:runShellScript"
        name   = "configureApplication"
        inputs = {
          runCommand = [
            "#!/bin/bash",
            "yum install -y docker",
            "systemctl start docker",
            "systemctl enable docker",
            "docker pull ${var.app_image}",
            "docker run -d -p 80:80 ${var.app_image}"
          ]
        }
      }
    ]
  })
}

resource "aws_ssm_association" "app_config" {
  name = aws_ssm_document.app_config.name
  
  targets {
    key    = "tag:Name"
    values = ["AppServer"]
  }
}
```

## Considerações de Segurança

- Limite permissões IAM ao mínimo necessário
- Criptografe dados sensíveis
- Use VPC endpoints para acessar serviços AWS sem tráfego pela internet
- Implemente grupos de segurança restritivos



