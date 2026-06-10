# Zabbix Auto-registro com Ansible

> Provisionamento automático do Zabbix Agent 7 em servidores Linux com auto-registro no Zabbix Server — zero intervenção manual no frontend.

---

## O problema que isso resolve

Em ambientes tradicionais, sempre que um novo servidor é adicionado à infraestrutura, alguém precisa manualmente instalar o agente, configurar o arquivo, acessar o frontend do Zabbix e cadastrar o host. Esse processo é lento, sujeito a erros e não escala.

Com este projeto, basta adicionar o IP do servidor no inventário e rodar o playbook. O resto acontece automaticamente.

---

## Como funciona
ansible-playbook site.yml
↓
Ansible acessa o servidor via SSH
↓
Instala e configura o Zabbix Agent 2
↓
Agente envia HostMetadata ao Zabbix Server
↓
Zabbix identifica o metadata e aplica a Action
↓
Host aparece no frontend com grupo e template

---

## Ambiente usado na lab - mas você adapta ao seu cenário

| Máquina | OS            | IP     | Função                         |
|---------|---------------|--------|--------------------------------|
| Ubuntu  | Ubuntu Server | seu ip | Nó de controle / Zabbix Server |
| Rocky   | Rocky Linux 9 | seu ip | Nó monitorado                  |

> O Zabbix 7 está rodando em Docker no Ubuntu com a porta `10051` exposta pro host.

---

## Pré-requisitos para o laboratório

- Ubuntu Server com Ansible instalado
- Zabbix Server 7 rodando e acessível
- Servidor Rocky Linux 9 acessível via SSH
- Usuário `ansible` criado no servidor remoto

---

## Estrutura do Projeto
/opt/ansible/
├── ansible.cfg              # configurações globais do Ansible
├── group_vars/
│   └── all.yml              # variáveis compartilhadas entre todos os hosts
├── inventory/
│   └── lab                  # lista de hosts do ambiente
└── roles/
└── zabbix_agent/
├── handlers/
│   └── main.yml     # reinicia o agente quando necessário
├── tasks/
│   └── main.yml     # tarefas de instalação e configuração
└── templates/       # reservado para templates futuros

---

## Configuração SSH

O Ansible precisa de acesso SSH ao servidor remoto sem senha. Para isso, criamos um usuário dedicado e configuramos autenticação por chave.

**1. Gerar a chave SSH no Ubuntu:**
```bash
ssh-keygen -t ed25519 -C "ansible-lab" -f ~/.ssh/ansible_lab -N ""
```

**2. Criar o usuário ansible no Rocky Linux:**
```bash
sudo useradd -m -s /bin/bash ansible
echo 'ansible ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/ansible
```

**3. Copiar a chave pública para o Rocky:**
```bash
ssh-copy-id -i ~/.ssh/ansible_lab.pub ansible@IP_DO_ROCKY
```

**4. Testar a conexão:**
```bash
ssh -i ~/.ssh/ansible_lab ansible@IP_DO_ROCKY "echo conectado"
```

---

## Arquivos

### ansible.cfg
> Defina o inventário, usuário remoto e configurações de privilege escalation.

```ini
[defaults]
inventory         = ./inventory/lab
remote_user       = ansible
private_key_file  = ~/.ssh/ansible_lab
host_key_checking = False
roles_path        = ./roles

[privilege_escalation]
become        = True
become_method = sudo
become_user   = root
```

---

### inventory/lab
> Liste os hosts que serão gerenciados pelo Ansible.

```ini
[lab]
NOME-DA-MAQUINA  ansible_host=IP_DA_MAQUINA

[lab:vars]
ansible_python_interpreter=/usr/bin/python3
```

---

### group_vars/all.yml
> Variáveis globais do projeto. Edite com os dados do seu ambiente.

```yaml
---
zabbix_server_ip: IP_DO_ZABBIX
zabbix_agent_version: "7.0"
zabbix_host_metadata: "linux-lab" 
```

> O `zabbix_host_metadata` é o valor que o Zabbix usa para identificar e registrar o host automaticamente. Você pode criar valores diferentes para cada tipo de servidor.

---

### roles/zabbix_agent/tasks/main.yml
> Instale e configure o Zabbix Agent 2 no servidor remoto.

```yaml
---
- name: Instalar repositório do Zabbix 7
  ansible.builtin.dnf:
    name: "https://repo.zabbix.com/zabbix/7.0/rocky/9/x86_64/zabbix-release-7.0-1.el9.noarch.rpm"
    state: present
    disable_gpg_check: true

- name: Instalar Zabbix Agent 2
  ansible.builtin.dnf:
    name: zabbix-agent2
    state: present

- name: Configurar Server
  ansible.builtin.lineinfile:
    path: /etc/zabbix/zabbix_agent2.conf
    regexp: '^Server='
    line: "Server={{ zabbix_server_ip }}"
  notify: restart zabbix-agent2

- name: Configurar ServerActive
  ansible.builtin.lineinfile:
    path: /etc/zabbix/zabbix_agent2.conf
    regexp: '^ServerActive='
    line: "ServerActive={{ zabbix_server_ip }}"
  notify: restart zabbix-agent2

- name: Configurar Hostname
  ansible.builtin.lineinfile:
    path: /etc/zabbix/zabbix_agent2.conf
    regexp: '^Hostname='
    line: "Hostname={{ inventory_hostname }}"
  notify: restart zabbix-agent2

- name: Configurar HostMetadata para auto-registro
  ansible.builtin.lineinfile:
    path: /etc/zabbix/zabbix_agent2.conf
    regexp: '^# HostMetadata='
    line: "HostMetadata={{ zabbix_host_metadata }}"
  notify: restart zabbix-agent2

- name: Habilitar e iniciar o Zabbix Agent 2
  ansible.builtin.systemd:
    name: zabbix-agent2
    enabled: true
    state: started
```

> Use `lineinfile` em vez de um template completo para alterar apenas as linhas necessárias, preservando o restante da configuração padrão do agente.

---

### roles/zabbix_agent/handlers/main.yml
> Reinicie o agente automaticamente quando a configuração é alterada.

```yaml
---
- name: restart zabbix-agent2
  ansible.builtin.systemd:
    name: zabbix-agent2
    state: restarted
```

---

### site.yml
> Ponto de entrada do projeto. Defina quais roles rodam em quais hosts.

```yaml
---
- name: Provisionar Zabbix Agent
  hosts: lab
  roles:
    - zabbix_agent
```

---

## Configuração no Zabbix Server

Antes de rodar o playbook, configure a Action de auto-registro no frontend. Ela é responsável por receber o host e aplicar grupo e template automaticamente.

Acesse `Alerts → Actions → Autoregistration actions → Create action`:

| Campo      |   Valor                            |
|------------|------------------------------------|
| Name       | Auto-registro Linux Lab            |
| Condition  | Host metadata contains `linux-lab` |
| Operação 1 | Add to host groups → Linux servers |
| Operação 2 | Link to templates → Linux by Zabbix agent active |

> Esta Action só precisa ser criada uma vez. Qualquer servidor novo que rodar o playbook será registrado automaticamente a partir daí.

---

## Como usar

```bash
cd /opt/ansible
ansible-playbook site.yml
```

Após rodar o playbook, acesse `Monitoring → Hosts` no Zabbix. O host estará lá, com grupo e template aplicados, sem nenhuma configuração manual.
