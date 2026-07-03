#  TryHackMe: Valenfield 

Neste laboratório, apresento o passo a passo detalhado da exploração do ambiente e captura da flag na sala **Valenfield** da plataforma TryHackMe. O objetivo deste *write-up* é documentar a resolução de forma didática, detalhando a metodologia ofensiva aplicada e o raciocínio técnico por trás de cada vetor de ataque descoberto.

🔗 **Link da Sala:** [TryHackMe - Valenfield](https://tryhackme.com/room/lafb2026e10)

---

## 1. Reconhecimento e Coleta de Informações

O primeiro passo fundamental em qualquer atividade de segurança ofensiva é o mapeamento da superfície de ataque do alvo. Após estabelecer a conexão segura com a infraestrutura do laboratório via **VPN do TryHackMe**, iniciei o processo de varredura.

### 1.1 Varredura de Portas com Nmap

Utilizei o **Nmap** para identificar portas lógicas abertas e mapear as versões dos serviços em execução no host alvo:

```bash
nmap -sV -sC -Pn <IP_DA_SALA>

```

**Resultado do Scan:**

* **Porta 22/tcp:** OpenSSH 9.6p1 (Ubuntu Linux)
* **Porta 5000/tcp:** Werkzeug httpd 3.0.1 (Ambiente Python 3.12.3)

<img width="1201" height="268" alt="Captura de tela 2026-06-23 104321" src="https://github.com/user-attachments/assets/cd08bc11-9ed2-4eb1-a4f3-287e1991e358" />


A presença do servidor **Werkzeug** na porta 5000 indica fortemente que a aplicação web backend foi construída utilizando o framework **Flask** ou **FastAPI**.

### 1.2 Enumeração de Diretórios 

Com o serviço web mapeado na porta 5000 (`http://<IP_DO_ALVO>:5000`), iniciei uma varredura de força bruta em diretórios utilizando o **Gobuster** com uma wordlist padrão:

```bash
gobuster dir -u http://10.64.162.121:5000 -w /usr/share/wordlists/dirb/common.txt

```

**Resultado do Gobuster:**

* `/login` (Status: 200)
* `/register` (Status: 200)
* `/dashboard` (Status: 302 -> Redirecionamento para /login)
* `/logout` (Status: 302)

<img width="1421" height="648" alt="Captura de tela 2026-06-23 105221" src="https://github.com/user-attachments/assets/dd3a7cc3-3802-4384-b957-e7ed6ff97cbe" />

A estrutura retornada aponta para um ecossistema clássico de autenticação de usuários.

---

## 2. Análise da Aplicação Web e Engenharia Social

Paralelamente à execução das ferramentas de automação, realizei a exploração manual da interface da aplicação. Trata-se de uma plataforma no estilo "aplicativo de namoro".

1. Realizei o cadastro de um usuário de testes na rota `/register`.
2. Efetuei a autenticação para acessar o painel principal (`/dashboard`).

<img width="1917" height="861" alt="Captura de tela 2026-06-23 104949" src="https://github.com/user-attachments/assets/39ad1aaa-540f-4236-b278-f6a764c08559" />


### A Pista Crucial

Navegando pelos perfis disponíveis na plataforma, encontrei uma conta sob o nome de usuário **"Cupid"**. Na biografia do perfil, constava a seguinte provocação:

> *"I keep the database secure. No peeking."* *(Eu mantenho o banco de dados seguro. Nada de espiar).*

<img width="1916" height="856" alt="Captura de tela 2026-06-23 105021" src="https://github.com/user-attachments/assets/a97850b4-edd0-4b26-84c4-fcde6643ce0f" />

Esta mensagem funcionou como uma dica explícita de desenvolvimento seguro inadequado: o foco do desafio envolveria, de alguma forma, o vazamento ou a leitura não autorizada do banco de dados do servidor.

---

## 3. Engenharia de Vulnerabilidade: Explorando LFI / Path Traversal

Utilizando o **Burp Suite**, analisei minuciosamente o tráfego HTTP gerado ao carregar componentes de estilização da página de perfis. Identifiquei uma chamada suspeita direcionada à API interna:

`GET /api/fetch_layout?layout=theme_modern.html`

<img width="1577" height="696" alt="Captura de tela 2026-06-23 105456" src="https://github.com/user-attachments/assets/e40b17a5-c37f-49a0-9fff-da15fadb192e" />


### 3.1 Fundamentação Teórica da Falha

O parâmetro `?layout=` indica que a aplicação recebe uma string do usuário e a utiliza diretamente no backend para abrir e ler arquivos em disco (ex: arquivos de templates HTML). Se o input não for devidamente sanitizado, o sistema fica vulnerável a **Local File Inclusion (LFI)** e **Path Traversal**.

Ao injetar a sequência de retrocesso de diretórios (`../`), é possível forçar o interpretador a sair do diretório padrão do aplicativo e navegar pela raiz do sistema operacional Linux (`/`).

### 3.2 Validação do LFI

Enviei a requisição para o módulo **Repeater** do Burp Suite e injetei uma carga excessiva de `../` seguida do caminho do arquivo de usuários nativos do Linux (`/etc/passwd`):

```http
GET /api/fetch_layout?layout=../../../../../../../../etc/passwd
...

```
<img width="1649" height="664" alt="Captura de tela 2026-06-23 105852" src="https://github.com/user-attachments/assets/bd83bbe7-d05c-436d-a6a4-fe099b41cbec" />

**Sucesso!** O servidor respondeu com o conteúdo íntegro do arquivo `/etc/passwd`. Além de confirmar o LFI, a leitura revelou a existência do usuário do sistema **`ubuntu`** (UID 1000).

---

## 4. Extração do Código-Fonte e Análise de Caixa Branca (White-Box)

Sabendo que o ecossistema é baseado em Python, o arquivo principal da aplicação costuma ser nomeado como `app.py` ou `main.py` no diretório raiz do projeto.

Utilizando a falha de Path Traversal recém-descoberta, tentei retroceder diretórios a partir da pasta de componentes web para ler o arquivo estrutural do servidor:

```http
GET /api/fetch_layout?layout=../../app.py HTTP/1.1

```
<img width="1692" height="656" alt="Captura de tela 2026-06-23 110643" src="https://github.com/user-attachments/assets/4e535370-ac19-43a6-bca6-c6205b09cd2b" />

O servidor interpretou o comando com sucesso e expôs o código-fonte completo do backend em texto claro na resposta HTTP.

### 4.1 Revisão de Código (Code Review)

Analisando a lógica de negócios codificada em `app.py`, localizei duas informações críticas de segurança:

1. **Credenciais Hardcoded:** No início do script, variáveis globais expunham a chave mestra de administração e o nome do banco de dados SQLite:
```python
ADMIN_API_KEY = "CUPID_MASTER_KEY_2024_XOXO"
DATABASE = 'cupid.db'

```


2. **Mapeamento de Rota Administrativa Privada:** Próximo ao fim do arquivo, foi identificada a rota `/api/admin/export_db`. Esta função permite o download direto do banco de dados, desde que a requisição forneça um cabeçalho HTTP customizado chamado `X-Valentine-Token` contendo a chave administrativa correspondente:


---

## 5. Controle de Acesso Insuficiente e Captura da Flag

Ao tentar acessar diretamente a rota administrativa no navegador (`http://10.64.162.121:5000/api/admin/export_db`), recebi, como esperado, uma mensagem de erro `403 Forbidden` devido à falta de credenciais no cabeçalho.

<img width="1704" height="501" alt="Captura de tela 2026-06-23 110819" src="https://github.com/user-attachments/assets/978d8923-8f7a-4255-bb76-08b0e9b31803" />

### 5.1 Forjando a Requisição no Burp Suite

Para contornar o bloqueio, utilizei o **Burp Suite Repeater** para estruturar uma requisição HTTP legítima direcionada à API de exportação, adicionando manualmente o token administrativo descoberto na fase de análise do código:

```http
GET /api/admin/export_db HTTP/1.1
Host: 10.64.162.121:5000
X-Valentine-Token: CUPID_MASTER_KEY_2024_XOXO

```

<img width="1639" height="719" alt="Captura de tela 2026-06-23 111118" src="https://github.com/user-attachments/assets/c095c74f-2a61-4395-9ae7-f74df4caa165" />


### 5.2 Coleta do Artefato e Flag Final

Ao enviar a requisição forjada, o servidor validou com sucesso a assinatura do cabeçalho `X-Valentine-Token`. O backend respondeu com o cabeçalho de download e o conteúdo binário puro estruturado do banco de dados SQLite (**`SQLite format 3`**).

<img width="740" height="237" alt="Captura de tela 2026-06-23 111208" src="https://github.com/user-attachments/assets/74d5007d-0ddc-41fc-8e5f-0dfeb376706e" />


Inspecionando a resposta, localizei com sucesso a flag do desafio armazenada no banco.


---

*Desenvolvido por mesm0x.*
