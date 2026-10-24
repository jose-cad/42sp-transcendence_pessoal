# Plano técnico

Status: aprovado pelo dono · 2026-10-17 · atualizado com os ajustes dos planos M0 a M7 em 2026-10-24
Obedece a [`00-constituicao.md`](00-constituicao.md) e implementa [`01-spec-produto.md`](01-spec-produto.md). Esta parte diz **como** o produto vai ser construído. A divisão em tarefas vem no próximo documento.

Marcações:
- **[decisão do dono]**: o dono do projeto decidiu.
- **[decisão dos líderes]**: os líderes técnicos decidiram, e o dono revisa neste documento.

---

## 1. Visão geral

```
                         navegador (Chrome)
                               │ HTTPS (443)
                         ┌─────▼─────┐
                         │   nginx   │  entrega o front e repassa /api e /hub
                         └─────┬─────┘
                               │
                    ┌──────────▼──────────┐
                    │   core (C#)         │  única porta do front
                    │   ASP.NET Core      │
                    └───┬────────┬────────┘
              REST      │        │ fila
            interno     │        │
   ┌────────────────────▼─┐   ┌──▼───────────────────────────┐
   │  cobrança (Java)     │   │          RabbitMQ            │
   │  Spring Boot         │◄──►  exchange "frota" (topic)    │
   │  bot + regras        │   │  exchange "evolution"        │
   └──────┬───────────────┘   └──▲───────────────▲───────────┘
          │ REST                 │               │
   ┌──────▼───────────┐          │        ┌──────┴───────────┐
   │ Evolution API    ├──────────┘        │ leitura (Python) │──► Gemini
   │ (WhatsApp)       │ eventos           │ IA + e-mail      │──► caixa de e-mail (IMAP)
   └──────────────────┘                   └──────────────────┘

   PostgreSQL: um container, um banco por serviço (core, cobranca, evolution)
   Arquivos: volume do Docker, controlado pelo core
```

Tudo sobe com `docker compose up --build`. **[decisão dos líderes]**

## 2. Stack

| Peça | Tecnologia | Por quê |
|---|---|---|
| Front | React 19 + TypeScript, Vite | **[decisão do dono: React]** Padrão de mercado. Vite gera o build estático que o nginx entrega. **[decisão dos líderes: TypeScript e Vite]** |
| Core | C# · .NET 10 (LTS) · ASP.NET Core · Entity Framework Core + Npgsql | **[decisão do dono: C#]** ASP.NET Core conta como framework de backend e o EF Core como ORM. |
| Cobrança | Java 25 (LTS) · Spring Boot 4 · Spring Data JPA · Spring AMQP | Regra de negócio pesada com agendamento: o Spring cobre os dois. **[decisão dos líderes, a partir da sugestão do dono de usar Java no chatbot]** |
| Leitura | Python 3.13 · SDK `google-genai` · `pika` · `imaplib` (biblioteca padrão) · Pydantic | **[decisão do dono: Python para a IA]** Não expõe HTTP, então não precisa de framework web. |
| WhatsApp | Evolution API v2 (container pronto) | **[decisão do dono]** O dono já usou em produção. Só transporte: nenhuma regra de negócio. |
| Fila | RabbitMQ 4 | **[decisão do dono: mensageria]** A Evolution API publica nele sem código extra. |
| Banco | PostgreSQL 18 | **[decisão do dono]** |
| Entrada | nginx | HTTPS, entrega do front, repasse para o core. |

Linguagens: C#, Java, Python e TypeScript. **[decisão do dono: no mínimo 3]**

As versões exatas das imagens e dos pacotes ficam fixadas na primeira tarefa de cada serviço, e nunca como `latest`.

### 2.1 Bibliotecas menores **[decisão dos líderes]**

- **Front:** React Router (rotas), TanStack Query (busca e cache dos dados da API), React Hook Form + Zod (formulários e validação no navegador), shadcn/ui + Tailwind CSS (componentes visuais), react-resizable-panels (layout estilo VS Code com painéis redimensionáveis), Recharts (gráficos), `@microsoft/signalr` (tempo real), jsPDF (PDF do painel de números).
- **Core:** ASP.NET Core Identity (senha, 2FA por app autenticador, bloqueio após tentativas erradas), SignalR (tempo real), `RabbitMQ.Client`, CsvHelper (CSV), Data Protection do próprio .NET (criptografia da senha do e-mail do banco).
- **Cobrança:** Flyway (versões do banco), `RestClient` do Spring (chamadas à Evolution API e ao core), JUnit 5 (testes).
- **Leitura:** `httpx` (busca o arquivo no core), pytest (testes).

## 3. Serviços

Cada serviço tem uma única responsabilidade, seu próprio container e, quando guarda dados, seu próprio banco. Nenhum serviço lê o banco de outro. **[decisão do dono: banco por serviço]**

### 3.1 Front (React)

A tela única da spec (seção 7) e as páginas públicas. Fala **só com o core**, por `/api` (REST) e `/hub` (SignalR). Valida todo formulário com Zod antes de enviar.

### 3.2 Core (C#)

É a única porta de entrada do front e o dono de:
- contas, login, 2FA, locadoras e equipe (módulo de organizações);
- cadastros: categorias de preço, carros, motoristas, locações (com os preços congelados);
- arquivos: upload, hash, visualização, download, exclusão;
- conversas e mensagens (o que aparece na aba de conversa);
- histórico de alterações das entidades acima;
- busca de motoristas e carros;
- exportação e importação (seção 7.6);
- tempo real: avisa o navegador, por SignalR, do que chega da fila (só tipo e ids, seção 4.3);
- configurações da locadora, incluindo a caixa de e-mail do banco (senha criptografada);
- limpeza de dados pessoais após 5 anos (seção 6.8).

Quando o front pede algo que é da cobrança (alertas, pagamentos, cálculos, números do painel), o core repassa o pedido para a cobrança por REST interno. Assim o login existe num lugar só e o front não precisa conhecer os outros serviços. No mercado, isso se chama **BFF** (*backend for frontend*). **[decisão dos líderes]**

### 3.3 Cobrança (Java): o módulo próprio

O dono de todas as regras da seção 6 que envolvem dinheiro, km e o bot:
- **todo cálculo de dinheiro** do sistema: proporcional da assinatura, multa de km, dias não usados, crédito na troca, caução e saldo final. Ficar num lugar só facilita testar todos (constituição, item 13);
- cobranças, pagamentos (comprovantes), crédito e saldo de caução;
- casamento do comprovante com os avisos de Pix do e-mail (seção 6.3);
- leituras de km, leitura suspeita, revisão;
- alertas: cria, lista com filtros, ordenação e paginação, e marca como tratados;
- o bot: decide o que responder a cada mensagem, conta as falhas seguidas de foto e envia lembretes;
- agendamentos: 9h, 12h e 15h de segunda, abertura das cobranças semanais, atrasos, 30 dias da caução;
- números do painel de analytics;
- **único serviço que fala com a Evolution API**: recebe os eventos dela, envia mensagens e gerencia a conexão (QR code) de cada locadora. Como o bot já precisa ler cada mensagem para decidir a resposta, é ele quem repassa ao core, no nosso formato de evento, o que o core precisa guardar. Assim o core não precisa conhecer o formato da Evolution.

Guarda uma cópia dos dados de motorista, carro e locação de que precisa, atualizada pelos eventos do core (consistência eventual: a cópia fica igual à original pouco depois, não no mesmo instante).

### 3.4 Leitura (Python)

Lê e devolve dados estruturados. Não decide nada de negócio.
- **Imagens:** recebe o pedido pela fila, busca o arquivo no core, manda ao Gemini e devolve o tipo (hodômetro, comprovante ou outra), os dados lidos (km, ou valor, data, horário e pagador) e a confiança da leitura. O Gemini responde num formato JSON definido por nós, que o Pydantic valida.
- **E-mail:** a cada 2 minutos, para cada locadora com caixa configurada, lê por IMAP os e-mails recentes do remetente do banco, sem alterar a caixa (só leitura). Usa o Gemini para extrair valor, data, horário e pagador de cada aviso de Pix e publica o resultado. **[decisão dos líderes: usar a IA também no e-mail, porque o formato do aviso do banco ainda é desconhecido (spec, 6.3, item "Em aberto")]**
- **Não tem banco.** Pode reenviar um aviso já enviado (por exemplo, depois de reiniciar). A cobrança ignora repetidos pelo identificador do e-mail (`Message-ID`).

### 3.5 Evolution API (WhatsApp)

Container pronto, sem código nosso. Uma instância por locadora, com o nome igual ao id da locadora. Configurada por variáveis de ambiente para publicar no RabbitMQ os eventos de mensagem recebida, conexão e QR code. Usa o seu próprio banco no Postgres.

### 3.6 Infra

- **nginx:** única porta exposta (443, com o 80 redirecionando para o 443). Entrega o build do React e repassa `/api` e `/hub` (com WebSocket) para o core. Não repassa nada de `/interno`.
- **Certificado HTTPS:** autoassinado, gerado com `openssl` na primeira subida se não existir. O Chrome mostra uma tela de aviso de certificado antes de entrar. Essa tela não é erro de console. **[decisão dos líderes]**
- **Postgres:** um script de inicialização cria os bancos `core`, `cobranca` e `evolution`, cada um com usuário e senha próprios, e cada usuário só acessa o seu banco. Isso garante na prática que um serviço não lê o banco do outro.
- **Criptografia entre os serviços:** todo tráfego interno também é criptografado (TLS, a mesma tecnologia do HTTPS). Um container `certs` roda uma vez na subida e, se ainda não existirem, gera com `openssl` uma autoridade certificadora própria do projeto e um certificado para cada serviço que recebe conexões. Cada serviço confia só nessa autoridade. **[decisão do dono]**
  - nginx → core: HTTPS.
  - core ↔ cobrança, cobrança e leitura → core: HTTPS no REST interno.
  - RabbitMQ: só a porta com TLS (5671); a porta sem criptografia (5672) fica desligada.
  - Postgres: SSL ligado, e os serviços conectam exigindo certificado válido (`sslmode=verify-full`).
  - Evolution API: HTTPS no REST, TLS para o RabbitMQ e SSL para o Postgres. As variáveis exatas da Evolution são conferidas na tarefa de infra.
- **Ordem de subida:** o compose usa *healthcheck* (teste de "está pronto") no Postgres e no RabbitMQ, e os serviços só sobem depois deles.
- **Rede:** todos os containers numa rede interna do Docker. Só o nginx publica porta na máquina.

## 4. Como os serviços conversam

### 4.1 Regras **[decisão dos líderes]**

1. O front fala só com o core.
2. **Evento pela fila** quando algo aconteceu e outros serviços podem querer saber ("locação iniciada", "mensagem recebida"). Quem publica não espera resposta.
3. **REST interno** quando quem pede precisa da resposta na hora (um cálculo para mostrar no formulário, a lista de alertas, confirmar um Pix e saber se deu certo). Só o core chama a cobrança, e só a cobrança e a leitura chamam o core.
4. **Arquivo nunca viaja na fila.** Ele é gravado no core, e a mensagem leva só o id. (No mercado, isso se chama *claim check*.)
5. Toda chamada REST interna leva o cabeçalho `X-Interno-Token`, com um segredo do `.env`. As chamadas que alteram dados levam também `X-Operador-Id` e `X-Operador-Nome`, para o histórico registrar quem fez a alteração.
6. **Nenhum evento se perde (outbox).** Core e cobrança não publicam direto na fila. Eles gravam o evento numa tabela `outbox`, na mesma transação do dado que mudou. Uma rotina em segundo plano lê essa tabela, publica no RabbitMQ e marca o evento como enviado; as linhas enviadas há mais de 1 dia são apagadas pela mesma rotina, e a exclusão da locadora apaga na hora os eventos dela que ainda estão na tabela (o envelope leva dado pessoal) **[decisão dos líderes]**. Sem isso, uma locação poderia ser salva sem que a cobrança ficasse sabendo, e o motorista nunca seria cobrado. A leitura não tem banco, então publica direto: no pior caso, um aviso de e-mail é lido de novo no ciclo seguinte.

### 4.2 RabbitMQ

- Exchange `frota`, do tipo *topic*: a *routing key* é o nome do evento.
- Uma fila durável por serviço consumidor (`core.eventos`, `cobranca.eventos`, `leitura.pedidos`), ligada só aos eventos que ele consome.
- A exchange `evolution` é da Evolution API. Só a cobrança consome dela. As filas `evolution.*` (conexão, QR code e mensagens recebidas) também nascem no `definicoes.json`, com os argumentos exatos que a Evolution declara (com argumentos diferentes ela derrubaria o canal); a espera crescente das tentativas vem pela política `evolution-espera`, que cobre as filas `^evolution\.`.
- Envelope de todo evento nosso, em JSON:

```json
{
  "id": "uuid do evento",
  "evento": "locacao.iniciada",
  "correlacao": "uuid que acompanha um fluxo de ponta a ponta (logs)",
  "locadoraId": "uuid",
  "ocorridoEm": "2026-10-19T12:00:00Z",
  "dados": { }
}
```

- A entrega é "pelo menos uma vez": um evento pode chegar repetido. Todo consumidor precisa aguentar repetição (por exemplo, ignorar uma mensagem de WhatsApp cujo id já foi gravado).

### 4.3 Eventos

| Evento | Publica | Consome | Dados principais |
|---|---|---|---|
| `locadora.excluida` | core | cobrança | — (a cobrança apaga os dados da locadora e a instância de WhatsApp) |
| `motorista.salvo` | core | cobrança | id, nome, telefone |
| `motorista.anonimizado` | core | cobrança | id |
| `carro.salvo` | core | cobrança | id, placa, modelo, dia de rodízio; km atual e km da última revisão **só na criação** (seção 6.5) |
| `locacao.iniciada` | core | cobrança | id, motorista, carro, plano, preço semanal e os 3 preços congelados, semanas de caução, valor de caução e proporcional pagos, data de retirada, km inicial, locação anterior (na troca), quem criou a locação e conferiu o pagamento da assinatura (`operadorId` e `operadorNome`, seção 6.5; já no contrato da T06 e preenchidos pela T24), percentual da multa por atraso congelado na locação (`multaAtrasoPercentual`; nasce na T58). Na troca, o valor de caução é só a diferença paga e o proporcional inclui o dia da troca (muda na T45) |
| `locacao.encerrada` | core | cobrança | id, data da devolução, km final, motivo (devolução ou troca), quem registrou o encerramento (`operadorId` e `operadorNome`; nascem na T45) |
| `whatsapp.envio_solicitado` | core | cobrança | mensagem id, telefone, texto ou arquivo id, operador |
| `conversa.mensagem_recebida` | cobrança | core | id do WhatsApp, telefone, motorista (se conhecido), texto, arquivo id, data, tipo de mídia (`midia`: imagem, vídeo, áudio, documento, figurinha, localização ou outra; vazio no texto; nasce na T30) |
| `conversa.mensagem_enviada` | cobrança | core | autor (bot ou operador), mensagem id, texto ou arquivo id, situação (enviada ou falhou) |
| `conversa.precisa_resposta` | cobrança | core | motorista ou telefone (acende o destaque vermelho) |
| `whatsapp.conexao_atualizada` | cobrança | core | situação, QR code (quando houver) |
| `alerta.criado` / `alerta.atualizado` | cobrança | core | alerta resumido (tipo, motorista, carro, detalhe, hora, tratado); a lista de tipos ganha `BLOQUEAR_VEICULO` na T58 |
| `painel.mudou` | cobrança | core | — (o front busca os números de novo) |
| `motorista.situacao_mudou` | cobrança | core | motorista id (o front atualiza o painel inferior) |
| `leitura.imagem_solicitada` | cobrança | leitura | arquivo id |
| `leitura.imagem_lida` | leitura | cobrança | arquivo id, tipo, km ou valor/data/horário/pagador, confiança, ou falha |
| `leitura.aviso_pix_lido` | leitura | cobrança | Message-ID, valor, data, horário, pagador (se houver) |

O core não repassa ao navegador os eventos `conversa.*`, `alerta.*`, `painel.mudou`, `motorista.situacao_mudou` e `whatsapp.conexao_atualizada`: manda por SignalR só um aviso com o tipo e os ids, só para os operadores da locadora do evento (um grupo do SignalR por locadora), e o front busca o dado na API. Nenhum dado pessoal nem o QR code passa pelo WebSocket.

### 4.4 REST interno

**Core oferece** (para a cobrança e a leitura):
- `POST /interno/arquivos`: grava um arquivo (por exemplo, a foto recebida no WhatsApp) e devolve id e hash.
- `GET /interno/arquivos/{id}`: devolve o conteúdo do arquivo.
- `GET /interno/caixas-email`: locadoras com caixa configurada, com os dados de acesso já descriptografados (só a leitura chama).

**Cobrança oferece** (só para o core):
- cálculos: assinatura, encerramento e troca. O formulário mostra o valor antes de salvar, e o core pede o cálculo de novo ao salvar, sem confiar no valor que veio do navegador;
- alertas: listar (com filtros, ordenação e paginação) e marcar como tratado;
- pagamentos: confirmar, recusar, atribuir a uma cobrança; multas: cancelar (spec, 6.3.1); cobranças: dar baixa manual;
- leituras de km: corrigir; revisões: registrar;
- situação do motorista (semana, multa, km × franquia, caução) e do carro (km atual, km para a revisão);
- caução: lançar desconto;
- painel: números por período;
- WhatsApp: conectar (`POST /interno/whatsapp/conectar`, devolve o QR code) e ver a situação da conexão (`GET /interno/whatsapp`).

O nome do operador vai codificado em `X-Operador-Nome` (`Uri.EscapeDataString` no core, decodificado em UTF-8 na cobrança), porque cabeçalho HTTP só leva ASCII: "João" viaja como `Jo%C3%A3o`.

Os caminhos exatos e os formatos de cada chamada são definidos nas tarefas, mas cada tela do front só pode chamar um caminho que esteja listado na tarefa correspondente do core (problema do projeto antigo: o front chamava rotas que não existiam).

## 5. Fluxos principais

**Foto do hodômetro ou comprovante**
1. O motorista manda a imagem. A Evolution API publica o evento na exchange `evolution`.
2. A cobrança baixa a imagem da Evolution, grava no core (`POST /interno/arquivos`), publica `conversa.mensagem_recebida` e `leitura.imagem_solicitada`.
3. O core grava a mensagem e mostra na conversa em tempo real.
4. A leitura lê com o Gemini e publica `leitura.imagem_lida`.
5. A cobrança aplica a regra: grava a leitura de km (calcula multa e revisão) ou cria o pagamento "a conferir" (procura o aviso do e-mail e cria o alerta "Conferir Pix"). Se a imagem não for legível, pede outra, e na segunda falha seguida cria o alerta "foto ilegível". Responde pelo bot e publica os eventos.

**Aviso de Pix no e-mail**
1. A leitura encontra um aviso novo e publica `leitura.aviso_pix_lido`.
2. A cobrança grava o aviso (ignora se o `Message-ID` já existe). Se houver um comprovante "a conferir" que case com ele, atualiza o alerta para "✓ e-mail do banco encontrado".

**Operador confirma o Pix**
1. Front → core → cobrança (`confirmar`).
2. A cobrança quita a cobrança, registra no histórico, manda a mensagem do bot ao motorista e publica `alerta.atualizado`, `motorista.situacao_mudou` e `painel.mudou`.
3. Se dois operadores clicarem ao mesmo tempo, o segundo recebe "já confirmado por Fulano" (seção 6.3).

**Operador responde na conversa**
1. O core grava a mensagem e publica `whatsapp.envio_solicitado`. O destaque vermelho some.
2. A cobrança envia pela Evolution e publica `conversa.mensagem_enviada` com a situação.

## 6. Dados

### 6.1 Regras gerais **[decisão dos líderes]**

- **Multiempresa:** toda tabela de negócio tem `locadora_id`, e toda consulta filtra por ela. No core, isso é um filtro global do EF Core, aplicado sem depender de cada desenvolvedor lembrar.
- **Concorrência:** controle otimista (cada linha tem uma versão; quem tenta salvar sobre uma versão antiga recebe erro em vez de sobrescrever). No Postgres, usa a coluna de sistema `xmin`, que o EF Core e o Hibernate suportam. Mais regras de unicidade no banco: CPF único por locadora, no máximo uma locação ativa por motorista e por carro (índice único parcial), um único aviso de e-mail por `Message-ID`.
- **Histórico de alterações:** uma tabela `historico_alteracoes` em cada serviço que tem dados (entidade, id, campo, valor anterior, valor novo, operador, data, motivo). No core é preenchida automaticamente ao salvar (o EF Core sabe quais campos mudaram). Na cobrança, cada correção grava o seu registro.
- **Dinheiro:** `numeric(12,2)` no banco, `decimal` no C#, `BigDecimal` no Java. Nunca ponto flutuante.
- **Datas:** gravadas em UTC. Regras de dia e de horário (segunda, 9h, rodízio) usam o fuso `America/Sao_Paulo`.
- **Versões do banco:** EF Core Migrations no core e Flyway na cobrança, aplicadas ao subir.

### 6.2 Banco `core`

`locadoras` (nome, CNPJ, endereço, semanas de caução, multa por atraso (`multa_atraso_percentual`, `numeric(5,2)`, padrão 0), caixa de e-mail: servidor, usuário, senha criptografada, remetente do banco) · usuários do Identity (com `locadora_id` e `eh_dono`) · `convites` (e-mail liberado pelo dono) · `categorias_preco` (nome, 3 preços) · `carros` · `motoristas` (com `criado_em` e `anonimizado_em`; os dados pessoais aceitam nulo, para a anonimização) · `locacoes` (plano, preço semanal e os 3 preços congelados, caução, `multa_atraso_percentual` congelada (`numeric(5,2)`, padrão 0), datas, km inicial e final, locação anterior, `motivo_fim`, `credito_usado`) · `arquivos` (a que pertence, tipo, nome original, caminho, tamanho, SHA-256, excluído em/por) · `conversas` (motorista ou telefone não cadastrado, precisa de resposta; o telefone aceita nulo) · `mensagens` (autor: motorista, bot ou operador; operador; texto ou arquivo; id do WhatsApp; situação) · `historico_alteracoes`. **[decisão dos líderes]** Conversas e mensagens não têm histórico de alterações (a mensagem já é o registro); o vínculo de número a um motorista fica no histórico do motorista.

### 6.3 Banco `cobranca`

Cópias: `motoristas_ref`, `carros_ref`, `locacoes_ref` (com `contrato_id`, `motivo_fim`, o acerto e o percentual da multa por atraso) · `cobrancas` (assinatura, semana, multa de km, multa por atraso ou saldo devedor; valor, valor pago, vencimento, situação, quem cancelou) · `pagamentos` (comprovante, valor, data, horário, pagador, situação, cobrança quitada, aviso de e-mail casado) · `quitacoes` (cobrança quitada, pagamento e origem: pagamento, crédito, caução ou baixa manual, esta com operador e motivo) · `avisos_pix_email` · `creditos` (por contrato) · `lancamentos_caucao` (entrada, desconto, devolução, crédito e saldo devedor do acerto; por contrato e, no acerto, ligados à cobrança) · `leituras_km` (carro, locação, km, origem: foto ou operador, arquivo, suspeita) · `leituras_pendentes` (foto de hodômetro esperando a leitura da IA) · `revisoes` · `alertas` (tipo, motorista, carro, detalhe, criado em, tratado por e em) · `estado_bot` (falhas seguidas de foto por motorista) · `instancias_whatsapp` · `envios` (fila de envio do WhatsApp; a linha some depois do envio) · `mensagens_recebidas` (id de cada mensagem recebida, para ignorar repetidas) · `pendencias_resposta` (telefone com resposta pendente) · `eventos_processados` (ids dos eventos já tratados, para ignorar repetidos) · `historico_alteracoes`.

`outbox` existe nos dois bancos (seção 4.1, regra 6).

### 6.5 Quem é dono de cada dado de fronteira **[decisão dos líderes]**

- **Pagamento da assinatura.** O operador confere esse pagamento fora do bot (spec, 6.2). Ao salvar a locação, o core pede o cálculo de novo à cobrança e envia em `locacao.iniciada` o que foi pago. A cobrança registra a caução como entrada em `lancamentos_caucao` e o proporcional como uma cobrança do tipo "assinatura", já paga e confirmada pelo operador que criou a locação. O proporcional entra no recebido do painel; a caução só quando é usada para pagar uma cobrança. Na troca, a cobrança aplica as regras da spec (6.6): a caução passa adiante, e a diferença e o crédito dos dias não usados entram no cálculo da assinatura da nova locação.
- **Km do carro.** O core manda o km atual e o km da última revisão só ao criar o carro. Depois disso, a dona desses valores é a cobrança: o km muda por foto ou por correção de leitura, e a revisão muda por "Registrar revisão". O formulário de edição do carro não mostra esses dois campos.
- **Leituras de km vindas de locação.** Ao receber `locacao.iniciada` e `locacao.encerrada`, a cobrança cria as leituras de km com origem "operador", a partir do km inicial e do km final. Na troca só de plano, a leitura do dia é o km final da locação antiga e também o km inicial da nova.
- **Número não cadastrado.** O bot responde com o texto padrão da seção 6.5 da spec e cria o alerta "mensagem para responder". Uma foto de número não cadastrado não vai para a IA, porque não há locação a que ela pertença. Vincular o número a um motorista (spec, 7.1) passa a usar esse telefone como o WhatsApp do motorista, com registro no histórico. Isso gera `motorista.salvo`, e a cobrança passa a reconhecer o número.
- **Anonimização de 5 anos (spec, 6.8; constituição, item 12).** O core apaga do motorista nome, CPF, RG, CNH, telefone, endereço e e-mail (a linha fica como "motorista removido"), apaga o texto das mensagens dele e todos os arquivos ligados a ele e às locações dele, e limpa o motivo e os valores anterior e novo do histórico de alterações dele. A cobrança, ao receber `motorista.anonimizado`, apaga nome e telefone de `motoristas_ref` e o nome do pagador em `pagamentos` e `avisos_pix_email`, os textos livres de operador ligados a ele (motivos e descrições de cobranças, quitações, lançamentos de caução, leituras e pagamentos, e o detalhe dos alertas), as linhas de controle do bot do telefone dele (pendências de resposta, estado do bot, envios e mensagens recebidas) e, no histórico de alterações, o motivo e os valores de campos de texto. Valores, datas e km ficam, sem ligação com uma pessoa identificável, para os números do painel.
- **Cópia ainda desatualizada.** Se o core pedir um cálculo de uma locação que a cobrança ainda não recebeu (por exemplo, uma troca logo após criar a locação), a cobrança responde "tente de novo em instantes" (HTTP 409) em vez de calcular com dado errado.

### 6.6 Arquivos **[decisão dos líderes]**

- Gravados num volume do Docker montado só no core, com nome gerado (UUID), nunca o nome enviado pelo usuário.
- SHA-256 calculado no upload. Ao abrir, o core recalcula e o front mostra "íntegro" ou "alterado".
- Download e visualização só por rota autenticada do core, que confere a locadora.
- Tipos: PDF, JPEG, PNG, WEBP, MP4. Limites: 10 MB para imagem, 20 MB para PDF, 300 MB para vídeo. São validados no navegador, no nginx (tamanho) e no core (tipo pelo conteúdo do arquivo e tamanho).
- Os documentos de motorista e de locação são enviados logo depois de salvar o cadastro, ligados a ele (não antes, soltos), para nenhum ficar órfão (sem dono, fora do alcance da limpeza de 5 anos). Consequência: o banco não impede uma locação sem documento; quem garante é o formulário. **[decisão dos líderes]**
- Excluir marca o arquivo como excluído e registra no histórico. O conteúdo some do disco na hora.

## 7. Segurança e segredos

- Senhas com o hash do Identity. 2FA por app autenticador (TOTP: o código de 6 dígitos que muda a cada 30 segundos). Bloqueio temporário após tentativas erradas e limite de requisições no login.
- Sessão em cookie `HttpOnly`, `Secure`, `SameSite=Strict`. Como front e API estão no mesmo endereço, não precisa de CORS nem de token no JavaScript. **[decisão dos líderes]**
- Excluir um operador invalida as sessões dele na hora (o Identity troca o *security stamp*, um carimbo que as sessões antigas deixam de reconhecer).
- Erros do uso normal (senha ou código errado, sem sessão, e-mail já usado, conflito entre dois operadores) respondem 200 com `{ok: false, motivo, mensagem}`; os que só acontecem burlando o front mantêm o status HTTP (400, 401, 403, 404, 429). Motivo: o Chrome escreve no console toda resposta 4xx (constituição, item 2). **[decisão do dono]**
- Validação dupla: Zod no front, validação do ASP.NET no core, Bean Validation na cobrança.
- `.env` fora do Git e `.env.example` versionado só com nomes e valores falsos: senhas do Postgres e do RabbitMQ, token interno, chave do Gemini, chave da Evolution API, chave do Data Protection (`DATA_PROTECTION_CHAVE`, gerada com `openssl rand -hex 32`): ela cifra as chaves do Data Protection guardadas no volume `chaves`.
- A senha da caixa de e-mail do banco é digitada pelo dono nas Configurações e gravada criptografada no banco. Nunca vai para o `.env` nem para o Git.

### 7.1 Proteção de dados pessoais (LGPD) **[decisão do dono: proteger contra interceptação e vazamento]**

- **Em trânsito:** criptografado em todo lugar, do navegador até o banco (seção 3.6).
- **Banco:** nenhuma porta do Postgres, do RabbitMQ ou da Evolution API é publicada na máquina, então só os containers da rede interna alcançam esses serviços. Cada serviço tem usuário e senha próprios, vindos do `.env`, e o `.env.example` não traz senha utilizável. Nenhum serviço usa o usuário administrador do Postgres.
- **Logs:** nunca entram nos logs nome, CPF, RG, CNH, telefone, endereço, e-mail, texto de mensagem, conteúdo de imagem, resposta do Gemini, senha, token ou chave. Os logs registram só ids (de motorista, locação, arquivo, evento) e o `correlacao` do envelope. O registro de dados sensíveis do EF Core (`EnableSensitiveDataLogging`) e o log de SQL com valores do Hibernate ficam desligados. A Evolution API roda com log mínimo, sem o conteúdo das mensagens. O nginx não grava log de acesso (`access_log off`), porque a URL leva CPF e texto de busca **[decisão dos líderes]**.
- **Erros:** a resposta de erro que vai ao navegador nunca traz detalhe interno (stack trace, SQL).
- **Acesso:** dados pessoais só com login e só da própria locadora (seção 6.1). Os arquivos só saem pela rota autenticada do core (seção 6.6).
- **Apagamento:** limpeza automática de 5 anos (seção 6.5) e exclusão imediata da locadora (spec, seção 5).
- **Fica fora:** criptografar as colunas do banco (CPF, RG etc.) ou o disco. A busca por CPF ficaria bem mais complexa, e a spec não pede.

## 8. Agendamentos **[decisão dos líderes]**

- **Cobrança** (`@Scheduled` do Spring, fuso `America/Sao_Paulo`): segunda 0h abre as cobranças da semana; 9h primeiro lembrete; 12h segundo lembrete; 15h alertas e cobranças em atraso e, depois dos alertas, verifica os bloqueios; todo dia 0h05 conta um dia nas multas por atraso; todo dia 3h limpa dados de números não cadastrados e avisos de e-mail sem pagamento com 5 anos; todo dia 8h confere revisão próxima e vencida e faz o acerto da caução dos contratos devolvidos há 30 dias.
- **Core** (serviço em segundo plano do .NET; a limpeza roda ao subir e a cada 24 horas): apaga dados pessoais de motoristas 5 anos após a última locação, apaga os arquivos e publica `motorista.anonimizado`.
- **Leitura:** laço com espera de 2 minutos para o e-mail.
- Cada serviço roda com uma única instância, então nenhum agendamento dispara duas vezes.

## 9. Módulos × serviços

| Módulo | Onde |
|---|---|
| Tempo real | SignalR no core + eventos da fila; o front reconecta sozinho e busca os dados de novo ao reconectar |
| Organizações | core |
| Framework front e back | React + ASP.NET Core (e Spring Boot) |
| Módulo próprio | cobrança |
| Microsserviços | core, cobrança, leitura, Evolution API, conversando por RabbitMQ |
| Analytics | números na cobrança, gráficos e exportação CSV/PDF no front |
| ORM | EF Core (e JPA/Hibernate) |
| Arquivos | core + front |
| Busca avançada | core (motoristas e carros) e cobrança (alertas) |
| Imagem | leitura |
| 2FA | core |
| Exportação e importação | core |

## 10. Testes **[decisão dos líderes]**

- **Obrigatório (constituição, item 13):** teste unitário de todo cálculo de dinheiro na cobrança (JUnit), usando os exemplos da spec como casos: R$900 da assinatura, multa de R$100 com 1310 km, 4 × R$100 de dias não usados, e as bordas (1250, 1251, 1500, 1501 km, rodízio no dia da retirada, troca no meio da semana).
- Regras de casamento do Pix (folga de 5 minutos, mais de um e-mail, nenhum) e de leitura suspeita: teste unitário na cobrança.
- Core: testes das regras de acesso (uma locadora não vê dados da outra) e do histórico.
- Leitura: teste da validação da resposta do Gemini com respostas gravadas (sem chamar a IA de verdade).

## 11. Repositório

```
frontend/                React
services/core/           C#
services/cobranca/       Java
services/leitura/        Python
infra/nginx/             configuração
infra/certs/             gerador da autoridade certificadora e dos certificados
infra/postgres/          script que cria os bancos
infra/rabbitmq/          configuração e definições (exchanges, filas, política)
infra/testes/            testes de fumaça da infra
infra/demo/              dados de demonstração (entra no M7)
docker-compose.yml
.env.example
README.md
```

## 12. Time **[decisão dos líderes]**

Cada cadete é dono de um serviço: escreve, revisa os PRs dos outros nele e responde por ele na avaliação. A divisão fina e o equilíbrio de carga (o core é o maior serviço) ficam para o documento de tarefas.

| Cadete | Papel | Dono de |
|---|---|---|
| 1 | Tech Lead | Infra (compose, nginx, Postgres, RabbitMQ, Evolution API) e o contrato de eventos (seção 4) |
| 2 | Product Owner + dev | Front |
| 3 | Project Manager + dev | Core |
| 4 | Dev | Cobrança (módulo próprio) |
| 5 | Dev | Leitura (Python) |

## 13. Riscos

- **Gemini gratuito:** o plano gratuito tem limite de requisições por minuto e por dia, e o Google pode usar os dados enviados para melhorar os modelos. Comprovantes trazem nome do pagador, que é dado pessoal. A Política de Privacidade precisa dizer isso.
- **Senha de app do Gmail:** ainda funciona em 2026, mas o Google trata como recurso legado. Se sumir, a alternativa é OAuth, bem mais trabalhosa.
- **WhatsApp não oficial:** a Evolution API depende de engenharia reversa do WhatsApp e pode parar de funcionar com uma atualização dele. O risco de bloqueio do número é baixo com o volume da demonstração.
- **Formato do aviso do banco:** desconhecido até o Pix de teste (spec, 6.3). A extração por IA reduz o risco, mas o critério de casamento só fecha depois do teste.
- **Certificado autoassinado:** o Chrome mostra uma tela de aviso antes de entrar. Ela não aparece no console, então não fere a constituição (item 2), mas precisa ser explicada na avaliação.
- **Peso na máquina da escola:** são 7 containers (nginx, core, cobrança, leitura, Evolution API, Postgres, RabbitMQ). É preciso testar a subida do zero numa máquina da escola cedo, não no fim.
- **Rede da escola:** ninguém testou ainda se WhatsApp, IMAP (porta 993) e Gemini funcionam a partir da rede da escola. Se algum estiver bloqueado, o plano B é usar o celular como roteador na demonstração.

## 14. Ajustes feitos na spec junto com este plano

1. Seção 10, "Demonstração": a automação de WhatsApp rodou **na empresa**, não na escola, e falta testar a rede da escola.
2. Seção 7.6, "E-mail do banco": incluído o **remetente** dos avisos, usado para filtrar a caixa.
3. Seção 6.8: envio de fotos e e-mails ao Gemini gratuito, informado na Política de Privacidade **[decisão do dono: plano gratuito]**, e dados pessoais criptografados em trânsito e fora dos logs **[decisão do dono]**.
4. Seção 10: "IA de imagem" virou "IA", porque ela lê também os e-mails.
