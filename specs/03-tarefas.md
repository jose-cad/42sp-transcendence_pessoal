# Tarefas

Status: rascunho para revisão · 2026-10-24
Quebra o [`02-plano-tecnico.md`](02-plano-tecnico.md) em tarefas para o time de 5 cadetes. Cada tarefa vira uma issue no GitHub. O passo a passo de teste e código de cada marco está num plano detalhado próprio, escrito antes de qualquer implementação (os planos de todos os marcos foram escritos antes de começar o primeiro).

## Time

| Cadete | Papel | Dono de |
|---|---|---|
| C1 | Tech Lead | infra, contrato de eventos, integração com a Evolution API, tempo real |
| C2 | Product Owner + dev | front |
| C3 | Project Manager + dev | core |
| C4 | Dev | cobrança (módulo próprio) |
| C5 | Dev | leitura (Python), painel de números, exportação e importação |

Quem é dono de um serviço revisa todo PR que mexe nele, mesmo quando a tarefa é de outro cadete (por exemplo, C5 escreve a exportação no core, e C3 revisa).

## Como trabalhar uma tarefa

1. A issue no GitHub tem o código da tarefa no título (`T12 · Cadastro, login e sessão`) e as etiquetas do marco e do cadete.
2. Uma branch por tarefa: `feat/t12-login`, `fix/...`, `docs/...`.
3. Commits no padrão Conventional Commits.
4. PR com `Closes #<issue>`, revisado pelo dono do serviço, e merge por squash.

## Definição de pronto (vale para toda tarefa)

- Os critérios de "pronto quando" da tarefa passam, com teste automatizado sempre que o critério puder ser testado.
- Todo formulário é validado no front (Zod) e no backend.
- Toda consulta filtra pela locadora de quem está logado.
- Toda alteração de dado de negócio grava o histórico (quem, quando, antes, depois).
- Nenhum dado pessoal, senha, token ou chave aparece em log (plano, seção 7.1).
- Nenhum erro ou aviso no console do Chrome nas telas tocadas.
- Nada além do que a tarefa pede (constituição, item 10).
- O código é explicável: o dono do projeto lê o PR e entende cada parte.

## Pontos de atenção

Situações que a spec não cita, mas que vão acontecer de verdade. Cada uma tem um teste na tarefa indicada.

1. **O próprio número da locadora manda mensagem** (o operador responde pelo celular, ou o bot recebe o eco da própria resposta): a cobrança ignora mensagens com `fromMe`, grupos e status (`status@broadcast`). Sem isso, o bot responde a si mesmo em laço. → T30
2. **O 9º dígito do celular:** o WhatsApp entrega alguns números brasileiros sem o 9 depois do DDD. A busca do motorista pelo telefone compara com e sem o 9. → T30
3. **Dois operadores na mesma ação ao mesmo tempo** (confirmar o mesmo Pix, tratar o mesmo alerta, criar locação com o mesmo carro): um consegue, e o outro recebe uma mensagem clara, sem erro 500. → T13, T24, T29, T40
4. **Gemini fora do ar ou no limite do plano gratuito:** a leitura tenta de novo algumas vezes com espera crescente. Se não conseguir, publica falha de sistema, que não conta como "foto ilegível" para o motorista. → T35, T38
5. **Navegador perde a conexão e volta:** ao reconectar, o front busca de novo conversas, alertas e contador, porque os eventos do intervalo não são reenviados. → T34

---

## M0 · Fundação

Pronto quando: tudo sobe com `docker compose up --build`, só com tráfego criptografado, `https://localhost` mostra o layout vazio, e o teste na escola foi feito.

### T01 · Compose, Postgres e RabbitMQ
Dono: C1 · Depende de: — · Plano: 3.6, 11 · Constituição: 1, 5
- Pastas `infra/certs/` (gerador da CA e dos certificados), `infra/rabbitmq/` (conf e definições) e `infra/testes/` (smoke tests); as demais pastas da seção 11 nascem nas tarefas de cada serviço (o Git não guarda pasta vazia). `docker-compose.yml`, `.env.example` sem nenhum valor utilizável e `.gitignore` com `.env`, conferindo que não há linha `arquivos/` sem barra inicial (a pasta do projeto antigo é `arkivos/`).
- Postgres com script que cria os bancos `core`, `cobranca` e `evolution`, cada um com usuário próprio. RabbitMQ com plugin de gerenciamento desligado para fora da rede.
- *Healthcheck* nos dois. Nenhuma porta publicada além das do nginx.

Pronto quando: os dois sobem saudáveis; o usuário `cobranca` não consegue abrir o banco `core`; `docker compose ps` não mostra porta publicada.

### T02 · Certificados e TLS em tudo
Dono: C1 · Depende de: T01 · Plano: 3.6 · Constituição: 3
- Container `certs`: gera com `openssl`, se ainda não existirem, a autoridade certificadora interna e os certificados de nginx, core, cobrança, RabbitMQ, Postgres e Evolution API.
- Postgres com SSL obrigatório. RabbitMQ só na porta 5671. nginx no 443 e redirecionando o 80.

Pronto quando: conexão sem TLS ao Postgres e ao RabbitMQ é recusada; `http://localhost` redireciona para `https://`.

### T03 · Esqueleto do core
Dono: C3 · Depende de: T02 · Plano: 3.2, 6.1, 7
- ASP.NET Core (.NET 10) com HTTPS no Kestrel, EF Core + Npgsql com `sslmode=verify-full` e migrations aplicadas ao subir.
- Erro genérico para o navegador e log sem dados sensíveis (`EnableSensitiveDataLogging` desligado).
- `GET /api/saude`. Projeto de testes xUnit.

Pronto quando: `https://localhost/api/saude` responde 200 pelo nginx; `dotnet test` roda.

### T04 · Esqueleto da cobrança
Dono: C4 · Depende de: T02 · Plano: 3.3, 4.1
- Spring Boot 4 (Java 25) com HTTPS, Flyway, Postgres com SSL e RabbitMQ com TLS.
- Filtro do `X-Interno-Token` em todo `/interno/**`. `GET /interno/saude`. Log de SQL com valores desligado.

Pronto quando: uma chamada de dentro da rede (container `curl`, ferramenta de teste nova) a `/interno/saude` recebe 200; sem o token, recebe 401; `./mvnw test` roda. O cliente HTTP do core nasce na primeira chamada de verdade (T23/T24).

### T05 · Esqueleto da leitura
Dono: C5 · Depende de: T02, T06 · Plano: 3.4
- Python 3.13, `pika` com TLS, consumidor da fila `leitura.pedidos`, configuração por variáveis de ambiente, log sem conteúdo.

Pronto quando: sobe, conecta com TLS e fica consumindo; `pytest` roda.

### T06 · Contrato de eventos
Dono: C1 · Depende de: T01 · Plano: 4.2, 4.3
- `docs/eventos.md`: o envelope e cada evento da seção 4.3, com todos os campos, tipos JSON e exemplo.
- Topologia: a exchange `frota` e as filas `core.eventos`, `cobranca.eventos` e `leitura.pedidos`, com as ligações de cada uma.

Pronto quando: C3, C4 e C5 aprovam o PR (é o contrato dos três); cada evento tem exatamente um publicador.

### T07 · Outbox e consumidor no core
Dono: C3 · Depende de: T03, T06 · Plano: 4.1 (regra 6), 4.2
- Tabela `outbox`, gravação na mesma transação do dado, publicador em segundo plano que marca como enviado.
- Consumidor da fila `core.eventos` que ignora evento cujo `id` já foi processado.

Pronto quando: há teste mostrando que um rollback não deixa evento na outbox, e que um evento repetido é processado uma vez só.

### T08 · Outbox e consumidor na cobrança
Dono: C4 · Depende de: T04, T06 · Plano: 4.1 (regra 6), 4.2
- Os mesmos critérios da T07, em Spring.

Pronto quando: os mesmos testes da T07 passam em JUnit.

### T09 · Esqueleto do front e layout
Dono: C2 · Depende de: T02 · Spec: 7 · Plano: 2.1, 3.1
- Vite + React 19 + TypeScript, Tailwind + shadcn/ui, React Router, TanStack Query. O nginx entrega o build.
- Layout da tela única com painéis redimensionáveis (coluna esquerda, área de abas, painel inferior) e rodapé com os links de Privacidade e Termos, ainda sem conteúdo.

Pronto quando: `https://localhost` mostra o layout; o console do Chrome fica limpo.

### T10 · Evolution API no compose
Dono: C1 · Depende de: T02, T04 · Plano: 3.5, 3.6
- Imagem com versão fixa, banco `evolution`, publicação no RabbitMQ (exchange `evolution`, eventos de mensagem recebida, conexão e QR code) com TLS, HTTPS no REST, log mínimo, chave da API no `.env`.

Pronto quando: uma instância criada por `curl` de dentro da rede gera QR code, conecta o número de teste, e uma mensagem recebida aparece na exchange `evolution`.

### T11 · Teste na escola
Dono: C1 · Depende de: T10 · Plano: 13 · Spec: 10
- Numa máquina da escola: subir o projeto do zero, conectar o WhatsApp de teste, chamar o Gemini e abrir conexão IMAP na porta 993.

Pronto quando: um relato na issue diz o que funcionou, o tempo de subida e o uso de memória. Se algo for bloqueado, o plano B (celular como roteador) é testado.

---

## M1 · Contas e locadoras

Pronto quando: duas pessoas criam duas locadoras, cada dono convida um operador, e nenhum deles vê nada da outra locadora.

### T12 · Cadastro, login e sessão
Dono: C3 · Depende de: T07 · Spec: 5 · Plano: 7
- Identity: o cadastro (nome, nome da locadora, e-mail, senha) cria a locadora e torna a pessoa dona dela. Login e logout.
- Cookie `HttpOnly`, `Secure` e `SameSite=Strict`. Bloqueio após 5 senhas erradas e limite de requisições no login.

Pronto quando: há testes de cadastro → dono, das flags do cookie, do bloqueio e do 429 ao estourar o limite.

### T13 · Equipe e isolamento por locadora
Dono: C3 · Depende de: T12, T16 · Spec: 5 · Plano: 6.1, 7
- Filtro global por `locadora_id` no EF Core.
- O dono cadastra e-mails convidados, e quem se cadastra com esse e-mail entra na locadora. O dono exclui operadores, e o acesso deles cai na hora (*security stamp*). O dono não pode excluir a si mesmo.

Pronto quando: há teste de operador da locadora A tratado como se o recurso da B não existisse, de sessão de operador excluído recusada na requisição seguinte, e de dois donos excluindo o mesmo operador ao mesmo tempo sem erro 500.

### T14 · Editar e excluir locadora
Dono: C3 · Depende de: T13, T16 · Spec: 5, 7.6
- Nome, CNPJ (validado), endereço e semanas de caução (1; 1,5; 2…). Só o dono edita.
- Excluir pede confirmação dupla (digitar o nome da locadora), apaga todos os dados da locadora (os arquivos no disco saem na T19) e publica `locadora.excluida`.

Pronto quando: há teste de operador comum recebendo 403 e de exclusão apagando as linhas de todas as tabelas daquela locadora.

### T15 · 2FA e Minha conta
Dono: C5 · Depende de: T12, T16 · Spec: 5, 7.6 · Plano: 7
- Ativar 2FA: mostra o QR code e confirma com o primeiro código. Login com 2FA pede o código. Minha conta: nome, troca de senha.

Pronto quando: há teste de login com código TOTP certo e errado, e de 2FA ativado só depois de confirmado.

### T16 · Histórico de alterações
Dono: C1 · Depende de: T12 · Spec: 7.2, 6.3.1 · Constituição: 13 · Plano: 6.1
- Interceptador do EF Core que grava `historico_alteracoes` (entidade, id, campo, anterior, novo, operador, quando, motivo) a cada alteração.
- `GET /api/historico/{entidade}/{id}`.

Pronto quando: há teste mostrando que editar um campo gera uma linha com antes e depois, e que um motivo informado é gravado.

### T17 · Telas de conta e locadora
Dono: C2 · Depende de: T09, T12–T15 · Spec: 5, 7.6, 7.7
- Login (com o passo do 2FA), cadastro normal e por convite, Minha conta, Configurações > Locadora (editar e excluir com confirmação dupla) e > Equipe. Rotas protegidas.

Pronto quando: o fluxo completo funciona no Chrome sem erro de console, e a validação do navegador recusa o mesmo que o backend.

---

## M2 · Cadastros e arquivos

Pronto quando: o operador cadastra categoria, carro e motorista, cria uma locação com todos os documentos e vê o valor da assinatura calculado.

### T18 · Categorias de preço
Dono: C5 · Depende de: T13, T16 · Spec: 6.1, 7.6
- Criar e editar (nome livre, três preços semanais: planos 1250, 1500 e livre).

Pronto quando: há teste de preço negativo ou zero recusado e de edição gravando histórico.

### T19 · Arquivos
Dono: C1 · Depende de: T13, T16, T20, T21 · Spec: 8 (Arquivos), 7.2 · Plano: 6.6
- Upload em fluxo contínuo (sem carregar tudo na memória) para o volume, com nome UUID, SHA-256 e tipo conferido pelo conteúdo.
- Limites: imagem 10 MB, PDF 20 MB, vídeo 300 MB, conferidos também no nginx.
- Visualizar e baixar por rota autenticada, com verificação do hash. Excluir, com histórico. `POST` e `GET /interno/arquivos`.

Pronto quando: há testes de arquivo de outra locadora dando 404, de byte alterado no disco aparecendo como "alterado", de `.exe` renomeado para `.pdf` recusado e de arquivo acima do limite recusado.

### T20 · Carros
Dono: C3 · Depende de: T18 · Spec: 8, 6.1 (rodízio) · Plano: 6.5
- Placa (antiga e Mercosul), modelo, ano, categoria, categorias da Uber, licenciamento em PDF. Km atual e km da última revisão só na criação.
- Rodízio calculado pelo último dígito da placa. Publica `carro.salvo`.

Pronto quando: há teste do rodízio para os 10 finais de placa nos dois formatos e de que o `PUT` não aceita os campos de km (o formulário de edição vira tela na T43, junto com o painel "Motorista e carro").

### T21 · Motoristas
Dono: C3 · Depende de: T07, T13, T16 · Spec: 8
- Nome, CPF (com dígito verificador), RG, CNH, telefone, endereço, e-mail opcional, documentos escaneados. CPF e telefone únicos por locadora (o bot acha o motorista pelo telefone). Publica `motorista.salvo`.

Pronto quando: há teste de CPF inválido recusado e de CPF repetido na mesma locadora devolvendo o cadastro existente.

### T22 · Cópias na cobrança
Dono: C4 · Depende de: T08, T20, T21 · Plano: 3.3, 6.3
- Consome `motorista.salvo` e `carro.salvo` e mantém `motoristas_ref` e `carros_ref`. O km do carro só é aceito na criação.

Pronto quando: há teste mostrando que um `carro.salvo` de edição não sobrescreve o km atual.

### T23 · Cálculo da assinatura
Dono: C4 · Depende de: T04 · Spec: 6.1, 6.2 · Constituição: 13
- `POST /interno/calculos/assinatura`: valor do dia = preço ÷ 6; proporcional = dias depois da retirada até domingo, sem o rodízio; caução = preço × semanas.

Pronto quando: há testes do exemplo da spec (Básico 1250, retirada na quarta, rodízio na sexta → R$300 + R$600 = R$900), de retirada no domingo (proporcional zero), de rodízio no dia da retirada e de caução de 1,5 semana.

### T24 · Nova locação
Dono: C3 · Depende de: T20, T21, T23, T14, T19 · Spec: 6.2, 8 · Plano: 6.1, 6.5
- Motorista (novo ou pelo CPF existente), carro livre, plano, data de retirada, km inicial, contrato, vistoria e vídeo de perícia.
- Ao salvar, pede o cálculo de novo à cobrança e congela o preço semanal e os três preços da categoria. Índice único parcial: uma locação ativa por motorista e uma por carro. Publica `locacao.iniciada`.

Pronto quando: há teste de duas locações simultâneas com o mesmo carro (uma passa, a outra recebe mensagem clara) e de mudança na categoria sem efeito na locação já criada.

### T25 · Locação na cobrança
Dono: C4 · Depende de: T23, T22, T24, T08 · Spec: 6.2 · Plano: 6.5
- Consome `locacao.iniciada`: grava `locacoes_ref`, o lançamento de entrada da caução, a cobrança "assinatura" já paga e a leitura de km inicial.

Pronto quando: há teste de que um evento repetido não duplica a caução.

### T26 · Abas, upload e visualizador
Dono: C2 · Depende de: T09, T19 · Spec: 7.2, 8
- Abas da área de trabalho: abrir, fechar e ir para a aba se ela já estiver aberta.
- Componente de upload com barra de progresso e validação de tipo e tamanho no navegador.
- Aba de documento: PDF, imagem ou vídeo, Baixar e o selo "íntegro" ou "alterado".

Pronto quando: as abas abrem, fecham e vão para a aba que já estiver aberta, e o componente de upload e a aba de documento ficam prontos. Os critérios "abrir o mesmo documento duas vezes não cria duas abas" e "arquivo acima do limite é recusado antes do envio" são testados na T27, que tem a tela onde isso acontece.

### T27 · Formulários de cadastro
Dono: C2 · Depende de: T26, T18, T20, T21, T24 · Spec: 7.1 (+ Novo), 7.6, 8
- + Novo: Nova locação (com o valor calculado aparecendo ao escolher carro, plano e data), Novo carro, Nova categoria. Configurações > Categorias de preço.

Pronto quando: o valor mostrado na tela bate com o exemplo da spec, a locação criada aparece no core, abrir o mesmo documento duas vezes não cria duas abas, e um arquivo acima do limite é recusado antes do envio.

---

## M3 · Conversa no WhatsApp

Pronto quando: o operador conecta o número pelo QR code, conversa com um motorista pelo site em tempo real, e uma pergunta do motorista acende o destaque vermelho e gera o alerta.

### T28 · Conexão de WhatsApp por locadora
Dono: C1 · Depende de: T08, T10, T24, T12 · Spec: 7.6 · Plano: 3.3, 3.5
- Na cobrança: `instancias_whatsapp`, endpoints internos para conectar (devolve o QR code) e ver a situação. Consome os eventos de conexão e de QR code da Evolution e publica `whatsapp.conexao_atualizada`.
- No core: `GET /api/whatsapp` e `POST /api/whatsapp/conectar` (só o dono conecta), que a tela da T34 usa.
- Ao receber `locadora.excluida`, apaga a instância e os dados da locadora.

Pronto quando: há teste com os eventos da Evolution gravados (sem número de verdade) gerando o evento nosso.

### T29 · Alertas
Dono: C4 · Depende de: T08, T22 · Spec: 7.4 · Plano: 4.4
- Tabela, criação, lista com filtros (tipo, tratados), ordenação e paginação, marcar como tratado (quem e quando). Publica `alerta.criado` e `alerta.atualizado`.

Pronto quando: há teste de paginação e filtro, e de dois operadores tratando o mesmo alerta (o segundo recebe "já tratado por Fulano"; testado na cobrança, a tela é da T43).

### T30 · Receber mensagens
Dono: C4 · Depende de: T22, T28, T29, T19, T31 · Spec: 6.5, 7.1 · Plano: 5, 6.5
- Consome as mensagens recebidas da Evolution. Ignora repetidas (pelo id do WhatsApp), `fromMe`, grupos e status.
- Baixa a mídia e grava no core. Acha o motorista pelo telefone (com e sem o 9º dígito). Publica `conversa.mensagem_recebida`.
- Mensagem que não é imagem, ou que vem de número não cadastrado: resposta padrão, alerta "mensagem para responder" e `conversa.precisa_resposta`.

Pronto quando: há testes de cada caso ignorado, do 9º dígito, de número não cadastrado e de texto gerando o alerta.

### T31 · Enviar mensagens
Dono: C1 · Depende de: T28, T19, T22 · Spec: 7.2 · Plano: 4.3
- Na cobrança: um serviço de envio (texto ou arquivo) pela Evolution, usado pelo bot e pelo consumidor de `whatsapp.envio_solicitado`. Publica `conversa.mensagem_enviada` com a situação (enviada ou falhou).

Pronto quando: há teste de falha da Evolution resultando em situação "falhou", sem erro no consumidor.

### T32 · Conversas no core
Dono: C3 · Depende de: T07, T21, T30, T31, T33, T24 · Spec: 6.5, 7.1, 7.2
- Consome `conversa.*` e grava conversas e mensagens. Lista de conversas com prévia, ordenada pela mais recente, com o destaque.
- Responder: grava a mensagem, publica `whatsapp.envio_solicitado` e apaga o destaque. Enviar documento ao motorista.
- Vincular número não cadastrado a um motorista: troca o telefone dele, grava no histórico e publica `motorista.salvo`.

Pronto quando: há teste de que a resposta apaga o destaque, e de que o vínculo grava histórico e publica o evento.

### T33 · Tempo real
Dono: C1 · Depende de: T07, T13 · Spec: 9 (tempo real) · Plano: 4.3
- Hub SignalR em `/hub`, autenticado pelo cookie, com um grupo por locadora. Repassa ao grupo os eventos listados no plano (4.3). O nginx repassa o WebSocket.

Pronto quando: há teste de que um operador da locadora B não recebe evento da A.

### T34 · Telas de conversa e WhatsApp
Dono: C2 · Depende de: T26, T32, T33 · Spec: 7.1, 7.2, 7.6
- Lista de conversas, aba de conversa (motorista, bot e operador diferenciados, com o nome do operador), campo de resposta, "Enviar ao motorista no WhatsApp" na aba de documento, vínculo de número não cadastrado.
- Configurações > WhatsApp com o QR code ao vivo.
- Conexão SignalR com reconexão automática, recarregando os dados ao reconectar.

Pronto quando: dois navegadores logados veem a mesma conversa atualizar ao vivo; desligar e religar a rede recarrega a lista sem erro de console do nosso código (com a rede desligada, o próprio Chrome escreve no console a falha de rede e do WebSocket; o teste tolera só essas mensagens, só nessa janela).

---

## M4 · Rotina de segunda-feira

Pronto quando: um motorista manda o hodômetro e o comprovante pelo WhatsApp, o operador recebe o alerta "Conferir Pix" com a situação do e-mail do banco e confirma com um clique, e os lembretes e alertas das 9h, 12h e 15h disparam.

### T35 · Leitura de imagens
Dono: C5 · Depende de: T05, T19 · Spec: 6.3, 9 (imagem) · Plano: 3.4
- Consome `leitura.imagem_solicitada`, busca o arquivo no core e chama o Gemini pedindo JSON no formato nosso, validado pelo Pydantic.
- Publica `leitura.imagem_lida`: tipo (hodômetro, comprovante, outra), km ou valor, data, horário e pagador, e a confiança. Limite ou queda do Gemini: tenta de novo com espera crescente e, se não conseguir, publica falha de sistema.

Pronto quando: há testes com respostas gravadas do Gemini (válida, JSON quebrado, campo faltando e erro 429).

### T36 · Caixa de e-mail do banco no core
Dono: C5 · Depende de: T14, T19 · Spec: 7.6 · Plano: 3.4, 4.4, 7
- Servidor, usuário, senha (criptografada pelo Data Protection, com as chaves num volume) e remetente do banco, editáveis só pelo dono.
- `GET /interno/caixas-email`. A tela fica em Configurações > E-mail do banco.

Pronto quando: há teste de que a senha não aparece em texto no banco nem em nenhuma resposta de `/api`.

### T37 · Leitura de e-mails
Dono: C5 · Depende de: T35, T36 · Spec: 6.3 · Plano: 3.4
- Laço de 2 minutos: para cada caixa, IMAP com SSL em modo só leitura (`EXAMINE` e `BODY.PEEK`, que não marcam nada como lido), e-mails dos últimos 2 dias vindos do remetente.
- O Gemini extrai os dados. Publica `leitura.aviso_pix_lido` com o `Message-ID`. Falha de login numa caixa não para as outras.

Pronto quando: há teste com um servidor IMAP falso mostrando que nenhuma mensagem ganha a marcação de lida, e que uma caixa com senha errada não interrompe as outras.

### T38 · Bot: fluxo da imagem
Dono: C4 · Depende de: T30, T35, T39, T40 · Spec: 6.3 (foto ilegível) · Plano: 5
- Imagem de motorista com locação ativa: publica `leitura.imagem_solicitada`.
- Ao receber `leitura.imagem_lida`: hodômetro vai para a T39, comprovante vai para a T40. Outra imagem, ilegível ou com baixa confiança: o bot pede outra. Na 2ª falha seguida, alerta "foto ilegível". Falha de sistema avisa o motorista que a leitura vai demorar, sem contar como falha dele.

Pronto quando: há teste de uma falha, de duas falhas seguidas (alerta) e de falha e acerto (contador zerado).

### T39 · Km, multa e revisão
Dono: C4 · Depende de: T25, T29, T31, T22 · Spec: 6.3 (km e multa, leitura suspeita, foto em outro dia), 6.3.1, 6.4 · Constituição: 13
- Toda foto atualiza o km do carro. Para a semana, vale a primeira leitura a partir de segunda 0h. Km da semana = leitura − leitura anterior.
- Leitura menor que a anterior, ou mais de 5.000 km na semana: alerta "leitura suspeita", sem multa.
- Multa pela tabela da spec, que vira cobrança e mensagem do bot, com o alerta "km excedido". Revisão: 9.000 km gera alerta e mensagem; 10.000 km sem revisão gera "revisão vencida".
- Endpoints internos: corrigir leitura (com motivo, histórico e recálculo), cancelar multa e registrar revisão.

Pronto quando: há testes de 1310 km no Básico 1250 dando R$100, das bordas 1250, 1251, 1500 e 1501 nos planos 1250 e 1500, do plano livre sem multa, das duas leituras suspeitas, de 9.000 e 10.000 km, e da correção recalculando a multa.

### T40 · Pagamentos e casamento com o e-mail
Dono: C4 · Depende de: T39, T29, T31 · Spec: 6.3 (conferência, a que cobrança pertence) · Constituição: 13
- Comprovante vira pagamento "a conferir" e gera o alerta. Um comprovante idêntico a outro já enviado é recusado, com aviso ao operador.
- `avisos_pix_email` com o `Message-ID` único. Casamento: mesmo valor, mesma data, diferença de até 5 minutos e aviso ainda não casado. O alerta mostra ✓, ⚠ ou ✗, e é atualizado se o e-mail chegar depois.
- Confirmar ou recusar com controle otimista. Confirmado: quita a cobrança aberta mais antiga de mesmo valor, ou vira crédito. Valor sem cobrança correspondente: o operador atribui. O bot avisa o motorista nos dois casos.

Pronto quando: há testes da folga (5:00 casa, 5:01 não), de dois avisos possíveis, de e-mail chegando depois, de comprovante duplicado, de dois operadores confirmando juntos, de crédito e de atribuição.

### T41 · Agendamentos de segunda
Dono: C4 · Depende de: T39, T40 · Spec: 6.3 (horários) · Plano: 8
- Fuso `America/Sao_Paulo`. 0h: abre as cobranças da semana (a partir da primeira segunda depois da retirada). 9h: lembrete a todos com locação ativa. 12h: lembrete só para quem tem pendência. 15h: um alerta por pendência e cobranças em atraso. Diário: revisão próxima e vencida.
- Comprovante enviado, mesmo a conferir, conta como pago.

Pronto quando: há testes com relógio fixo de quem recebe o lembrete das 12h e de quais alertas saem às 15h.

### T58 · Multa por atraso e bloqueio
Dono: C4 · Depende de: T41, T14, T24 · Spec: 6.3 (multa por atraso e bloqueio), 7.4, 7.6 · Plano: 4.4, 6.2, 6.3, 8 · Constituição: 13
- A locadora configura a multa por atraso (% do preço semanal por dia; padrão 0 = sem multa; só o dono). A locação congela o percentual ao ser criada, e `locacao.iniciada` o leva à cobrança. O tipo de alerta `BLOQUEAR_VEICULO` entra no contrato.
- Todo dia 0h05, a cobrança `SEMANA` ou `MULTA_KM` aberta e vencida ganha um dia de `MULTA_ATRASO`, a partir do dia seguinte ao vencimento. O valor é sempre recalculado dos dias de atraso (preço × % × dias, arredondado só no fim). O que o motorista já pagou da multa não é cobrado de novo (paga a multa e a semana segue aberta: no dia seguinte ela volta a aberta com o restante).
- Quando a origem é paga (Pix confirmado ou baixa manual), a multa para de crescer e, se sobrar multa, nasce uma vez o alerta `MULTA_NAO_PAGA`. Multa cancelada pelo operador não volta nem é recriada. A multa não gera multa sobre ela, nem lembrete das 12h, nem alerta das 15h (quem avisa é a cobrança de origem e, na segunda seguinte, o bloqueio). Comprovante a conferir não conta o dia.
- Segunda 15h, depois dos alertas: Pix com 7 dias ou mais de atraso, ou a 2ª segunda seguida sem km, gera o alerta `BLOQUEAR_VEICULO` ao operador e a mensagem do bot ao motorista ("O veículo poderá ser bloqueado."). O sistema não bloqueia o carro.

Pronto quando: há testes do valor por dia, do dia em que começa, do comprovante a conferir que não conta o dia, do pagamento parcial da multa com a semana aberta (continua crescendo, com o restante), da semana paga parando a multa com o alerta uma vez só, da multa cancelada que não volta, da multa sem lembrete nem alerta das 15h, e dos dois bloqueios.

### T42 · Painel do motorista e ações no core
Dono: C3 · Depende de: T32, T39, T40 · Spec: 7.3, 6.3.1 · Plano: 3.2, 4.4
- Rotas `/api` que repassam à cobrança, com `X-Operador-*`: situação do motorista e do carro, histórico de km, revisões, pagamentos e multas, alertas, confirmar, recusar e atribuir Pix, corrigir leitura, cancelar multa, registrar revisão. O lançamento de desconto de caução fica no M5 (T45/T46/T47), onde a caução é tratada na cobrança.
- `POST /api/conversas` (abre ou cria a conversa do motorista ou do número) e o repasse do histórico da cobrança.
- Locações anteriores do motorista.

Pronto quando: há teste de que a cobrança recebe o operador correto, e de que a 409 da cobrança chega ao navegador como `ok: false`, com mensagem clara.

### T43 · Telas da rotina
Dono: C2 · Depende de: T34, T42, T36, T58 · Spec: 7.3, 7.4, 6.3, 7.6
- Aba Alertas: lista, filtros (inclusive tratados), caixa de tratado e contador ao vivo. Clicar abre a conversa e muda o painel inferior.
- Alerta "Conferir Pix" com o comprovante a um clique, a situação do e-mail e os botões confirmar e recusar.
- Painel "Motorista e carro" com links (documentos, contrato, pagamentos e multas, locações anteriores, histórico de km e revisões, histórico de alterações), editar, Registrar revisão, corrigir leitura (com motivo), cancelar multa e atribuir Pix. Configurações > E-mail do banco e a configuração da multa por atraso (só o dono).

Pronto quando: o roteiro do marco funciona com dois operadores logados, e o contador atualiza nos dois.

---

## M5 · Encerramento e troca

Pronto quando: o operador encerra uma locação e troca carro ou plano de outra, os valores batem com a spec e, 30 dias depois, sai o alerta certo da caução.

### T44 · Cálculos de encerramento e troca
Dono: C4 · Depende de: T23, T39, T40 · Spec: 6.6, 6.7 · Constituição: 13
- Dias não usados: do dia da devolução (inclusive) até domingo, sem o rodízio, paga ou não: a cobrança da semana cai para os dias usados. Última semana com o km final.
- Troca: a caução passa adiante (com a diferença paga ou devolvida no fim), o crédito dos dias não usados abate a assinatura nova, e o dia da troca é cobrado uma vez só.
- `POST /interno/calculos/encerramento` e `/troca`. Locação ainda não recebida devolve 409.

Pronto quando: há testes do exemplo da spec (devolução na quarta, rodízio na sexta: 4 × R$100), de semana não paga, de troca para caução maior e menor, e de troca no dia do rodízio.

### T45 · Encerrar e trocar no core
Dono: C3 · Depende de: T24, T42, T44, T46 · Spec: 6.6, 6.7, 8 (Encerramento)
- Encerramento: km final (a data é a de hoje), perícia e vídeo. Troca de carro: o mesmo do carro devolvido, mais km inicial, vistoria e vídeo do carro novo. Troca só de plano: apenas o km do dia.
- Recalcula na cobrança (só na troca; o encerramento não chama a cobrança ao salvar) e publica `locacao.encerrada` (e `locacao.iniciada`, na troca) na mesma transação.
- Repasses à cobrança dos lançamentos de caução e da baixa manual, e `documentosObrigatorios` (os documentos que cada locação exige).

Pronto quando: há teste de que a troca grava as duas locações e os dois eventos, ou nada.

### T46 · Caução, baixa e pós-encerramento
Dono: C4 · Depende de: T44, T40, T41, T58 · Spec: 6.6, 6.7, 6.3.1
- Consome `locacao.encerrada`: tira o motorista da rotina e gera a multa da última semana.
- Caução e crédito por contrato (a locação e as trocas que vieram dela). Lançar desconto de caução (reparo ou multa de trânsito) a qualquer momento.
- Baixa manual, total ou parcial, em qualquer cobrança, com motivo obrigatório no histórico.
- No 30º dia: saldo positivo gera o alerta "devolver caução"; saldo negativo gera a cobrança do saldo devedor e o alerta "cobrar saldo devedor". Multa não paga entra como desconto.

Pronto quando: há testes com relógio fixo dos dois casos do 30º dia, de multa não paga descontada e de motorista encerrado sem lembrete.

### T47 · Telas de encerramento, troca, caução e baixa
Dono: C2 · Depende de: T43, T45 · Spec: 7.3, 6.6, 6.7
- Ações "Encerrar locação" e "Trocar carro/plano" com o valor calculado. Aba Caução com o lançamento de descontos, baixa manual na aba Financeiro e lista "Faltando" a partir de `documentosObrigatorios`.

Pronto quando: os valores na tela batem com os testes da T44.

---

## M6 · Painel, busca e dados

Pronto quando: o painel mostra os números com filtro de período e atualiza ao vivo, a busca acha motoristas encerrados, e os dados são exportados e importados.

### T48 · Números do painel
Dono: C5 · Depende de: T40, T41, T42 · Spec: 7.5
- Na cobrança, por período: faturamento esperado × recebido por semana, multas por semana (valor e quantidade), inadimplência, km médio por carro por semana e frota por situação (alugado, livre, perto da revisão). Publica `painel.mudou` quando um número muda. No core, a rota `GET /api/painel`.

Pronto quando: há teste com dados fixos conferindo cada número.

### T49 · Tela do painel
Dono: C5 · Depende de: T48, T42, T33, T43 · Spec: 7.5
- Aba Painel com gráficos de barras, linha e pizza (Recharts), filtro de período, atualização com `painel.mudou` e exportação em CSV e PDF.

Pronto quando: confirmar um Pix em outro navegador atualiza o gráfico, e o CSV e o PDF trazem os números filtrados.

### T50 · Busca avançada
Dono: C3 · Depende de: T21, T24, T32 · Spec: 7.1
- Por nome, CPF, e-mail ou placa, com filtros (locação ativa ou encerrada), ordenação e paginação.

Pronto quando: há teste de busca encontrando um motorista encerrado, e de paginação estável com ordenação.

### T51 · Busca e ficha do motorista
Dono: C2 · Depende de: T50, T16, T43 · Spec: 7.1, 7.2, 7.3
- Busca na coluna esquerda com filtros e paginação. Ficha do motorista, com os documentos dele e de cada locação. Os links do painel inferior e a aba de histórico de alterações já foram feitos na T43.

Pronto quando: dois cliques levam da busca a qualquer documento de um motorista (spec, objetivo 2).

### T52 · Exportação e importação
Dono: C5 · Depende de: T42, T20, T18, T24 · Spec: 7.6
- No core: exportar motoristas, carros, locações e pagamentos (estes vêm da cobrança) em CSV, JSON e XML. Importar carros por CSV, validando linha a linha: se houver qualquer erro, nada é importado, e o relatório lista o erro de cada linha. **[decisão dos líderes: tudo ou nada]** Inclui a rota interna de cobranças (para os pagamentos exportados). A tela fica em Configurações. Só o dono exporta e importa.

Pronto quando: há testes dos três formatos e de um CSV com uma linha ruim (nada importado, erro apontado na linha).

### T53 · Limpeza de 5 anos no core
Dono: C3 · Depende de: T32, T45, T19, T54, T50, T52 · Spec: 6.8 · Constituição: 12 · Plano: 6.5, 8
- Rotina diária: motoristas com a última locação encerrada há mais de 5 anos têm os dados pessoais apagados, as mensagens e os arquivos removidos, e `motorista.anonimizado` é publicado.

Pronto quando: há teste com relógio fixo (4 anos e 364 dias: nada muda; 5 anos: apagado).

### T54 · Anonimização na cobrança
Dono: C4 · Depende de: T08, T48 · Spec: 6.8 · Plano: 6.5
- Consome `motorista.anonimizado`: apaga nome e telefone da cópia e o nome do pagador em pagamentos e avisos. Valores, datas e km ficam.

Pronto quando: há teste de que os números do painel não mudam depois da anonimização.

---

## M7 · Entrega

Pronto quando: o projeto passa no ensaio completo da avaliação, numa máquina da escola.

### T55 · Privacidade e Termos
Dono: C2 · Depende de: T09, T17 · Spec: 6.8, 7.7, 5 · Constituição: 6
- Conteúdo real: dados coletados e por quê, envio ao Gemini gratuito, retenção de 5 anos, exclusão da locadora, direitos do titular pela LGPD, contato, transferência internacional (Gemini), cookies (só os necessários), quem acessa e quem exporta. Acessíveis sem login, no rodapé de todas as telas.

Pronto quando: o PO e o PM revisam o texto contra a spec.

### T59 · Dados de demonstração
Dono: C1 · Depende de: T19, T24, T46, T48, T50, T53 · Spec: 10 · Constituição: 11
- Script (`bash infra/demo/demo.sh`) que cria contas, carros e locações pelas rotas do core e insere as semanas passadas nos bancos: uma locadora de demonstração com 12 semanas de histórico, só com dados fictícios. Os motoristas fictícios usam o DDD 20, que não existe, para o bot não alcançar ninguém.

Pronto quando: num banco zerado, o painel mostra 12 semanas com os valores esperados do plano e a busca tem duas páginas.

### T56 · README
Dono: C3 · Depende de: todas, inclusive a T59 · Constituição: 9 · Spec: 9
- Primeira linha em itálico no formato do subject (sem os sinais `< >`), papéis do time, módulos com justificativa e pontos, justificativa do módulo próprio (as 4 perguntas do subject), uso de IA, como subir (`.env` e `docker compose up --build`).
- Títulos das seções em inglês, como no subject, e texto em português. Também as seções que o subject pede e o quadro não citava: Project Management, Technical Stack, Database Schema, Features List, Individual Contributions. Uso de IA: mantém o texto atual e acrescenta o Gemini no produto.

Pronto quando: cada item da seção VI do subject está presente.

### T57 · Ensaio da avaliação
Dono: C1 · Depende de: todas, inclusive a T59 · Constituição: 1, 2, 8, 11
- Numa máquina da escola: `git clone`, `.env` a partir do `.env.example`, um comando. Roteiro de demonstração por módulo, com dois operadores simultâneos e o console do Chrome aberto em todas as telas.
- Treino de 5 mudanças ao vivo, sem IA, numa branch descartável (subject, capítulo VII). Relato na issue.

Pronto quando: todos os módulos do README funcionam de ponta a ponta. Módulo que falhar sai do README (constituição, item 11).

---

## Cobertura

| Spec | Tarefas |
|---|---|
| 5 · Quem usa e cadastro de conta | T12, T13, T14, T15, T17 |
| 6.1 · Termos (categoria, plano, rodízio, valor do dia) | T18, T20, T23 |
| 6.2 · Assinatura | T23, T24, T25, T27 |
| 6.3 · Rotina de segunda (inclui multa por atraso e bloqueio) | T35, T37, T38, T39, T40, T41, T43, T58 |
| 6.3.1 · Correções (inclui a baixa manual) | T16, T39, T40, T42, T43, T46 |
| 6.4 · Revisão | T39, T41, T43 |
| 6.5 · Mensagens que não são foto | T30, T32, T34 |
| 6.6 · Troca | T44, T45, T46, T47 |
| 6.7 · Encerramento | T44, T45, T46, T47 |
| 6.8 · Retenção | T53, T54, T55 |
| 7.1 · Coluna esquerda | T27, T32, T34, T50, T51 |
| 7.2 · Área de trabalho | T16, T26, T34, T43, T51 |
| 7.3 · Motorista e carro | T42, T43, T47 |
| 7.4 · Alertas | T29, T43, T58 |
| 7.5 · Painel de números | T48, T49 |
| 7.6 · Configurações | T14, T15, T17, T18, T28, T34, T36, T43, T52, T58 |
| 7.7 · Páginas públicas | T17, T55 |
| 8 · Cadastros e arquivos | T19, T20, T21, T24, T26, T27, T45 |
| 10 · Restrições e demonstração | T01, T02, T10, T11, T57, T59 |

| Módulo | Tarefas |
|---|---|
| Tempo real | T33, T34 |
| Organizações | T12, T13, T14, T17 |
| Framework front e back | T03, T09 |
| Módulo próprio | T28–T31, T38–T41, T44, T46, T58 |
| Microsserviços | T01–T08, T10 |
| Analytics | T48, T49 |
| ORM | T03 |
| Arquivos | T19, T26 |
| Busca avançada | T29, T50, T51 |
| Imagem | T35 |
| 2FA | T15 |
| Exportação e importação | T52 |

| Constituição | Tarefas |
|---|---|
| 1 · Um comando | T01, T57 |
| 2 · Chrome limpo | definição de pronto, T57 |
| 3 · HTTPS em tudo | T02 |
| 4 · Validação dupla | definição de pronto |
| 5 · Segredos fora do Git | T01 |
| 6 · Privacidade e Termos | T55 |
| 7 · Banco bem modelado | T06 e cada tarefa com tabela |
| 8 · Vários usuários | T13, T24, T29, T33, T40 |
| 9 · README | T56 |
| 10 · Explicável | definição de pronto |
| 11 · Módulo só funcionando | T57, T59 |
| 12 · LGPD | T13, T53, T54, T55 |
| 13 · Dinheiro testado e rastreável | T16, T23, T39, T40, T44, T46, T58 |
| 14, 15 · Spec antes, branch e PR | como trabalhar uma tarefa |
