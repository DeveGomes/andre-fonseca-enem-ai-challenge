# Checklist de Funcionalidades

> Para colar no Notion (cada `- [ ]` vira um to-do).
> Cada item é **testável**: descreve o que você deve conseguir fazer na aplicação rodando, não o que você deve programar.
> Um item só é marcado depois de testado **no navegador** — não basta o código compilar.

**Prioridades:** `P0` sem isso a entrega está incompleta · `P1` esperado, faz diferença na avaliação · `P2` se sobrar tempo

**Definição de pronto:** funciona no deploy, não no seu localhost · o caminho de erro foi testado, não só o feliz · você sabe explicar por que está assim.

---

## Fase 0 — Fundação e deploy `P0`

> Deploy é a **primeira** entrega, não a última. Nada aqui espera a aplicação estar pronta.

- [ ] Repositório público criado com o nome `andre-fonseca-enem-ai-challenge`
- [ ] `.gitignore` com `.env` e `node_modules` **no primeiro commit**
- [ ] Monorepo com `apps/api` e `apps/web` rodando localmente
- [ ] `.env.example` nos dois apps, com os nomes das variáveis e sem valores
- [ ] Cluster no MongoDB Atlas criado e a API conecta nele
- [ ] Chave do Gemini gerada e funcionando num teste isolado
- [ ] **API publicada no Render**, respondendo numa rota de health check
- [ ] **Front publicado na Vercel**, carregando em branco sem erro no console
- [ ] O front em produção consegue chamar a API em produção (CORS liberado só para o domínio da Vercel)
- [ ] `git log` mostra commits semânticos pequenos, não um commit gigante

✅ **Fecha a fase quando:** existe uma URL pública onde front e back conversam.

---

## Fase 1 — Autenticação `P0`

- [ ] Criar conta com nome, e-mail e senha
- [ ] Senha com menos de 8 caracteres é recusada, com mensagem visível
- [ ] E-mail já cadastrado devolve erro claro (não um 500)
- [ ] A senha está **hasheada** no banco — confira no Atlas, a olho nu
- [ ] Nenhuma resposta da API devolve `passwordHash`, nem no registro
- [ ] Login com credenciais corretas entra na plataforma
- [ ] Login errado mostra *"e-mail ou senha incorretos"* — **sem** revelar se o e-mail existe
- [ ] Token é guardado e enviado automaticamente nas chamadas seguintes
- [ ] Recarregar a página (F5) **não** desloga
- [ ] Logout limpa o token e volta para a tela de login
- [ ] Acessar `/dashboard` sem estar logado redireciona para o login
- [ ] Chamar uma rota protegida sem token devolve **401**

🔒 **Teste de segurança obrigatório:** tente acessar uma rota protegida colando um token inventado. Deve dar 401.

---

## Fase 2 — Banco de questões `P0`

- [ ] Script `npm run seed:questions` importa da enem.dev para o Mongo
- [ ] O script respeita o rate limit (pausa entre chamadas) e não toma 429
- [ ] Rodar o script **duas vezes não duplica** questão (upsert por `externalId`)
- [ ] Falha em um lote é registrada e a importação **continua**
- [ ] Pelo menos 3 anos importados (~500 questões)
- [ ] As 4 disciplinas têm questões no banco
- [ ] Questões com imagem guardaram as URLs em `files`
- [ ] Índice único em `externalId` criado

✅ **Fecha a fase quando:** dá para consultar o banco e ver questões de todas as disciplinas.

---

## Fase 3 — Simulado e correção `P0`

### Criar
- [ ] Tela de configuração: escolher disciplina (ou "todas") e tamanho (5/10/20)
- [ ] Simulado é criado com a quantidade certa de questões
- [ ] Nenhuma questão se repete dentro do mesmo simulado
- [ ] Filtro sem questões suficientes mostra mensagem clara, **não** entrega simulado menor calado
- [ ] Iniciar um simulado com outro em andamento pergunta se quer retomar ou descartar

### Responder
- [ ] As questões aparecem com enunciado, imagem (quando houver) e as 5 alternativas
- [ ] Dá para selecionar e **trocar** a alternativa antes de enviar
- [ ] Dá para navegar entre as questões sem perder o que já foi marcado
- [ ] Deixar em branco é permitido
- [ ] F5 no meio do simulado **não** perde o progresso

### Corrigir
- [ ] Enviar mostra a nota **imediatamente** (acertos, total, percentual)
- [ ] Questão em branco conta como erro
- [ ] Reenviar um simulado já enviado é recusado (**409**)
- [ ] A tentativa fica salva no banco com respostas e nota

🔒 **Teste de segurança obrigatório** — este é o que o avaliador testa primeiro:
- [ ] Abra a aba **Network** durante o simulado: o payload das questões **não** contém `correctAlternative` nem `isCorrect`
- [ ] Pegue o id de uma tentativa sua, crie **outro usuário** e tente abrir essa tentativa pela URL → deve dar **403**

---

## Fase 4 — IA Tutora `P0`

- [ ] Ao enviar, a nota aparece na hora, **sem esperar** a IA
- [ ] Questões certas mostram card verde, sem explicação
- [ ] Questões erradas mostram skeleton enquanto o feedback é gerado
- [ ] Os cards vão preenchendo **um a um** (polling de 3s)
- [ ] O polling **para** quando não há mais feedback pendente
- [ ] O feedback traz: status, tipo de erro, explicação e ponto-chave
- [ ] Os `tipoErro` vêm da lista fechada — nunca uma etiqueta inventada
- [ ] Questão com imagem: a explicação demonstra que a IA **viu** a imagem
- [ ] Imagem que não carrega gera feedback marcado como **parcial**, com aviso na tela
- [ ] JSON inválido vindo da IA não quebra a tela — vira estado de falha
- [ ] Feedback que falha mostra botão **"Tentar novamente"** e ele funciona
- [ ] Um feedback com falha **não** impede os outros de carregarem
- [ ] As chamadas ao Gemini são sequenciais (não tomam 429 num simulado de 20)
- [ ] Reabrir a tentativa depois mostra os feedbacks salvos, sem gerar de novo

⚠️ **Teste obrigatório de resiliência:** desligue a chave do Gemini (troque por uma inválida no `.env`) e envie um simulado. **A nota deve aparecer normalmente**, com os feedbacks em estado de falha. Se a tela quebrar, a regra da seção 9 não foi implementada.

---

## Fase 5 — Dashboard e histórico `P0`

- [ ] Dashboard mostra: simulados feitos, média de acerto e melhor disciplina
- [ ] Gráfico de evolução com as últimas tentativas
- [ ] Desempenho por disciplina
- [ ] **Perfil de erros** — distribuição dos `tipoErro`
- [ ] Lista dos últimos simulados com data, disciplina e nota
- [ ] Clicar num simulado antigo abre o resultado com os feedbacks
- [ ] Os dados são **só do usuário logado** (teste com duas contas)

### Estados vazios — a primeira tela que o avaliador vê `P0`
- [ ] Conta recém-criada mostra dashboard **convidando** ao primeiro simulado, sem quebrar
- [ ] Gráfico de evolução some (ou avisa) com menos de 2 tentativas
- [ ] Blocos sem dados não aparecem zerados nem com `NaN`

---

## Fase 6 — IA Curadora `P1`

- [ ] Botão "simulado focado nos meus erros" **desabilitado** para quem não tem histórico, com explicação
- [ ] Habilita após o primeiro simulado enviado com ao menos um erro
- [ ] A justificativa da IA aparece **antes** do simulado começar
- [ ] O simulado focado prioriza as disciplinas em que o usuário mais erra
- [ ] O simulado sai com o **tamanho pedido**, mesmo se a IA sugerir pesos incoerentes
- [ ] Disciplina inexistente sugerida pela IA é ignorada, sem quebrar
- [ ] Falha da Curadora **ainda entrega** um simulado (critério padrão), avisando que não personalizou

⚠️ **Teste obrigatório:** invalide a chave do Gemini e peça um simulado focado. Você deve receber um simulado normal com aviso — nunca uma tela de erro.

---

## Fase 7 — Entrega `P0`

### README
- [ ] **Link do deploy no topo**
- [ ] Explicação da ideia de IA (as três funções e por que cada uma existe)
- [ ] Passo a passo para rodar localmente, testado do zero
- [ ] Lista das variáveis de ambiente
- [ ] Prints ou GIFs: login, dashboard, simulado, **feedback da IA**, simulado focado
- [ ] Aviso sobre o cold start do Render (~50s na primeira chamada)
- [ ] Seção de decisões de arquitetura
- [ ] Seção de evolução futura (o escopo cortado da §10 das regras)

### Verificação final
- [ ] **Nenhuma chave no repositório** — rode `git log -p | grep -i "api_key\|secret"`
- [ ] Clone o repositório numa pasta nova e siga o próprio README: tem que funcionar
- [ ] Crie uma conta nova **no deploy** e faça o fluxo inteiro
- [ ] Teste no celular — a interface não pode quebrar
- [ ] Sem erro vermelho no console do navegador
- [ ] Sem `console.log` esquecido no código

---

## Fase 8 — Bônus `P2`

- [ ] Conversa contextual "ainda não entendi" (§5.5)
- [ ] Limite de 5 perguntas por questão funcionando
- [ ] A IA recusa pergunta fora do escopo da questão
- [ ] Testes unitários no serviço de correção (é a lógica mais crítica e a mais fácil de testar)
- [ ] Testes no validador de resposta da IA
- [ ] Tópico fino nas questões, via script de enriquecimento
- [ ] Skeleton e transições mais caprichados
- [ ] Acessibilidade básica: navegação por teclado no simulado

---

## Painel de progresso

| Fase | Prioridade | Status |
|---|---|---|
| 0 — Fundação e deploy | P0 | ⬜ |
| 1 — Autenticação | P0 | ⬜ |
| 2 — Banco de questões | P0 | ⬜ |
| 3 — Simulado e correção | P0 | ⬜ |
| 4 — IA Tutora | P0 | ⬜ |
| 5 — Dashboard e histórico | P0 | ⬜ |
| 6 — IA Curadora | P1 | ⬜ |
| 7 — Entrega | P0 | ⬜ |
| 8 — Bônus | P2 | ⬜ |

---

## Os cinco testes que mais valem

Se o tempo apertar e você só puder validar cinco coisas, valide estas — são as que quebram projeto de estágio:

1. **Gabarito não vaza** — aba Network durante o simulado
2. **Tentativa de outro usuário dá 403** — troca de id na URL
3. **Chave do Gemini inválida não derruba a aplicação** — nota continua aparecendo
4. **Conta nova não quebra o dashboard** — estado vazio
5. **O README funciona num clone limpo** — o avaliador vai fazer exatamente isso
