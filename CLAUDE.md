@AGENTS.md

## Agent skills

### Ordem de construção (arquitetura)

Construir sempre nesta ordem: domínio → aplicação → apresentação →
infraestrutura. Infra é sempre a última camada tocada — nunca verificar ou
configurar Supabase, Docker ou outras dependências externas antes de
domínio/aplicação/apresentação existirem. Ver `docs/arquitetura.md` e
`docs/adr/0001-ordem-de-construcao-clean-architecture.md` caso existam.
