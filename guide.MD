# Prompt para IA: Implementar Sistema de Streaks com Supabase

## 📋 CONTEXTO DO PROJETO

Estamos desenvolvendo um **Web App de Estudo Gamificado** para universitários moçambicanos. O app é offline-first (PWA), com flashcards, simulados, fóruns, e um **sistema de Streaks** (contador de dias consecutivos de estudo).

O projeto usa:
- **Frontend:** React 18 + TypeScript + Tailwind CSS
- **Backend:** Supabase (PostgreSQL gerenciado + Auth + Realtime)
- **Database:** PostgreSQL com Row Level Security (RLS)

---

## 🎯 TAREFA: Implementar o Sistema de Streaks Completo

Você deve **implementar APENAS** o sistema de streaks (não todo o app). Aqui está o que precisa fazer:

---

## 📐 ESTRUTURA A IMPLEMENTAR

### **1. SQL Schema (PostgreSQL/Supabase)**

Crie um arquivo `schema.sql` com:

- ✅ Tabela `user_streaks` (streak atual + longest + freeze tokens)
- ✅ Tabela `streak_history` (histórico diário)
- ✅ Tabela `streak_badges` (catálogo de badges)
- ✅ Tabela `user_badges` (badges conquistadas)
- ✅ Tabela `streak_notifications` (alertas)
- ✅ VIEW `streak_leaderboard` (ranking top 50)
- ✅ RLS Policies (segurança por usuário)
- ✅ Índices para performance

**Requisitos:**
- Cada tabela tem `created_at` e `updated_at`
- RLS garante que usuário A não vê dados do usuário B
- Leaderboard e badges são públicos
- UNIQUE constraints apropriados

**Arquivo de saída:** `schema.sql`

---

### **2. PostgreSQL Functions (RPC)**

Crie um arquivo `functions.sql` com:

#### **a) register_study_activity(p_user_id, p_minutes, p_activities)**
- Registra quando usuário estuda (flashcard, simulado, etc.)
- Calcula diferença de dias: `today - last_study_date`
  - Se 0 dias: já estudou hoje, não incrementa
  - Se 1 dia: incrementa streak em 1
  - Se >1 dias: quebra streak, volta a 1
- Insere/atualiza `streak_history`
- Checa badges conquistados
- Notifica milestones (5, 7, 10, 14, 21, 30, 50, 100, 365)
- Retorna JSON: `{ success: true, currentStreak, longestStreak, daysDiff }`

#### **b) freeze_streak(p_user_id)**
- Congela streak por 1 dia (não deixa quebrar se não estudar)
- Verifica se usou limite de freezes (máximo 2/mês)
- Se já usou 2: retorna erro
- Se <2: incrementa `freeze_token`, atualiza `last_freeze_date`
- Notifica usuário
- Retorna JSON: `{ success: true/false, message }`

#### **c) check_and_award_badges(p_user_id, p_streak_value)**
- Busca badges não conquistados com `required_streak_days <= p_streak_value`
- Insere cada badge novo em `user_badges`
- Notifica cada badge conquistado

#### **d) notify_milestone(p_user_id, p_streak_value)**
- Insere notificação: "🎉 Parabéns! X dias de estudo consecutivo!"

#### **e) notify_freeze_used(p_user_id)**
- Insere notificação: "Streak congelado por 1 dia"

#### **f) notify_badge_earned(p_user_id, p_badge_name)**
- Insere notificação: "🏆 Conquistou badge 'X'!"

#### **g) is_milestone(p_streak INT) RETURNS BOOLEAN**
- Retorna true se streak é 5, 7, 10, 14, 21, 30, 50, 100, 365
- Retorna false caso contrário

**Arquivo de saída:** `functions.sql`

---

### **3. Edge Function (Cron Job)**

Crie um arquivo `edge-function-daily-reset.ts` que:

- Roda todo dia à meia-noite (00:00 UTC)
- **1º de cada mês:** reseta `freeze_token = 0` para todos
- **Diariamente:** busca usuários com `last_study_date < today` e `current_streak > 0`
- Para cada usuário:
  - Quebra streak: `current_streak = 0` (a menos que tenha freeze)
  - Insere notificação: "😔 Seu streak de X dias foi quebrado"
  - Decrementa `freeze_token` se usou

**Arquivo de saída:** `daily-streak-reset.ts`

---

### **4. React Frontend (Hooks + Componentes)**

Crie um arquivo `useStreak.ts` com:

#### **Hook: useStreak()**
- Busca dados de streak do usuário via Supabase
- Realtime subscription (atualiza se outro cliente modifica)
- Função `registerStudy(minutes, activities)`: chama RPC `register_study_activity`
- Função `freezeStreak()`: chama RPC `freeze_streak`
- Função `refetch()`: recarrega dados do servidor
- Retorna: `{ streak, loading, error, registerStudy, freezeStreak, refetch }`

**Requisitos:**
- TypeScript com tipos completos
- Error handling robusto
- Cache local (não chamar RPC a cada render)
- Realtime updates via `supabase.from().on()`

**Arquivo de saída:** `useStreak.ts`

---

### **5. Componentes React (UI)**

Crie arquivo `StreakWidget.tsx` com componentes:

#### **<StreakWidget />**
- Props: `streak: number`, `longest: number`, `showAnimation: boolean`
- Mostra contador grande (CSS animado quando incrementa)
- Mostra melhor resultado
- Opcional: ícone de fogo (🔥)

#### **<BadgesPanel />**
- Props: `badges: Badge[]`, `streakValue: number`
- Lista badges conquistados (com ícone/imagem)
- Mostra próximos badges (quantos dias faltam)

#### **<LeaderboardView />**
- Props: `leaderboard: LeaderboardEntry[]`, `userRank: number`
- Tabela com top 50 usuários
- Mostra posição do usuário atual (highlight)
- Colunas: rank, username, current_streak, longest_streak

#### **<FreezeButton />**
- Props: `freezeCount: number`, `maxFreeze: number`, `onFreeze: () => void`
- Botão desabilitado se freeze_count >= maxFreeze
- Mostra quantos freezes restam
- Loading state enquanto processa

#### **<NotificationCenter />**
- Props: `notifications: Notification[]`
- Lista notificações recentes
- Botão X para fechar cada uma
- Ícones diferentes por tipo (milestone, broken, badge)

**Arquivo de saída:** `StreakWidget.tsx`

---

### **6. Hooks Adicionais**

Crie arquivo `useLeaderboard.ts` com:

#### **Hook: useLeaderboard(limit = 50)**
- Busca leaderboard via query Supabase
- Realtime subscription (atualiza quando alguém estuda)
- Função `getUserPosition(userId)`: retorna rank e percentile
- Retorna: `{ leaderboard, loading, error, userPosition }`

**Arquivo de saída:** `useLeaderboard.ts`

---

### **7. Types/Interfaces (TypeScript)**

Crie arquivo `types.ts` com interfaces:

```typescript
interface UserStreak {
  id: string;
  user_id: string;
  current_streak: number;
  longest_streak: number;
  last_study_date: string | null;
  freeze_token: number;
  max_freeze_per_month: number;
  last_freeze_date: string | null;
  created_at: string;
  updated_at: string;
}

interface StreakBadge {
  id: number;
  name: string;
  description: string;
  icon_url: string;
  required_streak_days: number;
  reward_points: number;
}

interface UserBadge {
  id: string;
  user_id: string;
  badge_id: number;
  earned_at: string;
}

interface StreakNotification {
  id: string;
  user_id: string;
  notification_type: 'milestone' | 'broken' | 'badge' | 'freeze_used';
  message: string;
  streak_value: number | null;
  read: boolean;
  created_at: string;
}

interface LeaderboardEntry {
  user_id: string;
  username: string;
  current_streak: number;
  longest_streak: number;
  rank_current: number;
  rank_longest: number;
}

interface UserPosition {
  rank: number;
  totalParticipants: number;
  percentile: number;
}
```

**Arquivo de saída:** `types.ts`

---

### **8. Exemplo de Uso**

Crie arquivo `example-usage.tsx` com um exemplo completo de como usar:

```typescript
// Exemplo: Dashboard com todos os componentes
function StreakDashboard() {
  const { streak, registerStudy, freezeStreak } = useStreak();
  const { leaderboard, userPosition } = useLeaderboard(50);
  
  return (
    <div>
      <StreakWidget streak={streak?.current_streak} longest={streak?.longest_streak} />
      <BadgesPanel streakValue={streak?.current_streak} />
      <FreezeButton freezeCount={streak?.freeze_token} onFreeze={freezeStreak} />
      <LeaderboardView leaderboard={leaderboard} userRank={userPosition?.rank} />
    </div>
  );
}
```

**Arquivo de saída:** `example-usage.tsx`

---

## 📝 REQUISITOS TÉCNICOS

### **Supabase Setup**
- Assumir que projeto Supabase já existe
- Usar `SUPABASE_URL` e `VITE_SUPABASE_ANON_KEY` do `.env`
- Chamar funções RPC via `supabase.rpc()`
- Usar realtime subscriptions via `supabase.from().on()`

### **Segurança (RLS)**
- ✅ Usuário A nunca consegue ver dados do usuário B
- ✅ Serviço pode usar `SECURITY DEFINER` para operações privilegiadas
- ✅ Leaderboard é público (VIEW acessível por todos)

### **Performance**
- ✅ Índices em colunas frequentemente consultadas
- ✅ Cache local no React (não refetch a cada render)
- ✅ Realtime updates em vez de polling
- ✅ Limitar leaderboard a top 100

### **TypeScript**
- ✅ Tipos completos em tudo (sem `any`)
- ✅ Validar responses do Supabase
- ✅ Error handling tipado

### **Testes**
- ✅ Testes unitários para funções PostgreSQL (via SQL asserts)
- ✅ Testes do hook `useStreak` (mock Supabase)
- ✅ Exemplo de teste: "registrar estudo incrementa streak"

---

## 🎨 DESIGN/ESTILO

- ✅ Use **Tailwind CSS** para styling
- ✅ Componentes responsivos (mobile-first)
- ✅ Animações suaves (CSS transitions)
- ✅ Dark mode support (opcional mas bom)
- ✅ Loading states e skeletons

---

## 📦 ESTRUTURA DE ARQUIVOS ESPERADA

```
streaks-system/
├── sql/
│   ├── schema.sql          # Tabelas + RLS + índices
│   └── functions.sql       # Funções RPC
├── edge-functions/
│   └── daily-streak-reset/
│       └── index.ts        # Cron job
├── src/
│   ├── hooks/
│   │   ├── useStreak.ts    # Hook principal
│   │   └── useLeaderboard.ts
│   ├── components/
│   │   └── StreakWidget.tsx # Todos os componentes
│   ├── types/
│   │   └── types.ts        # Interfaces TypeScript
│   └── example/
│       └── example-usage.tsx # Exemplo de uso
├── tests/
│   ├── useStreak.test.ts
│   └── functions.test.sql
├── README.md               # Documentação de implementação
└── .env.example           # Variáveis necessárias
```

---

## 📖 DOCUMENTAÇÃO ESPERADA

Crie um **README.md** que explique:

1. **Como setup Supabase** (passo a passo)
2. **Como executar o SQL schema**
3. **Como deployar Edge Functions**
4. **Como usar cada hook**
5. **Como usar cada componente**
6. **Exemplo completo (dashboard)**
7. **Troubleshooting**

---

## ⚠️ COISAS IMPORTANTES

### **NÃO faça:**
- ❌ Não crie backend Node.js/Express (Supabase RPC é suficiente)
- ❌ Não use banco de dados adicional (só PostgreSQL Supabase)
- ❌ Não ignores RLS (segurança é crítica)
- ❌ Não crie tipos `any` (TypeScript completo)
- ❌ Não chame `register_study_activity` multiple vezes no mesmo dia (custa)

### **SIM faça:**
- ✅ Use Supabase client library official
- ✅ Implemente RLS policies corretamente
- ✅ Trate erros graciosamente
- ✅ Valide inputs (frontend + backend)
- ✅ Cache dados quando apropriado
- ✅ Use realtime para updates ao vivo

---

## 🎯 CRITÉRIOS DE SUCESSO

Quando terminar, o código deve:

- ✅ Compilar sem erros TypeScript
- ✅ RLS policies permitem apenas operações seguras
- ✅ Streak incrementa quando estuda (1 dia depois)
- ✅ Streak quebra se passar 2+ dias sem estudar
- ✅ Freeze congela streak por 1 dia (máximo 2/mês)
- ✅ Badges são conquistados automaticamente nos milestones
- ✅ Leaderboard mostra top 50 e posição do usuário
- ✅ Notificações são enviadas para eventos importantes
- ✅ Componentes renderizam corretamente com dados reais
- ✅ Realtime updates funcionam (múltiplos clientes)
- ✅ Performance é boa (sem N+1 queries)

---

## 📞 DÚVIDAS ESPERADAS

**P: Como faço para testar localmente?**
R: Use `supabase start` (CLI local), depois execute SQL schema e functions, depois rode testes.

**P: Preciso de Node.js backend?**
R: Não! Supabase RPC (funções PostgreSQL) fazem tudo que precisa.

**P: Como faço realtime?**
R: Use `supabase.from('tabela').on('*', callback).subscribe()`

**P: Posso usar Redux/Zustand?**
R: Não precisa, hook `useStreak` é suficiente (ou use Zustand se quiser cache global).

**P: E autenticação?**
R: Use `supabase.auth` (já existe, integra com RLS automaticamente).

---

## 🚀 ENTREGA FINAL

Quando terminar, você deve ter:

1. ✅ SQL schema pronto para executar
2. ✅ PostgreSQL functions (RPC) completas
3. ✅ Edge Function (cron job) deployável
4. ✅ React hooks e componentes funcionais
5. ✅ TypeScript types completos
6. ✅ Testes (unitários + integração)
7. ✅ README com instruções
8. ✅ Exemplo de uso (dashboard)

---

## 🎓 RESUMO RÁPIDO

- **O quê:** Sistema de Streaks (contador de dias consecutivos)
- **Onde:** Supabase (PostgreSQL + Auth + Realtime)
- **Como:** RPC functions + React hooks + Realtime subscriptions
- **Saída:** SQL + TypeScript + React componentes
- **Tempo estimado:** 4-6 horas (dependendo da experiência)

---

**BOA SORTE! 🚀 Qualquer dúvida, consulte a arquitetura em `estrutura_streaks_supabase.md`**
