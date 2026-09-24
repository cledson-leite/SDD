# Descrição de Produto — Sistema de Atendimento e Gestão (Rigoni Contábil)

> Modelo de descrição para entrada no `wayfinder`. Reorganizado por assunto (não por ordem cronológica de escrita), com nomenclatura unificada entre seções que descrevem a mesma coisa. Pontos que ficam propositalmente em aberto (pra serem resolvidos na etapa de grilling) estão marcados no fim, em "Notas / pontos em aberto".

## 0. Escopo desta rodada

Implementação atual: **apenas o setor Simples Nacional** ganha painel/dashboard de funcionário nesta rodada.

⚠️ Em aberto: o backbone de atendimento via WhatsApp/URA (seções 6 e 7) é descrito de forma multi-setor (menu com todos os setores, transferência entre qualquer setor, fila e balanceamento por setor) — falta decidir se esse backbone inteiro já é construído multi-setor nesta rodada (com os demais setores só sem painel de funcionário ainda) ou se também fica restrito ao Simples Nacional por enquanto.

## 1. Papéis e acesso (RBAC via Supabase)

Papéis humanos, com login por e-mail e senha:

- **Admin** — acesso geral, todos os setores.
- **Líder de setor** — acesso restrito ao seu setor.
- **Funcionário** — acesso restrito ao seu setor.

Cada usuário tem nome, setor e papel. Cadastro:

- Admin e líder de setor são cadastrados apenas por admin.
- Funcionário é cadastrado pelo líder do respectivo setor.
- Em ambos os casos é gerada uma senha provisória; o usuário pode trocar a própria senha e fazer logout.

A **URA não é um papel de usuário com login** — é um ator de sistema (service role) que atende via WhatsApp. Suas permissões de leitura/escrita são específicas sobre colunas do cadastro de cliente, descritas na seção 3.

## 2. Setores

Os setores do sistema são os que constam em `organograma.pdf`, com as seguintes exceções e ajustes:

- Excluir os setores **Patriki** e **Expedição**.
- O setor "Presumido/Real" deve ser renomeado para **TRIBUTÁRIO** — esse é o nome a ser usado no sistema.

Regra de extração do PDF: em cada card da primeira linha, a primeira linha de texto é o nome do setor e a segunda linha de texto é o nome do líder do setor.

## 3. Cadastro de cliente

Base: planilha `CONTROLE SIMPLES NACIONAL`, reorganizada de forma intuitiva, legível e funcional, com os seguintes acréscimos.

| Coluna | Obrigatória | Tipo / valores possíveis | Quem atualiza |
|---|---|---|---|
| Empresa | Sim | texto | Admin (cadastro) |
| CNPJ | Sim | texto | Admin (cadastro) |
| Regime (nova) | Sim | ⚠️ a confirmar | Admin (cadastro) |
| Contrato social / Alterações contrato social (nova) | Sim | referência(s) para arquivo em storage — ⚠️ confirmar se 1 coluna com os dois tipos de documento ou 2 colunas separadas | Admin (cadastro) |
| Forma de envio | Não | ⚠️ a confirmar (orienta o canal/forma de entrega das guias) | Setor Simples Nacional, URA |
| Nome e contato padrão (WhatsApp) | Não | texto/telefone | Setor Simples Nacional, URA |
| Outros nomes e outros contatos | Não | lista texto/telefone | Setor Simples Nacional, URA |
| Documento obrigatório | Não | positivo / negativo / não se aplica | Setor Simples Nacional |
| DAS | Não | pendente (padrão) / pronto / enviado / sem movimento | Setor Simples Nacional, URA (e automaticamente pelo job da seção 9) |
| Data de envio | Não | data | URA (automaticamente pelo job da seção 9, no sucesso do envio) |
| Tem parcelamento | Não | positivo / negativo | Setor Simples Nacional |
| Parcelamento | Não | pendente / enviado / interno | Setor Simples Nacional, URA (e automaticamente pelo job da seção 9) |
| Observações gerais | Não | texto livre | Funcionário do setor (via tela de lista, seção 10) |

Regras de acesso:

- Cadastro de cliente (criação) é feito apenas por admin.
- O setor Simples Nacional pode atualizar as colunas opcionais.
- A URA pode atualizar: forma de envio, nome e contato padrão, outros nomes e outros contatos, DAS, data de envio, parcelamento.
- Qualquer setor pode visualizar o cadastro, quando necessário — exceto as colunas de contrato social/alterações, que nunca aparecem nas telas do setor Simples Nacional (ver seção 10).

## 4. Documentos: contrato social e alterações

Contrato social e alterações contratuais ficam vinculados ao cadastro do cliente e são persistidos no storage do Supabase.

- Contrato social = contrato original.
- Alteração contratual = precisa ser identificada de forma específica — ⚠️ escolher a melhor forma de referenciar (por data da alteração, por descrição da alteração feita, ou por número da alteração). Exemplos de ambos os tipos de documento estão na pasta `docs/referencia/contratos`.

## 5. Integrações Serpro

- Busca de guias de imposto e guias de parcelamento via API do Serpro, nos endpoints e payloads específicos de cada uma.
- Busca de CND (Certidão Negativa de Débitos) via API do Serpro, endpoint e payload específicos.

## 6. Atendimento via WhatsApp (URA)

Fluxo de identificação:

1. Cliente manda mensagem. O sistema verifica se o número está cadastrado como contato padrão de algum cliente e, se não, se está na lista de "outros contatos".
2. Se encontrado, usa o nome cadastrado (padrão ou outro) e oferece um menu com tarefas pré-programadas: contrato social/alterações, guia de impostos, guia de parcelamento, CND, falar com atendente.
3. Ao escolher uma opção, a URA segue o fluxo de coleta de informação necessário para a requisição correspondente. O CNPJ, quando necessário, não é perguntado — é buscado no mesmo cadastro de onde veio o contato.
4. Ao concluir a busca/geração do artefato correspondente (seção 5), a URA responde a mensagem do cliente com o artefato.

Se o contato não estiver cadastrado:

1. A URA pergunta se o contato já é cliente.
2. Se responder que não, transfere para a recepção (sempre avisando o cliente sobre a transferência, com texto amigável).
3. Se responder que sim, pede o CNPJ e busca esse CNPJ no cadastro.
   - Se não houver contato padrão cadastrado para esse cliente, pergunta se quer se cadastrar como contato padrão; se sim, pede o nome de contato e salva nome + número de WhatsApp no campo correspondente.
   - Se já houver contato padrão diferente e esse número não estiver em "outros contatos", segue o mesmo caminho, mas salvando como "outros contatos".
   - Se o contato recusar o cadastro, transfere para a recepção.

Fallback: se o cliente não responder ao que foi pedido, não escolher nenhuma opção pré-programada, não escolher nenhum setor de transferência, e escrever algo como "quero falar com atendente" (ou similar), transfere para a recepção.

## 7. Transferência e fila de atendimento

- Ao transferir, o cliente recebe mensagem informando a transferência e o nome/setor do atendente que vai atendê-lo. O funcionário do setor escolhido recebe um aviso com os dados do cliente (nome do contato, empresa, CNPJ, de onde foi transferido, resumo do assunto) e assume a conversa.
- A distribuição segue balanceamento mensal de atendimento entre os funcionários do setor.
- Se não houver funcionário disponível, o cliente entra em fila: recebe sua posição, e o setor visualiza o cliente na fila (com as informações já coletadas) junto do tempo de espera no formato `hh:mm:ss` — a hora só aparece a partir de 1h de espera.
- Quando há fila, o balanceamento mensal não decide a ordem — sempre é atendido o mais antigo na fila assim que um funcionário fica livre. O atendimento, porém, ainda conta no cálculo do balanceamento.
- O menu de setores para transferência usa os mesmos setores e exceções da seção 2, exibindo "nome do setor (nome do líder)" — ex.: "Simples Nacional (Leo)".
- Funcionários podem transferir o cliente para outros setores quando necessário para dar continuidade ao atendimento.

## 8. Painel do funcionário

- O funcionário pode disparar os mesmos serviços do menu pré-programado (seção 6) oferecido ao cliente, usando o CNPJ já vinculado à conversa.
- O funcionário pode alternar de aba sem fechar a conversa em andamento — a conversa deve continuar visível, de alguma forma, em alguma parte da tela.
- Em todas as conversas: o balão do cliente mostra o nome do contato padrão de WhatsApp; o balão do atendente mostra nome do atendente e seu setor.

## 9. Jobs agendados (envio automático)

**Lembrete de documento obrigatório** — todo dia 01, 05, 10 e 15 de cada mês, para clientes com "documento obrigatório" = negativo: enviar mensagem solicitando o envio dos documentos. Se "documento obrigatório" for positivo ou não se aplica, não envia.

**Envio de guia de imposto (DAS)** — agendado nos mesmos dias (01/05/10/15), respeitando a "forma de envio" cadastrada:

- Só tenta enviar quando DAS = pronto.
- Em caso de sucesso: DAS → enviado, e "data de envio" recebe a data do envio.
- Se DAS estiver pendente (estado padrão) e "documento obrigatório" for positivo, envia um aviso ao setor Simples Nacional informando que ainda há clientes com DAS não pronto.
- Se "documento obrigatório" for negativo ou não se aplica, nenhum aviso é enviado (nem ao cliente, nem ao setor) — mesma regra se DAS estiver como "sem movimento".

**Envio de guia de parcelamento** — agendado nos dias 20/22/24/26/28, para clientes com "tem parcelamento" = positivo:

- Só tenta enviar quando parcelamento = pendente.
- Em caso de sucesso: parcelamento → enviado.
- Se "tem parcelamento" for negativo, ou for positivo mas parcelamento estiver marcado como "interno", nada é enviado ao cliente.

Em ambos os envios (DAS e parcelamento), usa as integrações da seção 5 para obter o PDF, seguindo a "forma de envio" cadastrada para cada cliente.

## 10. Dashboard do setor Simples Nacional

Cards de resumo do cadastro de clientes:

- **DAS emitidas** — DAS = enviada OU sem movimento.
- **Sem documento** — documento obrigatório = negativo.
- **DAS pendente** — documento obrigatório = positivo E DAS = pendente.
- **Sem movimento** — DAS = sem movimento.
- **Esperando envio DAS** — documento obrigatório = positivo E DAS = pronto.
- **Com parcelamento** — tem parcelamento = positivo.
- **Parcelamento pendente** — tem parcelamento = positivo E parcelamento = pendente.
- **Parcelamento interno** — tem parcelamento = positivo E parcelamento = interno.
- **Parcelamento enviado** — tem parcelamento = positivo E parcelamento = enviado.

Cada card tem seu próprio gráfico (tipo de gráfico a escolher conforme as diretrizes de design mais adequadas a cada assunto).

Cada card é clicável e leva a uma lista de clientes com a respectiva classificação. Cada linha da lista mostra: CNPJ, nome da empresa, os campos usados para essa classificação, um campo de observações gerais (editável em caixa de texto) e um campo de ação para o funcionário trocar/atualizar o valor do campo classificador.

No topo do dashboard, uma busca inteligente por CNPJ ou nome da empresa (sugestões a partir de 3 caracteres digitados, buscando no cadastro persistido). Ao escolher uma empresa, mostra as informações do cliente com os campos opcionais disponíveis para edição.

Em nenhuma dessas telas (lista por card ou busca) os campos de contrato social/alterações são exibidos.

## 11. Auditoria

Todas as conversas são armazenadas, formando histórico completo, com rastreabilidade e auditoria.

Todas as ações do sistema são registradas e persistidas para compor uma aba de auditoria, permitindo consultar: ações num período, ações de um contato específico, ações de um setor específico, ou ações de um funcionário específico num período — como um "protocolo" de tudo que foi feito no sistema.

## 12. Mensageria (WhatsApp)

- Respostas da URA **dentro de uma conversa ativa** (dentro da janela de sessão do WhatsApp) usam template próprio do sistema, não o template pago do WhatsApp Business — para reduzir custo de mensageria.
- Os disparos proativos da seção 9 (lembretes automáticos, iniciados pela empresa fora de uma conversa ativa) precisam usar template aprovado do WhatsApp Business — exigência da política do Meta/WhatsApp para mensagens iniciadas pela empresa fora da janela de 24h de sessão.

## 13. Requisitos não-funcionais

- Disponibilidade: 95%.
- Usuários simultâneos: 25.000 (pico de 30.000).
- Vazão esperada: 1.000 rps (pico de 3.000 rps).

⚠️ Ver nota sobre esses números na seção "Notas / pontos em aberto".

## 14. Design

Cada setor tem sua própria aba, com uma tela principal em formato de dashboard específico do setor.

Paleta de cores deve transmitir profissionalismo, segurança/confiança e satisfação — aplicação: Rigoni Contábil (escritório de contabilidade).

## 15. Referências

Documentos e arquivos de referência estão em `docs` e `docs/referencia` (inclui `organograma.pdf`, a planilha `CONTROLE SIMPLES NACIONAL`, e exemplos de contrato social/alterações contratuais).

## 16. Regra geral de execução

Antes de qualquer documentação ou implementação, ler sempre `docs/testes.md` e `docs/arquitetura.md` e tratar as regras lá contidas como gerais, a serem seguidas irrestritamente.

---

## Notas / pontos em aberto (propositalmente não resolvidos aqui — para o wayfinder perguntar)

- Escopo do backbone de atendimento (seção 0): multi-setor completo já nesta rodada, ou restrito ao Simples Nacional?
- "Documento obrigatório = não se aplica": não tem bucket próprio no dashboard (seção 10) — em qual card esse cliente deveria aparecer, se em algum?
- Tratamento de falha nos envios automáticos (seção 9): o texto só descreve o caminho de sucesso. O que acontece quando o envio falha (erro de API do Serpro, falha no envio do WhatsApp)? Fica pendente pro próximo ciclo, gera alerta, tenta de novo?
- Seção 6, cliente que afirma "já é cliente" mas informa um CNPJ que não bate com nenhum cadastro: o que acontece (pergunta de novo, transfere pra recepção, outro caminho)?
- Seção 6, item de fallback: como detectar "algo como 'quero falar com atendente'" — lista de palavras-chave, ou classificação mais flexível?
- Seção 13 (NFRs): os números de usuários simultâneos e RPS parecem altos para uma ferramenta interna de um único setor de um escritório de contabilidade — vale confirmar se são reais antes de travarem decisões de arquitetura.
- Seção 3: tipo/valores da coluna "Regime" e da coluna "Forma de envio" ainda não definidos — e confirmar se contrato social/alterações ficam numa coluna só ou em duas.
