

# 🚀 TryHackMe: Valenfield 

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

<img width="1201" height="268" alt="image" src="https://github.com/user-attachments/assets/5fa2df4a-11f1-4ad4-aa5b-ff6d7ecbf383" />


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

<img width="1421" height="647" alt="image" src="https://github.com/user-attachments/assets/5f0ef461-67c1-441c-b4be-0b6432a11258" />

A estrutura retornada aponta para um ecossistema clássico de autenticação de usuários.

---

## 2. Análise da Aplicação Web e Engenharia Social

Paralelamente à execução das ferramentas de automação, realizei a exploração manual da interface da aplicação. Trata-se de uma plataforma no estilo "aplicativo de namoro".

1. Realizei o cadastro de um usuário de testes na rota `/register`.
2. Efetuei a autenticação para acessar o painel principal (`/dashboard`).

<img width="1915" height="843" alt="Captura de tela 2026-06-23 104931" src="https://github.com/user-attachments/assets/1e5652d2-3c64-4572-a94a-88546ffe1cff" />

<img width="1917" height="861" alt="Captura de tela 2026-06-23 104949" src="https://github.com/user-attachments/assets/5f78accd-db94-49c6-9200-d287ded507e2" />

### A Pista Crucial

Navegando pelos perfis disponíveis na plataforma, encontrei uma conta sob o nome de usuário **"Cupid"**. Na biografia do perfil, constava a seguinte provocação:

> *"I keep the database secure. No peeking."* *(Eu mantenho o banco de dados seguro. Nada de espiar).*

<img width="1916" height="856" alt="image" src="https://github.com/user-attachments/assets/245ecdd4-f1dc-4e90-8e5b-7572e74a96f8" />

Esta mensagem funcionou como uma dica explícita de desenvolvimento seguro inadequado: o foco do desafio envolveria, de alguma forma, o vazamento ou a leitura não autorizada do banco de dados do servidor.

---

## 3. Engenharia de Vulnerabilidade: Explorando LFI / Path Traversal

Utilizando o **Burp Suite**, analisei minuciosamente o tráfego HTTP gerado ao carregar componentes de estilização da página de perfis. Identifiquei uma chamada suspeita direcionada à API interna:

`GET /api/fetch_layout?layout=theme_classic.html`

<img width="1001" height="183" alt="image" src="https://github.com/user-attachments/assets/e911d35d-e649-4910-ad9d-6a147e0eaab9" />
<img width="1577" height="696" alt="image" src="https://github.com/user-attachments/assets/e79021ec-8abb-459b-ae1d-23c8e03fab4b" />


### 3.1 Fundamentação Teórica da Falha

O parâmetro `?layout=` indica que a aplicação recebe uma string do usuário e a utiliza diretamente no backend para abrir e ler arquivos em disco (ex: arquivos de templates HTML). Se o input não for devidamente sanitizado, o sistema fica vulnerável a **Local File Inclusion (LFI)** e **Path Traversal**.

Ao injetar a sequência de retrocesso de diretórios (`../`), é possível forçar o interpretador a sair do diretório padrão do aplicativo e navegar pela raiz do sistema operacional Linux (`/`).

### 3.2 Validação do LFI

Enviei a requisição para o módulo **Repeater** do Burp Suite e injetei uma carga excessiva de `../` seguida do caminho do arquivo de usuários nativos do Linux (`/etc/passwd`):

```http
GET /api/fetch_layout?layout=../../../../../../../../etc/passwd
...

```

<img width="1648" height="663" alt="image" src="https://github.com/user-attachments/assets/b337ffca-f0eb-4a9d-9f63-6e78d7a12dd1" />

**Sucesso!** O servidor respondeu com o conteúdo íntegro do arquivo `/etc/passwd`. Além de confirmar o LFI, a leitura revelou a existência do usuário do sistema **`ubuntu`** (UID 1000).

---

## 4. Extração do Código-Fonte e Análise de Caixa Branca (White-Box)

Sabendo que o ecossistema é baseado em Python, o arquivo principal da aplicação costuma ser nomeado como `app.py` ou `main.py` no diretório raiz do projeto.

Utilizando a falha de Path Traversal recém-descoberta, tentei retroceder diretórios a partir da pasta de componentes web para ler o arquivo estrutural do servidor:

```http
GET /api/fetch_layout?layout=../app.py HTTP/1.1

```

<img width="1548" height="367" alt="image" src="https://github.com/user-attachments/assets/c3079e7e-7cdd-4d8e-92b6-d9adbd57dcc2" />
<img width="1692" height="656" alt="image" src="https://github.com/user-attachments/assets/a758305f-c150-4f93-980c-721fe5561e6b" />

O servidor interpretou o comando com sucesso e expôs o código-fonte completo do backend em texto claro na resposta HTTP.

### 4.1 Revisão de Código (Code Review)

Analisando a lógica de negócios codificada em `app.py`, localizei duas informações críticas de segurança:

1. **Credenciais Hardcoded:** No início do script, variáveis globais expunham a chave mestra de administração e o nome do banco de dados SQLite:
```python
ADMIN_API_KEY = "CUPID_MASTER_KEY_2024_XOXO"
DATABASE = 'cupid.db'

```


2. **Mapeamento de Rota Administrativa Privada:** Próximo ao fim do arquivo, foi identificada a rota `/api/admin/export_db`. Esta função permite o download direto do banco de dados, desde que a requisição forneça um cabeçalho HTTP customizado chamado `X-Valentine-Token` contendo a chave administrativa correspondente:

<img width="698" height="198" alt="image" src="https://github.com/user-attachments/assets/dc93c411-62fa-48d6-8d30-1cb9f18e106a" />

---

## 5. Controle de Acesso Insuficiente e Captura da Flag

Ao tentar acessar diretamente a rota administrativa no navegador (`http://10.64.162.121:5000/api/admin/export_db`), recebi, como esperado, uma mensagem de erro `403 Forbidden` devido à falta de credenciais no cabeçalho.

<img width="1703" height="501" alt="image" src="https://github.com/user-attachments/assets/1b22a556-9d72-44f8-89ed-cb2184e7ad7d" />

### 5.1 Forjando a Requisição no Burp Suite

Para contornar o bloqueio, utilizei o **Burp Suite Repeater** para estruturar uma requisição HTTP legítima direcionada à API de exportação, adicionando manualmente o token administrativo descoberto na fase de análise do código:

```http
GET /api/admin/export_db HTTP/1.1
Host: 10.64.162.121:5000
X-Valentine-Token: CUPID_MASTER_KEY_2024_XOXO

```

<img width="1638" height="718" alt="image" src="https://github.com/user-attachments/assets/d5f6bf74-fcc8-4ccf-8ed8-47c5fb5cec9d" />


### 5.2 Coleta do Artefato e Flag Final

Ao enviar a requisição forjada, o servidor validou com sucesso a assinatura do cabeçalho `X-Valentine-Token`. O backend respondeu com o cabeçalho de download e o conteúdo binário puro estruturado do banco de dados SQLite (**`SQLite format 3`**).

<img width="740" height="237" alt="image" src="https://github.com/user-attachments/assets/c9c79eef-f1bd-4927-941d-3d036b1c37b1" />

Inspecionando a resposta, localizei com sucesso a flag do desafio armazenada no banco.


---

*Desenvolvido por mesm0x.*
