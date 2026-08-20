# 🤖 Assistente Virtual para Microempreendedores

Assistente virtual baseado em **Inteligência Artificial** desenvolvido para apoiar microempreendedores e pessoas interessadas em iniciar um negócio, com foco no contexto local de **Garanhuns - PE**.

O sistema utiliza **RAG (Retrieval-Augmented Generation)** para consultar uma base de conhecimento especializada antes de gerar suas respostas, permitindo fornecer informações mais contextualizadas sobre empreendedorismo, MEI, legislação e gestão de pequenos negócios.

## 🎯 Objetivo

Facilitar o acesso a informações relevantes para microempreendedores por meio de um assistente conversacional integrado ao **WhatsApp**, aproximando tecnologias de Inteligência Artificial das necessidades da comunidade local.

## ⚙️ Tecnologias

* **n8n** — automação e orquestração dos fluxos
* **OpenAI / GPT-5** — geração das respostas
* **RAG** — recuperação de informações da base de conhecimento
* **Supabase + pgvector** — armazenamento e busca vetorial
* **Google Drive** — armazenamento dos documentos
* **Redis** — memória das conversas
* **WAHA** — integração com WhatsApp
* **Docker** — infraestrutura dos serviços

A arquitetura e os componentes utilizados são descritos no projeto de pesquisa e implementados no workflow disponibilizado neste repositório.

## 🔄 Como funciona

### 1. 📚 Alimentação da base

Documentos são armazenados no Google Drive. Quando um arquivo é **adicionado ou atualizado**, o workflow é acionado automaticamente.

O conteúdo é extraído, dividido em fragmentos e transformado em embeddings. Esses dados são então armazenados no **Supabase**, permitindo consultas por similaridade semântica.

### 2. 💬 Atendimento

O usuário envia uma mensagem pelo WhatsApp.

```text
WhatsApp
    ↓
WAHA
    ↓
n8n
    ↓
Redis → Histórico da conversa
    ↓
Supabase → Busca na base de conhecimento
    ↓
OpenAI → Geração da resposta
    ↓
WAHA
    ↓
WhatsApp
```

O agente recupera informações relevantes da base de conhecimento e utiliza o histórico da conversa para gerar uma resposta contextualizada.

## 🧠 Arquitetura RAG

A utilização de RAG permite que o modelo consulte documentos previamente selecionados antes de formular uma resposta.

A base de conhecimento pode conter materiais como:

* Cartilhas do Sebrae;
* Legislação relacionada ao MEI;
* Informações sobre empreendedorismo;
* Práticas comerciais;
* Informações econômicas do contexto local.

Isso permite reduzir a dependência do conhecimento geral do modelo e gerar respostas mais contextualizadas.

## 📁 Estrutura do Workflow

O workflow do n8n contempla:

* Monitoramento de arquivos no Google Drive;
* Extração de conteúdo;
* Divisão dos documentos em fragmentos;
* Geração de embeddings;
* Armazenamento no Supabase;
* Recebimento de mensagens via webhook;
* Gerenciamento da memória com Redis;
* Agente de Inteligência Artificial;
* Consulta à base vetorial;
* Envio das respostas pelo WhatsApp.

## 📊 Status do projeto

🚧 **Protótipo funcional em desenvolvimento e testes.**

Os testes iniciais demonstraram que o sistema consegue receber mensagens pelo WhatsApp, consultar a base de conhecimento e retornar respostas de forma autônoma.

### Próximos passos

* [ ] Realizar testes com microempreendedores reais;
* [ ] Ampliar a base de conhecimento;
* [ ] Adicionar materiais regionais;
* [ ] Criar métricas para avaliação das respostas;
* [ ] Avaliar encaminhamento para atendimento humano quando necessário.

## 👥 Projeto

Desenvolvido pelo **PET Criativação — UFAPE**.

O projeto investiga a aplicação de agentes de Inteligência Artificial como ferramenta de apoio ao desenvolvimento de microempreendedores no contexto local.
