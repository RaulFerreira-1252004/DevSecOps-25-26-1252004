# CE1 – Provisionamento com Vagrant e Ansible

Raul Ferreira - 1252004

## 1. Objetivo

O presente exercício tem como objetivo a automatização do provisionamento de uma infraestrutura composta por três máquinas virtuais utilizando **Vagrant** e **Ansible**, cumprindo os seguintes requisitos:

- Instalação automática de dependências (Java, Tomcat, Git, etc.)
- Download e execução da base de dados H2 (modo TCP + Web Console)
- Clonagem e build de uma aplicação Gradle
- Deploy do ficheiro WAR no servidor Tomcat
- Execução da base de dados H2 como serviço `systemd`
- Execução dos playbooks exclusivamente a partir da máquina `ansible`
- Utilização de módulos Ansible apropriados (`apt`, `git`, `get_url`, `copy`, `template`, `systemd`, etc.)

---

## 2. Arquitetura da Solução

A infraestrutura é composta por três máquinas virtuais:

| Máquina   | IP               | Função |
|------------|-----------------|--------|
| ansible   | 192.168.56.10   | Máquina de controlo (execução dos playbooks + build da aplicação) |
| app       | 192.168.56.11   | Servidor Tomcat (deploy da aplicação) |
| h2        | 192.168.56.12   | Servidor Base de Dados H2 |

### Portos encaminhados para o host:

| Serviço | Guest | Host |
|----------|-------|------|
| Tomcat   | 8080  | 8080 |
| H2 Web   | 8082  | 8082 |

---

## 3. Estrutura do Projeto

    CE1/
    │
    ├── Vagrantfile
    ├── README.md
    └── ansible/
        ├───artifacts   
        │   └─── app.war
        ├───inventory
        │   └─── hosts.ini
        ├───playbooks
        │   └─── site.yml
        └───roles
            ├───app
            │   ├───defaults
            │   │   └─── main.yml
            │   ├───handlers
            │   │   └─── main.yml
            │   └───tasks
            │       └─── main.yml
            ├───build
            │   ├───defaults
            │   │   └─── main.yml
            │   └───tasks
            │       └─── main.yml
            ├───h2
            │   ├───defaults
            │   │   └─── main.yml
            │   ├───handlers
            │   │   └─── main.yml
            │   ├───tasks
            │   │   └─── main.yml
            │   └───templates
            │       └─── h2.service.j2 
            └── ansible.cfg

Foi utilizada uma abordagem baseada em **roles**, permitindo modularidade e melhor organização.

---

## 4. Execução do Exercício

### 4.1 Iniciar as máquinas virtuais

No PowerShell (Windows) vamos entrar no diretorio /CE1, e executar:

    vagrant up

Assim que terminar podemos verificar se a execução foi concluida com sucesso através de:

    vagrant status

E verificar o output:

    Current machine states:
    
    app                       running (virtualbox)
    h2                        running (virtualbox)
    ansible                   running (virtualbox)
    
    This environment represents multiple VMs. The VMs are all listed
    above with their current state. For more information about a specific
    VM, run `vagrant status NAME`.

Se App, h2 e ansible VMs estiverem **running**, podemos proceder ao entrar na ansible VM para executar o playbook.

    vagrant ssh ansible

Dentro da máquina ansible, mudar para o diretorio correto, e executar o playbook:

    cd /vagrant/ansible
    ansible-playbook -i inventory/hosts.ini playbooks/site.yml

Este comando:

- Constrói a aplicação Gradle

- Instala e configura o servidor H2

- Instala Tomcat e faz deploy da aplicação

---

## 5. Implementação dos Requisitos
### 5.1 Instalação de dependências

Foi utilizado o módulo: 

    ansible.builtin.apt

Permite instalação idempotente de pacotes como:

- openjdk

- git

- tomcat10

- unzip

- curl

Justificação:

O módulo apt garante idempotência e evita execução repetida desnecessária.

---

### 5.2 Clonagem e Build da Aplicação

Foi utilizada a role build, executada na máquina ansible.

Clonagem:

    ansible.builtin.git

Repositório utilizado:

    https://bitbucket.org/pssmatos/tut-basic-gradle.git

Build:

    ./gradlew clean assemble --no-daemon

O artefacto gerado (WAR) é copiado para:

    /vagrant/ansible/artifacts/app.war

O build é realizado na máquina de controlo conforme requisito.
O diretório /vagrant é partilhado entre VMs, facilitando a transferência do artefacto.

---

### 5.3 Deploy da Aplicação no Tomcat

Na role app:

    Instalação do Tomcat via apt

Deploy do WAR usando:

    ansible.builtin.copy

Destino:

    /var/lib/tomcat10/webapps/app.war

O serviço é gerido com:

    ansible.builtin.systemd

Tomcat é instalado via gestor de pacotes para maior simplicidade e manutenção.

---

### 5.4 Instalação e Execução do H2

Na role h2:

Download do ficheiro .jar via:

    ansible.builtin.get_url

- Criação de utilizador dedicado (h2)

- Criação de diretório /var/lib/h2

- Criação de serviço systemd através de template

Execução do H2 como serviço
    
    ExecStart=/usr/bin/java -cp /opt/h2/h2.jar org.h2.tools.Server -tcp -tcpPort 9092 -tcpAllowOthers -web -webPort 8082 -webAllowOthers

Foi utilizado org.h2.tools.Server em vez de -jar para permitir:

- tcpAllowOthers

- webAllowOthers

Isto permite acesso remoto a partir do host.

---

## 6. Validação do Funcionamento
### 6.1 Verificação de serviços

Na máquina ansible:

    ansible -i inventory/hosts.ini appservers -m ping
    ansible -i inventory/hosts.ini dbservers -m ping

Resultado esperado: pong

Verificação de serviços:

    systemctl status tomcat10
    systemctl status h2

Ambos devem estar active (running).

---

### 6.2 Acesso via Browser

Aplicação:

    http://localhost:8080

O que deve ser visivel:

![8080.png](CE1%20imagens/8080.png)

Consola H2:

    http://localhost:8082

Output:

![8082.png](CE1%20imagens/8082.png)

---

## 7. Idempotência

A execução repetida do comando:

    ansible-playbook -i inventory/hosts.ini playbooks/site.yml

Não provoca reinstalações desnecessárias, demonstrando idempotência.

---

## 8. Conclusão

Foi implementado com sucesso um ambiente totalmente automatizado utilizando Vagrant e Ansible, cumprindo todos os requisitos propostos:

- Separação clara de responsabilidades por VM

- Utilização de módulos Ansible apropriados

- Execução controlada a partir da máquina ansible

- Serviços configurados como systemd

- Acesso remoto funcional à aplicação e à base de dados

A solução é reproduzível, modular e extensível.