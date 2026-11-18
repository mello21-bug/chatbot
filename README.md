# 🤖 Assistente Virtual para Microempreendedores

Assistente virtual com **Inteligência Artificial** que apoia microempreendedores e pessoas que querem abrir um negócio, com foco no contexto local de **Garanhuns - PE**. O atendimento acontece pelo **WhatsApp**.

O sistema usa **RAG (Retrieval-Augmented Generation)**: antes de responder, o agente consulta uma base de conhecimento (cartilhas do Sebrae, legislação do MEI, gestão de pequenos negócios), o que torna as respostas mais contextualizadas e reduz o risco de a IA inventar informações.

Projeto desenvolvido no **PET Criativação, UFAPE**.

> 🚧 **Status:** protótipo funcional em desenvolvimento e testes.

---

## Sumário

1. [Como funciona](#-como-funciona)
2. [Tecnologias](#️-tecnologias)
3. [Arquitetura dos fluxos](#-arquitetura-dos-fluxos)
4. [Estrutura do repositório](#-estrutura-do-repositório)
5. [Como executar](#-como-executar)
6. [Credenciais necessárias](#-credenciais-necessárias)
7. [Segurança e privacidade](#-segurança-e-privacidade)
8. [Limitações conhecidas](#-limitações-conhecidas)
9. [Próximos passos](#-próximos-passos)

---

## 🔄 Como funciona

O workflow do n8n (`workflow-n8n.json`) contém **dois fluxos** que trabalham juntos:

1. **Alimentação da base:** quando um PDF é adicionado ou atualizado numa pasta do Google Drive, o conteúdo é extraído, dividido em fragmentos, transformado em *embeddings* e guardado no Supabase.
2. **Atendimento:** quando o usuário envia uma mensagem no WhatsApp, o agente de IA busca os trechos relevantes na base, usa o histórico da conversa e responde pelo próprio WhatsApp.

---

## ⚙️ Tecnologias

| Tecnologia | Função |
| --- | --- |
| **n8n** | Orquestra os dois fluxos (automação e agente de IA) |
| **WAHA** | Conecta o WhatsApp e expõe uma API para receber e enviar mensagens |
| **OpenAI (GPT-5)** | Gera as respostas do agente |
| **OpenAI Embeddings** | Transforma textos em vetores para a busca semântica |
| **Supabase + pgvector** | Armazena os fragmentos e faz a busca por similaridade |
| **Redis** | Memória das conversas (histórico por usuário) |
| **Google Drive** | Guarda os documentos que alimentam a base |
| **Docker** | Executa os serviços (n8n, WAHA, Redis) |

---

## 🧭 Arquitetura dos fluxos

### Fluxo 1: Alimentação da base de conhecimento

```
Google Drive (pasta "Pasta Fonte")
   │  verifica a cada 1 minuto: arquivo criado ou atualizado
   ▼
Download file → Extract from File (PDF)
   ▼
Supabase Vector Store (modo insert)
   ├─ Default Data Loader
   │     └─ Character Text Splitter (divide em fragmentos)
   └─ Embeddings OpenAI (gera os vetores)
```

### Fluxo 2: Atendimento pelo WhatsApp

```
Usuário (WhatsApp)
   ▼
WAHA ── webhook (POST) ──► n8n
                            │
   Webhook ► Dados ► Switch ► AI Agent ► HTTP Request ► HTTP Request1
                                │             │               │
                                │        sendSeen (lida)   sendText (resposta)
                                │             └──────── WAHA ───────► WhatsApp
                                │
                                ├─ OpenAI Chat Model (gpt-5)
                                ├─ Redis Chat Memory (histórico por usuário)
                                └─ Supabase Vector Store (busca na base, como ferramenta)
```

| Nó | O que faz |
| --- | --- |
| **Webhook** | Recebe (POST) as mensagens enviadas pelo WAHA |
| **Dados** | Extrai só o necessário: sessão, `chatId`, nome, id da mensagem, evento, texto e `fromMe` |
| **Switch** | Deixa seguir apenas eventos do tipo `message` |
| **AI Agent** | Recebe o texto, decide quando consultar a base e gera a resposta |
| **Redis Chat Memory** | Guarda o histórico usando o `chatId` como chave (expira em 1 hora, janela de 10 mensagens) |
| **Supabase Vector Store (tool)** | Busca semântica na base; o agente a usa quando a pergunta é sobre o tema |
| **HTTP Request** | Chama o WAHA (`/api/sendSeen`) para marcar a mensagem como lida |
| **HTTP Request1** | Chama o WAHA (`/api/sendText`) para enviar a resposta |

---

## 📁 Estrutura do repositório

```
chatbot/
├── README.md             # este arquivo
├── workflow-n8n.json     # workflow para importar no n8n (renomeie o arquivo atual)
├── docker-compose.yml    # [CONFIRME/ADICIONE] n8n, WAHA e Redis
└── .env.example          # [ADICIONE] modelo das variáveis (sem valores reais)
```

---

## 🚀 Como executar

### Pré-requisitos

- Docker e Docker Compose
- Conta no **Supabase** (projeto com a extensão `pgvector`)
- Chave de API da **OpenAI**
- Conta Google com acesso ao **Google Drive**
- Um número de WhatsApp para conectar ao WAHA (use um **número de teste**)

### Passo 1: Preparar o banco (Supabase)

No **SQL Editor** do Supabase, crie a extensão, a tabela e a função de busca usadas pelo n8n.
*(Exemplo padrão para embeddings da OpenAI, com 1536 dimensões. **[CONFIRME]** a dimensão do modelo que você usa. Se a tabela `documents` já existe, pule este passo.)*

```sql
create extension if not exists vector;

create table if not exists documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(1536)
);

create or replace function match_documents (
  query_embedding vector(1536),
  match_count int default null,
  filter jsonb default '{}'
) returns table (id bigint, content text, metadata jsonb, similarity float)
language plpgsql as $$
#variable_conflict use_column
begin
  return query
  select id, content, metadata,
         1 - (documents.embedding <=> query_embedding) as similarity
  from documents
  where metadata @> filter
  order by documents.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

### Passo 2: Subir os serviços

```bash
docker compose up -d
```

Os containers precisam estar na **mesma rede Docker**. O workflow acessa o WAHA em `http://waha:3000`, ou seja, pelo **nome do serviço**, não por `localhost`.

### Passo 3: Conectar o WhatsApp (WAHA)

1. Acesse o painel do WAHA (porta 3000 por padrão).
2. Inicie a sessão `default` e escaneie o **QR code** com o WhatsApp.
3. Configure o webhook do WAHA para o n8n, com o evento `message`:
   `http://n8n:5678/webhook/webhook`
4. Defina uma **chave de API forte** para o WAHA (não use valores padrão).

### Passo 4: Importar o workflow no n8n

1. Abra o n8n (`http://localhost:5678`).
2. Menu **⋯ → Import from file** e escolha `workflow-n8n.json`.
3. Crie as credenciais da tabela abaixo e associe-as aos nós.
4. Nos nós **HTTP Request** e **HTTP Request1**, configure a chave de API do WAHA.
5. No nó **Google Drive Trigger**, selecione **a sua pasta** de documentos.
6. **Ative** o workflow (interruptor *Active*).

### Passo 5: Testar

1. Coloque um **PDF** na pasta do Drive e aguarde cerca de 1 minuto. Confira em **Table Editor → documents** se os fragmentos apareceram.
2. Envie uma mensagem para o número conectado ao WAHA, por exemplo: *"Como me formalizo como MEI?"*
3. Acompanhe a execução em **Executions** no n8n.

---

## 🔑 Credenciais necessárias

Crie estas credenciais dentro do n8n. **Nunca coloque valores reais no repositório.**

| Credencial | Usada em |
| --- | --- |
| Google Drive OAuth2 | Gatilhos do Drive e download do arquivo |
| OpenAI API | Modelo de chat e embeddings |
| Supabase API | Armazenamento e busca vetorial |
| Redis | Memória das conversas |
| Chave de API do WAHA | Chamadas de envio de mensagem |

---

## 🔒 Segurança e privacidade

- **Segredos fora do repositório:** credenciais ficam no n8n e a chave do WAHA em variável de ambiente ou credencial, nunca escrita no workflow.
- **Antes de publicar o workflow:** exporte-o **sem dados de teste** (remova o bloco `pinData`), porque ele pode conter números de telefone, nomes e mensagens reais.
- **Privacidade (LGPD):** as conversas contêm dados pessoais. Defina por quanto tempo o histórico é guardado (hoje a memória no Redis expira em 1 hora).
- **Uso consciente de IA:** o assistente é apoio informativo e **não substitui** orientação de contador ou do Sebrae.

---

## ⚠️ Limitações conhecidas

- **Só PDFs:** o fluxo de alimentação extrai texto apenas de PDF.
- **Atualização de arquivo:** em modo `insert`, atualizar um documento pode gerar fragmentos duplicados na base.
- **Sem filtros de mensagem:** mensagens de grupos, mídias (áudio, imagem) e do próprio número ainda não são tratadas.
- **Webhook sem autenticação:** quem conhecer a URL pode enviar requisições falsas.
- **Dependência do formato do WAHA:** o nó *Dados* usa campos específicos do payload; mudar o motor do WAHA pode exigir ajustes.
- **Sem avaliação sistemática** da qualidade das respostas e **sem teste com usuários reais** ainda.
- **WAHA** é uma solução de integração não oficial com o WhatsApp; para produção, avaliar a API oficial.

---

## 🛣️ Próximos passos

- [ ] Realizar testes com microempreendedores reais
- [ ] Ampliar a base com materiais regionais
- [ ] Criar métricas para avaliar as respostas
- [ ] Encaminhar para atendimento humano quando necessário
- [ ] Filtrar mensagens (`fromMe`, grupos e mídias) no *Switch*
- [ ] Autenticar o webhook
- [ ] Evitar duplicatas ao atualizar documentos
- [ ] Marcar a mensagem como lida logo após o recebimento
- [ ] Reforçar o *system prompt*: escopo, "não sei" e orientação sem inventar leis ou valores

---

## 👥 Projeto

Desenvolvido pelo **PET Criativação, UFAPE**, para investigar o uso de agentes de IA como apoio ao desenvolvimento de microempreendedores no contexto local.

**Autoria:** Letícia (integração WhatsApp ↔ WAHA ↔ n8n e montagem do workflow) e **[NOME DO COLEGA]** (alimentação da base de conhecimento). **[AJUSTE conforme a divisão real]**

