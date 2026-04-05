# 📦 n8n — Gerenciador de Arquivos Dropbox

> Workflow n8n que autentica usuários via tela de login customizada, varre recursivamente um Dropbox Business e exibe os **top 50 arquivos** ranqueados por tamanho e antiguidade, tudo em uma interface web gerada diretamente pelo n8n.


## 📸 Preview
<img width="503" height="491" alt="Captura de Tela 2026-04-05 às 18 25 59" src="https://github.com/user-attachments/assets/033fab60-77f3-486c-be9c-1a3e9ea4401b" />


![Gravação de Tela 2026-04-05 às 18 30 13](https://github.com/user-attachments/assets/febd73c7-4376-43cc-8ab0-2cab291c67f8)

<img width="2639" height="8000" alt="Untitled design (95)" src="https://github.com/user-attachments/assets/19509f9e-e05a-4ba3-a31f-2341a877aa4e" />

---

## ✨ Funcionalidades

- **Tela de login customizada** servida diretamente pelo webhook n8n  
- **Autenticação via DataTable** (usuário + senha armazenados no n8n)  
- **Varredura recursiva** do Dropbox com paginação automática (`list_folder/continue`)  
- **Namespace Dropbox Business** — suporte a team folders via `root_namespace_id`  
- **Score inteligente** — arquivos ranqueados por `60% tamanho + 40% antiguidade`  
- **Filtro mínimo de 100 MB** para focar no que realmente ocupa espaço  
- **Pastas excluídas** configuráveis para ignorar áreas específicas  
- **Dashboard com uso de espaço** (total, usado, livre, % do time)  
- **Cópia de link Dropbox** direto da tabela com um clique  
- **Filtro e ordenação em tempo real** sem recarregar a página  

---

## 🏗️ Arquitetura do Fluxo

```
[GET /dropbox-lista]
       │
       ▼
 Respond to Webhook ──► Tela de Login (HTML)
                               │
                     POST /dropbox-auth
                               │
                               ▼
                         Get Login (DataTable)
                               │
                         Check Login (Code)
                               │
                    ┌──────────┴──────────┐
                    │ válido              │ inválido
                    ▼                    ▼
             GRANT ACCESS (200)    DENY ACCESS (401)
             Dashboard HTML        Tela login c/ erro
                    │
          POST /dropbox-processar
                    │
                    ▼
            Get Name Space (API)
                    │
               Edit Fields
                    │
             List Folders (API) ◄──────────────┐
                    │                          │
               Aggregator (Code)              Pages (API)
                    │                          │
              More Pages? ──── has_more=true ──┘
                    │ has_more=false
               Get Space (API)
                    │
              Space Used (Code)
                    │
              Output (Code) ──► HTML Final
                    │
          Respond to Webhook1 (200)
```

---

## 📋 Pré-requisitos

| Requisito | Versão mínima |
|---|---|
| n8n | `2.14+` |
| Node Dropbox OAuth2 API | nativo no n8n |
| DataTable (n8n feature) | nativo no n8n |

---

## ⚙️ Configuração Passo a Passo

### 1. Criar o App no Dropbox

1. Acesse [dropbox.com/developers/apps](https://www.dropbox.com/developers/apps)
2. **Create app** → API: **Scoped Access** → Acesso: **Full Dropbox**
3. Dê um nome (ex: `n8n-file-manager`) → **Create app**

### 2. Configurar o Redirect URI

Na aba **Settings** → **OAuth 2** → **Redirect URIs**, adicione:

```
https://<seu-dominio-n8n>/rest/oauth2-credential/callback
```

### 3. Ativar Permissões

Na aba **Permissions**, marque:

- `account_info.read`
- `files.metadata.read`
- `files.content.read`
- `sharing.read`

> Clique em **Submit** após marcar.

### 4. Criar a Credencial no n8n

1. **Credentials → New** → busque `Dropbox OAuth2 API`
2. Cole **App key** (Client ID) e **App secret** (Client Secret) da aba Settings do app
3. Clique em **Connect my account** → autorize no popup
4. Salve como **`Dropbox account`**

### 5. Criar a DataTable de Usuários

No n8n, crie uma **DataTable** chamada `dropbox` com as colunas:

| Coluna | Tipo   |
|--------|--------|
| `id`   | number |
| `user` | string |
| `pwd`  | string |

Adicione os usuários autorizados diretamente na tabela.

> ⚠️ **Segurança:** O campo `pwd` armazena a senha em texto plano. Para ambientes mais sensíveis, considere implementar hash (ex: bcrypt via Code node).

### 6. Importar o Workflow

1. No n8n, vá em **Workflows → Import from file**
2. Selecione o arquivo `Listar_Arquivos_Dropbox.json`
3. Após importar, configure os campos substituíveis abaixo

---

## 🔧 Campos a Substituir Após Importar

| Placeholder | Onde | O que colocar |
|---|---|---|
| `YOUR_CREDENTIAL_ID` | Nós com Dropbox | Selecione sua credencial `Dropbox account` na UI |
| `YOUR_DATATABLE_ID` | Nó `Get Login` | Selecione sua DataTable `dropbox` na UI |
| `YOUR_FOLDER_PATH` | Nó `List Folders` | Caminho raiz a varrer, ex: `""` (tudo) ou `"/Clientes"` |
| `YOUR_FOLDER_PATH/SubFolder1` | Nó `Aggregator` | Pastas a **excluir** da varredura (pode deixar vazio) |
| `YOUR_COMPANY` | HTML das telas | Nome da sua empresa para exibir nas telas |

> 💡 Os campos de credencial e DataTable podem ser selecionados graficamente na UI do n8n após importar — não é necessário editar o JSON manualmente.

---

## 🌐 Endpoints Gerados

| Método | Path | Descrição |
|---|---|---|
| `GET` | `/webhook/dropbox-lista` | Serve a tela de login |
| `POST` | `/webhook/dropbox-auth` | Valida usuário e senha |
| `POST` | `/webhook/dropbox-processar` | Executa a varredura e retorna o dashboard |

---

## 🔒 Considerações de Segurança

- **Senhas em plaintext** — a DataTable armazena senhas sem hash. Adequado para uso interno; para acesso externo, implemente hash.
- **`allowedOrigins: "*"`** — os webhooks aceitam qualquer origem. Restrinja ao seu domínio em produção se necessário.
- **Sem rate limiting** — o endpoint de auth não possui proteção contra brute force. Considere adicionar um nó de rate limit ou IP allowlist.
- **Sessão stateless** — não há token de sessão; o acesso ao dashboard é via chamada direta ao webhook. Ideal para redes internas.

---

## 🗂️ Estrutura do Repositório

```
.
├── Listar_Arquivos_Dropbox.json   # Workflow n8n (sanitizado)
├── README.md                      # Esta documentação
└── LICENSE                        # MIT License
```

---

## 🤝 Contribuindo

Pull requests são bem-vindos! Para mudanças maiores, abra uma issue primeiro para discutir o que você gostaria de mudar.

---

## 📄 Licença

MIT © Veja [LICENSE] para detalhes.
