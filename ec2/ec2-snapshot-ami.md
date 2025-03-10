# EC2 Snapshots e AMI na AWS 🖥️💾

## O que são EC2 Snapshots? 📸
Snapshots são backups point-in-time dos volumes EBS (Elastic Block Store) na AWS. Eles capturam o estado exato de um volume em um momento específico, incluindo todos os dados armazenados no volume.
Características principais:
Incrementais: Apenas os blocos que mudaram desde o último snapshot são salvos
Persistentes: Armazenados no Amazon S3 (mas não acessíveis diretamente)
Regionais: Existem dentro de uma região específica da AWS
Compartilháveis: Podem ser compartilhados entre contas AWS

## O que são AMIs (Amazon Machine Images)? 🚀
AMIs são templates que contêm a configuração de software (sistema operacional, servidor de aplicação e aplicações) necessária para iniciar uma instância EC2.
Características principais:
Reutilizáveis: Permitem iniciar múltiplas instâncias com a mesma configuração
Regionais: Específicas para uma região AWS
Públicas ou privadas: Podem ser compartilhadas ou mantidas privadas
Baseadas em snapshots: Incluem snapshots dos volumes EBS

## Diferenças entre Snapshots e AMIs 🔄
| Snapshots | AMIs |
|-----------|------|
| Backup de volumes EBS | Template para criar instâncias EC2 |
| Não contém informações de boot | Contém informações de boot e configuração |
| Usado para backup/recuperação | Usado para implantação de novas instâncias |
| Pode ser criado de volumes em uso | Criada a partir de instâncias ou snapshots |

## Criando e Gerenciando Snapshots com Terraform 🛠️

### Criando um snapshot EBS 📸
```hcl
resource "aws_ebs_snapshot" "example_snapshot" {
  volume_id    = aws_ebs_volume.example_volume.id
  description  = "Snapshot do volume de produção"
  
  tags = {
    Name        = "snapshot-producao"
    Environment = "Production"
    CreatedBy   = "Terraform"
  }
}
```
### Criando snapshot com criptografia 🔒
```hcl
resource "aws_ebs_snapshot" "encrypted_snapshot" {
  volume_id    = aws_ebs_volume.example_volume.id
  description  = "Snapshot criptografado"
  
  tags = {
    Name = "snapshot-criptografado"
  }
}

resource "aws_ebs_snapshot_copy" "encrypted_copy" {
  source_snapshot_id = aws_ebs_snapshot.encrypted_snapshot.id
  description        = "Cópia criptografada do snapshot"
  encrypted          = true
  kms_key_id         = aws_kms_key.example_key.arn
}
```
### Política de ciclo de vida para snapshots 🖼️
```hcl
resource "aws_dlm_lifecycle_policy" "example_policy" {
  description        = "Política de ciclo de vida para snapshots diários"
  execution_role_arn = aws_iam_role.dlm_lifecycle_role.arn
  state              = "ENABLED"

  policy_details {
    resource_types = ["VOLUME"]

    schedule {
      name = "daily-snapshots"
      
      create_rule {
        interval      = 24
        interval_unit = "HOURS"
        times         = ["03:00"]
      }
      
      retain_rule {
        count = 7  # Manter snapshots por 7 dias
      }
      
      tags_to_add = {
        SnapshotCreator = "DLM"
      }
      
      copy_tags = true
    }

    target_tags = {
      Backup = "true"
    }
  }
}
```
### Criando e Gerenciando AMIs com Terraform 🖼️
Criando uma AMI a partir de uma instância EC2

```hcl
resource "aws_ami_from_instance" "example_ami" {
  name               = "ami-app-producao"
  source_instance_id = aws_instance.example_instance.id
  
  tags = {
    Name        = "AMI-App-Producao"
    Environment = "Production"
    CreatedBy   = "Terraform"
  }
}
```

# Copiando uma AMI para outra região

```hcl
resource "aws_ami_copy" "example_ami_copy" {
  provider           = aws.us-west-2
  name               = "ami-app-producao-copia"
  source_ami_id      = aws_ami_from_instance.example_ami.id
  source_ami_region  = "us-east-1"
  encrypted          = true 
  tags = {
    Name = "AMI-App-Producao-DR"
  }
}
``` 

# Compartilhando uma AMI com outra conta AWS

```hcl
resource "aws_ami_share" "example_ami_share" {
  ami_id      = aws_ami_from_instance.example_ami.id
  account_id  = "123456789012"
}
```

# Casos de Uso Comuns 🎯
1. Backup e Recuperação de Desastres:
- Snapshots regulares para backup de dados
- AMIs para recuperação rápida de ambientes completos

2. Migração entre Regiões:
- Copiar snapshots/AMIs entre regiões para migração
- Implementar estratégias multi-região

3. Implantação de Ambientes:
- AMIs como base para Auto Scaling Groups
- Imutabilidade de infraestrutura (criar novas instâncias em vez de atualizar)

4. Ambiente de Desenvolvimento:
- Criar ambientes de desenvolvimento idênticos à produção
- Testar atualizações antes de aplicar em produção

# Melhores Práticas 🌟
1. Automação:
- Automatize a criação de snapshots e AMIs
- Use o AWS Data Lifecycle Manager para gerenciar snapshots

2. Tagging:
- Aplique tags consistentes para facilitar o gerenciamento
- Inclua informações como data, ambiente e propósito

3. Retenção:
- Defina políticas claras de retenção para evitar custos desnecessários
- Remova snapshots e AMIs obsoletos

4. Segurança:
- Criptografe snapshots e AMIs
- Limite o compartilhamento apenas às contas necessárias

5. Testes:
- Teste regularmente a restauração de snapshots
- Valide AMIs antes de usá-las em produção

# Considerações de Segurança 🔒
1. Criptografia:
- Use criptografia para todos os snapshots e AMIs
- Considere o uso de chaves KMS gerenciadas pelo cliente

2. IAM:
- Restrinja permissões para criar/gerenciar snapshots e AMIs
- Use políticas de IAM para controlar o acesso

3. Compartilhamento:
- Revise regularmente permissões de compartilhamento
- Evite tornar AMIs públicas acidentalmente

4. Dados Sensíveis:
- Remova dados sensíveis antes de criar AMIs
- Use ferramentas como EC2 Image Builder para criar AMIs seguras

# Monitoramento e Custos 💰
1. Monitoramento:
- Configure alertas para falhas na criação de snapshots
- Monitore o uso e crescimento de snapshots

2. Custos:
- Snapshots são cobrados por GB armazenado
- Snapshots incrementais ajudam a reduzir custos
- AMIs não têm custo adicional além dos snapshots associados

3. Otimização:
- Remova snapshots e AMIs não utilizados
- Considere o uso de snapshots arquivados para dados raramente acessados

# Exemplo de Fluxo Completo com Terraform 🔄

# 1. Criar uma instância EC2
resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "AppServer"
    Backup = "true"
  }
}

# 2. Configurar política de ciclo de vida para snapshots automáticos
resource "aws_dlm_lifecycle_policy" "app_backup_policy" {
  description        = "Política de backup diário para servidores de aplicação"
  execution_role_arn = aws_iam_role.dlm_lifecycle_role.arn
  state              = "ENABLED"

  policy_details {
    resource_types = ["VOLUME"]

    schedule {
      name = "daily-snapshots"
      
      create_rule {
        interval      = 24
        interval_unit = "HOURS"
        times         = ["01:00"]
      }
      
      retain_rule {
        count = 7
      }
      
      tags_to_add = {
        SnapshotType = "Automated"
        CreatedBy    = "DLM"
      }
    }

    target_tags = {
      Backup = "true"
    }
  }
}

# 3. Criar AMI semanal para DR
resource "aws_ami_from_instance" "weekly_ami" {
  name               = "app-server-ami-${formatdate("YYYY-MM-DD", timestamp())}"
  source_instance_id = aws_instance.app_server.id
  
  tags = {
    Name        = "AppServerAMI-Weekly"
    Environment = "Production"
    CreatedBy   = "Terraform"
  }
  
  # Usando null_resource para programar a criação semanal
  depends_on = [null_resource.weekly_trigger]
}

resource "null_resource" "weekly_trigger" {
  triggers = {
    week = formatdate("YYYY-WW", timestamp())
  }
}

# 4. Copiar AMI para região secundária para DR
provider "aws" {
  alias  = "dr_region"
  region = "us-west-2"
}

resource "aws_ami_copy" "dr_ami_copy" {
  provider           = aws.dr_region
  name               = "dr-app-server-ami-${formatdate("YYYY-MM-DD", timestamp())}"
  source_ami_id      = aws_ami_from_instance.weekly_ami.id
  source_ami_region  = "us-east-1"
  encrypted          = true
  
  tags = {
    Name = "DR-AppServerAMI"
  }
}

## Conclusão
Snapshots e AMIs são componentes fundamentais para estratégias de backup, recuperação de desastres e implantação na AWS. Quando usados corretamente, eles fornecem flexibilidade, segurança e eficiência para gerenciar seus recursos EC2.
Ao automatizar a criação e o gerenciamento desses recursos com Terraform, você pode implementar práticas consistentes e confiáveis de infraestrutura como código, garantindo que seus ambientes sejam resilientes e facilmente reproduzíveis.