# Regras de Negócio

> Complementa o `PROJETO.md`. Aqui está **o que o sistema faz, com quais dados e o que acontece quando dá errado**.
> Este documento é a fonte da verdade na hora de implementar: se o código discordar daqui, um dos dois está errado.

---

## 0. Decisões já fechadas

| Decisão | Escolha |
|---|---|
| Configuração do simulado | usuário escolhe **disciplina + quantidade** (5, 10 ou 20) |
| Filtro por dificuldade | **não existe** — ver 1.1 |
| Cronômetro | **fora do escopo** |
| Feedback da IA | **progressivo** — nota na hora, explicações chegando uma a uma |
| Ativação da Curadora | a partir do 1º simulado corrigido com ao menos um erro |
| Conversa contextual | **fase 2** — construída por último, ver 5.5 |

### 1.1 Por que não há filtro de dificuldade

A API enem.dev não expõe dificuldade — os campos são `discipline`, `year`, `language`, `context`, `files`, `alternatives` e `correctAlternative`. Derivar dificuldade exigiria taxa de acerto real dos nossos usuários, que no lançamento é zero.

**Decisão:** filtro por **disciplina**. O ano existe no banco e aparece no card da questão ("ENEM 2019"), mas não vira filtro no MVP — é um campo a mais para validar e testar, com pouco ganho.

Isso é decisão consciente, não esquecimento: numa plataforma madura, dificuldade viria do histórico agregado (item de evolução futura no README).

---

## 2. Modelo de dados

Quatro coleções no MongoDB.

### 2.1 `users`

| Campo | Tipo | Regra |
|---|---|---|
| `_id` | ObjectId | |
| `name` | string | 2–80 caracteres |
| `email` | string | **único**, salvo sempre em minúsculas |
| `passwordHash` | string | bcrypt — **a senha nunca é salva em texto** |
| `createdAt` | Date | |

Índice único em `email`. A senha nunca sai do backend, em nenhuma resposta — nem no registro.

### 2.2 `questions` (espelho da enem.dev)

| Campo | Tipo | Observação |
|---|---|---|
| `externalId` | string | `"2023-1-ingles"` — **único**, torna a importação idempotente |
| `year` | number | 2009–2023 |
| `index` | number | número da questão na prova |
| `discipline` | enum | `linguagens` · `ciencias-humanas` · `ciencias-natureza` · `matematica` |
| `language` | string? | só para questões de idioma |
| `context` | string | enunciado (markdown) |
| `files` | string[] | URLs das imagens do enunciado |
| `alternativesIntroduction` | string | frase que introduz as alternativas |
| `alternatives` | array | `{ letter, text, file }` — **sem `isCorrect`** |
| `correctAlternative` | string | `A`–`E` — **nunca trafega antes da submissão** |
| `topic` | string? | tópico fino (ex.: "Funções"); opcional, ver 6.2 |
| `stats` | object | `{ timesAnswered, timesCorrect }` — alimenta métricas futuras |

Índices: `externalId` (único), `discipline`, `topic`.

> A API devolve `isCorrect` dentro de cada alternativa **e** `correctAlternative` fora. Guardamos só o segundo — dois lugares com a mesma verdade é convite para inconsistência.

### 2.3 `attempts` (tentativa = um simulado feito)

| Campo | Tipo | Observação |
|---|---|---|
| `userId` | ObjectId | dono da tentativa |
| `config` | object | `{ discipline, size, focused: boolean }` |
| `curation` | object? | preenchido só se `focused` — `{ foco[], justificativa }` |
| `questions` | array | snapshot: `{ questionId, order }` |
| `answers` | array | ver abaixo |
| `score` | object | `{ correct, total, percent }` — só após submissão |
| `status` | enum | `in_progress` → `submitted` → `feedback_done` |
| `startedAt` / `submittedAt` | Date | |

Cada item de `answers`:

| Campo | Observação |
|---|---|
| `questionId` | referência |
| `chosen` | `A`–`E` ou `null` (em branco) |
| `isCorrect` | calculado na submissão |
| `feedback` | objeto da IA Tutora (ver 2.4); `null` se acertou |
| `feedbackState` | `not_needed` · `pending` · `done` · `failed` |
| `messages` | conversa contextual sobre a questão — ver 5.5 *(fase 2)* |

**Por que a tentativa nasce no banco antes de o usuário responder:** o gabarito fica no servidor, o usuário pode atualizar a página sem perder o simulado, e ninguém consegue forjar uma tentativa que nunca existiu.

### 2.4 `feedback` (embutido na resposta)

| Campo | Tipo |
|---|---|
| `status` | string — veredito curto |
| `tipoErro` | string[] — etiquetas |
| `explicacao` | string |
| `pontoChave` | string |
| `partial` | boolean — `true` quando a imagem não pôde ser lida |
| `generatedAt` | Date |

Fica **embutido** na tentativa, não em coleção separada: feedback só existe no contexto de uma resposta, e sempre é lido junto dela. Coleção à parte só adicionaria um `join` sem nenhum ganho.

---

## 3. Autenticação

| Regra | Valor |
|---|---|
| Senha mínima | 8 caracteres |
| Hash | bcrypt, custo 10 |
| Token | JWT, validade 7 dias |
| Payload do token | `{ sub: userId, email }` — **nunca** dados sensíveis |
| Armazenamento no front | `localStorage`, enviado em `Authorization: Bearer` |

Regras:

- e-mail duplicado no registro → **409 Conflict**;
- login inválido → **401**, com mensagem genérica *"e-mail ou senha incorretos"*. Nunca diga "esse e-mail não existe": isso entrega ao atacante quais e-mails estão cadastrados;
- toda rota exceto registro e login exige token válido;
- logout é do lado do cliente (descarta o token). Sem blacklist — é complexidade desnecessária aqui, e você deve saber dizer isso na entrevista.

**Regra de propriedade:** toda tentativa acessada é checada contra o `userId` do token. Trocar o id na URL e ler o simulado de outra pessoa é a falha mais comum em projeto de estágio — e a mais fácil de o avaliador testar.

---

## 4. Simulados

### 4.1 Criar

Usuário escolhe disciplina (ou "todas") e tamanho (5, 10 ou 20). O backend sorteia questões do **nosso banco**, sem repetir questão dentro da mesma tentativa.

Regras:

- se não houver questões suficientes para o filtro, devolve **422** com mensagem clara — nunca um simulado menor sem avisar;
- questões já respondidas pelo usuário em tentativas anteriores são **despriorizadas**, não proibidas (com banco finito, proibir acabaria travando o usuário assíduo);
- o usuário pode ter **uma** tentativa `in_progress` por vez. Começar outra pergunta se ele quer retomar ou descartar a anterior.

### 4.2 Responder

- resposta em branco é permitida e conta como erro (é assim no ENEM);
- respostas podem ser alteradas livremente até a submissão;
- o payload das questões **nunca** inclui `correctAlternative` nem `isCorrect` — a sanitização acontece no backend, em um único lugar, não em cada controller.

### 4.3 Submeter e corrigir

A correção é **determinística e instantânea**: compara `chosen` com `correctAlternative`. Sem IA, sem espera.

```
score.correct = nº de acertos
score.total   = nº de questões
score.percent = arredondado, 0-100
```

Submeter é **irreversível**. Tentativa já `submitted` não aceita novas respostas — devolve **409**.

---

## 5. Feedback da IA Tutora (progressivo)

### 5.1 O fluxo

```
usuário submete
  └─▶ backend corrige e responde NA HORA (nota + acertos/erros)
  └─▶ em segundo plano, gera um feedback por erro, SEQUENCIALMENTE
        └─▶ front busca a tentativa a cada 3s e vai preenchendo os cards
```

O usuário vê a nota imediatamente e os cards de explicação aparecem um a um, com skeleton enquanto carregam.

**Por que sequencial e não tudo de uma vez:** o free tier do Gemini limita requisições por minuto. Seis chamadas paralelas podem tomar **429** e derrubar os seis feedbacks. Uma de cada vez é mais lento no total, mas entrega todas — e como a tela já é progressiva, o usuário não percebe a diferença.

**Por que polling e não WebSocket/SSE:** polling de 3 segundos resolve o problema com um `setInterval` e zero infraestrutura extra. Conexão persistente em plano gratuito que hiberna é fonte de dor sem retorno. Essa é a resposta certa para "por que não usou WebSocket?".

O polling **para** quando nenhum feedback está `pending`.

### 5.2 Estados de cada feedback

| Estado | Significa | Como aparece na UI |
|---|---|---|
| `not_needed` | acertou | card verde, sem explicação |
| `pending` | na fila ou gerando | skeleton animado |
| `done` | pronto | card completo |
| `failed` | IA falhou após retry | mensagem + botão **"Tentar novamente"** |

`failed` **não é erro de sistema** — é estado previsto de um recurso externo. Um feedback que falha não afeta os outros, e nunca impede o usuário de ver a nota.

### 5.3 Multimodal e fallback

Se a questão tem imagem, o backend baixa, converte para base64 e envia junto ao prompt.

- download com timeout de 10s;
- falhou → segue **só com o texto** e marca `partial: true`;
- `partial` vira um aviso visível: *"Esta questão tem imagem que não pôde ser analisada — a explicação pode estar incompleta."*

Nunca fingimos que vimos a imagem. Um feedback confiante sobre um gráfico invisível é pior que feedback nenhum.

### 5.4 Contrato com o Gemini

- resposta em **JSON** com schema fixo;
- o backend **valida** antes de salvar. JSON inválido ou campo faltando → conta como falha e entra no retry;
- **1 retry** com espera crescente; falhou de novo → `failed`;
- `tipoErro` sai de uma **lista fechada** definida no prompt: `interpretação` · `conceito` · `cálculo` · `fórmula` · `atenção` · `pegadinha` · `vocabulário`. Etiqueta livre viraria 200 rótulos diferentes e destruiria a agregação do dashboard.


### 5.5 Conversa contextual — "ainda não entendi" *(fase 2)*

> **Ordem de implementação:** esta feature é a **última** a ser construída, depois que auth, simulado, feedback e dashboard estiverem funcionando **e deployados**. É ganho de produto, não requisito — o requisito 4 do desafio já está cumprido pela Tutora e pela Curadora.

Depois de ler a explicação, o aluno pode continuar travado. Dentro do próprio card da questão errada existe um botão **"Ainda não entendi"** que abre uma conversa **já carregada com o contexto**:

```
contexto enviado à IA (montado pelo backend, não pelo front):
  enunciado + imagem · alternativas · a que o aluno marcou
  · a correta · a explicação que a Tutora já deu · as mensagens anteriores
```

Assim o aluno pergunta *"por que a C também não serviria?"* sem precisar explicar nada — a IA já sabe do que se trata.

**Por que não um chat genérico na lateral:** um chat que não conhece seu histórico nem a questão é um ChatGPT com outra roupa, e obriga o aluno a redigitar o problema. A conversa contextual nasce no ponto exato da dúvida e fecha o ciclo *erra → entende → treina*.

#### Regras

| Regra | Valor | Motivo |
|---|---|---|
| Escopo do prompt | **travado na questão em discussão** | a IA recusa educadamente assunto fora dela; não é assistente de uso geral |
| Limite | **5 perguntas por questão** | protege a cota do free tier e evita conversa infinita |
| Formato da resposta | **texto corrido**, não JSON | aqui é conversa, não dado agregável — diferente da Tutora de propósito |
| Persistência | embutida na resposta da tentativa | ao reabrir o histórico, a conversa continua lá |
| Pré-requisito | feedback com `feedbackState: done` | sem explicação inicial não há o que aprofundar |
| Falha da IA | mensagem de erro no chat, com reenvio | não afeta o feedback já salvo |

#### Dado adicional

Cada item de `answers` ganha:

| Campo | Tipo | Observação |
|---|---|---|
| `messages` | array | `{ role: 'user' \| 'assistant', content, createdAt }` |

Nada além disso: o contexto da questão **não** é duplicado dentro das mensagens — ele é remontado a cada chamada a partir dos dados que já existem. Guardar o prompt inteiro em cada mensagem incharia o documento e criaria duas fontes de verdade.

#### Por que reaproveita tudo

Não exige coleção nova, módulo novo nem integração nova: o contexto já está salvo na tentativa, o módulo `ai` já fala com o Gemini e o tratamento de falha já existe. É **um endpoint** que recebe a pergunta e o id da resposta, remonta o contexto e devolve texto. Estimativa: **3 a 4 horas**, com a UI.


---

## 6. IA Curadora

### 6.1 Quando pode ser usada

Precisa de pelo menos **uma tentativa submetida com ao menos um erro**. Sem isso, não há perfil a ler: o botão aparece desabilitado, explicando que é preciso fazer um simulado primeiro.

### 6.2 Como funciona

```
1. banco   → agrega erros por disciplina, tópico e tipoErro
2. IA      → recebe esse resumo e devolve a ESTRATÉGIA (JSON)
3. backend → valida e traduz a estratégia em query
4. banco   → seleciona as questões
5. UI      → mostra a justificativa antes de o simulado começar
```

Saída da IA:

| Campo | Conteúdo |
|---|---|
| `foco` | `[{ disciplina, topico?, peso }]` — peso = quantas questões |
| `justificativa` | 1–2 frases, mostradas ao aluno |
| `evitar` | tópicos já dominados |

**O backend manda, a IA sugere.** Se a soma dos pesos não bater com o tamanho pedido, se vier disciplina inexistente, ou se faltarem questões do tópico, o backend **ajusta e completa** pelo critério padrão. A prova sai sempre com o tamanho certo.

**Sobre `topic`:** a API não traz tópico. No MVP a Curadora opera com **disciplina + tipoErro**, que existem de graça. Enriquecer as questões com tópico fino é um script offline de importação (lotes de ~20 questões por chamada) — melhoria desejável, **não bloqueante**.

### 6.3 Falha

Se a Curadora falhar, o simulado é montado pelo critério padrão da disciplina com pior desempenho, avisando que a personalização não pôde ser feita. **Nunca** deixamos o usuário sem simulado por causa da IA.

---

## 7. Dashboard

O que aparece, tudo derivado das tentativas:

| Bloco | Conteúdo | Sem dados ainda |
|---|---|---|
| Resumo | simulados feitos · média de acerto · melhor disciplina | convite para o primeiro simulado |
| Evolução | % de acerto nas últimas 10 tentativas | escondido até a 2ª tentativa |
| Desempenho por disciplina | acertos/total em cada área | escondido |
| **Perfil de erros** | distribuição de `tipoErro` | escondido |
| Últimos simulados | data, disciplina, nota, link | lista vazia com call to action |
| Ações | novo simulado · **simulado focado nos meus erros** | focado desabilitado |

O **perfil de erros** é o bloco que justifica a arquitetura inteira: ele só existe porque a Tutora devolve `tipoErro` estruturado. É a prova visual de que as duas IAs conversam.

**Estado vazio é requisito, não detalhe.** O avaliador vai criar uma conta nova — a primeira tela que ele vê é o dashboard zerado. Se ela estiver quebrada ou feia, é a primeira impressão do projeto.

---

## 8. Importação das questões

Script executado **fora do fluxo do usuário** (`npm run seed:questions`).

| Regra | Valor |
|---|---|
| Paginação | 50 questões por chamada |
| Rate limit | 10 req/10s → **espera de 1,2s** entre chamadas |
| Idempotência | upsert por `externalId` — rodar duas vezes não duplica |
| Falha em um lote | registra e continua; não aborta a importação inteira |
| Escopo inicial | anos mais recentes primeiro (2023 → 2019) |

Importar 5 anos (~900 questões) leva poucos minutos e já sustenta o produto. Rodar o resto depois é só executar o script de novo.

---

## 9. Erros da API — contrato

| Situação | Status |
|---|---|
| Payload inválido | 400 |
| Sem token / token expirado | 401 |
| Tentativa de outro usuário | 403 |
| Recurso inexistente | 404 |
| E-mail já cadastrado · tentativa já submetida | 409 |
| Filtro sem questões suficientes | 422 |
| Falha ao falar com o Gemini | **não vira erro HTTP** — vira `feedbackState: failed` |
| Limite de 5 perguntas atingido na conversa | 429 |

A última linha é a regra mais importante: **falha da IA nunca derruba a requisição do usuário.** Ele vê a nota, o histórico e o simulado de qualquer jeito. A IA enriquece o produto; ela não é pré-requisito para ele funcionar.

---

## 10. Fora do escopo (consciente)

Registrado aqui para virar seção "evolução futura" do README — e para você responder com firmeza se perguntarem:

- cronômetro e simulação de tempo de prova;
- redação e correção de texto dissertativo;
- dificuldade calculada por taxa de acerto real;
- recuperação de senha por e-mail;
- refresh token / blacklist de logout;
- ranking ou qualquer recurso social;
- prova completa de 180 questões;
- conversa contextual **genérica** (fora do escopo de uma questão).

Cada item aqui foi **decidido**, não esquecido. Saber justificar um corte de escopo vale mais que entregar metade de dez features.
