# CONTEXT.md — NIT Módulo Efetivo
> Gerado em 28/06/2026 · Máx 200 linhas · Atualizar a cada sessão

---

## Stack
- Frontend: Vanilla JS + HTML + CSS — GitHub Pages (repo: nit-efetivo-logistics)
- DB: Firebase RTDB — mesmo projeto NIT (`nit-operacional-default-rtdb.firebaseio.com`)
- Auth: Firebase Auth Google + whitelist `/efetivo_roles/{emailKey}`
- Backend Fase 2: FastAPI/Python no Railway (relatório mensal, proxy campo)

---

## Entidades Firebase

```
/efetivo/
  recursos/{id}/        nome, matricula, cargo, telefone, bairro, transporte
                        turno_padrao, status, qrToken, obs, updatedAt, updatedBy
  viaturas/{id}/        nome, liderId, membrosIds{}, status
  escalas/{pushKey}/    data, turno, label, horarioInicio, horarioFim, status
                        supervisao{ supervisores{id:{funcao,contato}}, auxiliares{}, motociclistas{}, monitores{} }
                        viaturasEscaladas{ viaturaId: true }
  operacoes/{id}/       escalaId, nome, bairro, horario, ordem
  postos/{id}/          escalaId, operacaoId, numero, local, bairro, horario
                        tipoAcao, alocacao{tipo,id,nome}, qruPessoas, obs
  qr_index/{token}/     recursoId   ← índice público para lookup de QR
  log/{id}/             tipo, recursoId, payload, operadorEmail, timestamp
  templates/operacoes/{id}/   nome, bairro, horarioPadrao, recorrencia, diasSemana[], postosPadrao[]

/efetivo_roles/{emailKey}     "monitor" | "supervisor" | "admin"
```

---

## Terminologia operacional
- **QRU** = posto de trabalho / ocorrência ("Tem algo para mim?")
- **QTH** = localização / endereço ("Onde você está?")
- Cada posto = 1 QRU com N pessoas (`qruPessoas`)
- Viatura exibe QTH = quantidade de QRUs sob sua responsabilidade na escala

---

## Turnos padrão (BRT)
```
Manhã:  05:30–11:30  inicioMin=330  fimMin=690
Tarde:  10:30–16:30  inicioMin=630  fimMin=990
Noite:  15:30–21:30  inicioMin=930  fimMin=1290
Overlap Manhã/Tarde: 10:30–11:30
Overlap Tarde/Noite: 15:30–16:30
Especial: horário livre definido pelo supervisor
```

---

## Perfis de acesso
| Role | Capacidades |
|---|---|
| monitor | Visualizar, abrir turno, adicionar postos/operações/supervisão |
| supervisor | Tudo + cadastrar recursos, viaturas, atestados, encerrar turno |
| admin | Tudo + configurar postos fixos, templates, gerir roles |
| (sem role) | Modo Campo apenas — consulta QTH read-only |

---

## Conexões Firebase
- Plano atual: Spark (100 simultâneas)
- Supervisores/monitores: WebSocket SDK persistente (5–15 slots)
- Agentes Modo Campo: one-time read (não mantém WebSocket aberto)
- Fase 2: agentes roteados via FastAPI/Admin SDK (zero slots)

---

## Modo Campo — fluxo
1. Agente acessa `efetivo.html?modo=campo` ou QR `?modo=campo&t=TOKEN`
2. Sem auth obrigatória — leitura pública de recursos/escalas/postos
3. Busca por nome (mínimo 2 chars) → filtra `State.recursos`
4. Múltiplos resultados → agente confirma pela matrícula
5. Resultado: QTH (endereço), QRU nº, operação, horário, contato supervisor, link Maps
6. Não tem posto designado → mensagem "Aguarde designação do supervisor"

---

## Arquitetura JS (NIT_EFETIVO namespace)
```
NIT_EFETIVO.Auth     — login Google, role, emailToKey
NIT_EFETIVO.DB       — listeners onValue, writers com log
NIT_EFETIVO.State    — store em memória
NIT_EFETIVO.Campo    — busca de QTH (modo campo)
NIT_EFETIVO.UI       — renderEscala, renderRecursos, renderMetricas, renderResultadoCampo, toast
NIT_EFETIVO.Modals   — abrirCriarEscala, abrirAddOperacao, abrirAddPosto, etc.
NIT_EFETIVO.Actions  — mudarStatus, encerrarEscala
NIT_EFETIVO.Log      — write() append-only
```

---

## Funções PROTEGIDAS (não alterar após estabilização)
- `NIT_EFETIVO.Log.write()` — log append-only, nunca deletar entradas
- `DB.db.ref('efetivo/log').push()` — nunca usar set/update/remove no log
- Firebase config — não alterar authDomain/databaseURL sem testar Auth
- `emailToKey()` — padrão de chave deve ser idêntico em todo o sistema

---

## Pendentes Fase 1
- [ ] Geração de QR Code no cadastro de recurso (qrToken + exibição)
- [ ] Listener para operacoes/postos filtrado por escalaId (otimização)
- [ ] Encerramento de turno: resetar status dos recursos escalados → disponivel
- [ ] Templates de operação recorrente (ciclofaixa, posto Parangaba, etc.)
- [ ] Firebase Security Rules completas

## Pendentes Fase 2
- [ ] Endpoint FastAPI `/api/campo/qth?t=TOKEN` (proxy sem conexão SDK)
- [ ] Relatório mensal → xlsx (aggregar postos do mês por turno/bairro)
- [ ] Exportação da escala como PDF printável (substitui WhatsApp+Excel)
- [ ] Integração com Módulo Semáforos (`/efetivo/recursos/` como fonte de equipes)
- [ ] Atestados / licenças / ausências

---

## Histórico
- 28/06/2026: Arquitetura definida. MVP codificado: Auth, Modo Campo,
  Escala (criar/visualizar/adicionar operação+posto+supervisão),
  Banco de Recursos (listar/filtrar/cadastrar/mudar status), Métricas básicas.
