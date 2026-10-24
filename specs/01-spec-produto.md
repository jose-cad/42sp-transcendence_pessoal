# Spec do produto: gestão de frota para locadoras de carros de aplicativo

Status: rascunho para revisão · 2026-10-24
Obedece a [`00-constituicao.md`](00-constituicao.md). Esta spec diz **o quê** e **por quê**. Tecnologia, arquitetura e divisão de tarefas ficam para o plano técnico e para as tarefas.

Itens marcados **[decisão dos líderes]** foram decididos pelos líderes técnicos e não vieram direto do dono do produto.

---

## 1. Intenção

Locadoras pequenas e médias alugam carros por semana para motoristas de Uber e 99. A rotina semanal é toda manual: cada segunda, cada motorista manda pelo WhatsApp o comprovante do Pix e a foto do hodômetro, e alguém da locadora confere valor por valor no extrato, calcula os km rodados, cobra quem não pagou, calcula multa de quilometragem e controla quando cada carro precisa de revisão. Os documentos (contrato, vistoria, vídeo de perícia) ficam espalhados em pastas.

O produto automatiza essa rotina. O motorista continua usando só o WhatsApp. Um bot recebe as fotos, uma IA lê o km e os dados do Pix, e o sistema calcula km, multa e revisão e manda lembretes. O sistema também procura o aviso daquele Pix no e-mail do banco e mostra se achou. O operador só precisa confirmar cada Pix com um clique, a partir de um alerta pronto, e tratar o que foge do normal. Ele tem num painel único a conversa, os dados do motorista e do carro e todos os documentos.

**Contexto do projeto:** é o ft_transcendence da 42 São Paulo, refeito do zero com SDD. É um projeto de grupo (4 a 5 pessoas), feito por uma pessoa simulando um time de 5. Precisa seguir o subject v19 à risca e somar no mínimo 14 pontos de módulos.

## 2. Objetivos

1. Na segunda-feira, o operador não abre nenhuma foto nem calcula nada. Ele confirma cada Pix no extrato com um clique, a partir de um alerta que já traz nome, valor e horário, e trata as exceções: não pagou, km excedido, foto ilegível, mensagem que não é foto.
2. Qualquer documento de qualquer motorista ou carro é encontrado em dois cliques, a partir da busca, sem navegar em pastas.
3. Todo valor em dinheiro (proporcional da assinatura, multa de km, devolução) é calculado pelo sistema de forma igual para todos, e toda alteração fica rastreada.
4. Nenhum carro passa da troca de óleo (10 mil km) sem que operador e motorista tenham sido avisados.

## 3. Não objetivos

- **Avaliação do motorista** (documentos, endereço, garagem, perfil na Uber/99): acontece fora do sistema. O sistema só começa quando o motorista já foi aprovado.
- **Login de motorista:** o motorista nunca acessa o site. O canal dele é só o WhatsApp.
- **Contrato digital:** o contrato é assinado em papel e o sistema guarda a cópia escaneada. O sistema não gera nem assina contratos.
- **Receber ou enviar dinheiro, ou acessar banco:** o sistema não faz Pix, não cobra cartão e não se conecta a banco. Ele lê comprovantes e os avisos de Pix no e-mail, calcula valores e pede a confirmação do operador. Devoluções são feitas por fora, pelo operador.
- **Cobrança após o alerta das 15h:** ligar, cobrar e bloquear o carro acontecem fora do sistema.
- **Multas de trânsito como fluxo próprio:** entram apenas como desconto na devolução da caução.
- **WhatsApp oficial (Meta):** usa uma automação própria num container, não a API oficial.
- **Níveis de acesso:** só existem operador e dono (seção 5).
- **Idiomas:** interface só em português do Brasil.

## 4. Anti-objetivos (fracasso mesmo que funcione)

- O operador ter que olhar foto de rotina para conferir km ou valor.
- Alguém alterar valor, contrato ou cadastro sem deixar rastro.
- O histórico mentir: uma edição apagar com qual plano ou preço uma semana passada foi cobrada.
- Um documento original se perder ou ser alterado sem que dê para perceber.
- Qualquer parte que o dono do projeto não consiga explicar na avaliação.

## 5. Quem usa

| Quem | Como acessa | O que faz |
|---|---|---|
| **Operador** | Site, com login (e-mail + senha + 2FA) | Tudo: cadastros, locações, conversas, alertas, preços, configurações, exceto o que é só do dono (linha abaixo). |
| **Dono** | Site, com login | É um operador com poderes a mais: incluir e excluir operadores, editar e excluir a locadora, conectar o WhatsApp, configurar o e-mail do banco, a caução e a multa por atraso, exportar e importar dados. Quem cria a conta da locadora vira o dono. |
| **Motorista** | Só WhatsApp | Recebe lembretes, manda fotos do hodômetro e comprovantes de Pix, conversa com a locadora. |

**Cadastro de conta:**
- Uma pessoa se cadastra no site (nome, nome da locadora, e-mail, senha), cria a locadora e vira dona dela. (Decisão do dono.)
- O dono cadastra o e-mail de um novo operador. A pessoa se cadastra com esse e-mail, define a própria senha e entra já vinculada à locadora. **[decisão dos líderes]**
- O dono pode excluir operadores, e o acesso deles é cortado na hora. O dono não pode excluir a si mesmo, e não existe transferência de posse. **[decisão dos líderes]**
- O dono pode editar e excluir a locadora (exigência do módulo de organizações). A exclusão pede confirmação dupla e apaga na hora todos os dados daquela locadora, porque é a locadora deixando de ser cliente. A Política de Privacidade descreve isso. **[decisão dos líderes]**
- O 2FA é opcional: cada operador ativa o seu em "Minha conta". **[decisão dos líderes]**
- Cada operador pertence a uma única locadora. **[decisão dos líderes]**
- Cada locadora só vê os próprios dados.

## 6. Glossário e regras de negócio

### 6.1 Termos

- **Locadora:** a empresa cliente do sistema (multi-empresa).
- **Categoria de preço:** grupo criado pela locadora (ex.: Básico, Médio, Luxo) com três preços semanais, um para cada plano. Cada carro pertence a uma categoria. O nome é livre. Os três preços precisam crescer (1250 < 1500 < livre); senão a multa de km sairia zero ou negativa. **[decisão dos líderes]**
  - Exemplo: Básico (Fiat Mobi) 600 / 700 / 800; Médio (Fiat Cronos) 700 / 800 / 900; Luxo (VW Virtus) 800 / 900 / 1000. Valores ilustrativos.
- **Plano:** franquia semanal de km. São três: **1250 km**, **1500 km** e **livre**.
- **Locação (contrato):** um motorista + um carro + um plano, com preço semanal, caução, data de retirada e, quando encerra, data de devolução. Um motorista tem no máximo uma locação ativa. Um carro está em no máximo uma locação ativa.
- **Semana:** segunda a domingo.
- **Rodízio:** dia da semana em que o carro não pode circular em São Paulo, definido pelo final da placa: segunda 1 e 2, terça 3 e 4, quarta 5 e 6, quinta 7 e 8, sexta 9 e 0. O sistema calcula pela placa. **[decisão dos líderes: cálculo automático]**
- **Valor do dia:** preço semanal ÷ 6. O valor semanal paga os 6 dias que não são rodízio. O motorista pode rodar no dia do rodízio, mas os km contam para a franquia. Os valores proporcionais são calculados como preço semanal × dias ÷ 6 e arredondados só no fim, para centavos (metade para cima). **[decisão dos líderes]**

### 6.2 Assinatura (início da locação)

- O operador registra a locação no dia da retirada.
- Na assinatura o motorista paga:
  - **caução:** preço semanal × N semanas, com N definido pela locadora (1; 1,5; 2...) **[decisão dos líderes: N configurável por locadora]**;
  - **dias proporcionais:** valor do dia × dias da semana corrente **depois** do dia da retirada até domingo, **sem contar o rodízio** do carro. O dia da retirada não é cobrado.
- Exemplo: Básico, plano 1250 (R$600, dia = R$100), retirada na quarta, rodízio na sexta. Cobra quinta, sábado e domingo = R$300, mais caução de 1 semana = R$600, total R$900.
- O sistema mostra esse valor calculado no formulário. A conferência desse pagamento é feita pelo operador, fora do bot. **[decisão dos líderes]**
- O preço semanal e os três preços da categoria ficam congelados na locação. Mudar a tabela depois não afeta locações em andamento. **[decisão dos líderes]**
- Na retirada, o operador registra o **km inicial** do carro. **[decisão dos líderes: necessário para calcular a primeira semana]**

### 6.3 Rotina de segunda-feira

A partir da primeira segunda depois da retirada, toda segunda o motorista manda pelo WhatsApp:
1. o **comprovante do Pix** da semana que começa, no valor do preço semanal (pagamento adiantado);
2. a **foto do hodômetro**.

Horários, iguais para todas as locadoras:
- **9h:** lembrete automático para todos os motoristas com locação ativa.
- **12h:** segundo lembrete, só para quem ainda tem pendência: semana não paga, foto do hodômetro não enviada ou multa de km não paga.
- **15h:** alerta para o operador, um por pendência. Daí em diante o tratamento é fora do sistema e o operador marca o alerta como tratado.

**Conferência do Pix** (decisão do dono: o sistema nunca acessa banco; ele lê o e-mail só para sugerir, e quem confirma é sempre o operador)
1. O comprovante pode vir em foto ou PDF. A IA lê nele o valor, a data, o horário e, se houver, o nome do pagador. (Decisão do dono.)
2. O pagamento fica **a conferir**, e o operador recebe um alerta "Conferir Pix: João · R$600 · 10h", com o comprovante a um clique.
3. **Casamento com o e-mail do banco:** o sistema lê a caixa de e-mail da locadora (ex.: Gmail), onde chegam os avisos de Pix recebido, e procura um aviso com o mesmo valor, na mesma data e com horário próximo (folga de até 5 minutos **[decisão dos líderes]**), que ainda não tenha sido casado com outro comprovante. O resultado aparece no próprio alerta:
   - "✓ e-mail do banco encontrado (R$600 · 10:00)", quando acha exatamente um;
   - "⚠ mais de um e-mail possível", quando acha mais de um;
   - "✗ nenhum e-mail encontrado", quando não acha nenhum. Se o e-mail chegar depois, o alerta é atualizado.
4. O operador **confirma** ou **recusa** com um clique. Quando o e-mail bate, basta conferir o alerta. Quando não bate, ele olha o extrato. Isso fica no histórico. O sistema **nunca confirma sozinho**.
5. **Em aberto:** fazer um Pix de teste para o e-mail da locadora e ver o formato do aviso do banco. Se ele trouxer o nome ou o CPF do pagador, isso entra no critério do passo 3.
6. Confirmado: o bot avisa o motorista. Recusado: o bot avisa o motorista que o pagamento não foi encontrado e a cobrança volta a ficar aberta.
7. Para os lembretes das 12h e o alerta das 15h, um comprovante enviado (mesmo ainda a conferir) conta como "pagou": o motorista fez a parte dele.
8. O sistema recusa sozinho, com aviso ao operador, um comprovante idêntico (mesmo valor, data e horário) a outro já enviado, o que impede reaproveitar o mesmo comprovante. **[decisão dos líderes]**

**A que cobrança um Pix pertence** **[decisão dos líderes]**
- O motorista tem **cobranças**: uma por semana (o preço semanal, aberta na segunda às 0h) e uma por multa de km (aberta quando a multa é calculada). Cada cobrança está **aberta**, **paga** ou **em atraso** (aberta depois das 15h do dia em que venceu).
- Pix são aceitos em qualquer dia. Um pagamento confirmado quita a cobrança aberta mais antiga cujo valor seja igual ao do Pix. Se não houver nenhuma, por exemplo um Pix adiantado no domingo (raro), ele fica como **crédito** e quita a próxima cobrança que abrir.
- Um Pix com valor que não bate com nenhuma cobrança vai para o operador decidir a qual cobrança ele pertence.
- Atribuído a uma cobrança maior, o Pix abate o que pagou e a cobrança continua aberta com o restante; maior que ela, a sobra vira crédito. (Decisão do dono.)
- Os lembretes das 12h e o alerta das 15h valem para cobranças abertas no dia. Cobranças em atraso aparecem no painel do motorista e nos números de inadimplência.

**Foto do hodômetro em outro dia** **[decisão dos líderes]**
- Fotos são aceitas em qualquer dia e sempre atualizam o km do carro (para revisão), salvo leitura suspeita (abaixo).
- Para o cálculo da semana vale a **primeira leitura recebida a partir de segunda às 0h**.
- Sem foto na segunda, o hodômetro segue os mesmos lembretes (9h, 12h) e o alerta das 15h do Pix; o operador cobra por fora e não lança km no sistema. Na 2ª segunda seguida sem km, bloqueio (ver "Multa por atraso e bloqueio"). Se a foto só chegar numa semana seguinte, o km medido é dividido pelo número de semanas e cada semana é comparada com a franquia. (Decisão do dono.)

**Leitura suspeita** **[decisão dos líderes]**
- Se a leitura for menor que a anterior, ou se o km da semana passar de 5.000, o sistema não calcula multa e gera o alerta "leitura suspeita".
- O operador corrige a leitura (fica no histórico) e o sistema recalcula.
- Leitura suspeita não muda o km do carro. **[decisão dos líderes]**
- Se a IA indicar baixa confiança na leitura, a foto é tratada como ilegível.

**Km e multa:**
- **km da semana** = leitura de hoje − leitura anterior (ou o km inicial, na primeira semana).
- A franquia é **sempre cheia**, mesmo em semana quebrada (primeira semana, semana da devolução, semana de troca). Se o contrato diz 1250, são 1250 km, tenha rodado 4 dias ou 6. (Decisão do dono.)
- Na devolução e na troca, a última semana é calculada com o km final registrado pelo operador, que fecha a semana da devolução. Se a devolução for numa segunda antes da foto, ele vale como a leitura daquela segunda. Se houver multa, ela vira uma cobrança. Se não for paga até o fim dos 30 dias, entra como desconto da caução. **[decisão dos líderes]**
- Plano 1250: até 1250 não há multa. De 1251 a 1500, multa = preço do plano 1500 − preço do plano 1250. Acima de 1500, multa = preço livre − preço 1250.
- Plano 1500: acima de 1500, multa = preço livre − preço 1500.
- Plano livre: nunca há multa. A foto continua obrigatória para o controle de manutenção.
- Exemplo (Básico, plano 1250): rodou 1310 km, multa = 700 − 600 = R$100.
- Quando há multa, o bot informa o valor e o motorista paga com um **segundo Pix**, separado, conferido do mesmo jeito. A multa segue os mesmos lembretes das 12h e 15h.
- A multa vence no dia da leitura (no dia seguinte, se a leitura chegar depois das 15h). Multa cancelada pelo operador não é recriada por correção posterior. **[decisão dos líderes]**
- Km excedido gera um alerta informativo para o operador.

**Foto ilegível:** se a IA não conseguir ler a foto ou não conseguir identificar se é hodômetro ou comprovante, o bot pede outra. Na segunda falha seguida, gera alerta "foto ilegível" para o operador. Falha de sistema da IA gera o alerta "foto ilegível" com o detalhe "leitura automática indisponível", sem contar como falha do motorista. Foto de motorista sem locação ativa vai para o operador (mensagem para responder). **[decisão dos líderes]**

**Multa por atraso e bloqueio**
- A locadora configura um percentual do preço semanal por dia (0 = sem multa). O padrão é 0 e o percentual fica congelado na locação, como os preços: a multa precisa estar no contrato de cada locadora. **[decisão dos líderes]**
- Vale para semana e multa de km não pagas. Conta um dia a cada 0h, a partir do dia seguinte ao vencimento, numa cobrança "multa por atraso" que cresce a cada dia enquanto a cobrança de origem estiver sem pagar. Comprovante enviado conta como pago, como nos lembretes. Multa por atraso não gera multa sobre ela. (Decisão do dono.)
- O que o motorista já pagou da multa não é cobrado de novo. Exemplo: a semana de R$ 600 venceu na segunda, com multa de 2% ao dia (R$ 12 por dia). Na quinta a multa está em R$ 36 e o João paga os R$ 36. Na sexta a multa vale R$ 48 e falta pagar R$ 12.
- Quando a cobrança de origem é paga, a multa para de crescer. Se ainda houver multa em aberto, o operador recebe uma vez o alerta "multa não paga".
- Multa cancelada pelo operador não volta a crescer nem é recriada.
- A multa por atraso não gera lembrete das 12h nem alerta das 15h: quem avisa é a cobrança de origem e, na segunda seguinte, o bloqueio.
- **Bloqueio do veículo**, sempre na segunda às 15h: Pix com 7 dias ou mais de atraso, ou a 2ª segunda seguida sem km. O operador recebe o alerta "bloquear veículo" e o bot avisa o motorista de que o veículo poderá ser bloqueado. O sistema não bloqueia o carro; o operador bloqueia por fora. (Decisão do dono.)

### 6.3.1 Correções pelo operador **[decisão dos líderes]**

- **Podem ser corrigidos**, sempre com registro no histórico (quem, quando, antes, depois e motivo): leitura de km, confirmação ou recusa de pagamento, cancelamento de multa e dados cadastrais de motorista e carro.
- **Não podem ser editados:** carro, plano e preço de uma locação. Para mudar, faz-se uma troca (6.6).
- Toda correção de km recalcula a multa daquela semana.
- O operador pode dar baixa manual, total ou parcial, em qualquer cobrança (Pix em partes, de outra conta, dinheiro), com motivo obrigatório no histórico. O sistema continua gerando os alertas. (Decisão do dono.)

### 6.4 Revisão

- O carro guarda o km da última revisão.
- Ao chegar a **9.000 km** desde a última revisão: alerta "revisão próxima" para o operador e mensagem para o motorista.
- Ao chegar a **10.000 km** (troca de óleo) sem revisão registrada: alerta "revisão vencida". **[decisão dos líderes]**
- O operador registra a revisão feita (data e km), e a contagem recomeça.

### 6.5 Mensagens que não são foto

Qualquer mensagem do motorista que não seja foto de hodômetro ou de comprovante (dúvida, pedido, "quebrei na marginal"):
- o bot responde que a locadora vai retornar assim que possível **[decisão dos líderes: texto da resposta]**;
- o nome do motorista ganha um destaque vermelho na lista de conversas;
- gera um alerta "mensagem para responder".

O destaque some quando um operador responde. **[decisão dos líderes]**

Enquanto o destaque estiver aceso, novas mensagens do mesmo motorista não geram outra resposta nem outro alerta. A resposta de um operador reabre o ciclo. (Decisão do dono.)

### 6.6 Troca de carro ou de plano

- Toda troca **encerra a locação atual e abre uma nova** para o mesmo motorista. Não se edita locação existente.
- A caução passa para a nova locação sem devolução. Se a nova exigir caução maior, o motorista paga a diferença na assinatura da nova. Se exigir menor, a sobra é devolvida junto com a caução no encerramento final. **[decisão dos líderes]**
- O dia da troca é cobrado uma vez só, na locação nova (não fica dia grátis nem cobrado em dobro). **[decisão dos líderes]**
- Os dias não usados da semana já paga na locação antiga viram **crédito abatido** no valor de assinatura da nova. **[decisão dos líderes]**
- **Troca de carro:** exige km final, perícia e vídeo do carro devolvido, e km inicial, vistoria e vídeo do carro novo. **Troca só de plano:** exige apenas a leitura de km do dia. **[decisão dos líderes]**
- A caução e o crédito são do contrato (a locação e as trocas que vieram dela). Um contrato novo começa do zero; usar a caução antiga no contrato novo é combinado pelo operador, fora do sistema. Descontos (reparo, multa de trânsito) podem ser lançados nela a qualquer momento. A retenção de 30 dias só acontece no encerramento final. **[decisão dos líderes]**
- Na troca, todos os preços passam a ser os da tabela atual da categoria (semana e multa de km). A caução continua a mesma se a categoria não mudou; se mudou, vale a caução da categoria nova, com a diferença paga na troca ou a sobra devolvida no encerramento final. (Decisão do dono.)
- Na troca com mudança de categoria vale a configuração de hoje da locadora: caução nova = preço novo × semanas de caução atuais, e a multa por atraso atual. Exemplo: o João alugou um Básico (R$ 600) com caução de 1 semana; a locadora passou a pedir 2 semanas; na troca para o Médio (R$ 700), a caução exigida é 2 × 700 = R$ 1.400 e ele paga R$ 800 de diferença. (Decisão do dono.)
- As locações anteriores ficam no histórico do motorista.

### 6.7 Encerramento

- Devolução só de manhã: devolução à tarde é agendada para o dia seguinte. O controle de horário é do operador. O sistema registra a **data efetiva** da devolução.
- O operador registra o encerramento com o km final e os uploads de perícia e vídeo de encerramento. O encerramento e a troca são registrados no dia da devolução, depois da perícia e do vídeo; a data é a do registro. (Decisão do dono.)
- O motorista sai da rotina: não recebe mais lembretes de pagamento nem de revisão, nem alertas das 15h, multa por atraso ou bloqueio. Na troca ele continua na rotina, e as cobranças em aberto da locação antiga continuam a ser cobradas. Foto do hodômetro que chega depois do encerramento não vira leitura; comprovante continua valendo. **[decisão dos líderes]**
- **Dias não usados:** valor do dia × dias da semana corrente **a partir do dia da devolução (inclusive)** até domingo, **sem contar o rodízio**. O dia da devolução não é cobrado. O motorista deve só os dias usados da semana da devolução. Se a semana já foi paga, os dias não usados entram no saldo devolvido com a caução (exemplo: 4 × R$100); se não foi, a cobrança da semana cai para os dias usados. Devolução na segunda: nada daquela semana é cobrado. Vale igual na troca (6.6). (Decisão do dono.)
  - Exemplo: rodízio na sexta, pagou a semana na segunda, devolveu na quarta. Volta quarta, quinta, sábado e domingo = 4 × R$100.
- **Caução:** fica retida por **30 dias**, por causa das multas de trânsito que chegam cerca de 20 dias depois. Nesse período o operador lança descontos (reparo ou multa de trânsito, com valor e descrição).
- **Valor a devolver** = caução + dias não usados − descontos (reparos, multas de trânsito, multa de franquia não paga).
- No 30º dia (30 dias corridos depois da devolução), o contrato é acertado. Cobranças em aberto do contrato (semana, multa de km, multa por atraso) entram como desconto; o crédito do contrato soma. Saldo zero não gera alerta. **[decisão dos líderes]**
  - se o saldo for positivo, o operador recebe o alerta "devolver caução", faz o Pix por fora e marca como tratado;
  - se for negativo, o sistema gera uma **cobrança do saldo devedor** e um alerta "cobrar saldo devedor" para o operador. A cobrança em si acontece por fora, e o operador marca como tratado. (Decisão do dono.)

### 6.8 Retenção de dados

- Nada é apagado no encerramento. Conversa, documentos, vídeos e pagamentos continuam consultáveis pela busca.
- Os dados pessoais de um motorista são apagados automaticamente **5 anos após o encerramento da última locação** dele, **ou após o cadastro, se ele nunca alugou** (decisão do dono). Conversas de números não cadastrados são apagadas 5 anos após a última mensagem. Os números agregados do painel podem permanecer anonimizados. **[decisão dos líderes]**
- O prazo e o motivo aparecem na Política de Privacidade.
- As fotos e os e-mails de aviso de Pix são enviados a uma IA externa gratuita (Gemini), que, no plano gratuito, pode usar esse conteúdo para melhorar os próprios modelos. A Política de Privacidade informa isso, e a demonstração usa só dados de teste. (Decisão do dono: o projeto é acadêmico e não tem verba para o plano pago.)
- Dados pessoais trafegam criptografados em todo o sistema, inclusive entre as partes internas, e nunca são gravados em logs. (Decisão do dono.)

## 7. Telas

Desenho de referência: estilo VS Code, numa tela única depois do login.

```
┌───────────────┬──────────────────────────────────────────────┐
│ [+ Novo ▾]    │ [💬 Carlos ✕][📄 Contrato Carlos ✕][Painel ✕] │  ← abas da área de trabalho
│ [Painel]      ├──────────────────────────────────────────────┤
│ [Config.]     │                                              │
│ 🔍 busca      │   ÁREA DE TRABALHO                           │
│               │   (conversa, documento, cadastro, painel…)   │
│ ● Carlos      │                                              │
│   Ana         ├──────────────────────────────────────────────┤
│ ● Bruno       │ [Motorista e carro] [Alertas (5)]            │  ← painel inferior
│   ...         │  dados do motorista  │  dados do carro       │
└───────────────┴──────────────────────────────────────────────┘
  rodapé: Política de Privacidade · Termos de Uso
```

### 7.1 Coluna esquerda

- **+ Novo:** Nova locação, Novo carro, Nova categoria de preço.
- **Painel:** abre os números (seção 7.5).
- **Configurações:** abre a aba de configurações (seção 7.6).
- **Busca:** por nome, CPF, e-mail ou placa, com filtros, ordenação e paginação. Inclui motoristas com locação encerrada. A busca diferencia acentos ("Joao" não acha "João"). Clicar num resultado abre a ficha do motorista, com os documentos dele e de cada locação. **[decisão dos líderes]**
- **Lista de conversas:** um item por motorista, com prévia da última mensagem, ordenada pela mais recente, e destaque vermelho em quem tem mensagem para responder. Mensagens de números não cadastrados aparecem como "Não cadastrado", e o operador pode vinculá-las a um motorista. **[decisão dos líderes]**

### 7.2 Área de trabalho (centro, com abas)

Funciona como os arquivos abertos no VS Code: cada coisa aberta é uma aba que pode ser fechada.
- **Clicar num motorista** abre a aba da conversa dele, ou vai para ela se já estiver aberta.
- **Conversa:** mensagens do motorista, respostas do bot (diferenciadas visualmente) e respostas de operadores (com o nome de quem respondeu). Tem um campo para o operador responder, e a resposta chega ao motorista no WhatsApp. Atualiza em tempo real para todos os operadores. Áudios recebidos no WhatsApp são guardados e tocam na conversa. (Decisão do dono.)
- **Documento ou vídeo:** abre numa aba com visualização (PDF, imagem ou vídeo) e os botões **Baixar** e **Enviar ao motorista no WhatsApp**. Mostra se o arquivo está íntegro (impressão digital/hash confere).
- **Cadastros e encerramento:** formulários em aba (seção 8).
- **Histórico de alterações** de um motorista, carro ou locação: lista de quem mudou o quê, quando, antes e depois.
- **Ficha do motorista:** aberta pela busca, com os documentos dele e de cada locação.

### 7.3 Painel inferior: aba "Motorista e carro"

Mostra os dados de quem está na aba de conversa ativa.
- **Motorista:** dados cadastrais, plano, preço semanal, situação da semana (aberta, a conferir, paga, em atraso), multa pendente, km da última semana contra a franquia (ex.: "1.310 / 1.250"), caução. Links para: documentos escaneados, contrato, vistoria inicial, vídeo de perícia, pagamentos e multas, locações anteriores, histórico de alterações.
- **Carro:** placa, modelo, categoria, dia de rodízio, km atual, km para a próxima revisão. Links para: licenciamento (PDF do Detran), histórico de km, revisões, histórico de alterações.
- Botão de editar em cada lado, e ações "Encerrar locação", "Trocar carro/plano" e "Registrar revisão".
- Todo link abre na área de trabalho.

### 7.4 Painel inferior: aba "Alertas"

- Uma lista única de alertas pendentes com tipo, motorista, carro, detalhe e hora.
- Filtros por tipo, inclusive "tratados".
- Caixa ☐ para marcar como tratado (registra quem e quando).
- Clicar num alerta abre a conversa do motorista e muda o painel para "Motorista e carro". **[decisão dos líderes]**
- O contador na aba atualiza em tempo real.

Tipos de alerta: conferir Pix, semana não paga (15h), foto do hodômetro não enviada (15h), multa de km não paga (15h), km excedido, leitura suspeita, foto ilegível, mensagem para responder, revisão próxima, revisão vencida, devolver caução, cobrar saldo devedor, bloquear veículo, multa não paga (multa por atraso em aberto).

### 7.5 Painel de números (analytics)

Abre como aba. Todos os números com filtro de período e atualização em tempo real. **[decisão dos líderes: lista de indicadores]**
- Faturamento semanal: esperado × recebido (gráfico de barras por semana). Esperado é a soma das semanas e das assinaturas daquela semana, sem as canceladas; multas ficam fora, porque têm número próprio. Recebido é o que foi pago dessas cobranças, contado na semana da cobrança, mesmo quando pago depois. A caução não é faturamento: só conta quando é usada para pagar uma cobrança.
- Multas de km por semana (valor e quantidade).
- Inadimplência: semanas pagas em atraso ou não pagas. Cada semana mostra quantas cobranças foram pagas em dia (até as 15h do vencimento), pagas em atraso (depois disso) e não pagas (ainda abertas e já vencidas). Cobrança que ainda não venceu não conta.
- Km médio rodado por carro na semana (linha ao longo do tempo): média simples entre os carros que têm km naquela semana. Quando a foto chega numa semana seguinte, o km é dividido igualmente entre as semanas. A leitura suspeita fica de fora até o operador corrigir.
- Frota por situação: alugado, livre, perto de revisão. É a situação de agora, e não depende do período. Alugado é o carro com locação ativa; livre, os outros; perto de revisão, quem rodou 9.000 km ou mais desde a última revisão (inclui as vencidas). Perto de revisão é um número à parte, porque um carro alugado também pode estar perto da revisão.
- Exportação dos dados do painel em CSV e PDF. O PDF traz tabelas, sem os gráficos.

O painel é visto pelo dono e pelos operadores. (Decisão do dono.)

### 7.6 Configurações

- **Locadora:** nome, CNPJ e endereço; editar ou excluir a locadora (só o dono).
- **Equipe** (só o dono): incluir e excluir operadores.
- **Caução** (só o dono): número de semanas.
- **Multa por atraso** (só o dono): percentual do preço semanal por dia (padrão 0).
- **E-mail do banco** (só o dono): dados de acesso à caixa que recebe os avisos de Pix (só leitura) e o endereço do remetente desses avisos (o banco), usado para filtrar a caixa. **[decisão dos líderes: remetente]**
- **Categorias de preço:** criar e editar.
- **WhatsApp** (só o dono): conectar o número da locadora (cada locadora tem o seu, via QR code). **[decisão dos líderes]** Os operadores veem a situação da conexão. (Decisão do dono.)
- **Exportar e importar** (só o dono): exportar dados (motoristas, carros, locações, pagamentos) em CSV, JSON e XML; importar carros em lote por CSV, com validação linha a linha. **[decisão dos líderes: o que se importa]** Os "pagamentos" exportados são as cobranças com o que foi pago. A importação de carros usa colunas fixas: placa, modelo, ano, categoria, categorias da Uber, km atual e km da última revisão.
- **Minha conta:** dados pessoais, senha, ativar 2FA.

### 7.7 Páginas públicas

Login, cadastro, Política de Privacidade e Termos de Uso. Privacidade e Termos ficam acessíveis no rodapé de todas as telas, inclusive sem login. As páginas dizem que o Frota é um projeto acadêmico da 42 São Paulo e dão o contato `jose-cad@student.42sp.org.br`. (Decisão do dono.) Pedidos do motorista sobre os próprios dados vão para a locadora, que decide sobre esses dados (controladora, na LGPD); o Frota é o operador. **[decisão dos líderes: a divisão entre controladora e operadora]**

## 8. Cadastros

Todos são feitos pelo operador.

**Novo carro:** placa, modelo, ano, categoria de preço, categorias da Uber (X, Comfort, Black, só informativo), km atual, km da última revisão, upload do licenciamento (PDF).

**Nova locação:**
- **Motorista:** nome, CPF, RG, número da CNH, telefone (é o WhatsApp), endereço, e-mail opcional. Upload dos documentos escaneados.
  - Se o CPF já existe na locadora, reaproveita o cadastro. **[decisão dos líderes]**
- **Carro:** escolhido entre os carros livres. Mostra categoria e rodízio.
- **Plano** e **data de retirada** (de hoje até 6 dias atrás). **[decisão dos líderes]**
- **Km inicial.**
- **Uploads:** contrato assinado (escaneado), vistoria inicial, vídeo de perícia.
- **Valor calculado** para pagar na assinatura (seção 6.2).

**Encerramento:** km final, uploads de perícia e vídeo de encerramento. A data é a do registro, feito no dia da devolução, depois da perícia e do vídeo. Depois, durante os 30 dias, descontos da caução.

**Troca:** perícia e vídeo de encerramento do carro devolvido (na troca de carro); vistoria e vídeo do carro novo. Na troca só de plano não há nenhum documento. Perícia de encerramento e vídeo de encerramento são dois tipos de arquivo novos.

**Arquivos (regras gerais):**
- Tipos aceitos: PDF, imagens e vídeo, com limite de tamanho validado no navegador e no servidor (limites ficam com quem construir).
- Áudios recebidos no WhatsApp também são guardados (formato OGG, até 16 MB, só como arquivos vindos do WhatsApp) e tocam na conversa. Mídia grande demais ou de tipo não aceito não é guardada; a conversa avisa para ver no celular. (Decisão do dono.)
- Os documentos são enviados logo depois de salvar o cadastro, ligados a ele (não antes, soltos). Assim nenhum documento fica órfão (sem dono, fora do alcance da limpeza de 5 anos). Consequência: o sistema não impede salvar uma locação sem documento; quem garante é o formulário. **[decisão dos líderes]**
- Barra de progresso no upload.
- Visualização no próprio sistema.
- Cada arquivo guarda uma impressão digital (hash) para provar que não foi alterado.
- Só operadores da locadora acessam.
- O operador pode excluir um arquivo enviado por engano, e a exclusão fica no histórico.
- O operador nunca navega em pastas: tudo é achado a partir do motorista, do carro ou da locação.

## 9. Módulos do subject (meta: 16 pontos)

| Módulo | Tipo | Pts | Onde aparece no produto |
|---|---|---|---|
| Recursos em tempo real (WebSockets ou similar) | Major | 2 | Conversas, alertas e contador ao vivo para todos os operadores. Trata desconexão e reconexão. |
| Sistema de organizações | Major | 2 | Locadoras; dono inclui e exclui operadores; criar, editar e excluir locadora. |
| Framework no frontend e no backend | Major | 2 | Escolha no plano técnico. |
| Módulo próprio: automação de cobrança via WhatsApp | Major | 2 | Bot (o WhatsApp é só transporte; a lógica é nossa), lembretes agendados, casamento do comprovante com o e-mail do banco, cálculo de km, multa e revisão, alertas. Exige justificativa no README. Risco: é o módulo com mais chance de ser recusado. |
| Backend em microsserviços | Major | 2 | Cada responsabilidade num serviço e num container próprio, conversando por fila de mensagens. Detalhes no plano técnico. |
| Painel de analytics com visualização de dados | Major | 2 | Seção 7.5. |
| ORM | Minor | 1 | Escolha no plano técnico. |
| Upload e gestão de arquivos | Minor | 1 | Seção 8, "Arquivos". |
| Busca avançada (filtros, ordenação, paginação) | Minor | 1 | Busca e lista de alertas. |
| Reconhecimento e classificação de imagem | Minor | 1 | IA classifica a foto (hodômetro, comprovante, outra) e extrai km ou valor e data, por chamada a uma IA gratuita (ex.: Gemini). |
| 2FA | Minor | 1 | Login dos operadores com código de app autenticador. |
| Exportação e importação de dados | Minor | 1 | Seção 7.6. |
| **Total** | | **18** | Mínimo exigido: 14. A margem cobre módulos recusados na avaliação. Sem o módulo próprio, ficam 16. |

## 10. Restrições

- **Subject v19** e a constituição.
- **Restrições do dono para o plano técnico:** no mínimo **3 linguagens de programação**, **containers** e **mensageria** (fila de mensagens entre serviços). A escolha de quais é do plano técnico.
- **Time simulado de 5 pessoas:** as tarefas serão divididas como num time real da 42 (PO, PM, Tech Lead, desenvolvedores), com papéis no README. Na prática há um único autor de commits, o que não atende "commits de todos os membros" do subject. É uma limitação conhecida, aceita porque o projeto não será avaliado de fato.
- **WhatsApp:** automação não oficial, rodando num container. Usar um número de teste, porque números de automação podem ser bloqueados pelo WhatsApp.
- **IA:** serviço externo gratuito, usado para ler as fotos e os e-mails de aviso de Pix. A chave fica no `.env`.
- **Repositório público:** o projeto é acompanhado pelo GitHub (como um avaliador da 42 acompanharia). Nenhuma credencial de e-mail, de IA ou de WhatsApp vai para o repositório, nem em exemplo. O sistema nunca se conecta a banco.
- **Demonstração:** precisa funcionar numa máquina da escola subindo com um comando. Usa WhatsApp e IA reais: o dono já rodou automação de WhatsApp em container com sucesso, na empresa. Ainda falta testar se WhatsApp, e-mail e IA funcionam a partir da rede da escola. Na demonstração basta uma locadora com número conectado. Um script (`bash infra/demo/demo.sh`) cria uma locadora de demonstração com 12 semanas de histórico, só com dados fictícios. Os motoristas fictícios usam o DDD 20, que não existe, para o bot não alcançar ninguém. (Decisão do dono.)

## 11. Fica com quem construir

- Tecnologias, linguagens, divisão em serviços, formato das mensagens na fila, banco.
- Onde e como os arquivos ficam guardados (desde que fora do banco, com hash e acesso controlado).
- Limites de tamanho de arquivo, layout visual fino, cores, componentes.
- Como agendar os lembretes (9h, 12h, 15h, 30 dias) e a limpeza de 5 anos.
- Textos exatos do bot, respeitando o conteúdo descrito.
- Como o e-mail do banco é lido e interpretado (respeitando o item "Em aberto" da seção 6.3).
- Conteúdo jurídico das páginas de Privacidade e Termos, coerente com a seção 6.8.
