# Deliberate — Contexto do Backend

## 1. Visão geral

**Deliberate** é um aplicativo móvel para dividir despesas de viagens e da casa entre amigos ou moradores. O sistema registra quem pagou o quê, calcula automaticamente o saldo de cada pessoa e sugere o menor número possível de transferências para quitar tudo, sem burocracia.

O backend é a **fonte da verdade**: guarda os dados, executa todos os cálculos financeiros, sincroniza clientes (inclusive os que ficaram offline), registra o histórico auditável e dispara notificações.

### Stack atual

- .NET (ASP.NET Core Web API) — solução `Deliberate.slnx`
- Banco de dados: **PostgreSQL** com **EF Core** (migrations versionadas)
- Autenticação: e-mail + senha, com telefone como identificador alternativo (ver 3.0)
- **Moeda única por ora: BRL (R$).** Multimoeda/câmbio e OCR/IA ficam para uma fase futura.
- Projetos: `Deliberate.API` (camada HTTP) e `Deliberate.Domain` (regras de negócio)
- OpenAPI habilitado em desenvolvimento

### Princípios

1. **Dinheiro nunca usa `float`/`double`.** Usar `decimal` de alta precisão, com política de arredondamento explícita.
2. **Regras de negócio ficam no `Domain`**, sem depender de HTTP, banco ou serviços externos.
3. **Tudo é auditável.** Nenhuma alteração financeira acontece sem registro no feed de atividades.
4. **Cálculos são determinísticos e testáveis.** Mesma entrada gera sempre o mesmo resultado.
5. **Soma exata:** a soma das cotas de uma despesa deve ser exatamente igual ao total dela. Sobras de centavos são distribuídas por regra definida e determinística.

---

## 2. Glossário

| Termo                       | Significado                                                                                     |
| --------------------------- | ----------------------------------------------------------------------------------------------- |
| **Usuário**                 | Pessoa com conta no app.                                                                        |
| **Grupo**                   | Contexto de divisão (ex.: "Viagem para a Praia", "Casa"). Tem membros.                          |
| **Membro**                  | Vínculo de um usuário com um grupo.                                                             |
| **Despesa**                 | Gasto pago por um ou mais membros e dividido entre participantes.                               |
| **Pagador**                 | Membro que desembolsou o dinheiro da despesa.                                                   |
| **Participante**            | Membro que consome/deve parte da despesa.                                                       |
| **Cota (share)**            | Valor final que cada participante deve em uma despesa.                                          |
| **Saldo**                   | Total pago menos total devido por um membro no grupo. Positivo = a receber, negativo = a pagar. |
| **Transferência sugerida**  | Pagamento "de A para B" resultante da simplificação de dívidas.                                 |
| **Liquidação (settlement)** | Registro de que um pagamento real foi feito entre dois membros.                                 |

---

## 3. Domínio e funcionalidades

### 3.0 Usuários, autenticação e permissões

- Login por **e-mail e senha**; o **telefone** também pode ser usado como identificador de login (e-mail e telefone únicos no sistema).
- A **verificação de identidade é feita por e-mail** (confirmação de endereço). O telefone **não** é verificado por SMS por ora; é apenas um login alternativo.
- Senhas armazenadas apenas como hash seguro (ex.: ASP.NET Identity). Acesso via token (JWT + refresh token).
- **Todo membro de um grupo precisa ter conta.** Não existem membros "fantasma".
- **Papéis dentro de um grupo:**
  - **Dono (owner):** quem criou o grupo, mas **um grupo pode ter vários donos**. Donos podem **editar e excluir despesas** e **confirmar liquidações**, além de gerenciar membros, convites e configurações do grupo. Um dono pode promover outro membro a dono. Um grupo nunca pode ficar sem pelo menos um dono.
  - **Membro:** vê o painel e o feed do grupo e pode lançar novas despesas, mas não edita nem exclui despesas e não confirma liquidações.
- Toda checagem de permissão é feita no backend, nunca confiada ao cliente.

### 3.1 Grupos e contextos

- Criação de grupos temáticos com nome e, opcionalmente, descrição/período. Todos os valores são em BRL.
- Gestão de membros: convite de usuários **já cadastrados**, entrada, saída e remoção por um dono.
- **Saída voluntária:** um membro com saldo diferente de zero **não pode sair** do grupo; precisa quitar o saldo antes.
- **Remoção por um dono:** permitida mesmo com saldo diferente de zero. Nesse caso o saldo do removido é **distribuído igualmente entre os demais membros ativos** (ajuste de saldo auditável, com soma zero), e o removido fica com saldo 0. Não se pode remover o último dono nem o único membro restante.
- **Convites:** o dono convida um usuário cadastrado e o backend gera um convite com **código** e **link** únicos. O convidado é avisado por:
  - **e-mail** (com o link/código);
  - **notificação push** no app.

  O convidado entra no grupo aceitando o convite pela notificação, abrindo o link ou digitando o código no app. Convites têm status (pendente, aceito, recusado, expirado, revogado) e **prazo de validade padrão de 3 dias**; o código/link deve ser difícil de adivinhar e invalidado após o uso ou a revogação.

- **Painel do grupo** (endpoint de resumo) deve retornar:
  - gasto total do grupo;
  - gasto por pessoa (quanto pagou e quanto consumiu);
  - saldo de cada membro (quem está devendo e quem tem a receber);
  - transferências sugeridas para quitação.

### 3.2 Despesas e modelos de divisão

Uma despesa tem: descrição, valor, data, categoria (opcional), pagador(es), participantes, método de divisão, anexos opcionais.

| Método             | Regra                                                                                               |
| ------------------ | --------------------------------------------------------------------------------------------------- |
| **Igualitária**    | Total ÷ nº de participantes.                                                                        |
| **Porcentagem**    | Cada participante tem um %; a soma deve ser 100%.                                                   |
| **Cotas (partes)** | Cada participante tem um peso (ex.: família com filhos = 3). Valor = total × peso ÷ soma dos pesos. |
| **Valores exatos** | Cada participante tem um valor fixo; a soma deve ser igual ao total.                                |

Regras:

- Validar sempre a consistência (soma de % = 100, soma de valores = total, pesos > 0).
- Distribuição de centavos residuais: determinística (ex.: maiores restos primeiro, desempate por ordem estável de id).
- Despesa pode ter **múltiplos pagadores** (ex.: dois membros pagaram parte cada).

### 3.3 Assimetria de consumo ("Não bebi, não pago")

- Uma despesa pode ser composta de **itens** (ex.: nota fiscal com vários itens).
- Cada item tem valor próprio e seus próprios participantes e método de divisão.
- A cota final de um membro na despesa = soma das suas cotas em cada item.
- Valores comuns (taxa de serviço, couvert, gorjeta, desconto) são rateados proporcionalmente ao consumo de cada membro, ou conforme método escolhido.
- A soma dos itens (+ taxas − descontos) deve fechar com o total da despesa.

### 3.4 Motor de simplificação de dívidas (obrigatório)

Objetivo: **minimizar o número de transferências** necessárias para zerar todos os saldos.

- Entrada: saldos líquidos de cada membro.
- Saída: lista de transferências `(devedor → credor, valor)`.
- Exemplo: João deve 50 a Maria, Maria deve 50 a Pedro → sugerir apenas João → Pedro (50).
- Abordagem base: calcular saldo líquido, separar devedores e credores e casar valores (guloso por maiores valores). Mínimo absoluto é NP-difícil; a heurística gulosa gera no máximo `n − 1` transferências e é aceitável. Pode-se otimizar depois, por exemplo detectando subconjuntos que se zeram.
- Invariantes (devem ser cobertos por testes):
  - a soma dos saldos é 0 (tolerância zero em centavos);
  - após aplicar as transferências, todos os saldos são 0;
  - nenhuma transferência tem valor ≤ 0;
  - ninguém aparece como devedor e credor ao mesmo tempo.
- Liquidações confirmadas e ajustes de saldo (distribuição de dívida de membro removido) entram no cálculo do saldo antes de simplificar.

### 3.6 Despesas recorrentes

- Despesas modelo (aluguel, luz, internet) com periodicidade **semanal ou mensal** (dia da semana ou dia do mês de vencimento; em meses curtos, usar o último dia do mês), data de início/fim opcional e divisão padrão.
- Um job em segundo plano gera as despesas reais a cada ciclo, de forma **idempotente** (nunca duplicar o lançamento de um mesmo ciclo).
- Deve ser possível pausar, editar ou encerrar a recorrência. Mudanças afetam apenas ciclos futuros.
- Despesas geradas aparecem no feed com origem "recorrência automática".

### 3.7 Feed de atividades (auditoria)

- Log **imutável e cronológico** (append-only) de eventos do grupo: despesa criada/editada/excluída, membro entrou/saiu, liquidação registrada, recorrência criada/alterada, etc.
- Cada evento guarda: autor, data/hora (UTC), tipo, entidade afetada e **estado antes/depois** (diff) nas edições.
- Exclusões de despesas são **lógicas (soft delete)**, para manter o histórico verificável.
- Eventos nunca são editados ou removidos.

### 3.8 Pagamento via PIX

- Dada uma transferência sugerida, gerar o **payload PIX "Copia e Cola"** (padrão EMV/BR Code) e a imagem/dados de **QR Code** com valor já preenchido.
- Requer a chave PIX do recebedor (cadastrada no perfil do usuário).
- O PIX é um **facilitador**: o app não processa o pagamento. O pagador pode sinalizar que pagou, mas a **liquidação só passa a valer quando o dono do grupo a confirma**. Valor sempre em BRL.

### 3.9 Modo offline e sincronização

O cliente mobile cria e edita despesas sem internet e sincroniza depois. O backend precisa ser projetado para isso:

- Operações de escrita **idempotentes**, com um **ID gerado pelo cliente** (GUID) para evitar duplicação em reenvios.
- Cada registro guarda data de criação no cliente e data de recebimento no servidor.
- Endpoint de sincronização:
  - recebe um lote de operações pendentes do cliente;
  - devolve o resultado por operação (aceita, rejeitada, conflito);
  - permite ao cliente buscar alterações desde um cursor/timestamp (delta sync).
- **Conflitos: a última escrita vence** (_last write wins_). Não há bloqueio otimista nem revisão manual. A ordem é definida pelo **horário de chegada da operação ao servidor**; o relógio do cliente nunca decide quem vence (o horário do cliente é guardado só como informação).
- Como só donos editam/excluem despesas, conflitos reais devem ser raros. Mesmo assim, toda sobrescrita é registrada no feed com estado antes/depois, preservando a auditoria.
- Exclusão offline de despesa também segue "última escrita vence" (soft delete).
- O feed de atividades registra a data real do evento e a data da sincronização.

### 3.10 Lembretes de cobrança

- Notificações push automáticas, amigáveis e bem-humoradas, para membros com saldo devedor pendente.
- Um job agendado avalia saldos e frequência e dispara os lembretes.
- Regras: respeitar frequência máxima (sem spam), permitir silenciar por grupo/usuário e parar quando a dívida for quitada.
- Textos variam por templates/tons.
- Requer registro de tokens de dispositivo (FCM/APNs) por usuário.

### 3.11 Comprovantes e OCR (OCR/IA adiado)

> Anexos podem vir antes; **OCR/IA fica para uma fase futura**, sem provedor definido. Câmbio/multimoeda também está adiado: por ora tudo é BRL, mas evite decisões que dificultem adicionar moeda depois.

- Upload de anexos (foto de nota fiscal, comprovante) vinculados à despesa, em armazenamento de objetos (não no banco).
- Fluxo OCR/IA: o cliente envia a imagem, o backend extrai itens e valores (serviço externo de OCR/visão) e devolve uma **proposta de despesa** estruturada (itens, quantidades, valores, total, taxas).
- A proposta é só **sugestão**: o usuário revisa e confirma antes de virar despesa.
- Processamento pode ser assíncrono (job + status), por poder ser lento.

---

## 4. Requisitos transversais

- **Autenticação e autorização:** só membros do grupo acessam seus dados; editar/excluir despesa, confirmar liquidação e gerir membros são restritos aos donos do grupo.
- **Arquitetura:** monólito em **camadas simples** (API → Application/serviços → Domain → Infraestrutura/EF Core), sem CQRS/MediatR por ora. O `Domain` não depende de nenhuma outra camada.
- **Concorrência:** última escrita vence; sem controle otimista.
- **Persistência:** PostgreSQL via EF Core; valores monetários como `numeric(18,2)` (ou centavos em `bigint`); datas em `timestamptz`.
- **Datas:** persistir em UTC.
- **Validação:** rejeitar dados inconsistentes (somas, participantes fora do grupo).
- **Testes:** cobertura forte em divisão de despesas, simplificação de dívidas, arredondamento. Esses testes são o núcleo de qualidade do sistema.
- **Processos em background:** recorrências, lembretes, OCR.
- **Integrações externas:** OCR/IA, push (FCM/APNs), armazenamento de arquivos.

---

## 5. Fora do escopo do backend

- Interface, navegação e armazenamento local do app (a lógica de fila offline no cliente é responsabilidade do app; o backend só oferece contrato idempotente de sync).
- Processamento real de pagamentos (o PIX é apenas gerado, não intermediado).
- Captura da câmera e pré-processamento de imagem no dispositivo.

---

### Decisões

- Login por e-mail e senha; telefone é só um login alternativo. Identidade verificada por e-mail.
- Sem membros sem conta.
- PostgreSQL + EF Core.
- Conflitos offline: última escrita vence, pelo horário de chegada ao servidor.
- Donos do grupo (pode haver vários) editam/excluem despesas e confirmam liquidações.
- Convites por e-mail, push, código ou link.
- Arquitetura em camadas simples.
- Qualquer membro lança despesas; só donos editam/excluem.
- Recorrência: semanal e mensal.
- Sem planos ou limites de uso por ora.
- Validade padrão do convite: 3 dias.
- Apenas BRL por ora; câmbio e OCR/IA adiados.
