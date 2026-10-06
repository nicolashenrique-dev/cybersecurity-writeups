# Biohazard — TryHackMe Writeup

> **Plataforma:** [TryHackMe — Biohazard](https://tryhackme.com/room/biohazard)  
> **Categoria:** Enumeração web · Puzzles · Codificações · Esteganografia · Linux  
> **Data da resolução:** 06/10/2026  
> **Resultado:** acesso por SSH, troca de usuário e obtenção de root.

Gostei muito de resolver esta room. Já conhecia e tinha jogado Resident Evil, então reconhecer os cenários, os personagens e a lógica de coletar itens para abrir novas áreas tornou a experiência especialmente divertida. Parabéns aos criadores por conectar a ambientação do jogo aos desafios técnicos.

Esta writeup registra meu caminho, incluindo os pontos em que precisei de dicas. Organizei as etapas por dependência para facilitar a leitura; durante a resolução, voltei várias vezes às salas já visitadas.

> **Spoilers:** há soluções, credenciais e flags do laboratório. Os comandos documentam uma instância autorizada do TryHackMe, cujo IP nesta sessão era `10.112.161.203`.

## Sumário

- [Visão geral](#visão-geral)
- [Reconhecimento](#reconhecimento)
- [Explorando a mansão](#explorando-a-mansão)
- [Os quatro crests e o FTP](#os-quatro-crests-e-o-ftp)
- [As imagens e a helmet key](#as-imagens-e-a-helmet-key)
- [Retorno às salas e acesso SSH](#retorno-às-salas-e-acesso-ssh)
- [De umbrella_guest a root](#de-umbrella_guest-a-root)
- [Erros e aprendizados](#erros-e-aprendizados)
- [Registro de evidências](#registro-de-evidências)

## Visão geral

```text
HTTP e comentários HTML
  → exploração das salas e coleta de itens
  → quatro crests → credenciais FTP
  → imagens: steghide, strings e binwalk
  → frase secreta → arquivo GPG → helmet key
  → novas salas → credenciais SSH
  → umbrella_guest → weasker → sudo → root
```

O desafio é um CTF de puzzles. Os itens e as portas da mansão fazem parte da mecânica do jogo: não estou classificando cada pista ou formulário como uma vulnerabilidade real de uma aplicação.

## Reconhecimento

Executei uma varredura TCP completa com SYN scan e scripts padrão:

```bash
nmap 10.112.161.203 -p- -sS -sC
```

```text
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
|_http-title: Beginning of the end
```

O scan identificou três portas abertas. Neste comando não utilizei `-sV`; portanto, a tabela não é um levantamento detalhado de versões. Mais tarde, a conexão FTP apresentou o banner `vsFTPd 3.0.3`.

![Resultado do Nmap](assets/Captura_de_tela_20261006_165244.png)

Comecei pela aplicação HTTP. Enquanto a varredura era executada, também fui navegando e inspecionando o código-fonte.

## Explorando a mansão

### Comentários HTML e mapa das salas

Um comentário indicava a primeira sala:

```html
<!-- It is in the /diningRoom/ -->
```

Na exploração seguinte encontrei uma pista codificada:

```text
SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=
```

A decodificação Base64 revelou `How about the /teaRoom/`. Na sala de chá, obtive o lock pick e uma indicação para visitar `/artRoom/`. Ali, um mapa ajudou a organizar as salas:

![Mapa das salas](assets/Captura_de_tela_20261006_155607.png)

| Sala | Papel na resolução |
|---|---|
| `/diningRoom/` | Emblema, pista cifrada e shield key |
| `/teaRoom/` | Lock pick e indicação da art room |
| `/artRoom/` | Mapa |
| `/barRoom/` | Partitura, emblema dourado e pista `rebecca` |
| `/diningRoom2F/` | Pista para a joia azul |
| `/tigerStatusRoom/` | Crest 1 |
| `/galleryRoom/` | Crest 2 |
| `/armorRoom/` | Crest 3 |
| `/attic/` | Crest 4 |
| `/studyRoom/` | Área revisitada após obter a helmet key |

Os nomes acima são os do mapa; algumas páginas internas usam caminhos adicionais, visíveis nas capturas.

### Partitura e emblema dourado

Na sequência do bar, encontrei uma partitura representada por uma string Base32. Ao decodificá-la, obtive o item `music_sheet`. Seguindo os links e a mecânica da sala, cheguei ao `gold_emblem`.

![Partitura codificada](assets/Captura_de_tela_20261006_161403.png)

O valor `rebecca`, obtido na interação com o puzzle dos emblemas, foi importante para a próxima etapa. A captura registra o emblema dourado e a orientação para retornar à página anterior:

![Emblema dourado](assets/Captura_de_tela_20261006_161830.png)

### Vigenère e shield key

Ao usar o emblema dourado na dining room, apareceu este texto:

```text
klfvg ks r wimgnd biz mpuiui ulg fiemok tqod. Xii jvmc tbkg ks tempgf tyi_hvgct_jljinf_kvc
```

Inicialmente suspeitei de ROT13, mas essa hipótese não resolveu a mensagem. Depois de consultar uma dica, reconheci a cifra de Vigenère e usei `rebecca` como chave.

O resultado indicava uma shield key na dining room e o nome da página `the_great_shield_key`.

![Vigenère com a chave rebecca](assets/Captura_de_tela_20261006_164550.png)

Essa etapa me ajudou a distinguir **codificação**, como Base64, de uma **cifra com chave**, como Vigenère.

### ROT13 e joia azul

No código-fonte de `/diningRoom2F/`, outra mensagem estava em ROT13. Ela orientava a procurar `sapphire.html` na dining room do primeiro andar. Seguindo essa indicação, obtive a blue jewel.

![Pista ROT13 no código-fonte](assets/Captura_de_tela_20261006_170242.png)

## Os quatro crests e o FTP

As salas forneciam quatro fragmentos que precisavam ser decodificados e reunidos na ordem indicada. A shield key permitiu avançar pela armor room e pelo attic.

| Fragmento | Sala | Decodificações, na ordem | Resultado |
|---|---|---|---|
| Crest 1 | Tiger status room | Base64 → Base32 | `RlRQIHVzZXI6IG` |
| Crest 2 | Gallery room | Base32 → Base58 | `h1bnRlciwgRlRQIHBh` |
| Crest 3 | Armor room | Base64 → binário → hexadecimal | `c3M6IHlvdV9jYW50X2h` |
| Crest 4 | Attic | Base58 → hexadecimal | `pZGVfZm9yZXZlcg==` |

O crest 3 tinha uma camada adicional: o primeiro resultado representava números binários; estes produziam uma sequência hexadecimal, que então revelava o fragmento.

Juntando os quatro resultados:

```text
RlRQIHVzZXI6IGh1bnRlciwgRlRQIHBhc3M6IHlvdV9jYW50X2hpZGVfZm9yZXZlcg==
```

A decodificação Base64 final revelou:

```text
FTP user: hunter, FTP pass: you_cant_hide_forever
```

Conectei ao serviço e baixei os arquivos individualmente:

```text
ftp 10.112.161.203
Name: hunter
Password: [senha obtida nos crests]
ftp> ls
ftp> get 001-key.jpg
ftp> get 002-key.jpg
ftp> get 003-key.jpg
ftp> get helmet_key.txt.gpg
ftp> get important.txt
```

![Login FTP e arquivos disponíveis](assets/Captura_de_tela_20261006_174046.png)

O arquivo `important.txt` relacionava o arquivo cifrado à helmet key e mencionava `/hidden_closet/`, uma área ainda bloqueada. Guardei essa pista para depois.

## As imagens e a helmet key

Cada imagem guardava um fragmento de outra mensagem. A técnica necessária variava entre os arquivos.

### 001-key.jpg: steghide

```bash
steghide extract -sf 001-key.jpg
cat key-001.txt
```

Resultado extraído:

```text
cGxhbnQ0Ml9jYW
```

![Extração com steghide](assets/Captura_de_tela_20261006_175005.png)

Meu log registra uma tentativa malsucedida e depois uma extração bem-sucedida. Como a entrada no prompt de passphrase não é exibida, não é possível reconstruí-la apenas pelo log.

### 002-key.jpg: strings

```bash
strings 002-key.jpg
```

Entre as strings legíveis estava o segundo fragmento:

```text
5fYmVfZGVzdHJveV9
```

![Fragmento encontrado com strings](assets/Captura_de_tela_20261006_174708.png)

Aqui, não precisei repetir a técnica usada no primeiro arquivo. A informação já aparecia na leitura das sequências de caracteres imprimíveis.

### 003-key.jpg: binwalk

Nesta etapa recorri a uma dica e conheci melhor o uso do Binwalk para identificar conteúdo incorporado a arquivos.

```bash
binwalk -e 003-key.jpg
```

A versão utilizada identificou um ZIP no offset decimal `1930`, equivalente a `0x78A`, com o arquivo `key-003.txt`. Embora houvesse um aviso na extração, o diretório gerado continha o texto que consegui ler:

```bash
cat _003-key.jpg.extracted/key-003.txt
```

```text
3aXRoX3Zqb2x0
```

Meu primeiro uso como root foi bloqueado pelo extrator. A execução registrada que avançou ocorreu como usuário comum. O nome da pasta e a saída correspondem ao Binwalk 2.x do meu ambiente.

### Reunindo a frase secreta

A concatenação produziu:

```text
cGxhbnQ0Ml9jYW5fYmVfZGVzdHJveV93aXRoX3Zqb2x0
```

Decodificando Base64:

```text
plant42_can_be_destroy_with_vjolt
```

Usei essa frase no prompt de descriptografia:

```bash
gpg --decrypt helmet_key.txt.gpg
```

Obtive a helmet key, necessária para continuar a exploração das salas.

![Descriptografia da helmet key](assets/Captura_de_tela_20261006_180118.png)

## Retorno às salas e acesso SSH

### Study room

Com a helmet key, retornei à study room. Ao examinar o livro, baixei `doom.tar.gz` e extraí seu conteúdo:

```bash
tar -xzvf doom.tar.gz
cat eagle_medal.txt
```

```text
SSH user: umbrella_guest
```

![Study room liberada](assets/Captura_de_tela_20261006_180219.png)

### Hidden closet e outra mensagem cifrada

Na exploração de `/hidden_closet/`, encontrei uma nova mensagem:

```text
wpbwbxr wpkzg pltwnhro, txrks_xfqsxrd_bvv_fy_rvmexa_ajk
```

![Hidden closet](assets/Captura_de_tela_20261006_181113.png)

Voltei à cifra de Vigenère e utilizei um resolvedor automático. Obtive a indicação de senha do usuário `weasker`:

```text
stars_members_are_my_guinea_pig
```

Mantive `weasker` porque essa é a grafia real da conta no laboratório, mesmo que o personagem do jogo seja conhecido como Wesker. Tentar usar essa senha para a conta SSH inicial não funcionou: as credenciais pertenciam a usuários diferentes.

### Login como umbrella_guest

A combinação que funcionou foi:

| Campo | Valor |
|---|---|
| Usuário | `umbrella_guest` |
| Senha | `T_virus_rules` |

A senha vinha do **wolf medal**, acessível pelo link **EXAMINE** na hidden closet, ao lado da leitura do MO disk. A captura da sala registra os dois links. A origem da senha foi conferida durante a organização desta writeup com uma [referência publicada](https://medium.com/@arthDetroja/tryhackme-room-based-on-game-evil-resident-writeup-biohazard-dad85ea8e692); o valor já constava nas minhas anotações. O arquivo `eagle_medal.txt`, extraído do TAR/GZip, fornecia o usuário, não a senha.

```bash
ssh umbrella_guest@10.112.161.203
id
```

```text
uid=1001(umbrella_guest) gid=1001(umbrella) groups=1001(umbrella)
```

O acesso inicial foi por credenciais, não por exploração de uma vulnerabilidade no serviço SSH.

## De umbrella_guest a root

### A pista em .jailcell

Listei os arquivos ocultos da home e encontrei `.jailcell`:

```bash
ls -la
cd .jailcell
cat chris.txt
```

O diálogo terminava com `MO disk 2: albert`, uma pista relacionada à cifra. No meu caminho, eu já havia usado o resolvedor automático antes de chegar aqui.

![Pista albert no arquivo chris.txt](assets/Captura_de_tela_20261006_183012.png)

### Troca para weasker

Em `/home`, encontrei os usuários `hunter`, `umbrella_guest` e `weasker`. Li `weasker_note.txt` e usei a senha obtida anteriormente para trocar de usuário:

```bash
su weasker
id
```

A saída mostrou `uid=1000(weasker)` e participação no grupo `sudo`.

![Troca de usuário bem-sucedida](assets/Captura_de_tela_20261006_183430.png)

### Conferindo a permissão sudo

```bash
sudo -l
```

Trecho relevante do log:

```text
User weasker may run the following commands on umbrella_corp:
    (ALL : ALL) ALL
```

Essa regra permitia executar qualquer comando como qualquer usuário e grupo. **Não era uma regra NOPASSWD:** o log mostra o pedido de senha. Como eu já tinha a credencial de `weasker`, consegui elevar os privilégios.

```bash
sudo su
cd /root
cat root.txt
```

![Shell root e conclusão do desafio](assets/Captura_de_tela_20261006_183733.png)

A leitura de `/root/root.txt` concluiu a sequência do laboratório.

<details>
<summary>Flag final — spoiler</summary>

```text
3c5794a00dc56c35f2bf096571edf3bf
```

</details>

## Erros e aprendizados

- **Nem todo texto estranho usa a mesma codificação.** Precisei distinguir Base32, Base58, Base64, ROT13 e Vigenère, além de respeitar a ordem das camadas.
- **Organizar os itens economiza tempo.** O mapa e a associação entre sala, chave e resultado evitaram perder o contexto ao voltar a áreas anteriores.
- **Dicas também fazem parte do aprendizado.** Usei ajuda em Vigenère e Binwalk; registrar isso deixa claro o que aprendi durante o desafio.
- **A ferramenta depende do arquivo.** `steghide`, `strings` e `binwalk` resolveram problemas diferentes nas três imagens.
- **Ler a saída é mais importante que presumir sucesso.** O Binwalk emitiu um aviso, mas havia um arquivo extraído para inspecionar.
- **Formato e estado do download importam.** Tentei `unzip` em um arquivo TAR/GZip. A extração correta foi com `tar -xzvf`. Renomear um `.crdownload` não garante que o download esteja completo, mesmo que neste caso a extração tenha funcionado.
- **Usuário e senha precisam permanecer associados.** Cheguei a inverter seu uso no SSH; a conexão correta usa `ssh usuario@host` e solicita a senha depois.
- **Grupo sudo e política sudo são coisas diferentes.** Confirmei a autorização efetiva com `sudo -l` antes de usar `sudo su`.
- **A origem da evidência precisa ser registrada.** Na próxima resolução, vou anotar a URL ou arquivo de cada credencial no momento em que a encontrar.

O principal ganho deste CTF foi combinar observação, organização e ferramentas simples para avançar por uma sequência longa de dependências. A ambientação em Resident Evil deixou esse processo muito mais envolvente para mim.

## Registro de evidências

As capturas ao longo do texto foram selecionadas para ilustrar decisões e resultados. A [galeria completa](EVIDENCIAS.md) preserva as 33 capturas fornecidas, com seus nomes originais. Os logs brutos e arquivos de trabalho permanecem na pasta local do laboratório; não foram incluídos aqui porque contêm ruído de terminal e listagens pessoais sem relação com a resolução.

---

*Por [Nicolas Henrique](https://github.com/nicolashenrique-dev). Room: [Biohazard](https://tryhackme.com/room/biohazard).*
