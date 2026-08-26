# Criação de Emenda (v3) — Componentes, Estados e Regras de Negócio

> Documentação gerada a partir do protótipo estático `pages/criacao-emenda/index.html`.
> Protótipo sem integração real: todos os dados (vereadores, locais, fiscais, empresas, artistas etc.) são mocks em memória, definidos no `<script>` da página.

---

## 1. Visão geral da página

A página implementa o fluxo de criação de uma **Emenda Parlamentar**, organizado em **6 blocos sequenciais** + 1 bloco de **Resumo**, navegáveis por uma barra lateral fixa (sidenav) com scrollspy.

Encadeamento conceitual dos blocos (cardinalidades):

```
Vereador → EventManagement (1 evento : N eventos planejados)
EventManagement → ContractingRequest (N eventos planejados : 1 contrato)
LegalPerson (empresa) → Artist (empresa define quais artistas ficam disponíveis)
Artist + EventPlannedSchedule → EventPlanned (1 evento planejado : 1 artista)
ContractingRequest.amount ≥ soma dos cachês + despesas extraordinárias
ContractingRequest.paymentForm → ContractInstallment → InstallmentSubprefecture
  (soma das parcelas == valor do contrato)
```

Aviso fixo no topo do conteúdo:
- **"Emenda apta à execução — sem etapa de aprovação"**: diferente do fluxo padrão de contratação, emendas parlamentares **não passam por aprovação da coordenadoria** — os eventos planejados já nascem prontos para contratação assim que a emenda é criada.

### Diferenças desta versão (v2/v3) em relação à v1 (conforme comentário do código-fonte)
- O vereador autor da emenda passa a ser informado já na criação do evento (`EventManagement`).
- O bloco "Coordenadoria de Emendas" saiu; a unidade requisitante virou campo do bloco Contrato.
- "Artistas agenciados" e "Eventos planejados" foram unificados em um único bloco: cada evento planejado é criado escolhendo o artista e definindo o cronograma diretamente.
- Novo bloco "Valor do contrato".
- O contrato passou a receber os dados gerais da contratação (fiscais, unidade requisitante, SEI mãe, justificativas) diretamente no bloco Contrato.

---

## 2. Layout e componentes estruturais

### 2.1 Sidenav (`.sidenav`)
| Componente | Descrição |
|---|---|
| `sidenav__title` | Título fixo "Criação de Emenda". |
| `sidenav__order` | Cabeçalho com número da emenda (`requestNumber`, mock `2026/0087`), nome do vereador (`requestCouncilor`, atualizado dinamicamente) e chip de status da emenda (`requestStatus`). |
| `sidenav__progress` | Texto "`N` de 6 blocos corretos" (`progressText`) + barra de progresso (`progressBar`) — preenchida proporcionalmente à quantidade de blocos com status `valid`. |
| `sidenav__nav` | Lista de âncoras para os 6 blocos + Resumo. Cada item tem ícone de status e rótulo textual do status, atualizados via `paintStatus()`. Destaque do item ativo por *scroll spy* (`IntersectionObserver`, `rootMargin: -40% / -50%`). |
| `sidenav__legend` | Botão "Legenda" que expande um painel (`legend__panel`) explicando os 3 estados visuais: **Salvo** (✓), **Alterações não salvas** (✎), **Incorreto** (!). |

### 2.2 Status da emenda (ciclo de vida)
Estado simplificado, **sem etapa de aprovação**:

| Status interno | Label exibido | Ícone |
|---|---|---|
| `draft` | Emenda em rascunho | ✎ |
| `created` (após clique em "Criar emenda" com tudo válido) | Emenda criada | ✓ |

### 2.3 Barra de ações (`.actionbar`, fixa)
- `savedLabel`: horário do último bloco salvo ("Salvo às HH:MM").
- `globalStatusLabel`: resumo textual do progresso geral (ex.: "3/6 blocos corretos", "✓ Pronto para criar").
- Botão **"Criar emenda"** (`btnCreate`): dispara `validateAll()`; só conclui a criação se **todos** os 6 blocos estiverem com status `valid`.

### 2.4 Toast (`#toast`)
Notificação temporária (4s) usada para confirmar ações: criar/descartar contrato, selecionar/remover empresa, salvar bloco, definir valor do contrato, criar emenda, validar todos os blocos.

---

## 3. Modelo de status por bloco

Cada um dos 6 blocos possui um `<section class="card" data-status="...">` cujo estado é recalculado a cada `refresh()` pela função `evaluate(section)`, em ordem de precedência:

1. **`invalid`** — há erro visível (contradição entre dados já preenchidos, ou bloco já tentado salvar com pendências).
2. **`draft`** — há alterações não salvas (comparado ao último snapshot) e nenhum erro visível ainda.
3. **`valid`** — nenhum erro e nada pendente de salvar.
4. **`empty`** — nada preenchido e o bloco nunca foi salvo.
5. **`partial`** — começou a preencher, faltam campos obrigatórios, mas nada errado visível ainda.

Ícones/labels (`STATUS_META`):

| Status | Ícone | Label do chip |
|---|---|---|
| `empty` | ○ | Salvo |
| `partial` | ◐ | Salvo |
| `draft` | ✎ | Alterações não salvas |
| `invalid` | ! | Incorreto |
| `valid` | ✓ | Salvo |

### 3.1 Tipos de erro de validação (por momento em que ficam visíveis)
| Tipo | Helper | Quando aparece |
|---|---|---|
| `required` | `missing(field, msg)` | Campo obrigatório não informado — só some depois que o bloco é salvo (`attempted`). |
| `soft` | `soft(field, msg)` | Regra sobre valor em digitação (ex.: tamanho mínimo de texto) — só aparece depois que o bloco é salvo. |
| `live` | `wrong(field, msg)` | Contradição inequívoca entre dados já informados (ex.: datas invertidas, formato de campo) — aparece **imediatamente**, mesmo sem salvar. |

### 3.2 Mecanismo de "rascunho" (dirty tracking)
- `snapshots` (Map) guarda a última serialização salva de cada bloco (`serializeSection`), concatenando `índice:valor` de todo `input/select/textarea` do formulário.
- `isDirty(section)` compara a serialização atual com o snapshot.
- Adicionar/remover linhas dinâmicas (fiscais, eventos planejados, datas de cronograma, parcelas, subprefeituras) também conta como alteração, pois desloca os índices dos campos seguintes.
- Botão **"Salvar alterações"** de cada bloco fica desabilitado enquanto o bloco não estiver `dirty`.
- Salvar um bloco: tira novo snapshot, marca o bloco em `attempted` (liberando todos os erros `required`/`soft` para exibição) e mostra toast de sucesso ou de pendências.

---

## 4. Bloco 1 — Evento da Emenda (`EventManagement`)

**Regra de negócio central:** um único evento (`EventManagement`) agrupa todos os eventos planejados desta emenda (1 evento : N eventos planejados).

### Componentes
| Campo | Tipo | Obrigatório | Observações |
|---|---|---|---|
| `councilorId` | select | Sim | Vereador autor da emenda. Gravado no `EventManagement` na criação e propagado aos eventos planejados. |
| `councilorInfo` (readonly) | painel de leitura | — | Mostra mandato, teto de emendas do exercício, valor já comprometido, saldo disponível e quantidade de eventos unificados do vereador. |
| `eventMode` | radio (choice-list) | Sim | Escolha excludente: **"Vincular ao evento unificado do vereador"** (padrão, `unified`) vs **"Criar um evento próprio para esta emenda"** (`new`). |
| `eventManagementId` (subform `unified`) | select | Sim, se `eventMode=unified` | Evento guarda-chuva já existente do vereador selecionado. |
| `eventManagementInfo` (readonly) | painel de leitura | — | Tipo de projeto, vigência, vereador do evento, eventos planejados já vinculados. |
| `eventName` (subform `new`) | texto | Sim, se `eventMode=new` | Mínimo 5 caracteres (regra *soft*). |
| `projectType` (subform `new`) | texto, desabilitado | — | Fixo: "Emenda Parlamentar". |
| `eventStartDate` / `eventEndDate` (subform `new`) | date | Sim, se `eventMode=new` | Fim não pode ser anterior ao início (regra *live*). |
| `eventDescription` (subform `new`) | textarea | Não | Objeto cultural, público e território atendidos. |
| `placeId` | select | Sim | **Único local habilitado** para toda a emenda: usado depois para restringir fiscais (Bloco 2) e locais do cronograma (Bloco 4). |

### Regras de validação
- Vereador obrigatório.
- Se `eventMode=unified`: evento unificado obrigatório; se o evento escolhido pertencer a outro vereador → erro *live* ("Este evento pertence a outro vereador").
- Se `eventMode=new`: nome (mín. 5 caracteres), início e fim de vigência obrigatórios; fim < início → erro *live*.
- Local do evento obrigatório.

### Efeitos colaterais / dependências
- Trocar o vereador recarrega as opções de `eventManagementId` (apenas eventos do vereador escolhido), preservando um valor "stale" (fora da lista) com rótulo "— de outro vereador" para não perder a seleção silenciosamente.
- O local escolhido aqui é a **única fonte de locais habilitados** para: linhas de fiscal por local (Bloco 2) e locais de cronograma dos eventos planejados (Bloco 4).

---

## 5. Bloco 2 — Contrato (`ContractingRequest`)

**Regra de negócio central:** o contrato é criado **sem artista e sem valor** — recebe apenas os dados gerais da contratação. Os eventos planejados desta emenda são anexados a ele (N eventos planejados : 1 contrato).

### Componentes
| Componente | Descrição |
|---|---|
| `contractCode` (hidden) | Código do contrato (mock `CT-2026-004312`), preenchido só ao clicar em "Criar contrato". |
| `contractInfo` (readonly) | Painel: código, status ("Rascunho"), objeto, unidade requisitante, quantidade de eventos planejados vinculados, valor do contrato. |
| Botão **"Criar contrato"** | Preenche `contractCode` (mock) e exibe toast. Enquanto não criado, bloco não pode ser considerado válido. |
| Botão **"Descartar contrato"** | Visível só quando há contrato criado; limpa `contractCode`. |
| **Múltiplos fiscais por local** (`inspectorMode`, radio) | **Sim** (padrão, `multiple`): um fiscal + suplente por cada local habilitado. **Não** (`single`): um único fiscal para todo o contrato. |
| `fiscalRows` (subform `multiple`) | Lista dinâmica de linhas fiscal/suplente/local. Botão "+ Adicionar fiscal". Cada linha pode ser removida. |
| `fiscalSingleId` / `substituteSingleId` (subform `single`) | Fiscal único e suplente para todo o contrato. |
| `registrationResponsible` | Texto, valor inicial mockado ("Salete de Campos Dias") — usuário autenticado cadastrando a solicitação. |
| `requestingUnitId` | Select — Unidade Requisitante (substituiu o antigo bloco "Coordenadoria de Emendas"). |
| `initialResponsibleId` | Select — Responsável pela solicitação inicial. |
| `legalFoundation` | Select — Fundamento Legal da Contratação (Editais, Inexigibilidade art. 74 II, Dispensa de licitação, Emenda parlamentar). |
| `contractObject` | Texto — objeto do contrato, mín. 5 caracteres (regra *soft*). |
| `parentProccessSEI` | Texto — Processo SEI Mãe; formato obrigatório `00000.000000/0000-00` (regra *live* de formato). |
| `contractObservation` | Texto livre, opcional. |
| **Pedido de Reserva** (`reserveRequest`, radio) | **Padrão** vs **Global** (padrão selecionado). Global: uma única reserva cobre todos os eventos planejados do contrato. |
| `justContracting` (RTE) | Justificativa da contratação — obrigatória, mín. 20 caracteres de texto puro (regra *soft*). |
| `justContractorChoice` (RTE) | Justificativa da escolha do contratado — obrigatória, mín. 20 caracteres (regra *soft*). |
| `justContractingValue` (RTE) | Justificativa do valor da contratação — **opcional aqui** (a memória de cálculo do valor global fica no Bloco 5); se preenchida, mín. 20 caracteres. |

### Regras de validação
- Contrato deve existir (`contractCode` preenchido) — senão erro em bloco (`contractBlock`).
- Modo `multiple`: ao menos uma linha de fiscal preenchida; cada linha exige fiscal e local; local deve estar entre os locais habilitados no Bloco 1 (senão erro *live*); um mesmo local não pode ter dois fiscais (erro *live*); suplente deve ser diferente do fiscal (erro *live*).
- Modo `single`: fiscal obrigatório; suplente diferente do fiscal (erro *live*).
- Responsável pelo cadastro, unidade requisitante, responsável pela solicitação inicial e fundamento legal obrigatórios.
- Objeto do contrato obrigatório (mín. 5 caracteres).
- SEI mãe obrigatório e com formato validado por regex.
- Justificativa da contratação e da escolha do contratado obrigatórias (mín. 20 caracteres); justificativa do valor é opcional, mas se preenchida também exige mín. 20 caracteres.

### Editor de texto rico (RTE) — componente reutilizado nas 3 justificativas
- `contenteditable` espelhado em um `<textarea>` oculto (fonte de verdade para serialização/validação).
- Toolbar: negrito, itálico, sublinhado, lista numerada, lista com marcadores, alinhar à esquerda/centro, inserir link (via `prompt`), limpar formatação, seletor de bloco (Normal / Título / Subtítulo).
- Estado vazio (`is-empty`) tratado como string vazia mesmo que reste apenas marcação HTML sem texto.

---

## 6. Bloco 3 — Empresa (`LegalPerson`)

**Regra de negócio central:** a empresa agenciadora contratada define quais artistas ficam disponíveis para os eventos planejados do Bloco 4.

### Componentes
| Componente | Descrição |
|---|---|
| `companyId` (hidden) | Empresa selecionada. |
| `companyPanel` (readonly) | Razão social, nome fantasia, CNPJ, quantidade de artistas agenciados. Mensagem de vazio quando nenhuma empresa selecionada. |
| Botão **"Selecionar empresa"** / **"Trocar empresa"** | Abre modal de seleção (`companyModal`). |
| Botão **"Remover empresa"** | Visível só quando há empresa selecionada; limpa a seleção. |

### Modal de seleção de empresa (`#companyModal`)
- Campo de busca (`companySearch`) filtra por razão social, nome fantasia ou CNPJ (case-insensitive, live).
- Tabela de seleção única (radio por linha) com colunas: Razão social/Nome fantasia, CNPJ, quantidade de Artistas.
- Mensagem de rodapé indica a empresa marcada ou "Nenhuma empresa selecionada".
- Botão **"Selecionar empresa"** do modal fica desabilitado até haver seleção pendente.
- Fecha por: botão Cancelar, clique no backdrop, ou tecla `Escape`.
- Confirmar seleção → toast informando quantos artistas agenciados ficam disponíveis; se a empresa não mudou, toast apenas confirma que foi mantida.

### Regras de validação
- Empresa é obrigatória para o bloco ser válido.

### Efeitos colaterais / dependências
- **Trocar/remover a empresa não limpa os eventos planejados já criados** — apenas os artistas ficam órfãos ("fora dos agenciados da empresa") até serem corrigidos, e o validador do Bloco 4 aponta o artista como inválido (`wrong`).

---

## 7. Bloco 4 — Artistas e eventos planejados (`EventPlanned` / `Artist` / `EventPlannedSchedule`)

**Regra de negócio central:** cada evento planejado é criado escolhendo **um** artista agenciado (1 evento planejado : 1 artista) e definindo seu cronograma de apresentações. Tipo do evento planejado: `AMENDMENT_EVENT`.

### 7.1 Painel "Artistas agenciados pela empresa" (somente leitura)
- Tabela: Artista/grupo, Tipo (Grupo/Coletivo ou Pessoa física), Atrações cadastradas (`ArtistEventType`, com duração e valor sugerido), quantidade de eventos planejados nesta emenda que já usam aquele artista.
- Mensagens de vazio: "selecione a empresa" (sem `companyId`) ou "esta empresa não possui artistas agenciados".
- Serve só de referência; a seleção do artista acontece dentro de cada card de evento planejado.

### 7.2 Lista "Eventos planejados (mínimo 1)"
- Botão **"+ Adicionar evento planejado"** (desabilitado sem empresa selecionada).
- Mensagem de vazio quando não há nenhum evento planejado.

Cada card de evento planejado (`.ep`) contém:

| Campo | Tipo | Obrigatório | Regras |
|---|---|---|---|
| `artist` (select) | — | Sim | Restrito aos artistas agenciados pela empresa do Bloco 3; se artista não pertencer mais à empresa (mudança de empresa) → erro *live*. |
| `aet` (select — atração) | — | Sim | Opções dependem do artista escolhido (`ArtistEventType`); ao escolher, sugere automaticamente duração e valor cadastrados (só preenche campos vazios, não sobrescreve). |
| `epTitle` | texto | Sim | Mín. 5 caracteres (regra *soft*). |
| `epDescription` | textarea | Não | Detalhes da atração, público-alvo, acessibilidade. |
| Botão "Remover evento" | — | — | Remove o card inteiro. |

Cabeçalho do card é recalculado dinamicamente (`renumberEvents`): número do evento, artista + atração escolhidos, quantidade de datas, subtotal em R$.

### 7.3 Cronograma do evento planejado (mínimo 1 data)
Botão "+ Data" adiciona linha de cronograma (`.schedule-row`):

| Campo | Tipo | Obrigatório | Regras |
|---|---|---|---|
| `place` (select) | — | Sim | Restrito aos locais habilitados no Bloco 1; local fora da lista → erro *live*. |
| `start-date` | date | Sim | — |
| `end-date` | date | Não | Preencher só em temporadas/recorrências; término anterior ao início → erro *live*. |
| `all-day` (switch) | checkbox | — | Quando marcado, desabilita e dispensa horário de início e duração. |
| `start-time` | time | Sim, se não for dia inteiro | — |
| `duration` (min) | number | Sim, se não for dia inteiro | Deve ser > 0. |
| `amount` (cachê, R$) | number | Sim | Deve ser > 0. |

- **Regra de conflito de agenda:** não pode haver duas datas iguais (mesma data + horário + local) dentro do **mesmo** evento planejado (erro *live*).
- Botão "Remover data" por linha.

### Regras de validação do bloco (resumo)
- Sem empresa selecionada → erro de bloco pedindo para selecionar a empresa antes.
- Ao menos 1 evento planejado obrigatório.
- Por evento planejado: artista válido e agenciado pela empresa; atração selecionada; título (mín. 5 caracteres); ao menos 1 data no cronograma.
- Por data do cronograma: data de início obrigatória; término ≥ início; se não for dia inteiro, horário e duração (>0) obrigatórios; local obrigatório e habilitado; valor do cachê obrigatório e > 0; sem duplicidade de slot (data+horário+local) dentro do mesmo evento.

---

## 8. Bloco 5 — Valor do contrato (`ContractingRequest.amount`)

**Regra de negócio central:** valor global a ser empenhado no contrato — **não pode ser menor que a soma dos cachês dos eventos planejados** (+ despesas extraordinárias).

### Componentes
| Componente | Descrição |
|---|---|
| `contractValueInfo` (readonly) | Quantidade de eventos planejados e datas, soma dos cachês, despesas extraordinárias, valor mínimo do contrato (cachês + extraordinárias), margem sobre o mínimo (destacada em vermelho se negativa, verde se ≥ 0). |
| `contractValue` (R$) | Obrigatório, > 0. |
| `extraordinaryValue` (R$) | Opcional; não pode ser negativo (erro *live*); custos fora do cachê (transporte, cenografia, acessibilidade). |
| Botão **"Usar a soma dos eventos planejados"** | Preenche `contractValue` com `schedulesTotal() + extraordinaryAmount()`; toast de aviso se não houver nenhum cachê informado ainda. |
| `contractValueJustification` (textarea) | Obrigatória **somente se** houver despesas extraordinárias > 0; caso contrário opcional. |

### Regras de validação
- Despesas extraordinárias, se preenchidas, não podem ser negativas.
- Se despesas extraordinárias > 0 → justificativa obrigatória.
- Valor do contrato obrigatório e > 0.
- Valor do contrato não pode ser menor que o mínimo exigido (soma dos cachês + despesas extraordinárias) — erro *live*, mensagem inclui o valor mínimo calculado.

---

## 9. Bloco 6 — Parcelamento (`ContractingRequest.paymentForm` / `ContractInstallment` / `InstallmentSubprefecture`)

**Regra de negócio central:** define como o valor do contrato será pago e, em cada parcela, quanto cabe a cada subprefeitura. **A soma de todas as parcelas tem de fechar exatamente com o valor do contrato.**

### Componentes
| Componente | Descrição |
|---|---|
| `paymentInfo` (readonly) | Forma de pagamento, valor do contrato, data de entrega do kit + pagamento previsto (kit + 30 dias), e (quando aplicável) quantidade de parcelas/subparcelas, total das parcelas e saldo a distribuir (destacado se ≠ 0). |
| **Forma de pagamento** (`paymentMethod`, radio, obrigatório) | `unica` (Parcela Única), `parcelado` (Parcelado), `bilheteria` (Reversão de Bilheteria), `outros` (Outros), `sem-cache` (Sem Cachê). Regra fixa de negócio (texto de ajuda): pagamento ocorre no **30º dia** após entrega de toda a documentação. |
| `kitDate` | date — Data prevista para entrega do kit de documentação; chip lateral mostra a data formatada (`kitDateChip`). Tooltip explica que o pagamento ocorre 30 dias depois. |
| `parcelasPanel` / `parcelasList` | Visível somente para `unica` e `parcelado` (`PAYMENT_WITH_INSTALLMENTS`). |
| `boxOfficeShare` / `boxOfficeMinimum` (subform `bilheteria`) | Percentual de reversão (obrigatório, 0–100) e valor mínimo garantido (opcional, ≥ 0). |
| `paymentOtherDescription` (subform `outros`) | Obrigatório, mín. 15 caracteres (regra *soft*). |
| Texto informativo (subform `sem-cache`) | Sem valor a pagar ao contratado; despesas extraordinárias, se houver, continuam no valor do contrato. |
| `paymentObservation` | Texto livre, opcional (condições, retenções, documentos exigidos). |

### 9.1 Estrutura de parcelas × subprefeituras
- **Parcela Única**: exatamente 1 parcela; ao trocar para esta forma, todas as parcelas além da primeira são removidas automaticamente.
- **Parcelado**: 2+ parcelas obrigatórias; botão "+ Adicionar parcela" habilitado; botão "Remover parcela" só aparece por linha quando há 2+ parcelas.
- Cada parcela (`.parcela`) tem uma tabela de subprefeituras com valor (`InstallmentSubprefecture`): select de subprefeitura + valor R$; botão "+ Adicionar novo valor por subprefeitura" e remoção por linha (ícone de lixeira).
- Título e metadados de cada parcela são renumerados dinamicamente (`renumberParcelas`): "1ª parcela", "2ª parcela"... com total de subprefeituras e soma da parcela.

### Regras de validação
- Forma de pagamento obrigatória.
- `bilheteria`: percentual obrigatório, > 0 e ≤ 100; valor mínimo garantido, se informado, não pode ser negativo.
- `outros`: descrição obrigatória, mín. 15 caracteres.
- `sem-cache`: sem validações adicionais.
- `unica`/`parcelado`: ao menos 1 parcela; `parcelado` exige ao menos 2 parcelas (erro *live*, sugere voltar para Parcela Única se só houver 1); cada parcela precisa de ao menos 1 subprefeitura com valor; subprefeitura obrigatória e sem duplicidade dentro da mesma parcela (erro *live*); valor da subprefeitura obrigatório e > 0; **soma total das parcelas deve bater com o valor do contrato** (tolerância de R$ 0,005) — divergência gera erro *live* mostrando a diferença exata.

---

## 10. Bloco Resumo (`sec-resumo`)

- Consolida os 6 blocos em grupos de leitura, cada um com o chip de status do bloco correspondente.
- Repete todos os principais campos preenchidos (vereador, evento, fiscais, contrato, empresa, cada evento planejado com suas datas e subtotal, valor do contrato, parcelamento/subprefeituras).
- Bloco de totais: quantidade de eventos planejados, quantidade de datas programadas, valor do contrato.
- Botão **"Validar todos os blocos"**: marca todos os blocos como "tentados" (`attempted`), força exibição de todos os erros pendentes e foca o primeiro bloco inválido (ou confirma que está tudo certo via toast).
- Rodapé do resumo não tem estado próprio — só descreve o andamento agregado dos blocos ("Todos os blocos estão salvos e corretos", "N blocos incorretos", "N blocos com alterações não salvas", ou "N de 6 blocos corretos").

---

## 11. Regras de negócio transversais (cross-block)

1. **Sem etapa de aprovação**: ao contrário de contratações regulares, a emenda parlamentar não passa pela coordenadoria — eventos planejados nascem prontos para contratação.
2. **Local único por emenda**: o local escolhido no Bloco 1 restringe tanto os locais elegíveis para fiscais (Bloco 2) quanto para o cronograma dos eventos planejados (Bloco 4). Trocar o local não remove seleções já feitas em outros blocos — apenas passa a marcá-las como fora da lista (erro *live*), preservando o valor para correção manual explícita.
3. **Empresa define artistas elegíveis**: trocar/remover a empresa (Bloco 3) não apaga eventos planejados já criados (Bloco 4); artistas que deixam de pertencer à empresa ficam sinalizados como inválidos até correção.
4. **Contrato nasce vazio**: só existe após clique explícito em "Criar contrato"; não tem artista nem valor até os blocos seguintes serem preenchidos.
5. **Valor mínimo do contrato**: nunca pode ser menor que a soma dos cachês dos eventos planejados + despesas extraordinárias (Bloco 5), reforçando a integridade financeira do orçamento.
6. **Fechamento de parcelas**: soma das parcelas (Bloco 6) deve ser rigorosamente igual ao valor do contrato (Bloco 5), com tolerância de arredondamento de R$ 0,005.
7. **Criação da emenda só é permitida quando os 6 blocos estão com status `valid`** (salvos e sem erros); qualquer bloco com alterações não salvas ou erro pendente bloqueia a criação e foca o primeiro problema.
8. **Persistência local por sessão de navegação**: não há chamada de API — "salvar" e "criar emenda" apenas atualizam estado em memória e disparam toasts (`showToast`), simulando o comportamento esperado do backend real.
