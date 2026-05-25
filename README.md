# 🛡️ Full-Stack Cybersecurity Lab: Red & Blue Team Operations

## 📌 Objetivo do Projeto

Este projeto consiste na arquitetura, implementação e documentação de um laboratório completo de cibersegurança (Full-Stack). O objetivo é simular um ambiente corporativo real, aplicando na prática conceitos avançados de infraestrutura de TI, Sistemas Operacionais, Protocolos de Redes (TCP/IP), Auditoria de Sistemas e Ethical Hacking.

A infraestrutura foi desenhada sob a ótica de **Defense in Depth** (Defesa em Profundidade), permitindo a execução de operações ofensivas (Red Team) e a validação contínua da detecção e telemetria (Blue Team).

## 🏗️ Topologia e Arquitetura de Rede

O ambiente foi construído sobre um hypervisor utilizando o isolamento tático de redes para separar o tráfego de internet (Norte-Sul) do tráfego interno da rede local (Leste-Oeste). O Modo Promíscuo foi ativado ("Permitir Tudo") na rede LAN virtual para viabilizar o *Port Mirroring* necessário para o Sistema de Detecção de Intrusão (IDS).

| Máquina / Papel | SO / Tecnologia | IP (Rede Interna) | Função na Arquitetura |
| --- | --- | --- | --- |
| **Gateway / Perimeter** | pfSense (FreeBSD) | `10.0.10.1` | Roteamento, Firewall, Servidor DHCP e IDS de Borda (Snort WAN). |
| **Coração Corporativo** | Windows Server 2019 | `10.0.10.10` (Estático) | Controlador de Domínio (`SENAC.LOCAL`), GPOs e Alvo Primário. |
| **Torre de Controle** | Ubuntu Server 20.04 LTS | `10.0.10.20` (Estático) | Monitoramento Zabbix (Stack LAMP) e IDS Interno (Snort LAN). |
| **Ameaça (Red Team)** | Kali Linux | `10.0.10.52` (DHCP) | Máquina do atacante para Reconhecimento, Enumeração e Exploração. |

---

## 🛠️ Fase 1: Perímetro e Roteamento (pfSense)

A fundação do laboratório começou pelo isolamento e proteção da borda, aplicando os conceitos de *Gateways* e segurança de perímetro.

* **Configuração de Interfaces:** Identificação precisa das interfaces WAN (`em0` / Bridged) e LAN (`em1` / Host-Only) utilizando os endereços MAC do hypervisor para evitar inversão de rotas.
* **Serviços Base:** Configuração do IP do Gateway LAN para `10.0.10.1` e ativação do serviço DHCP com range reservado (`10.0.10.50` a `10.0.10.200`), deixando os primeiros IPs livres para os servidores.
* **IDS de Perímetro:** Instalação do pacote nativo do **Snort** via WebGUI para atuar no monitoramento do tráfego Norte-Sul (Internet ↔ LAN).
* **Solução de Problema (Boot Loop):** O ambiente entrou em loop de instalação contínua. A correção foi realizada ejetando a mídia ISO virtual via hypervisor para forçar o boot pelo disco rígido recém-gravado.

## 🏢 Fase 2: O Coração Corporativo (Active Directory)

Simulação de uma infraestrutura corporativa robusta com políticas de identidade e restrição de privilégios.

* **Promoção a DC:** Instalação das funções *AD DS* e *DNS Server*. Criação da floresta `SENAC.LOCAL` e configuração da credencial administrativa padrão.
* **Automação DevSecOps:** Abandono da interface gráfica (ADUC) para provisionamento de usuários. Utilização do PowerShell para criar e injetar a conta "Vitor Fernandes" diretamente na raiz do domínio com controle de senha em texto seguro.

```powershell
$Password = ConvertTo-SecureString "SENHA_AQUI" -AsPlainText -Force
New-ADUser -Name "Vitor Fernandes" -GivenName "Vitor" -Surname "Fernandes" -DisplayName "Vitor Fernandes" -UserPrincipalName "vitor@senac.local" -Enabled $true -ChangePasswordAtLogon $false -AccountPassword $Password -Path "CN=Users,DC=SENAC,DC=LOCAL"

```

* **Solução de Problema (Lockout de GPO):** Uma GPO bloqueando o *Command Prompt* foi aplicada acidentalmente na raiz do domínio, bloqueando o próprio Administrador. A correção arquitetônica exigiu reverter a política raiz, criar uma Unidade Organizacional (OU) chamada `Usuarios_Comuns`, mover a conta corporativa para lá e aplicar a GPO de restrição exclusivamente nesta OU.
* **Telemetria de SO:** Instalação do **Zabbix Agent** via MSI e apontamento para a Torre de Controle (`10.0.10.20`), validando a comunicação com o ZBX verde.

## 👁️ Fase 3: Blue Team e Monitoramento Leste-Oeste

Implementação do arsenal defensivo interno para garantir visibilidade total sobre ameaças lateralizadas.

* **Infraestrutura Zabbix (Stack LAMP):** Instalação do Apache2, MySQL, PHP e repositórios do Zabbix Server 6.0 LTS no Ubuntu.
* **Solução de Problema (Repositório Quebrado):** Erro de pacote não encontrado ao tentar instalar o Zabbix via repositório de versões mais recentes. Identificação da versão exata do SO (`lsb_release -a` retornando `20.04.6 LTS`) e injeção do repositório *Focal Fossa* adequado.
* **Solução de Problema (MySQL ERROR 1419 - SUPER PRIVILEGE):** O script de importação do banco de dados do Zabbix falhou devido à ausência de privilégios para criar *Triggers*. A correção envolveu o acesso como `root`, a exclusão do banco corrompido (`drop database zabbix;`) e a reinjeção contornando o bloqueio de segurança:

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 zabbix

```

* **IDS Interno (Snort):** Configuração do Snort no Linux para inspecionar tráfego não direcionado diretamente a ele.
* **Solução de Problema (Cegueira do IDS e Erro BPF):** O Snort não capturava tráfego Leste-Oeste devido ao bloqueio do switch virtual; resolvido ativando o modo *Allow All* no VirtualBox e o comando `ip link set enp0s3 promisc on` no Ubuntu. O erro fatal de inicialização `Can't set DAQ BPF filter` foi corrigido reordenando a sintaxe e incluindo a flag obrigatória `-c`:

```bash
sudo snort -A console -c /etc/snort/snort.conf -i enp0s3

```

## 🥷 Fase 4: Red Team (Reconhecimento e Enumeração)

Testes ofensivos para validar a eficácia das configurações de segurança do Windows Server e a detecção do IDS.

* **Reconhecimento Ativo (Nmap):** Execução de um *SYN Stealth Scan* para mapear a infraestrutura sem concluir o *TCP Handshake*.

```bash
sudo nmap -sS -sV -O 10.0.10.10

```

* **Análise do Alvo:** O Nmap identificou com sucesso o alvo como Windows Server 2019, listou portas críticas expostas (88 Kerberos, 445 SMB, 5985 WinRM) e vazou o nome do domínio através da porta 389 LDAP (`SENAC.LOCAL`).
* **Validação do IDS:** O Snort no Ubuntu interceptou a varredura ruidosa, validando o *Port Mirroring* na rede LAN. No entanto, o scan stealth (-sS) não disparou alertas severos, evidenciando a necessidade de implementação de regras (ET Rules) focadas em pré-processadores de *portscan*.
* **Enumeração SMB (Null Session):** Tentativa de listar usuários, grupos e políticas via ferramenta genérica de SMB.

```bash
enum4linux -a 10.0.10.10

```

* **Resultado Defensivo (Access Denied):** O laboratório comprovou na prática a evolução da segurança da Microsoft. A tentativa de *Null Session* falhou (`NT_STATUS_ACCESS_DENIED`), demonstrando que o Windows Server 2019 bloqueia acesso anônimo ao IPC$ por padrão.
* **Vazamento Residual:** Apesar do bloqueio, o *Handshake* do protocolo Server/SMB vazou informações sensíveis: a *build* exata do sistema operacional e o SID primário da rede (`S-1-5-21-...`), dados valiosos para uma futura exploração de *Golden Ticket*.

## 📚 Conceitos Acadêmicos Aplicados

* **Conectividade de Redes & Protocolos:** Roteamento de pacotes, DHCP, análise de tráfego ICMP, TCP SYN vs. Full Connect, Broadcast vs. Promiscuous Mode.
* **Sistemas Operacionais:** Administração de FreeBSD (pfSense), Windows Server (AD, GPO, PowerShell) e Linux (Ubuntu, gerenciamento de pacotes, manipulação de daemons, controle de privilégios MySQL).
* **Segurança e Auditoria:** Principle of Least Privilege (PoLP) através de GPOs, segmentação de rede (VLAN/Host-Only), logs de eventos e telemetria (Zabbix).
* **Ethical Hacking:** OSINT simulado, *Footprinting*, *Scanning*, *Enumeration* (SMB/RPC), bypass de defesas básicas e identificação de vetores de ataque em Active Directory.

---



Quando terminar de criar o seu repositório, avise-me! Podemos despausar o laboratório e seguir exatamente de onde paramos: testar a existência dos usuários utilizando o script agressivo de Kerberos do Nmap.
