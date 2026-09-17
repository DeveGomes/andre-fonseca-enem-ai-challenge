# ENEM AI Challenge — Documento Base do Projeto

> Documento vivo. Aqui fica **o que o projeto é, por que cada decisão foi tomada e como vamos trabalhar**.
> Regras de negócio detalhadas, identidade visual e checklist de funcionalidades entram depois, em documentos próprios.

---

## 1. Visão do produto

Uma plataforma web onde o estudante do ENEM:

1. cria conta e faz login (sessão persistente);
2. entra num **dashboard** com visão geral do seu desempenho;
3. realiza **simulados** montados a partir de provas **oficiais** do ENEM (2009–2023), escolhendo disciplina e tamanho;
4. ao submeter, recebe o resultado **e uma explicação individual de cada erro** — não só "errou", mas *por que* errou;
5. acompanha o **histórico** de simulados e a evolução ao longo do tempo;
6. pede um novo simulado **focado nos próprios erros**, montado a partir desse histórico.

**A tese do projeto:** a maioria das plataformas de estudo entrega nota. Nota não ensina. O que ensina é entender o raciocínio que te levou à alternativa errada. A IA aqui existe para fechar esse ciclo.

---

## 2. De onde vem o conteúdo e onde entra a IA

### 2.1 As questões vêm da API pública do ENEM

Usamos a [enem.dev](https://docs.enem.dev) como fonte das questões:

```
15 provas oficiais: 2009 → 2023  ·  ~180 questões cada  ·  ~2.700 questões
Sem autenticação  ·  Rate limit: 10 req / 10s  ·  Paginação limit/offset
```

**Por que questão oficial e não questão gerada por IA:** uma questão gerada por LLM pode vir com gabarito errado — e aí a plataforma ensina errado. Numa ferramenta de estudo isso é defeito fatal. Com prova oficial, o gabarito é verdade absoluta, a correção é confiável e o estudante treina com o que ele vai encontrar na prova de verdade. De quebra, some a latência de gerar questões sob demanda.

**Cache obrigatório no nosso banco.** Montar simulado batendo na API ao vivo é frágil (10 req/10s) e acopla nossa disponibilidade à deles. Importamos uma vez, guardamos no Mongo e servimos do nosso banco — o que também nos dá filtro por disciplina, ano e dificuldade, que a API não faz.

**Atenção:** a API devolve `correctAlternative` e `isCorrect` **junto** da questão. O backend precisa remover esses campos antes de enviar ao front — senão o gabarito vai de bandeja na aba Network.

### 2.2 IA Tutora — explica o erro

É aqui que a IA é insubstituível: não em produzir conteúdo, mas em explicar o raciocínio que levou o estudante à alternativa errada.

- **O que faz:** para cada questão errada, produz um **feedback estruturado**:

| Campo | Conteúdo |
|---|---|
| `status` | veredito curto (ex.: *quase certa*, *erro conceitual*, *desatenção*) |
| `suaResposta` | alternativa que o usuário marcou |
| `respostaCorreta` | alternativa correta |
| `tipoErro` | etiquetas classificando a falha (ex.: `interpretação`, `fórmula`, `pegadinha`, `desatenção`) |
| `explicacao` | por que a marcada está errada **e** por que a correta está certa |
| `pontoChave` | o detalhe exato do enunciado que decidia a questão |

- **Quando roda:** **depois** da submissão — nunca durante, para não virar cola.
- **Por que estruturado e não texto solto:** a resposta vem em **JSON com schema definido**. Isso dá três coisas: a UI renderiza cada pedaço do seu jeito (badge, tags, destaque), o backend valida antes de salvar, e os `tipoErro` viram **dados agregáveis** — é o que alimenta o dashboard ("você erra mais por interpretação do que por conta").

> É esse detalhe que conecta o requisito 3 (simulados/histórico) com o requisito 4 (IA). Sem isso, a IA seria um chatbox solto na lateral.

### 2.3 Questões com imagem — multimodal com fallback

Boa parte das questões é imagem (gráfico, tabela, fórmula renderizada). Medição feita na prova de 2023:

| Disciplina | Só texto | Com imagem |
|---|---|---|
| Linguagens | 42 | 2 |
| Humanas | 47 | 9 |
| Natureza | 19 | 16 |
| **Matemática** | 18 | **28** |
| **Total** | **126 (70%)** | **55 (30%)** |

Matemática é ~60% imagem. Se a IA só recebe texto, ela explica uma questão que não consegue ver — e alucina.

**Decisão: multimodal com fallback.** O backend baixa a imagem, converte para base64 e envia junto do prompt (o Gemini é multimodal nativo). Se o download falhar, a IA responde com o texto disponível e a **UI avisa que o feedback é parcial** — nunca fingimos que vimos a imagem.

```
questão → monta prompt de texto ──┐
       └→ tem imagem? baixa,      ├→ Gemini → JSON validado → salva
          converte base64 ────────┘
          (falhou? segue só com texto + marca feedback parcial)
```

Custo: as imagens do ENEM são pequenas (~300–800px), o que dá ~258 tokens por imagem — cerca de 20–30% a mais numa chamada que já ia acontecer. Irrelevante no free tier.

⚠️ O limite que importa não é dinheiro, é **requisições**: um simulado com 6 erros dispara 6 chamadas de uma vez. Os tetos por minuto/dia do free tier precisam ser conferidos em `aistudio.google.com/rate-limit` e tratados no backend (fila, retry ou processamento sequencial) — definido nas regras de negócio.

### 2.4 IA Curadora — monta o próximo simulado em cima dos seus erros

É a segunda função de IA, e ela **se alimenta do que a Tutora produziu**. Fecha o ciclo do produto:

```
erra  →  Tutora explica e classifica o erro (tipoErro)
      →  o perfil de erros do aluno vai se acumulando no banco
      →  Curadora lê esse perfil e define a estratégia do próximo simulado
      →  o aluno é cobrado de novo exatamente onde falha
```

Sem isso, o histórico é um museu: você olha a nota e não acontece nada. Com isso, o histórico **vira insumo** — é o que separa produto de demonstração.

#### A divisão de trabalho (importante)

A IA **não sorteia questão**. Isso seria uso ruim de LLM: sortear é `find` com filtro, determinístico e instantâneo.

| Etapa | Quem faz | Por quê |
|---|---|---|
| Agregar o histórico de erros por disciplina/tópico/`tipoErro` | **banco** (query) | é contagem, não raciocínio |
| Decidir a **estratégia**: quais tópicos priorizar, em que proporção, e justificar | **IA** | exige interpretar o padrão, não só contar |
| Selecionar as questões que atendem essa estratégia | **banco** (query) | é filtro, determinístico |
| Explicar ao aluno por que essa prova foi montada assim | **IA** (texto já gerado acima) | transparência: o aluno entende o plano |

Mesmo princípio da correção: **a IA entra onde há julgamento, não onde há cálculo.** Essa frase vale para as duas features e é a linha de defesa da arquitetura no bate-papo técnico.

#### Saída esperada da Curadora

Também **JSON com schema**, não texto solto:

| Campo | Conteúdo |
|---|---|
| `foco` | lista de tópicos priorizados, com peso (quantas questões de cada) |
| `justificativa` | por que esses tópicos — mostrada ao aluno antes de começar |
| `evitar` | tópicos já dominados, para não desperdiçar questões |

O backend **valida** esse JSON e usa os pesos para montar a query. Se a IA pedir um tópico que não existe no banco ou não houver questões suficientes, o backend completa com o critério padrão — a IA sugere, o backend manda.

#### Caso do usuário novo

Quem acabou de criar conta não tem histórico de erro nenhum. Nesse caso não há o que curar: o simulado é montado pelo filtro normal escolhido pelo usuário. A Curadora só entra em cena a partir do primeiro simulado corrigido — regra exata em `regras-de-negocio.md`.

### 2.5 Conversa contextual — fase 2

Dentro do card de feedback, um **"ainda não entendi"** abre conversa com a IA já carregada com o contexto daquela questão (enunciado, imagem, alternativa marcada, correta e a explicação dada). Não é chat genérico: o prompt fica travado no escopo da questão.

É a **última** coisa a ser construída, depois de tudo funcionando e deployado. Regras em `regras-de-negocio.md` §5.5.


## 3. Stack definida

| Camada | Escolha | Justificativa para a entrevista |
|---|---|---|
| Linguagem | **TypeScript** (front e back) | Diferencial pedido no desafio; contratos de dados tipados ponta a ponta |
| Backend | **NestJS** | Arquitetura modular explícita (módulo / controller / service / repository), DI nativa, `class-validator` nos DTOs. Você consegue apontar no código onde cada responsabilidade mora |
| Frontend | **React + Vite** | Build rápido, sem a complexidade de SSR que o Next.js traria e que você teria que defender sem necessidade |
| Estilo | **Tailwind CSS** | Velocidade de iteração; tokens definidos em `docs/design.md` |
| Banco | **MongoDB Atlas** | Free tier sem cold start, citado pela empresa como banco que usam. Modelo de documento encaixa bem em "tentativa com N questões e N feedbacks aninhados" |
| Questões | **API enem.dev** | ~2.700 questões oficiais de 2009–2023, sem autenticação; cacheadas no nosso banco |
| IA | **Google Gemini API** | Tier gratuito; multimodal (lê as imagens das questões); chamada **exclusivamente pelo backend** |
| Auth | **JWT + bcrypt, implementado à mão** | Auth de terceiro (Clerk, Supabase Auth) é caixa-preta na hora de explicar. Aqui você domina hash, payload, expiração e guard |
| Deploy | **Vercel** (web) + **Render** (api) + **Atlas** (db) | Todos free tier, integração direta com GitHub |

### Regra inegociável de segurança

A `GEMINI_API_KEY` vive **só no backend**, em variável de ambiente. O frontend nunca fala com o Google — ele fala com a nossa API, que fala com o Google. Se a chave estivesse no React, qualquer pessoa abriria o DevTools e a levaria embora. `.gitignore` com `.env` desde o **primeiro commit**.

---

## 4. Arquitetura

### 4.1 Desenho geral

```
┌──────────────┐        ┌──────────────────┐        ┌──────────────┐
│  React (SPA) │──────▶ │   NestJS (API)   │──────▶ │ Gemini API   │
│    Vercel    │  HTTP  │      Render      │  HTTP  │   Google     │
└──────────────┘  +JWT  └───┬──────────┬───┘        └──────────────┘
                            │          │  (só na importação)
                            │          ▼
                            │   ┌──────────────┐
                            │   │ API enem.dev │
                            │   └──────────────┘
                            ▼
                     ┌──────────────┐
                     │ MongoDB Atlas│
                     └──────────────┘
```

O backend é o único ponto que conhece a chave da IA e o banco. A enem.dev só é chamada na **importação** do banco de questões, nunca durante um simulado. O frontend é burro de propósito: renderiza e chama a API.

### 4.2 Estrutura de pastas (monorepo)

```
andre-fonseca-enem-ai-challenge/
├── apps/
│   ├── api/                    # NestJS
│   │   └── src/
│   │       ├── auth/           # login, registro, JWT, guards
│   │       ├── users/          # perfil do estudante
│   │       ├── exams/          # criação e entrega dos simulados
│   │       ├── attempts/       # tentativas, respostas, correção, histórico
│   │       ├── ai/             # integração Gemini: geração e tutoria
│   │       ├── common/         # filtros, interceptors, decorators
│   │       └── config/         # env, validação de variáveis
│   └── web/                    # React + Vite
│       └── src/
│           ├── pages/          # login, dashboard, simulado, resultado, histórico
│           ├── components/     # UI reaproveitável
│           ├── features/       # lógica por domínio (auth, exams, attempts)
│           ├── lib/            # cliente HTTP, helpers
│           └── hooks/
├── docs/                       # este doc, regras de negócio, design, checklist
└── README.md                   # vitrine do projeto
```

**Por que monorepo:** o avaliador clona um repositório só e roda os dois lados. Também deixa a separação front/back visível na primeira olhada — que é literalmente o critério 3 da avaliação.

**Por que `features/` no front:** agrupar por domínio em vez de por tipo de arquivo evita a pasta `components/` com 40 arquivos soltos no fim do projeto.

### 4.3 Módulos do backend e suas responsabilidades

| Módulo | Responsabilidade | Não é responsabilidade dele |
|---|---|---|
| `auth` | registrar, autenticar, emitir/validar JWT | saber o que é um simulado |
| `users` | dados do estudante | corrigir prova |
| `questions` | importar da enem.dev e consultar o banco local | saber quem é o usuário |
| `exams` | montar e servir um simulado | falar com o Gemini direto |
| `attempts` | receber respostas, calcular acertos, guardar histórico | gerar texto de explicação |
| `ai` | **único** que fala com o Gemini; monta prompt, valida resposta, trata falha | decidir regra de negócio; sortear questão |

A regra: `exams` e `attempts` **pedem** para o `ai`; o `exams` monta a prova a partir do `questions`; o `ai` não decide nada de negócio. Se amanhã trocarmos Gemini por outro provedor, só o módulo `ai` muda.

---

## 5. Fluxo principal (ponta a ponta)

```
0. (uma vez) Importação: enem.dev ──▶ nosso banco de questões
1. Usuário pede um simulado (disciplina + tamanho, ou "focar nos meus erros")
1b. Se focado ──▶ IA Curadora lê o perfil de erros e devolve a estratégia
2. API monta o simulado consultando o banco (sem gabarito no payload)
3. Usuário responde (gabarito NUNCA vai pro front nessa etapa)
4. Usuário submete
5. API corrige (comparação simples, sem IA — é determinístico)
6. Para cada erro ──▶ IA Tutora gera o feedback (com a imagem, quando houver)
7. Tela de resultado: nota **na hora**; cards de feedback chegam progressivamente
8. Tudo persistido ──▶ histórico, dashboard e perfil de erros (insumo da Curadora)
```

Dois pontos que valem ouro na entrevista:

- **A correção não usa IA.** Comparar resposta com gabarito oficial é determinístico. Usar LLM para isso seria caro, lento e não-confiável. A IA entra só onde é insubstituível: explicar.
- **O gabarito não trafega antes da submissão.** Se as respostas corretas fossem junto com as questões, bastaria abrir a aba Network para gabaritar o simulado.

---

## 6. Ambientes e deploy

| Ambiente | Web | API | Banco |
|---|---|---|---|
| Local | `localhost:5173` | `localhost:3000` | Atlas (cluster de dev) |
| Produção | Vercel | Render | Atlas |

**Deploy é a primeira entrega, não a última.** Assim que existir um "hello world" nos dois lados, ele vai pro ar. A partir daí, cada push atualiza o que já está publicado. Deixar deploy para o último dia é o erro que mais reprova candidato nesse tipo de desafio.

**Cold start:** o plano gratuito do Render hiberna após inatividade e a primeira requisição pode levar ~50s. Isso precisa ser tratado na UI (estado de carregando honesto) e avisado no README.

### Variáveis de ambiente

| Onde | Variável | Para quê |
|---|---|---|
| api | `MONGODB_URI` | conexão com o Atlas |
| api | `JWT_SECRET` | assinatura do token |
| api | `GEMINI_API_KEY` | acesso à IA |
| api | `CORS_ORIGIN` | liberar só o domínio do front |
| web | `VITE_API_URL` | endereço da API |

Cada app terá um `.env.example` versionado — com os nomes, sem os valores.

---

## 7. Como vamos trabalhar

O critério de avaliação nº 5 é explícito: **você precisa dominar o próprio código**, porque haverá um bate-papo técnico sobre as decisões de arquitetura. Por isso o projeto é **guiado, não entregue pronto**.

### A divisão

**Meu papel:**
- explicar cada conceito novo antes de você encostar no código (o que é DI no Nest, por que guard e não middleware, por que o schema do Mongo é assim);
- escrever as especificações: contratos de rota, formato de payload, estrutura de arquivo;
- revisar o que você escreveu, apontar problema e explicar o porquê;
- desbloquear você quando travar — explicando a causa, não só colando a correção;
- assumir o trabalho mecânico e sem aprendizado (boilerplate repetitivo, configuração de ferramenta), sempre avisando.

**Seu papel:**
- escrever o código das partes que importam;
- fazer os commits (semânticos e pequenos — é critério de avaliação);
- perguntar sempre que algo não fizer sentido. "Copiei e funcionou" é o pior resultado possível aqui;
- testar cada funcionalidade antes de marcar como pronta no checklist.

### O teste do bate-papo

Antes de dar qualquer etapa por concluída, você deve conseguir responder, sem olhar o código:

> *"Por que essa parte é assim e não de outro jeito?"*

Se a resposta não vier, a etapa não está pronta — voltamos e revisamos.

### Commits

Padrão semântico, pequenos e frequentes:

```
feat(auth): cria endpoint de registro com hash de senha
fix(exams): corrige gabarito vazando na listagem de questões
docs: adiciona prints da aplicação ao README
```

O histórico do Git é avaliado como narrativa da sua evolução. Três commits gigantes no último dia contam contra você.

---

## 8. O que ainda não está definido

Esta é a fundação. Os três documentos que a completam:

| # | Documento | Conteúdo |
|---|---|---|
| 1 | ~~`docs/regras-de-negocio.md`~~ | ✅ **pronto** |
| 2 | ~~`docs/design.md`~~ | ✅ **pronto** |
| 3 | ~~`docs/checklist.md`~~ | ✅ **pronto** |

Os três estão fechados — o próximo passo é a Fase 0 do checklist.

---

## 9. Prazo

Entrega: **22/09/2026, 09:00**. Hoje: **17/09/2026** — 5 dias.

O escopo acima cabe no prazo **se o deploy sair no primeiro dia** e o escopo não crescer no meio do caminho. Ideia nova que aparecer durante o desenvolvimento vai para uma lista de "se sobrar tempo", não para o escopo.
