# HdB-RIF — Relatório de Incidente Forense

[![Publicar documentação](https://github.com/MarcoA-013/HdB-RIF/actions/workflows/deploy.yml/badge.svg)](https://github.com/MarcoA-013/HdB-RIF/actions/workflows/deploy.yml)
[![MkDocs Material](https://img.shields.io/badge/docs-MkDocs%20Material-teal)](https://marcoa-013.github.io/HdB-RIF/)
[![GitHub Pages](https://img.shields.io/badge/site-GitHub%20Pages-blue)](https://marcoa-013.github.io/HdB-RIF/)
[![Licença](https://img.shields.io/badge/uso-educacional-green)](#)

> Estudo de caso completo de resposta a incidente e forense digital, cobrindo análise de tráfego de rede com Zeek e análise de disco com Autopsy.

---

## 📖 Documentação completa

**[marcoa-013.github.io/HdB-RIF](https://marcoa-013.github.io/HdB-RIF/)**

O site contém todas as análises com explicações didáticas, comandos comentados, interpretação dos resultados e referências técnicas — pensado para quem está iniciando em CTI e forense digital.

---

## O que este projeto cobre

### Parte 1 — Análise de Rede com Zeek
Análise de um PCAP capturado durante uma infecção real por **Emotet** (novembro de 2018). A evidência é proveniente do arquivo público [malware-traffic-analysis.net](https://www.malware-traffic-analysis.net/) (CTF 2018).

- Identificação do host comprometido via `dhcp.log`
- Rastreamento do vetor de entrada via `http.log` e `files.log`
- Análise do payload, módulo de spam e reconhecimento de Active Directory
- Relatório CTI com IoCs, mapeamento MITRE ATT&CK e Cyber Kill Chain

### Parte 2 — Análise Forense com Autopsy
Perícia sobre uma imagem de disco apreendida pela Delegacia de Crimes Cibernéticos. A imagem é o dataset público [NIST CFREDS](https://cfreds.nist.gov/) — notebook Dell Latitude CPi, Windows 98, ~2004.

- Verificação de integridade da imagem (hash MD5)
- Extração de artefatos do Windows (registro, histórico, programas executados)
- Identificação de ferramentas ofensivas e evidências de uso
- Construção de timeline forense
- Laudo Pericial formal

---

## Estrutura do repositório

```
HdB-RIF/
├── docs/                  ← fonte do site (Markdown)
│   ├── contexto/          ← Emotet, Greg Schardt, visão geral do caso
│   ├── ambiente/          ← configuração do lab (Zeek + Autopsy)
│   ├── parte1/            ← Q1–Q5 com comandos e interpretação
│   ├── parte2/            ← Q6–Q10 com comandos e interpretação
│   └── referencias/       ← MITRE ATT&CK, legislação, glossário
├── parte1_zeek/           ← IoCs exportados e logs de sessão
├── parte2_autopsy/        ← CSVs exportados do Autopsy (web history, timeline...)
├── roteiros/              ← roteiros de análise e melhorias
├── mkdocs.yml             ← configuração do site
└── requirements.txt       ← dependências Python
```

---

## Ferramentas utilizadas

| Ferramenta | Versão | Uso |
|---|---|---|
| Zeek | 6.0.4 | Análise de PCAP — `forenseLinux` |
| Autopsy | 4.21.0 | Análise forense de disco — `forenseWin` |
| MkDocs Material | 9.x | Documentação do projeto |

---

## Como rodar o site localmente

```bash
pip install -r requirements.txt
mkdocs serve
# Acesse http://localhost:8000
```

---

## Aviso

As evidências utilizadas neste projeto são dados públicos disponibilizados para fins educacionais:
- PCAP: [malware-traffic-analysis.net](https://www.malware-traffic-analysis.net/) — créditos a Brad Duncan
- Imagem de disco: [NIST CFREDS](https://cfreds.nist.gov/)

Todo o conteúdo é disponibilizado para fins educacionais.

---

*por [@MarcoA-013](https://github.com/MarcoA-013)*- **Q7** Proprietário, conta, último desligamento e último logon
- **Q8** Programas suspeitos e arquivos na lixeira
- **Q9** Timeline das evidências
- **Q10** Laudo pericial formal

---

## Estrutura do Repositório

```
HdB-RIF/
├── README.md                   ← este arquivo
├── parte1_zeek/
│   ├── iocs_emotet.txt         ← IoCs exportados da análise de rede
│   └── sessao_*.log            ← gravação de sessão terminal
└── parte2_autopsy/
    ├── hash_evidencia.csv      ← hash MD5 da imagem E01
    ├── sessao_*.log            ← log de sessão PowerShell
    ├── web_history.csv         ← histórico de navegação (887 itens)
    ├── run_programs.csv        ← programas executados (81 entradas)
    ├── shell_bags.csv          ← Shell Bags (51 entradas)
    └── timeline_schardt.csv    ← timeline exportada do Autopsy
```

---

## Pré-requisitos

| Componente | Versão utilizada | Observação |
|-----------|-----------------|-----------|
| Zeek | 6.0.4 | Instalado em `/opt/zeek/bin/` |
| Autopsy | 4.21.0 | Windows 64-bit |
| Java | 1.8.0_391 (64-bit) | Requisito do Autopsy |
| p7zip-full | qualquer | Necessário se `unzip` não suportar AES-256 |

---

## Parte 1 — Análise de PCAP com Zeek

**VM:** `forenseLinux` | **IP:** `192.168.98.10`

### Checklist do Ambiente

```bash
# 1. Verificar versão do Zeek
/opt/zeek/bin/zeek --version
# Saída esperada: Zeek version 4.x.x ou superior

# Se o comando não for encontrado:
find / -name "zeek" -type f 2>/dev/null
export PATH=$PATH:/opt/zeek/bin
echo 'export PATH=$PATH:/opt/zeek/bin' >> ~/.bashrc

# 2. Verificar zeek-cut
/opt/zeek/bin/zeek-cut --help | head -5

# 3. Verificar evidência
ls -lh /opt/estudodecaso/
# Esperado: 2018-CTF-from-malware-traffic-analysis.net-2-of-2.pcap.zip (~12M)

# 4. Espaço em disco (Zeek gera ~10% do PCAP em logs)
df -h ~/ && df -h /opt/estudodecaso/
# Mínimo necessário: 200 MB livres

# 5. RAM disponível
free -h

# 6. Usuário atual
whoami && id
```

> **Problema comum:** `unsupported compression method 99` ao descompactar.
> O InfoZip nativo não suporta AES-256. Solução:
> ```bash
> sudo apt-get install p7zip-full -y
> 7z x -Pinfected /opt/estudodecaso/*.zip -o~/analise_caso/
> ```

### Boas Práticas Forenses

```
┌──────────────────────────────────────────────────────────────────────┐
│ NUNCA processe direto em /opt/estudodecaso/ — trabalhe em cópia.     │
│ DOCUMENTE timestamp de cada comando (use script ou tee).             │
│ NÃO altere permissões ou atributos dos arquivos extraídos.           │
│ REGISTRE os hashes ANTES e DEPOIS de cada operação.                  │
│ Qualquer anomalia inesperada: anote antes de tentar corrigir.        │
└──────────────────────────────────────────────────────────────────────┘
```

```bash
# Criar diretório de trabalho isolado
mkdir -p ~/analise_caso && cd ~/analise_caso

# Iniciar gravação de sessão
script -a ~/analise_caso/sessao_$(date +%Y%m%d_%H%M%S).log

# Hashes do ZIP original
echo "[$(date)] Hash do arquivo original:"
md5sum /opt/estudodecaso/2018-CTF-from-malware-traffic-analysis.net-2-of-2.pcap.zip
sha256sum /opt/estudodecaso/2018-CTF-from-malware-traffic-analysis.net-2-of-2.pcap.zip
```

### Extração do PCAP

```bash
cd ~/analise_caso/

unzip -P infected \
  /opt/estudodecaso/2018-CTF-from-malware-traffic-analysis.net-2-of-2.pcap.zip \
  -d ~/analise_caso/

# Verificar e registrar hash do PCAP extraído
ls -lh ~/analise_caso/*.pcap
echo "[$(date)] Hash do PCAP extraído:"
md5sum ~/analise_caso/*.pcap
sha256sum ~/analise_caso/*.pcap
```

### Processamento com Zeek

```bash
cd ~/analise_caso/
/opt/zeek/bin/zeek -r *.pcap

# Zeek processa silenciosamente — nenhuma saída é o comportamento normal
# Verificar logs gerados:
ls -lhS *.log
```

> **Logs típicos de infecção via HTTP:**
> ```
> conn.log       ← todas as conexões
> http.log       ← requisições web
> dns.log        ← consultas DNS
> files.log      ← arquivos transferidos
> ssl.log        ← conexões TLS
> dhcp.log       ← negociações DHCP
> smtp.log       ← conexões e-mail
> pe.log         ← análise de executáveis
> smb_files.log  ← acessos SMB
> x509.log       ← certificados TLS
> ```

```bash
# Confirmar campos disponíveis em cada log antes de usar zeek-cut
grep "^#fields" dhcp.log
grep "^#fields" http.log
grep "^#fields" conn.log
grep "^#fields" files.log
```

> **Problema com PCAP em nanosegundos:**
> ```bash
> capinfos *.pcap | grep "File type"
> editcap -F pcap *.pcap pcap_convertido.pcap
> /opt/zeek/bin/zeek -r pcap_convertido.pcap
> ```

---

### Q1 — Endereço MAC do cliente em 172.17.1.129

```bash
grep "^#fields" dhcp.log   # confirmar nome do campo de IP nesta versão

/opt/zeek/bin/zeek-cut client_addr mac < dhcp.log \
  | grep "172.17.1.129" | sort -u
```

> **Resultado:**
> ```
> 172.17.1.129    00:1e:67:4a:d7:5c
> ```
> Prefixo OUI `00:1e:67` → **Intel** (NIC física ou virtualização não-VMware).

---

### Q2 — Hostname do cliente em 172.17.1.129

```bash
/opt/zeek/bin/zeek-cut client_addr mac host_name < dhcp.log \
  | grep "172.17.1.129" | sort -u
```

> **Resultado:**
> ```
> 172.17.1.129    00:1e:67:4a:d7:5c    Nalyvaiko-PC
> ```
> Hostname com padrão de usuário (não gerenciado por TI) — indica possível BYOD.
> Domínio corporativo `kyivartworks.com` identificado no `dhcp.log`.

---

### Q3 — URL que retornou um documento Microsoft Word

```bash
# Panorama geral dos MIME types
/opt/zeek/bin/zeek-cut resp_mime_types < http.log \
  | grep -v "^#\|-" | sort | uniq -c | sort -rn

# Extrair URL que entregou o Word doc
/opt/zeek/bin/zeek-cut host uri resp_mime_types < http.log \
  | grep -i "msword\|openxmlformats\|vnd.ms-word" \
  | awk '{print "http://" $1 $2, "\t[" $3 "]"}'
```

> **Resultado:**
> ```
> http://ifcingenieria.cl/QpX8It/BIZ/Firmenkunden/    [application/msword]
> ```
> - `ifcingenieria.cl` — site legítimo chileno comprometido como hospedeiro do dropper.
> - Caminho `/QpX8It/` com aparência de ID aleatório — padrão de painel C2.
> - `Firmenkunden` (alemão: "clientes corporativos") — isca de engenharia social direcionada.
> - URL sem extensão `.doc` — arquivo servido dinamicamente, evita detecção por extensão.

> **Se `resp_mime_types` retornar vazio (`-`):**
> ```bash
> # Alternativa 1: buscar por extensão na URI
> /opt/zeek/bin/zeek-cut host uri < http.log | grep -i "\.doc\|\.docx\|\.dot\b"
>
> # Alternativa 2: mime_type identificado pelo Zeek via magic bytes (mais confiável)
> /opt/zeek/bin/zeek-cut tx_hosts rx_hosts mime_type filename < files.log \
>   | grep -i "msword\|openxml\|word"
> ```

---

### Q4 — Tipo de Infecção

> **Resultado: Emotet** — Trojan bancário com módulo de spam e reconhecimento de Active Directory.

#### Passo 1 — Hashes dos arquivos transferidos

```bash
/opt/zeek/bin/zeek -r *.pcap \
  /opt/zeek/share/zeek/policy/frameworks/files/hash-all-files.zeek

/opt/zeek/bin/zeek-cut mime_type filename md5 sha1 < files.log \
  | grep -v "^#" | sort -u
```

> **Resultado:**
> ```
> application/msword    2018_11Details_zur_Transaktion.doc    -    -
> application/x-dosexec  6169583.exe                         -    -
> text/ini   kyivartworks.com\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\gpt.ini   -   -
> ```
> Hashes `-` por truncamento do PCAP. Os nomes são IoCs suficientes.
> - `.doc` em alemão ("Detalhes da Transação Nov 2018") — isca financeira.
> - `6169583.exe` — payload de segundo estágio, nome numérico aleatório (padrão Emotet).
> - `gpt.ini` via SMB → acesso ao GPO do AD `kyivartworks.com` (reconhecimento/lateral).

#### Passo 2 — Análise do executável (pe.log)

```bash
/opt/zeek/bin/zeek-cut ts machine compile_ts os subsystem is_exe < pe.log | grep -v "^#"
```

> **Resultado:**
> ```
> 1542056529.402998    I386    1542051170.000000    Windows 2000    WINDOWS_GUI    T
> ```
> - Arquitetura I386 (32-bit) — compatibilidade máxima.
> - `compile_ts: 1542051170` = **12/11/2018 ~19:32 UTC** — compilado ~1h30 antes da infecção.
> - Subsistema `WINDOWS_GUI` — banker trojan sem janela de console.

#### Passo 3 — Módulo de spam (smtp.log)

```bash
/opt/zeek/bin/zeek-cut ts id.orig_h id.resp_h mailfrom rcptto < smtp.log | grep -v "^#"
```

> **Resultado:** 11 conexões SMTP de `172.17.1.129` para 4 servidores distintos ~1h após a infecção.
> `108.177.98.108` (Google/Gmail — 7x), `185.93.245.68`, `41.204.202.10`, `123.125.50.11`.

#### Passo 4 — Reconhecimento de AD (smb_files.log)

```bash
/opt/zeek/bin/zeek-cut ts id.orig_h id.resp_h action name < smb_files.log | grep -v "^#"
```

> **Resultado:** 4 acessos `SMB::FILE_OPEN` ao GPO de `kyivartworks.com` no DC `172.17.1.2` entre 21:01 e 22:27 UTC.

#### Passo 5 — Infraestrutura TLS (x509.log)

```bash
/opt/zeek/bin/zeek-cut id certificate.subject certificate.issuer < x509.log | grep -v "^#"
```

> **Servidores identificados:** `smtp.gmail.com`, `msmtp.bigticket.ae`, `*.qiye.163.com`, `*.cpt2.host-h.net`, `cloud.q1whosting.com`.
> Diversidade de SMTP em múltiplos países — característica do módulo de spam do Emotet.

#### Linha do Tempo Confirmada

```
12/11/2018
~19:32 UTC  →  6169583.exe compilado (pe.log compile_ts)
~21:01 UTC  →  Doc Word baixado: ifcingenieria.cl/QpX8It/BIZ/Firmenkunden/
~21:01 UTC  →  SMB FILE_OPEN no GPO do AD kyivartworks.com — 1ª e 2ª vez
~22:17 UTC  →  SMB FILE_OPEN — 3ª vez
~22:27 UTC  →  SMB FILE_OPEN — 4ª vez
~22:30 UTC  →  Módulo spam ativo: 11 conexões SMTP para 4 servidores externos
```

---

### Q5 — Relatório CTI

#### IoCs Confirmados

```bash
cat > ~/analise_caso/iocs_emotet.txt << 'EOF'
=== CASO: Good Money Financial | Emotet | 12/11/2018 ===

ATIVO COMPROMETIDO
IP:        172.17.1.129
MAC:       00:1e:67:4a:d7:5c
Hostname:  Nalyvaiko-PC
Domínio:   kyivartworks.com

VETOR DE ENTRADA
URL:       http://ifcingenieria.cl/QpX8It/BIZ/Firmenkunden/
Arquivo:   2018_11Details_zur_Transaktion.doc
MIME:      application/msword

PAYLOAD
Arquivo:   6169583.exe
Tipo:      PE32 I386 WINDOWS_GUI
Compilado: 12/11/2018 ~19:32 UTC

C2 / SPAM
108.177.98.108   (Google/Gmail  — 7 conexões SMTP)
185.93.245.68    (1 conexão SMTP)
41.204.202.10    (1 conexão SMTP)
123.125.50.11    (1 conexão SMTP)

INFRAESTRUTURA TLS (x509.log)
smtp.gmail.com / msmtp.bigticket.ae
*.qiye.163.com / *.cpt2.host-h.net / cloud.q1whosting.com

RECONHECIMENTO AD (smb_files.log)
DC:    172.17.1.2
GPO:   kyivartworks.com\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\gpt.ini
Acessos: 4x entre 21:01 e 22:27 UTC
EOF
```

#### Tabela de IoCs para o Relatório

| Tipo | Valor | Fonte | Confiança |
|------|-------|-------|-----------|
| IP vítima | 172.17.1.129 | dhcp.log | Alta |
| MAC | 00:1e:67:4a:d7:5c | dhcp.log | Alta |
| Hostname | Nalyvaiko-PC | dhcp.log | Alta |
| Domínio AD | kyivartworks.com | dhcp/smb | Alta |
| URL maliciosa | http://ifcingenieria.cl/QpX8It/BIZ/Firmenkunden/ | http.log | Alta |
| Doc lure | 2018_11Details_zur_Transaktion.doc | files.log | Alta |
| Payload | 6169583.exe (PE32 I386, comp. 12/11/2018) | files.log | Alta |
| IP C2/spam | 108.177.98.108 | smtp.log | Alta |
| IP C2/spam | 185.93.245.68 | smtp.log | Alta |
| IP C2/spam | 41.204.202.10 | smtp.log | Alta |
| IP C2/spam | 123.125.50.11 | smtp.log | Alta |
| DC alvo | 172.17.1.2 (GPO kyivartworks.com) | smb_files.log | Alta |
| TLS C2 | smtp.gmail.com / msmtp.bigticket.ae | x509.log | Média |
| TLS C2 | *.qiye.163.com / cloud.q1whosting.com | x509.log | Média |

#### Mapeamento MITRE ATT&CK

| ID | Tática | Técnica | Evidência |
|----|--------|---------|-----------|
| T1566.001 | Initial Access | Phishing: Spearphishing Link | URL ifcingenieria.cl → http.log |
| T1204.002 | Execution | User Execution: Malicious File | Abertura do .doc com macro |
| T1059.005 | Execution | Command & Scripting: Visual Basic | Macro VBA no Word |
| T1547.001 | Persistence | Registry Run Keys / Startup Folder | Comportamento padrão Emotet |
| T1071.001 | C2 | Application Layer Protocol: Web | Conexões SMTP smtp.log |
| T1021.002 | Lateral Movement | Remote Services: SMB/Admin Shares | SMB FILE_OPEN no GPO — smb_files.log |
| T1078 | Defense Evasion | Valid Accounts | Emotet exfiltra credenciais locais |
| T1568 | C2 | Dynamic Resolution | Múltiplos IPs/domínios SMTP |

> Referência: https://attack.mitre.org/groups/G0046/ (Mealybug — operador Emotet)

#### Cyber Kill Chain

```
Reconnaissance    →  Alvo: kyivartworks.com / Good Money Financial
Weaponization     →  Macro VBA em .doc + 6169583.exe compilado no mesmo dia
Delivery          →  HTTP GET ifcingenieria.cl (site legítimo comprometido)
Exploitation      →  Usuário abre .doc → macro executa
Installation      →  6169583.exe instalado + persistência no registro
Command & Control →  11 conexões SMTP para Gmail, 163.com, bigticket.ae
Actions on Obj.   →  Spam + Reconhecimento AD + Propagação lateral potencial
```

#### Recomendações Imediatas

- Isolar `172.17.1.129` (preservar imagem de memória antes de desligar)
- Bloquear IPs C2 no firewall: `108.177.98.108`, `185.93.245.68`, `41.204.202.10`, `123.125.50.11`
- Bloquear domínio `ifcingenieria.cl` no proxy corporativo
- Varrer demais hosts da rede `172.17.0.0/16` com os mesmos IoCs
- Auditar o controlador de domínio `172.17.1.2` — verificar GPOs e contas comprometidas
- Resetar credenciais de todas as contas que fizeram logon em `Nalyvaiko-PC`
- Desabilitar macros Office via GPO em todo o domínio `kyivartworks.com`

---

## Parte 2 — Análise Forense com Autopsy

**VM:** `forenseWin` | **IP:** `192.168.98.30` | **Acesso:** RDP → usuário `Aluno`

### Checklist do Ambiente

```powershell
# 1. Versão do Autopsy
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" |
  Where-Object DisplayName -like "*Autopsy*" |
  Select-Object DisplayName, DisplayVersion
# Resultado: Autopsy 4.21.0

# 2. Java instalado (Autopsy exige JRE 64-bit)
java -version
# Resultado: java version "1.8.0_391"

# 3. Espaço em disco
Get-PSDrive C | Select-Object Used, Free
# Mínimo: 4 GB livres

# 4. Segmentos da imagem
Get-Item "C:\curso\estudodecaso\*" | Select-Object Name, Length, LastWriteTime
```

> **Ferramentas adicionais disponíveis no desktop da VM:**
> `FTK Imager` | `Nmap / Zenmap GUI` | `NetworkMiner` | `DB Browser (SQLite)` | `AccessData`

### Boas Práticas Forenses

```powershell
# Marcar imagens como read-only (executar após download do E02)
Set-ItemProperty "C:\curso\estudodecaso\4Dell Latitude CPi.E01" -Name IsReadOnly -Value $true
Set-ItemProperty "C:\curso\estudodecaso\4Dell Latitude CPi.E02" -Name IsReadOnly -Value $true

# Hash da imagem antes da análise
New-Item -ItemType Directory -Path "C:\curso\case_schardt" -Force
$hashAntes = Get-FileHash "C:\curso\estudodecaso\4Dell Latitude CPi.E01" -Algorithm MD5
$hashAntes | Export-Csv "C:\curso\case_schardt\hash_evidencia.csv" -NoTypeInformation

# Log de sessão
Start-Transcript -Path "C:\curso\case_schardt\sessao_$(Get-Date -Format 'yyyyMMdd_HHmmss').log" -Append
```

### Download do Segundo Segmento (E02)

```powershell
# Opção 1 — PowerShell nativo
Invoke-WebRequest `
  -Uri "https://cfreds-archive.nist.gov/images/4Dell%20Latitude%20CPi.E02" `
  -OutFile "C:\curso\estudodecaso\4Dell Latitude CPi.E02" `
  -UseBasicParsing

# Opção 2 — curl (se download travar)
curl.exe -L `
  -o "C:\curso\estudodecaso\4Dell Latitude CPi.E02" `
  "https://cfreds-archive.nist.gov/images/4Dell%20Latitude%20CPi.E02"

# Confirmar e marcar como read-only
Get-Item "C:\curso\estudodecaso\*" | Select-Object Name, Length
Set-ItemProperty "C:\curso\estudodecaso\4Dell Latitude CPi.E02" -Name IsReadOnly -Value $true
```

---

### Q6a — Integridade da Imagem (Hash MD5)

Hash esperado: `aee4fcd9301c03b3b054623ca261959a`

```powershell
Get-FileHash "C:\curso\estudodecaso\4Dell Latitude CPi.E01" -Algorithm MD5 |
  Select-Object Hash, Path

# Alternativa via certutil
certutil -hashfile "C:\curso\estudodecaso\4Dell Latitude CPi.E01" MD5
```

> **Resultado:** `AEE4FCD9301C03B3B054623CA261959A` — imagem íntegra.
> Hash divergente invalida a cadeia de custódia — não prosseguir e registrar a divergência.

---

### Configuração do Case no Autopsy

```
Autopsy → New Case
  Case Name:      GoodMoneyFinancial_Schardt
  Base Directory: C:\curso\case_schardt\

Add Data Source → Image File
  Path: C:\curso\estudodecaso\4Dell Latitude CPi.E01
  (Autopsy localiza o E02 automaticamente no mesmo diretório)

Módulos de ingestão — selecionar:
  ✅ Hash Lookup
  ✅ File Type Identification
  ✅ Recent Activity
  ✅ Keyword Search
  ✅ Windows Registry
  ✅ Extension Mismatch
  ✅ Encryption Detection
```

> A ingestão pode levar 10–30 minutos. Aguardar 100% em todos os módulos antes de gerar a timeline.

---

### Q6b — Sistema Operacional

```
Autopsy → Data Sources → [imagem] → vol → Windows\System32\config\SOFTWARE
  → Microsoft → Windows NT → CurrentVersion
  Campos: ProductName | CurrentVersion | CurrentBuildNumber | RegisteredOwner | InstallDate

Alternativa:
  Autopsy → Results → Operating System Information
```

> **Resultado esperado:** `Windows 98` / `Windows 98 Second Edition` (Dell Latitude CPi, ~1998).

---

### Q7 — Proprietário, Conta, Desligamento e Último Logon

#### Q7a — Proprietário Registrado

```
SOFTWARE → Microsoft → Windows NT → CurrentVersion
  Valor: RegisteredOwner
```
> **Resultado:** `Greg Schardt`

#### Q7b — Nome da Conta

```
SYSTEM → ControlSet001 → Control → ComputerName → ComputerName
  Valor: ComputerName
```

#### Q7c — Último Desligamento

```
SYSTEM → ControlSet001 → Control → Windows
  Valor: ShutdownTime  (FILETIME — Autopsy converte automaticamente)

Alternativa:
  Results → Recent Activity → Shutdown Time
```

#### Q7d — Último Usuário a Fazer Logon

```
SOFTWARE → Microsoft → Windows NT → CurrentVersion → Winlogon
  Valor: DefaultUserName  ou  LastLoggedOnUser

Alternativa: verificar datas de modificação em Windows\Profiles\
```

---

### Q8a — Programas Instalados com Potencial Malicioso

```
Autopsy → Results → Installed Programs

Registro manual:
  SOFTWARE → Microsoft → Windows → CurrentVersion → Uninstall
  SOFTWARE → Microsoft → Windows → CurrentVersion → App Paths

Sistema de arquivos (portáteis sem entrada no registro):
  Autopsy → Views → File Types → Executables
  (listar .exe e .dll fora dos diretórios padrão do SO)
```

> Classificar como: **Suspeitos** (scanners, sniffers, crackers), **Abusáveis** (admin remoto, VPN), **Normais**.

---

### Q8b — Arquivos na Lixeira

```
Autopsy → Results → Recycle Bin

Verificação manual:
  vol → RECYCLER  (Windows 98/2000/XP)
  (arquivo INFO2 contém metadados: nome original, data, tamanho)
```

> - Arquivo na lixeira → **não** permanentemente excluído (recuperável).
> - Arquivo em `unallocated space` → excluído da lixeira (pode ser recuperável).

---

### Q9 — Timeline das Evidências

```
Autopsy → Tools → Timeline → Generate Timeline
  → Selecionar intervalo relevante
  → List View para exportação

Autopsy → Timeline → Export → CSV
  Salvar: C:\curso\case_schardt\timeline_schardt.csv
```

> A timeline consolida: MAC times de arquivos, entradas de registro, histórico de browser,
> Recent Documents, eventos de logon e Prefetch (execuções de programas).
> Filtrar no Excel pela data do incidente e dias anteriores para identificar preparação.

---

### Artefatos Adicionais (Blocos de Melhoria)

#### Web History (887 itens)

```
Autopsy → Data Artifacts → Web History
→ Save Table as CSV → web_history.csv

Termos de busca: hack | crack | password | tool | exploit | scan | sniff
Sites de interesse: neworder.box.sk, netcat.be, packetstormsecurity.com
```

#### Run Programs (81 entradas)

```
Autopsy → Data Artifacts → Run Programs
→ Save Table as CSV → run_programs.csv

Colunas: Program Name | Run Count | Last Run | Arguments
```

> **Diferença crítica para o laudo:** `Installed Programs` prova instalação; `Run Programs` prova **uso efetivo**.

#### USB Device Attached (1 dispositivo)

```
Autopsy → Data Artifacts → USB Device Attached
→ Anotar: fabricante, produto, número de série, primeira e última conexão
```

> Verificar se a data de conexão coincide com datas de atividade suspeita (possível exfiltração).

#### Shell Bags (51 entradas)

```
Autopsy → Data Artifacts → Shell Bags
→ Save Table as CSV → shell_bags.csv

Colunas: Folder Path | Last Write
```

> Shell Bags registram pastas abertas no Explorer — mesmo após deleção dos arquivos.

#### Hashes das Evidências Digitais (prints)

```powershell
Get-ChildItem *.png | ForEach-Object {
  $h = Get-FileHash $_.FullName -Algorithm MD5
  "$($_.Name) | $($h.Hash)"
}
```

---

### Q10 — Laudo Pericial

#### Arcabouço Legal Aplicável

```
Lei nº 12.737/2012 (Lei Carolina Dieckmann)
  Art. 154-A — Invasão de dispositivo informático alheio
  Pena: detenção de 3 meses a 1 ano + multa

Lei nº 14.155/2021
  Agrava penas do art. 154-A quando praticado contra instituições financeiras
  Pena: reclusão de 1 a 4 anos + multa

Decreto-Lei nº 2.848/1940 — Código Penal
  Art. 153 — Divulgação de segredo
  Art. 299 — Falsidade ideológica (se aplicável)

Normas técnicas:
  ABNT NBR ISO/IEC 27037:2013 — Identificação, coleta, aquisição e preservação de evidência digital
  ABNT NBR ISO/IEC 27042:2015 — Análise e interpretação de evidência digital
  RFC 3227 — Guidelines for Evidence Collection and Archiving
  NIST SP 800-86 — Guide to Integrating Forensic Techniques into Incident Response
```

#### Estrutura do Laudo

```
LAUDO PERICIAL Nº: [XXX/2026/DCC]
Processo:         [número]
Data de Autuação: [data]
Solicitante:      Delegacia de Crimes Cibernéticos
Perito:           [nome] — Matrícula [número]

1. INTRODUÇÃO
   - Identificação do perito e autoridade requisitante
   - Suspeito: Greg Schardt
   - Contexto: incidente Good Money Financial → investigação policial → apreensão

2. MÉTODO
   Equipamento: Notebook Dell Latitude CPi
   SO:          [Q6b]
   Hash MD5:    aee4fcd9301c03b3b054623ca261959a
   Ferramenta:  Autopsy [versão]
   Módulos:     [listar ativados]

3. COLETA DE EVIDÊNCIAS
   - Imagem E01+E02 com hash verificado
   - Registros Windows extraídos
   - Timeline gerada e exportada

4. ANÁLISE E RESULTADOS
   Q6: Integridade confirmada. SO: [resultado]
   Q7: Proprietário / Conta / Desligamento / Último logon
   Q8a: Programas suspeitos com caminhos e timestamps de execução
   Q8b: Arquivos na lixeira com metadados completos
   Q9: Timeline — eventos correlacionados ao incidente

5. LIMITAÇÕES DA ANÁLISE
   - PCAP truncado (hashes indisponíveis — Parte 1)
   - Timestamps em horário local sem fuso confirmado
   - Dados voláteis (RAM, processos, rede) não coletados
   - Web History parcialmente analisada (887 itens totais → web_history.csv)
   - As duas partes são casos independentes para fins didáticos

6. DISCUSSÃO E CONCLUSÕES
   - Correlação entre evidências do notebook e o incidente na Good Money Financial
   - Achados que vinculam Greg Schardt ao crime investigado

7. ANEXOS
   - Hash MD5 verificado (Q6a)
   - Prints de cada achado de registro (Q7)
   - Lista de programas suspeitos com timestamps de execução (Q8a)
   - Lista completa da lixeira (Q8b)
   - Timeline CSV (Q9)
   - web_history.csv / run_programs.csv / shell_bags.csv
```

---

## Glossário

| Sigla | Significado |
|-------|------------|
| AD | Active Directory |
| C2 | Command and Control |
| CTI | Cyber Threat Intelligence |
| DHCP | Dynamic Host Configuration Protocol |
| DLL | Dynamic-Link Library |
| GPO | Group Policy Object |
| IoC | Indicator of Compromise |
| MAC | Media Access Control |
| MIME | Multipurpose Internet Mail Extensions |
| PCAP | Packet Capture |
| PE | Portable Executable |
| SMB | Server Message Block |
| SMTP | Simple Mail Transfer Protocol |
| TTP | Tactic, Technique and Procedure |
| VBA | Visual Basic for Applications |

---

## Log de Evidências

### Parte 1 — Zeek

| # | Data | Ação | Resultado | Arquivo |
|---|------|------|-----------|---------|
| 1 | 11/06/2026 | Hash ZIP original | (registrar) | sessao_20260611.log |
| 2 | 11/06/2026 | Hash PCAP extraído | (registrar) | sessao_20260611.log |
| 3 | 11/06/2026 | Zeek processado | Zeek 6.0.4, logs gerados | — |
| 4 | 11/06/2026 | Q1 — MAC | `00:1e:67:4a:d7:5c` | print_q1.png |
| 5 | 11/06/2026 | Q2 — Hostname | `Nalyvaiko-PC` | print_q2.png |
| 6 | 11/06/2026 | Q3 — URL Word doc | `http://ifcingenieria.cl/QpX8It/BIZ/Firmenkunden/` | print_q3.png |
| 7 | 12/06/2026 | Q4 — Tipo de infecção | **Emotet** | print_q4.png |
| 8 | 12/06/2026 | IoCs exportados | 4 IPs C2, 1 URL, 2 arquivos, 1 DC | iocs_emotet.txt |
| 9 | — | Q5 — Relatório CTI | A redigir | relatorio_cti.pdf |

### Parte 2 — Autopsy

| # | Data | Ação | Resultado | Arquivo |
|---|------|------|-----------|---------|
| 1 | — | Download E02 concluído | Tamanho: — | — |
| 2 | — | Hash MD5 verificado | ✅ bate com `aee4fcd9...` | print_hash.png |
| 3 | — | Case Autopsy criado | C:\curso\case_schardt | — |
| 4 | — | Ingestão concluída | Módulos: — | — |
| 5 | — | Q6a — Integridade | — | print_q6a.png |
| 6 | — | Q6b — SO | — | print_q6b.png |
| 7 | — | Q7a — Proprietário | — | print_q7a.png |
| 8 | — | Q7b — Nome conta | — | print_q7b.png |
| 9 | — | Q7c — Desligamento | — | print_q7c.png |
| 10 | — | Q7d — Último logon | — | print_q7d.png |
| 11 | — | Q8a — Prog. suspeitos | — | print_q8a.png |
| 12 | — | Q8b — Lixeira | — | print_q8b.png |
| 13 | — | Web History exportado | 887 itens | web_history.csv |
| 14 | — | Run Programs exportado | 81 entradas | run_programs.csv |
| 15 | — | Shell Bags exportado | 51 entradas | shell_bags.csv |
| 16 | — | USB Device Attached | 1 dispositivo | print_usb.png |
| 17 | — | Q9 — Timeline exportada | — | timeline_schardt.csv |
| 18 | — | Q10 — Laudo redigido | — | laudo_pericial.pdf |

---

*Caso: Good Money Financial — Análise conduzida em junho de 2026*
