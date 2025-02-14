# Projeto: Infraestrutura na AWS com Ansible

## Introdução
Este documento apresenta os passos realizados na configuração da infraestrutura na AWS utilizando Ansible para automação. O projeto envolveu a criação de instâncias EC2, configuração de credenciais, configuração do inventário dinâmico do Ansible e a tentativa de automação via SSH.

---

## Configuração Inicial
### 2.1 Criação da VPC e Sub-redes
No AWS Console, foram criadas as seguintes configurações:
- **VPC:** MinhaVPC
- **CIDR:** 10.0.0.0/16
- **Sub-rede pública:** SubnetPublica (CIDR 10.0.1.0/24)
- **Sub-rede privada:** SubnetPrivada (CIDR 10.0.2.0/24)

### 2.2 Configuração da Tabela de Rotas
Foi criada uma tabela de rotas `RotaPublica` associada à `SubnetPublica`, com os seguintes destinos:
- `0.0.0.0/0` → Internet Gateway (`igw-07de8b6bba0334951`)
- `10.0.0.0/16` → local

---

## Configuração das Credenciais AWS no WSL
### 3.1 Instalação do AWS CLI
```bash
sudo apt update && sudo apt install awscli -y
```

### 3.2 Configuração das Credenciais AWS
```bash
aws configure
```
Fornecendo:
- **AWS Access Key ID:** `SEU_ACCESS_KEY`
- **AWS Secret Access Key:** `SEU_SECRET_KEY`
- **Região:** `sa-east-1`
- **Formato de saída:** `json`

Para verificar as credenciais:
```bash
aws sts get-caller-identity
```

---

## Configuração da Instância EC2
Foi criada uma instância EC2 na região `sa-east-1` com as seguintes configurações:
- **AMI:** Ubuntu 20.04 LTS
- **Tipo:** `t2.micro`
- **Security Group:** Permissão para SSH (`22`) e acesso necessário para o Ansible
- **Chave SSH:** `nova-chave-recuperacao.pem`
- **Subnet:** `subnet-013ada43609fa8594`

**Comando utilizado para criar a instância:**
```bash
aws ec2 run-instances \
  --image-id ami-0080974613cf1e8c7 \
  --count 1 \
  --instance-type t2.micro \
  --key-name nova-chave-recuperacao \
  --security-group-ids sg-0ff44d8395d90dfbc \
  --subnet-id subnet-013ada43609fa8594
```
Após a criação da instância, o **endereço IP público foi identificado**:
```bash
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[PublicIpAddress]'
```

---

## Configuração do Ansible
### 5.1 Instalação do Ansible
```bash
sudo apt update && sudo apt install ansible -y
```

### 5.2 Configuração do Inventário Dinâmico
O inventário dinâmico foi configurado no arquivo `aws_ec2.yml`:
```bash
mkdir -p ~/aws-ansible-infra/inventory
nano ~/aws-ansible-infra/inventory/aws_ec2.yml
```

**Conteúdo do arquivo `aws_ec2.yml`:**
```yaml
plugin: aws_ec2
regions:
  - sa-east-1
filters:
  instance-state-name: running
hostnames:
  - public-ip-address
keyed_groups:
  - key: tags.Name
```

Testando a configuração do inventário:
```bash
ansible-inventory -i ~/aws-ansible-infra/inventory/aws_ec2.yml --list
```

Foi criado um **inventário estático `hosts.ini`**:
```bash
nano ~/aws-ansible-infra/inventory/hosts.ini
```

**Conteúdo do `hosts.ini`:**
```ini
[aws_ec2]
56.125.3.155 ansible_private_key_file=~/.ssh/nova-chave-recuperacao.pem ansible_user=ubuntu
```

Testando:
```bash
ansible-inventory -i ~/aws-ansible-infra/inventory/hosts.ini --list
```

---

##  Conexão SSH com a Instância
Após a criação da chave SSH (`nova-chave-recuperacao.pem`), foi necessário configurar o acesso à instância:
```bash
ssh -i ~/.ssh/nova-chave-recuperacao.pem ubuntu@56.125.3.155
```
Caso houvesse erro de permissão, a chave pública foi adicionada ao `authorized_keys`:
```bash
echo "ssh-rsa AAAAB3Nza..." >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```
Após essas configurações, a conexão SSH foi **estabelecida com sucesso**.

---

##  Teste do Ansible
```bash
ansible all -i ~/aws-ansible-infra/inventory/hosts.ini -m ping --private-key ~/.ssh/nova-chave-recuperacao.pem -u ubuntu
```

---

## Criação e Execução de um Playbook
### 8.1 Criando um Playbook Simples
```bash
nano ~/aws-ansible-infra/update.yml
```
**Conteúdo do `update.yml`:**
```yaml
- name: Atualizar pacotes do sistema
  hosts: aws_ec2
  become: true
  tasks:
    - name: Atualizar lista de pacotes
      apt:
        update_cache: yes
    - name: Atualizar pacotes instalados
      apt:
        upgrade: dist
```

### 8.2 Executando o Playbook
```bash
ansible-playbook -i ~/aws-ansible-infra/inventory/hosts.ini ~/aws-ansible-infra/update.yml --private-key ~/.ssh/nova-chave-recuperacao.pem -u ubuntu
```

---

## Conclusão
Este documento detalha todas as configurações realizadas até o momento, garantindo que a infraestrutura AWS esteja corretamente configurada e integrada ao Ansible.

---

## 🔗 Referências
- [Documentação Ansible](https://docs.ansible.com/ansible/latest/index.html)
- [AWS CLI User Guide](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)
- [Inventário Dinâmico AWS no Ansible](https://docs.ansible.com/ansible/latest/plugins/inventory/aws_ec2.html)

### **Status do Projeto:** *Finalizado e funcionando corretamente!*

