# 🛡️ Full-Stack Cybersecurity Lab: Red & Blue Team Operations

## 📌 Objetivo do Projeto

Este projeto consiste na arquitetura, implementação e documentação de um laboratório completo de cibersegurança (Full-Stack). O objetivo é simular um ambiente corporativo real, aplicando na prática conceitos avançados de infraestrutura de TI, Sistemas Operacionais, Protocolos de Redes (TCP/IP), Auditoria/ Monitoramento de Sistemas e Ethical Hacking.

A infraestrutura foi desenhada sob a ótica de **Defense in Depth** (Defesa em Profundidade), permitindo a execução de operações ofensivas (Red Team) e a validação contínua da detecção e telemetria (Blue Team).

---

## 🏗️ Topologia e Arquitetura de Rede

O ambiente foi construído sobre um hypervisor (VirtualBox) utilizando o isolamento tático de redes para segmentar e auditar o fluxo de dados. A arquitetura foi desenhada para simular não apenas os ativos de uma empresa, mas o comportamento físico dos equipamentos de infraestrutura (Switches e Roteadores).

### Fluxo de Tráfego: Norte-Sul vs. Leste-Oeste

Para garantir a Defesa em Profundidade (*Defense in Depth*), a rede foi estruturada levando em consideração os dois eixos de tráfego corporativo:

* **Tráfego Norte-Sul:** Representa os dados que entram ou saem da rede local para a internet. Este vetor é afunilado e controlado pelo nosso Gateway de borda (pfSense), que atua como a primeira linha de defesa contra ameaças externas utilizando um IDS na interface WAN.
* **Tráfego Leste-Oeste:** Representa a comunicação lateral, interna à rede LAN (ex: um computador infectado tentando atacar o Controlador de Domínio). Este tráfego viaja diretamente pelo *switch* interno e nunca chega ao Gateway. Para auditar esse vetor, foi necessário implementar visibilidade interna através de um IDS secundário.

### Visibilidade Interna: Port Mirroring e Modo Promíscuo

Em redes físicas modernas, um *switch* encaminha os pacotes de rede exclusivamente para a porta do dispositivo de destino (baseado na tabela MAC). Isso cria um "ponto cego" para sistemas de segurança corporativos, pois o IDS não consegue "escutar" a comunicação entre as outras máquinas.

Para resolver isso fisicamente, utiliza-se o **Port Mirroring** (ou porta SPAN), uma configuração no *switch* que espelha uma cópia de todo o tráfego da rede para uma porta de monitoramento dedicada. No nosso laboratório virtual, replicamos essa engenharia através das seguintes configurações:

* **Modo Promíscuo ("Permitir Tudo"):** Ativado na rede *Host-Only* do hypervisor. Essa configuração quebra o isolamento padrão do *switch* virtual, permitindo que os pacotes Leste-Oeste sejam espelhados em *broadcast*.
* **Interface do IDS (Ubuntu/Snort):** A placa de rede do servidor de segurança foi configurada em modo promíscuo no nível do Sistema Operacional (`ip link set promisc on`), forçando-a a capturar e inspecionar os pacotes que não eram originalmente destinados ao seu endereço IP.

### Tabela de Ativos e Segmentação

| Máquina / Papel | SO / Tecnologia | IP (Rede Interna) | Função na Arquitetura |
| --- | --- | --- | --- |
| **Gateway / Perimeter** | pfSense (FreeBSD) | `10.0.10.1` | Roteamento, Firewall, Servidor DHCP e IDS de Borda (Norte-Sul). |
| **Coração Corporativo** | Windows Server 2019 | `10.0.10.10` (Estático) | Controlador de Domínio (`SENAC.LOCAL`), GPOs e Alvo Primário. |
| **Torre de Controle (Blue Team)** | Ubuntu Server 20.04 LTS | `10.0.10.20` (Estático) | Monitoramento Zabbix (Stack LAMP) e IDS Interno via modo promíscuo (Leste-Oeste) - Snort. |
| **Ameaça Interna (Red Team)** | Kali Linux | `10.0.10.52` (DHCP) | Máquina do atacante para Reconhecimento, Enumeração e Exploração. |

---

## 🛠️ Fase 1: Perímetro e Roteamento (pfSense)

A fundação do laboratório começou pelo isolamento e proteção da borda, aplicando os conceitos de *Gateways* e segurança de perímetro.

* **Configuração de Interfaces (Console):** Identificação precisa das interfaces físicas WAN (`em0` / Bridged) e LAN (`em1` / Host-Only) utilizando os endereços MAC do hypervisor durante o *setup* do FreeBSD, evitando a inversão crítica de rotas.
* **Serviços Base de Rede:** Configuração do IP do Gateway LAN para `10.0.10.1` e ativação do serviço DHCP com *range* reservado (`10.0.10.50` a `10.0.10.200`), deixando os primeiros endereços livres para a alocação de servidores e ativos de infraestrutura.
* **Gerenciamento Seguro:** A administração do firewall foi isolada da interface externa. Toda a configuração gráfica foi realizada internamente via WebGUI, acessando o endereço `http://10.0.10.1` através do navegador da máquina Windows Server.
* **Regras de Tráfego e Controle de Acesso (Firewall > Rules):** O ambiente foi validado em cima das regras estruturais do pfSense para garantir a comunicação Norte-Sul:
  * **Regras da LAN (LAN Rules):** Validação e manutenção da regra *Default allow LAN to any rule* (IPv4). Esta foi a regra de tráfego exata que permitiu que as máquinas internas (como o Windows Server) acessassem a internet externa, comportamento comprovado com sucesso durante os testes de conectividade (`ping 8.8.8.8`).
  * **Regras da WAN (WAN Rules):** Manutenção do bloqueio padrão de entrada (*Default Deny Inbound*) e das opções *Block private networks* (RFC 1918) e *Block bogon networks*. Isso garantiu que a rede Bridged (física) não invadisse a nossa topologia de laboratório.


* **IDS de Perímetro (Snort):** Instalação do pacote nativo via *Package Manager* e ativação do serviço atrelado à interface WAN, atuando com as configurações base como o sensor de borda da nossa rede.
* **Solução de Problema (Boot Loop):** Durante o provisionamento, o ambiente entrou em *loop* de instalação contínua. A correção foi realizada ejetando a mídia ISO virtual de instalação diretamente pelo hypervisor, forçando a máquina a realizar o *boot* pelo disco rígido recém-gravado.

---

## 🏢 Fase 2: O Coração Corporativo (Active Directory e Windows Server 2019)

Simulação de uma infraestrutura corporativa robusta com políticas de identidade, restrição de privilégios e monitoramento ativo.

* **Endereçamento Estratégico e DNS:** Antes da promoção a domínio, o servidor foi isolado com IP estático (`10.0.10.10`) fora do *range* do DHCP do pfSense, garantindo a estabilidade da infraestrutura. O DNS primário foi apontado para *localhost* (`127.0.0.1`) e o secundário para o Gateway (`10.0.10.1`), preparando o terreno para a resolução de nomes da rede local.
* **Promoção a Domain Controller (DC):** Instalação das funções de *AD DS* (Active Directory Domain Services) e *DNS Server*. Criação da floresta `SENAC.LOCAL` com a definição do nome NetBIOS como `SENAC` e configuração da credencial administrativa de recuperação.
* **Automação DevSecOps (PowerShell):** Abandono da interface gráfica clássica (ADUC) para o provisionamento de usuários. Utilização da CLI para criar e injetar a conta operacional com controle rigoroso da string de senha.

```powershell
$Password = ConvertTo-SecureString ".Fumaxu951." -AsPlainText -Force
New-ADUser -Name "Vitor Fernandes" -GivenName "Vitor" -Surname "Fernandes" -DisplayName "Vitor Fernandes (Hardclock)" -UserPrincipalName "vitor@senac.local" -Enabled $true -ChangePasswordAtLogon $false -AccountPassword $Password -Path "CN=Users,DC=SENAC,DC=LOCAL"

```

* **Solução de Problema Arquitetônico (Lockout de GPO):** Durante a fase de blindagem, a política para desativar o *Command Prompt* (`User Configuration > Policies > Administrative Templates > System > Prevent access to the command prompt`) foi aplicada acidentalmente na *Default Domain Policy*. Isso causou um efeito cascata que bloqueou o próprio `SENAC\Administrator`.
* **A Correção:** Acesso ao `gpmc.msc` via atalho de execução para reverter a GPO raiz. Criação de uma segmentação lógica com uma Unidade Organizacional (OU) chamada `Usuarios_Comuns`, movimentação da conta corporativa para esta OU e aplicação de uma nova política (`Bloqueio_CMD_Usuarios`) exclusivamente para este escopo. Finalizado forçando a atualização na rede com o comando `cmd /c gpupdate /force`.


* **Telemetria de SO (Blue Team):** Instalação do **Zabbix Agent 2** (versão 6.0 LTS) via instalador MSI, apontando as conexões ativas e passivas para o IP da Torre de Controle (`10.0.10.20`).
  * **Ativação no Servidor via Interface Gráfica Web (WebGUI):** Ativação via Interface Gráfica Web (WebGUI): Acessando a interface gráfica do Zabbix pelo navegador do Windows Server através do endereço `http://10.0.10.20/zabbix`, (IP máquina Ubuntu) o alvo foi configurado manualmente no menu `Configuration > Hosts`. O dispositivo foi associado ao template `Windows by Zabbix agent` e atrelado à interface `10.0.10.10`, validando o fluxo de dados quando o ícone ZBX acendeu na cor verde.

---

## 👁️ Fase 3: Blue Team e Monitoramento Leste-Oeste (Ubuntu Server)

Implementação do arsenal defensivo interno para garantir visibilidade total sobre ameaças lateralizadas e centralização de logs. A máquina foi configurada com IP estático (`10.0.10.20`).

* **Infraestrutura Zabbix (Stack LAMP):** Instalação de dependências base (Apache2, MySQL, PHP) e repositórios oficiais do Zabbix Server 6.0 LTS no Ubuntu.
* **Solução de Problema (Repositório Quebrado):** O `apt` retornou erro de pacote não encontrado ao tentar instalar o Zabbix via repositório de versões mais recentes (22.04/24.04). Identificou-se a versão exata do SO base executando `lsb_release -a` (retornando `20.04.6 LTS`), o que permitiu a injeção do repositório *Focal Fossa* correto (`zabbix-release_6.0-4+ubuntu20.04_all.deb`).


* **Configuração e Troubleshooting do Banco de Dados:** Criação estrutural do banco relacional `zabbix` com *character set* `utf8mb4`.
* **Solução de Problema (Interrupção e SUPER PRIVILEGE):** O processo de injeção da estrutura SQL via `zcat` não possui barra de progresso e foi abortado prematuramente, gerando o erro de corrupção `table role already exists` em tentativas subsequentes. Ao tentar corrigir, o MySQL retornou `ERROR 1419` devido à ausência de privilégios (`SUPER PRIVILEGE`) do usuário comum para criar *Triggers*. A correção definitiva exigiu acessar o MySQL como `root`, forçar a variável `log_bin_trust_function_creators = 1`, deletar o banco corrompido (`drop database zabbix;`) e reinjetar diretamente no banco base:


```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 zabbix

```


* **Orquestração de Serviços (Daemons):** Edição do arquivo de configuração nativo (`nano /etc/zabbix/zabbix_server.conf`) para descomentar e inserir a credencial do banco (`DBPassword=.Fumaxu951.`). Os serviços foram então ativados para inicialização automática junto ao SO:
```bash
systemctl restart zabbix-server zabbix-agent apache2
systemctl enable zabbix-server zabbix-agent apache2

```


* **Acesso Inicial WebGUI:** O *setup* final e o provisionamento da interface do Zabbix foram concluídos via navegador (`http://10.0.10.20/zabbix`), utilizando as credenciais padrão de fábrica (`Admin` / `zabbix`) para acesso ao *Dashboard*.
* **IDS Interno (Snort):** Instalação do pacote Snort no Linux (`apt-get install snort`) associado à interface local `enp0s3`. Durante a instalação via *ncurses*, o CIDR foi rigorosamente configurado para monitorar a rede `10.0.10.0/24`.
* **Solução de Problema (Cegueira Leste-Oeste e Erro BPF):** Inicialmente, o IDS não capturava tráfego lateral devido ao bloqueio natural do switch virtual. A arquitetura foi ajustada ativando o modo *Allow All* no hypervisor e forçando a placa no SO com `ip link set enp0s3 promisc on`. Um erro fatal de inicialização (`Can't set DAQ BPF filter`) causado por má interpretação de sintaxe foi corrigido reordenando os parâmetros e incluindo a flag obrigatória `-c`:


```bash
sudo snort -A console -c /etc/snort/snort.conf -i enp0s3

```


* **Validação Operacional:** Com o console do Snort em modo de escuta, testes de conectividade (ICMP Echo Request/Reply) foram disparados do Windows Server (`10.0.10.10`) para o Gateway (`10.0.10.1`), para o próprio Ubuntu (`10.0.10.20`) e para a WAN (`8.8.8.8`). Todos os fluxos Leste-Oeste foram interceptados e alertados com sucesso na tela do Blue Team.

---

### 🗄️ Detalhamento Arquitetônico do SGBD (MySQL)

A sustentação da Torre de Controle (Zabbix Server) exigiu a implementação e calibração de um Sistema Gerenciador de Banco de Dados (SGBD) relacional robusto. Optou-se pela utilização do **MySQL Server**, gerenciado diretamente via CLI no Ubuntu Server.

#### 1. Instalação e Inicialização do Banco de Dados

Após o provisionamento da stack LAMP, o SGBD foi instanciado e configurado para aceitar conexões locais de forma segura. O primeiro passo consistiu na criação do banco de dados dedicado e na definição do conjunto de caracteres (*character set*) adequado para evitar problemas de codificação de strings nos logs monitorados:

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY '.Fumaxu951.';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;

```

* **Análise Teórica:** A escolha do *collation* `utf8mb4_bin` foi estritamente necessária porque o Zabbix lida com strings binárias e metadados complexos vindos de múltiplos sistemas operacionais. O uso de privilégios granulares para o usuário `zabbix` restrito ao escopo `localhost` segue o Princípio do Menor Privilégio (PoLP).

#### 2. Análise Crítica do Erro de Injeção e Falha de Privilégio (ERROR 1419)

Durante a fase de população do esquema relacional utilizando o arquivo compactado do Zabbix, o comando foi abortado prematuramente pelo operador devido à ausência de uma barra de progresso nativa na CLI do Linux. Essa interrupção corrompeu o dicionário de dados do banco de dados, gerando o erro de tabela duplicada (`table role already exists`) em tentativas subsequentes.

Ao tentar reexecutar o processo limpo, o motor do MySQL barrou a operação retornando o seguinte erro fatal:

> `ERROR 1419 (HY000): You do not have the SUPER privilege and binary logging is enabled (you *might* want to use the less safe log_bin_trust_function_creators variable)`

* **O Diagnóstico Técnico:** Este erro ocorre porque a injeção do Zabbix (`server.sql.gz`) não contém apenas instruções DDL comuns (como `CREATE TABLE`), mas também comandos para a criação de **Triggers** e funções armazenadas. Como o MySQL opera por padrão com o log binário ativado (`log_bin`), o SGBD bloqueia a criação de funções por usuários comuns que não possuem o privilégio máximo de `SUPER`, visando impedir a replicação de códigos maliciosos ou não determinísticos nas transações do banco.

#### 3. Engenharia de Correção e Hardening do SGBD

Para sanar a corrupção e contornar a restrição de segurança com integridade, foi necessário elevar temporariamente os privilégios administrativos diretamente no console do `root` global do MySQL:

```bash
# 1. Acesso ao console administrativo como Root global
sudo mysql -u root -p

```

```sql
# 2. Desativação temporária da restrição de criação de funções/triggers
SET GLOBAL log_bin_trust_function_creators = 1;

# 3. Eliminação do esquema relacional corrompido e recriação limpa
DROP DATABASE zabbix;
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
QUIT;

```

Com o motor parametrizado para confiar nos criadores de funções, a injeção da estrutura de tabelas e restrições de chaves estrangeiras foi reexecutada com sucesso via pipeline do Linux, canalizando a descompactação diretamente para o binário de execução do MySQL:

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -u zabbix -p zabbix

```

#### 4. Hardening Pós-Injeção

Após a validação de que todas as tabelas foram criadas de forma íntegra, o parâmetro global de segurança foi reativado no MySQL para garantir a conformidade com as diretrizes de auditoria de sistemas e impedir modificações não autorizadas no esquema:

```sql
SET GLOBAL log_bin_trust_function_creators = 0;

```
---

## 💻 Fase 4: Red Team (Reconhecimento e Enumeração)

Testes ofensivos arquitetados para validar a eficácia das configurações de segurança base do Windows Server e a capacidade de resposta do IDS interno, partindo de um cenário de acesso local. (Atacante interno)

* **Setup do Atacante:** VM com Kali Linux introduzida na rede LAN (Host-Only). A interface de rede foi configurada em modo promíscuo (*Allow All*) e recebeu o IP `10.0.10.52` via DHCP do próprio pfSense, garantindo comunicação direta Leste-Oeste com os alvos internos.
* **Reconhecimento Ativo (Nmap):** Execução de um *SYN Stealth Scan* com detecção de SO e versões para mapear a infraestrutura sem concluir o *TCP Handshake*, reduzindo os rastros de conexão no host alvo.
```bash
sudo nmap -sS -sV -O 10.0.10.10

```


* **Análise do Alvo:** O Nmap identificou com sucesso o alvo como Windows Server 2019 e listou as portas críticas do *Active Directory* (88 Kerberos, 135/139 RPC/NetBIOS, 445 SMB, 5985 WinRM). A porta 389 (LDAP) revelou proativamente o nome do domínio: `SENAC.LOCAL`.
* **Validação do IDS:** O Snort no Ubuntu (Interface Promíscua) validou o *Port Mirroring* na rede LAN. Ao executar um scan nmap o Snort detectou e alertou.

* **Enumeração SMB (Null Session / Black Box):** Tentativa de exploração anônima para listar usuários, grupos e políticas do domínio.
```bash
enum4linux -a 10.0.10.10

```


* **Resultado Defensivo (Access Denied):** O laboratório comprovou na prática a evolução da segurança da Microsoft. A tentativa de Sessão Nula falhou (`NT_STATUS_ACCESS_DENIED`), demonstrando que o Windows Server 2019 bloqueia acesso anônimo ao IPC$ e as *Named Pipes* por padrão.
* **Vazamento Residual:** Apesar do bloqueio, o *Handshake* do protocolo SMB revelou metadados críticos: o NetBIOS do domínio e o SID primário da rede (`S-1-5-21-...`), dado essencial para a futura forja de um *Golden Ticket* em possíveis explorações futuras.

---

## 📚 Conceitos Acadêmicos Aplicados

* **Conectividade de Redes & Protocolos (TCP/IP):**
  * **Topologia e Roteamento:** Isolamento de redes virtuais (Bridged vs. Host-Only), configuração de Gateways, NAT, e reservas de escopo DHCP.
  * **Análise de Tráfego:** Diferenciação prática entre conexões completas e furtivas (*TCP SYN Stealth Scan*), captura de tráfego ICMP e inspeção de pacotes.
  * **Engenharia de Switching:** Compreensão do comportamento de Tabelas MAC em switches (físicos/virtuais) e a aplicação do Modo Promíscuo atrelado ao *Port Mirroring* para viabilizar a visibilidade de tráfego Leste-Oeste.


* **Infraestrutura e Sistemas Operacionais:**
  * **Windows Server (Microsoft):** Arquitetura e hierarquia de *Active Directory* (Floresta, NetBIOS, SID/RID), automação de provisionamento de identidades via **PowerShell**, estruturação de Unidades Organizacionais (OUs) e a evolução do hardening nativo (Bloqueio de IPC$ e *Null Sessions* a partir do Server 2012+).
  * **Linux (Ubuntu/Debian):** Gerenciamento de dependências via `apt` atrelado ao *release* do SO (*Focal Fossa*), orquestração de *Daemons* (`systemctl`), manipulação de interfaces de rede via CLI (`ip link`) e administração de SGBD contornando bloqueios de privilégios restritos (*SUPER PRIVILEGE* no MySQL).
  * **FreeBSD:** Administração de *appliances* de rede dedicados (pfSense) para gestão de borda.


* **Segurança da Informação e Auditoria:**
  * **Defense in Depth (Defesa em Profundidade):** Posicionamento estratégico de sensores IDS para cobrir vetores Norte-Sul (WAN) e Leste-Oeste (LAN).
  * **Controle de Acesso e PoLP:** Aplicação do Princípio do Menor Privilégio através de GPOs hierárquicas, segmentando usuários comuns de administradores de domínio (Bloqueio de *Command Prompt*).
  * **Gerenciamento Seguro:** Aplicação do conceito de gerência *Out-of-Band*, onde o firewall de borda só aceita configurações através de uma interface web isolada na LAN interna.
  * **Telemetria:** Monitoramento de disponibilidade de ativos via Zabbix Agent para respostas a incidentes de SOC.


* **Ethical Hacking e Red Teaming:**
  * **Footprinting & Scanning:** Identificação de alvos e *OS Fingerprinting* (detecção de SO e *builds* exatas) burlando firewalls básicos.
  * **Enumeration (Enumeração):** Exploração do protocolo SMB/RPC para extração de metadados críticos da rede (SIDs de domínio) e compreensão técnica do erro `NT_STATUS_ACCESS_DENIED` como resposta a varreduras anônimas.
  * **Mapeamento de Ameaças:** Entendimento prático da diferença entre ataques *Zero-Knowledge* (Atacante externo cego) e *Insider Threat* (Ameaça interna com credenciais de baixo privilégio via *Phishing*).
