# AXIS Chatbot

Chatbot de suporte da **AXIS**, um site que reúne pontos de carregamento de veículos elétricos e ajuda motoristas a encontrá-los. O assistente responde dúvidas sobre os lugares de carregamento, os preços, a disponibilidade e o funcionamento do site.

> A AXIS é a plataforma: ela não é dona nem opera os carregadores. Os carregadores pertencem a comerciantes parceiros, que cadastram seus pontos e preços no site.

Projeto desenvolvido para o challenge da FIAP.

## O que o bot faz

- Informa **preços** (por kWh e por minuto), **status** (disponível, ocupado, em manutenção) e **fila** dos carregadores.
- Informa endereço, tipo de conector e horário de funcionamento dos lugares.
- Lembra do que o usuário disse antes, dentro da mesma conversa.
- Recusa assuntos fora do escopo e nunca revela dados pessoais de clientes ou de comerciantes.
- Quando não tem uma informação, diz isso claramente em vez de inventar.

## Tecnologias

- **Python**
- **LangGraph** e **LangChain** para o fluxo do agente
- **Gemini** (`gemini-3.6-flash`) como modelo de linguagem
- **Supabase** (banco do site, via Lovable Cloud) para ler os dados dos carregadores
- **FastAPI** e **Uvicorn** para expor o bot como API
- **python-dotenv** para carregar as variáveis do `.env`

## Como funciona

O bot é um grafo do LangGraph com dois nós:

```
START → modelo → (pediu ferramenta?) → sim → tools → volta para o modelo
                                     → não → END
```

- **`modelo`**: envia o system prompt e o histórico da conversa ao Gemini.
- **`tools`**: executa a ferramenta `precos`, que consulta a tabela `chargers` no Supabase e devolve os dados ao modelo.
- **Memória**: cada conversa é identificada por um `thread_id`, e o histórico é guardado com o `MemorySaver`.

### Privacidade

- A ferramenta pede ao banco **somente** as colunas necessárias para responder ao usuário (nome, preços, status, fila, endereço, conector e horário). Dados internos e pessoais nem chegam ao modelo.
- Só lugares com `is_published = true` são exibidos.
- O bot usa apenas a chave **pública (publishable)** do Supabase, de leitura. Nunca use chaves de administrador (`service_role` / `sb_secret_`).
- O system prompt reforça as regras de privacidade, mas a proteção principal é não entregar o dado sensível ao modelo.

## Como rodar

**1. Clone o repositório e entre na pasta**

```bash
git clone <URL_DO_REPOSITORIO>
cd <NOME_DA_PASTA>
```

**2. Crie e ative o ambiente virtual**

```bash
python -m venv .venv
```

Windows (PowerShell):

```powershell
.venv\Scripts\activate
```

Linux / macOS:

```bash
source .venv/bin/activate
```

**3. Instale as dependências**

```bash
pip install -r requirements.txt
```

**4. Crie o arquivo `.env`** na raiz do projeto (veja a seção abaixo).

**5. Inicie a API**

```bash
uvicorn main:app --reload
```

A API sobe em `http://127.0.0.1:8000`. A documentação interativa, onde dá para testar o bot sem front-end, fica em `http://127.0.0.1:8000/docs`.

## Variáveis de ambiente

Crie um arquivo `.env` na raiz com as variáveis abaixo, preenchendo os valores no seu ambiente:

```
GOOGLE_API_KEY=
SUPABASE_URL=
SUPABASE_PUBLISHABLE_KEY=
```

- `GOOGLE_API_KEY`: chave da API do Gemini.
- `SUPABASE_URL`: endereço do banco (termina em `.supabase.co`).
- `SUPABASE_PUBLISHABLE_KEY`: chave pública de leitura do Supabase.

> **Nunca** faça commit do `.env`. Ele já está listado no `.gitignore`.

## Como usar a API

### `POST /chat`

Envia uma mensagem do usuário e recebe a resposta do bot.

**Corpo da requisição (JSON):**

```json
{
  "pergunta": "O carregador do Café Jardins está disponível?",
  "thread_id": "usuario-123"
}
```

| Campo | Tipo | Descrição |
| --- | --- | --- |
| `pergunta` | texto | A mensagem que o usuário digitou. |
|