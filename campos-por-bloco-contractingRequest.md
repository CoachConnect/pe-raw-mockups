# Campos por bloco — comparação TabContracting (form atual) vs. mockup "Dossiê" (`acompanhamento/index.html`)

Fonte original: `apps/ui/src/modules/artistic-contracting/components/ContractSubmissionPending/Components/TabContracting/index.tsx`
Fonte da proposta: `pages/acompanhamento/index.html`

Nomes de campo em inglês, como declarados no código (`TContractingRequest`, `TContractingRequestForm` e tipos relacionados). Quando o dado não pertence a `TContractingRequest` (é apenas referenciado por id), o módulo/tipo dono está indicado entre colchetes.

---

## Cabeçalho / Ribbon (KPIs do topo)

| Label no mockup | Campo original | Tipo / módulo |
|---|---|---|
| `#11159` | `code` | `TContractingRequest` |
| Nome da peça/evento | `artist.nameGroup` + `eventManagement.name` | `TArtist` [artist] / `TEventPlannedV1.eventManagement` [planning] |
| Pill "Em homologação" | `status` | `TContractingRequest` (`EContractingRequestStatus`) |
| Pill "Documentação assinada enviada" | `contractingRequestSignableDocumentId` → `status` | `TContractingRequestSignableDocument.status` [artistic-contracting] (`EContractingSignableDocumentsStatus`) |
| Valor do contrato | `contractValue` | `TContractingRequest` |
| Apresentação (data/hora) | `eventPlannedSchedules[].startDate/startTime/endTime` | `TEventPlannedSchedule` [planning] |
| Unidade requisitante | `requestingUnitId` / `requestingUnit` | `TContractingRequest` → `TRequestingUnit` [basic-records] |
| Processo SEI | `proccessSEI` *(grafia com duplo "c", mantida no frontend)* | `TContractingRequest` |
| Certidões (contagem/vencimento) | `artist.artistTaxCertificates`, `artist.artistListAndSentenced` | [artist] |
| Pendências (contagem) | derivado (ver bloco 1) | — |

---

## Bloco 1 — Pendências desta análise (`#s-pendencias`)

Bloco **derivado/computado**, não é uma seção de campos próprios — agrega inconsistências de outros blocos:

| Alerta | Campo de origem |
|---|---|
| Processo SEI não informado | `proccessSEI` vazio |
| Parcela com vencimento inválido | `installments[].paymentDate` |
| Reabertura em aberto | `contractingRequestRewindHistories[]` / `rewindStatus` (`TContractingRequestRewindHistory`) |
| Certidões vencendo | `artist.artistTaxCertificates.*FileValidity`, `artist.artistListAndSentenced.*FileValidity` |

*No form atual (`TabContracting`), nada disso é exibido como painel único — cada pendência só aparece dentro da aba/seção correspondente.*

---

## Bloco 2 — Evento e apresentação (`#s-evento`)

Não corresponde a um único componente do form atual: mistura `Presentations` (apenas leitura de `eventPlanned`) com dados de `TArtistEventType` (ficha do espetáculo) e `TEventPlannedSchedule`.

| Label no mockup | Campo original | Tipo / módulo |
|---|---|---|
| Nome do evento (objeto) | `contractObject` | `TContractingRequest` |
| Unidade requisitante | `requestingUnitId` / `requestingUnit` | `TContractingRequest` → [basic-records] |
| Nome sugerido (do espetáculo) | `nameEvent` | `TArtistEventType` [artist] |
| Data e horário | `startDate`, `startTime`, `endTime`, `rRule`, `durationInMinutes` | `TEventPlannedSchedule` [planning] |
| Tipo de evento | `eventTypeId` / `eventTypes.name` | `TArtistEventType` [artist] |
| Categoria | `categories[]` | `TArtistEventType` → `TCategory` [basic-records] |
| Público alvo | `targetAudienceId` / `targetAudience.name` | `TArtistEventType` [artist] |
| Classificação indicativa | `ageRanges` | `TArtistEventType` [artist] |
| Participações especiais | `specialParticipation` | `TEventPlannedV1` [planning] (`ESpecialParticipation`) |
| Reversão de bilheteria | `hasTicketOffice` | `TEventPlannedSchedule` [planning] |
| Local | `placeId` / `place` / `placeAddress` | `TEventPlannedSchedule` → `TPlace` [place] |
| Valor da apresentação | `contractValue` (ou `totalValue` do evento) | `TContractingRequest` / `TEventPlannedV1` |
| Sinopse | `synopsis` | `TArtistEventType` [artist] |
| Release | `release` | `TArtistEventType` [artist] |
| Fotos para divulgação (+ crédito) | `publicityPhotos[].photoUrl` / `.photoCredit` | `TArtistEventTypePublicityPhoto` [artist] |

**Componente atual equivalente:** `Presentations` (somente exibição, sem binding de form) + `EventPlannedDispenseLegalOpinion` (não expõe campo, calcula `dispenseLegalOpinion`/`comparisonValue` via `checkIfDispenseLegalOpinion`) + `SpecialParticipation` (lê/grava via `eventPlannedService`, sem campo de form).

---

## Bloco 3 — Artista (`#s-artista`)

Não existe como bloco no form atual — nenhum destes campos é editado em `TabContracting`; pertencem ao cadastro do artista (módulo `artist`), apenas referenciado via `artistId`.

**Líder do grupo / dados pessoais** — `TNaturalPerson` [proponent] ou `TEventMember` (`isLeader: true`) [artist]:

| Label | Campo |
|---|---|
| Nome completo | `firstName` + `lastName` (ou `name`) |
| Nome artístico | `artisticName` |
| Nome social | `socialName` |
| Data de nascimento | `dateOfBirth` |
| CPF/CIN | `cPF` |
| RG | `rG` |
| RNE / RNM | (campo específico de estrangeiro, ver `TNaturalPerson`/`TArtist`) |
| Nacionalidade | `nationality` |
| Pessoa com deficiência | `isPCD` |
| Gênero | `gender` |
| Raça/Cor | `skinColor` |
| Grau de instrução | (ver `TNaturalPerson`) |
| E-mail / Telefone | (contato do proponente/artista) |
| Endereço de referência | `address` |

**Ficha técnica (membros do grupo)** — `TEventMember` [artist], array em `TArtistEventType.eventMembers` / `.members`:

| Label | Campo |
|---|---|
| Integrante | `name` |
| Nome artístico | `artisticName` |
| Nacionalidade | `nationality` |
| CPF/CIN | `cPF` |
| RG / RNM | `rG` |
| Idade | derivado de `dateOfBirth` |
| Função | `positionsAndFunctions` |

**Documentos do artista** — `TArtistDocument` [artist]: `{id, documentType, value, artistId}` (`documentType` = `EArtistCollectiveDocument` / `EArtistWithAgentDocument` / `EArtistWithoutAgentDocument`, etc.)

---

## Bloco 4 — Proponente (`#s-proponente`)

Não existe no form atual. Dados de Pessoa Jurídica — `TLegalPerson` [proponent]:

| Label | Campo |
|---|---|
| Razão social | `companyName` |
| Nome fantasia | `tradingName` |
| CNPJ | `cNPJ` |
| CCM | `ccm` |
| Endereço | `address` |
| Representante legal | `legalPersonResponsibles` |

Dados bancários do proponente — `bankAccounts` / `bankAccountLegalPersons` (`TBankAccount` [shared]: `bankName`, `agency`, `account`).

> Diferente do bloco 6 (Contrato/Pagamento), cujo `bankAccount` é a conta **snapshot** gravada no próprio `TContractingRequest` (`TContractingRequestBankAccount`), usada para o pagamento deste pedido especificamente.

---

## Bloco 5 — Certidões, impedimentos e apenados (`#s-certidoes`)

Não existe como bloco único no form atual. Vem de `TArtistTaxCertificates` e `TArtistListAndSentenced` [artist] — pares `<nome>FileValidity` / `<nome>File`:

| Certidão | Campo |
|---|---|
| CNPJ | `cnpjFileValidity` / `cnpjFile` |
| FGTS | `fgtsFileValidity` / `fgtsFile` |
| CND | `cndFileValidity` / `cndFile` |
| CNDT | `cndtFileValidity` / `cndtFile` |
| CTM | `ctmFileValidity` / `ctmFile` |
| CCM | `ccmFileValidity` / `ccmFile` |
| CADIN | `cadinFileValidity` / `cadinFile` |
| TCE | `tecFileValidity` / `tecFile` (`TArtistListAndSentenced`) |
| E-Sanções | `eSanctionsFileValidity` / `eSanctionsFile` |
| CNJ | `cnjFileValidity` / `cnjFile` |
| TCU | `tcuFileValidity` / `tcuFile` |
| CEIS | `ceisFileValidity` / `ceisFile` |
| Listagem de apenados | `listAndSentencedFileValidity` / `listAndSentencedFile` |

---

## Bloco 6 — Contrato e pagamento (`#s-contrato`)

Corresponde a `GeneralDataForm` + `PaymentForm` no form atual (`TabContracting`).

**Dados gerais do pedido** — `GeneralDataForm`:

| Label no mockup | Campo original |
|---|---|
| Processo SEI | `proccessSEI` (exibição) / `parentProccessSEI` (editável, só quando `reserveRequest == GLOBAL`) |
| Status de acompanhamento | `status` (`EContractingRequestStatus`) |
| Fundamento legal | `contractingRequestType` (deriva o texto de fundamentação) |
| Código do pedido | `code` |
| Responsável pelo pedido | `responsibleRequestId` / `responsibleRequest` |
| Responsável pelo cadastro | `requesterId` / `requester` |
| Fiscal | `contractingRequestSupervisors[].supervisorId` / `.supervisor` |
| Suplente | `contractingRequestSupervisors[].surrogateId` / `.surrogate` |
| Projeto especial | `specialProject` |
| Pedido de reserva | `reserveRequest` (`EReserveRequest`) |
| Nome da reserva | (nome da dotação/reserva vinculada — via `reserveAllocationId`/`reserveAllocationNumberId`) |
| Despesas extraordinárias | `hasExtraordinaryExpenses` [planning, `TEventPlannedV1`] |

**Dotação e valores / pagamento** — `PaymentForm`:

| Label no mockup | Campo original |
|---|---|
| Nº da dotação | (dotação orçamentária — via `reserveAllocationNumberId`) [reserve-allocation] |
| Valor do contrato | `contractValue` |
| Forma de pagamento | `formOfPayment` |
| Parcela (nº, valor, vencimento, situação) | `installments[].paymentNumber`, `.value`, `.paymentDate`, `.status` (`EInstallmentsContractingRequestStatus`) |
| Nota fiscal da parcela | fora de `TContractingRequest` — controlado por `showTaxInvoices` |
| Dados bancários (Banco/Agência/Conta) | `bankAccount.bankName` / `.agency` / `.account` (`TContractingRequestBankAccount`) |

Campos auxiliares de UI (não persistidos como estão): `_kitSentStartDate`, `_installmentsQuantity`, `showBankAccount`, `showTaxInvoices`.
Campos de observação: `observation`, `paymentObservation`.

---

## Bloco 7 — Documentos do contrato (`#s-documentos`)

Não existe no form atual (`TabContracting` não trata assinatura de documentos). Vem de `TContractingRequestSignableDocument` [artistic-contracting], referenciado por `contractingRequestSignableDocumentId`:

| Label no mockup | Campo original |
|---|---|
| Status geral da documentação | `TContractingRequestSignableDocument.status` (`EContractingSignableDocumentsStatus`) |
| Documento / Grupo / Assinatura (por linha) | `documents[].file`, `.documentModel`, `.status`, `.sentDate` (`TContractSignableDocumentSMC`) |
| Histórico de aprovação/reprovação | `histories[]` / `disapprovedHistories[]` (`TContractingRequestSignableDocumentHistory`) |

---

## Bloco 8 — Justificativas (`#s-justificativas`)

Corresponde a `JustificationsForm` no form atual — mapeamento direto:

| Label no mockup | Campo original |
|---|---|
| Justificativa da contratação | `justificationContracting` |
| Justificativa do valor | `justificationContractingValue` |
| Justificativa da escolha do contratado | `justificationChoiceContractor` |
| Observação | `observation` |

---

## Rail (coluna lateral do mockup)

| Bloco do mockup | Campo original | Tipo / módulo |
|---|---|---|
| Reabertura em aberto (solicitante, data, justificativa) | `justification`, `requestedDate`, `user.name`, `from`/`target` | `TContractingRequestRewindHistory` [artistic-contracting] |
| Andamento (timeline) | `title`, `changeDate`, `statusBefore`/`statusAfter`, `userName` | `TContractingRequestHistory` [artistic-contracting] |
| Contato do produtor (e-mail/telefone) | contato do artista/proponente | `TArtist`/`TLegalPerson`/`TNaturalPerson` |

---

## Componentes sem campo próprio de `TContractingRequest`

- **`Presentations`** — só exibição, prop `eventPlanned: TEventPlannedV1`, sem `control`/`name`.
- **`ScopeOfRepresentation`** — não usa `Controller`; estado local (`scopeOfRepresentationSelected`) só é copiado para o campo de form `scopeOfRepresentation` no `onSubmit` (`contractingRequestForm.context.tsx`).
- **`SpecialParticipation`** — CRUD direto via `eventPlannedService`, sem campo de form; dado pertence a `TEventPlannedV1.specialParticipation` / `.specialParticipations[]`.
- **`FaseProSubmissionMissingUserAlert`** — só alerta informativo, prop `contractingRequestId`.
- **`EventPlannedDispenseLegalOpinion`** — só alerta informativo, prop `eventPlannedId`, dado computado via `checkIfDispenseLegalOpinion`.

---

## Nota — campo SEI

`proccessSEI` (grafia com "cc" duplo) é o nome atual e correto do campo no frontend (`TContractingRequest`, `contractingRequest.ts:135`) — não foi renomeado. O commit `4807adb53c` ("refactor: remove unused proccessSEI property from request model") alterou **apenas o backend** (`UpdateAsDraft` DTO), removendo uma propriedade não usada; o frontend continua com `proccessSEI` (processo principal) e `parentProccessSEI` (processo SEI mãe, usado só em reserva `GLOBAL`).


---

## Endpoints [IGNORE THIS]

Features/
  ContractingRequest/
    Blocks/
      ReadOneComplete (Retorna os dados formatados em blocos)
      ValidateArtisticContracting/
      ValidateMovie/
      ValidateAmendment/
        Event (Endpoint: )
        Contract (Endpoint: )
        Proponent  (Endpoint: )
        Submit  (Endpoint: Rascunho ou Reaberto)