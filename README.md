
## 📸 Demonstração do Produto


https://github.com/user-attachments/assets/8cc0c474-010e-465d-8bd1-1cbf54281e47

# KiwiTrack — Documentação Completa do Projeto

---

## 1. Visão Geral

**KiwiTrack** é uma PWA (Progressive Web App) de rastreamento de horas de trabalho voltada para trabalhadores por hora na **Nova Zelândia** (especialmente detentores de Working Holiday Visa). O app roda como um "app nativo" instalado no celular via iOS Safari.

**Público-alvo:** Brasileiros, portugueses, espanhóis e ingleses que trabalham na NZ em funções como Cleaner, Builder, Kitchen Hand, e similares.

**Problema que resolve:**
- Controle preciso de horas para evitar "wage theft"
- Cálculo automático de Holiday Pay (8%) e feriados (1.5x)
- Alertas de limite de 90 dias de vínculo com mesmo empregador (WHV)
- Exportação de extrato em PDF para enviar ao empregador

---

## 2. Stack Tecnológica

| Tecnologia | Versão | Uso |
|---|---|---|
| Next.js | 16.2.4 | Framework principal (App Router) |
| React | 19.2.4 | UI |
| TypeScript | ^5 | Tipagem |
| Tailwind CSS | ^4 | Estilização (via PostCSS) |
| Supabase | ^2.104 | Auth + Database (PostgreSQL) |
| @supabase/ssr | ^0.10.2 | Integração SSR/Middleware |
| jsPDF | ^4.2.1 | Geração de PDF |
| jspdf-autotable | ^5.0.7 | Tabelas no PDF |

**Rodar localmente:**
```bash
cd kiwi-track
npm run dev   # http://localhost:3000
```

---

## 3. Estrutura de Arquivos

```
kiwi-track/
├── src/
│   ├── app/
│   │   ├── layout.tsx              # Root layout (PWA metadata, viewport)
│   │   ├── page.tsx                # Landing Page (pública)
│   │   ├── globals.css             # Tema iOS + Tailwind v4
│   │   ├── (auth)/
│   │   │   └── login/page.tsx      # Login + Signup (mesmo componente)
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx          # Tab bar inferior (Início/Relatórios/Perfil)
│   │   │   ├── dashboard/page.tsx  # Dashboard principal
│   │   │   ├── reports/page.tsx    # Extrato + exportação PDF
│   │   │   └── settings/page.tsx   # Perfil + gestão de empresas
│   │   └── auth/
│   │       └── callback/route.ts   # Handler OAuth callback Supabase
│   ├── components/
│   │   ├── QuickEntryModal.tsx     # Modal de lançamento de turno
│   │   └── LanguageSwitcher.tsx    # Seletor de idioma (UI apenas)
│   ├── lib/
│   │   ├── payroll.ts              # Cálculos financeiros NZ
│   │   ├── shift.ts                # Cálculos de turno e pausas
│   │   └── week.ts                 # Range da semana atual (Seg–Dom)
│   ├── constants/
│   │   └── nz-employment.ts        # Constantes legais NZ
│   ├── types/
│   │   └── domain.ts               # Interfaces TypeScript (Company, WorkSession)
│   ├── utils/
│   │   └── supabase/
│   │       ├── client.ts           # Supabase browser client
│   │       ├── server.ts           # Supabase server client
│   │       └── middleware.ts       # Supabase middleware client
│   └── middleware.ts               # Proteção de rotas Next.js
├── database_schema.sql             # Schema PostgreSQL completo
├── .env.local                      # Variáveis de ambiente (não commitar)
└── package.json
```

---

## 4. Design System

### Filosofia
O design imita o **iOS nativo** — glassmorphism, bordas arredondadas tipo squircle, cores Apple, e animações suaves. O app deve parecer um app nativo instalado, não um site.

### Cores Principais
| Nome | Hex | Uso |
|---|---|---|
| iOS Blue | `#007AFF` | Ações primárias, CTAs, ativo |
| iOS Green | `#34C759` | Ganhos, sucesso, holiday pay pago |
| iOS Purple | `#AF52DE` | Feriados públicos |
| iOS Orange | `#FF9500` | Alertas, holiday pay acumulado (cofre) |
| iOS Red | `#FF3B30` | Erros, exclusão, limite WHV |
| Background Dark | `#09090b` | Fundo principal (dark mode) |
| Surface Dark | `#18181b` | Cards, headers |
| Text Primary | `#FAFAFA` | Texto principal |
| Text Secondary | `#A1A1AA` | Subtítulos, labels |
| Text Muted | `#71717A` | Labels uppercase, metadados |

### Layout
- **Max width:** `max-w-md` (mobile-first, centrado em telas maiores)
- **Border radius padrão:** `rounded-[32px]` para cards, `rounded-[24px]` para sub-cards
- **Fonte:** Geist Sans (carregada via `next/font/google`)
- **Safe areas:** `pb-safe`, `pt-safe` para notch/home bar do iPhone

### Regras de UI
1. Nunca usar emojis de forma amadora em headers principais
2. Labels de seção: sempre `uppercase tracking-widest text-[12px] text-[#71717A]`
3. Botões primários: gradiente `from-[#007AFF] to-[#0056D6]` com shadow azul
4. Cards: `bg-[#18181b]` com `border border-white/5`
5. Inputs: `bg-[#09090b] border border-white/10` com `focus:border-[#007AFF]`
6. Sempre adicionar `active:scale-[0.98] transition-all` em botões clicáveis

---

## 5. Banco de Dados (Supabase / PostgreSQL)

### Tabelas

#### `public.users`
Espelha `auth.users`. Criada automaticamente via trigger ao registrar.
```sql
id uuid PRIMARY KEY (= auth.users.id)
email text NOT NULL
full_name text
stripe_customer_id text     -- futuro (monetização)
subscription_status text    -- futuro
created_at timestamptz
```

#### `public.companies`
Cada "empresa" representa um vínculo empregatício do usuário.
```sql
id uuid PRIMARY KEY
user_id uuid → public.users(id)
name text NOT NULL                          -- Ex: "Cleaner", "Builder Co"
hourly_rate numeric NOT NULL               -- Taxa horária em NZD
tracking_method text CHECK IN ('continuous','bulk')
pays_holiday_pay_weekly boolean DEFAULT true  -- PAYG vs. Accrued
is_archived boolean DEFAULT false
```

**`tracking_method`:**
- `continuous` — usuário registra entrada/saída ao longo do dia
- `bulk` — usuário lança blocos ou totais de horas de uma vez

**`pays_holiday_pay_weekly` (CRÍTICO):**
- `true` = Holiday Pay é pago toda semana junto com o salário (PAYG). O bruto já inclui os 8%.
- `false` = Holiday Pay fica "guardado" (accrued) com o empregador para ser pago no fim do contrato ou férias.

#### `public.work_sessions`
Cada registro de dia/turno trabalhado.
```sql
id uuid PRIMARY KEY
user_id uuid → public.users(id)
company_id uuid → public.companies(id)
work_date date NOT NULL
start_time time                             -- opcional (modo continuous)
end_time time                              -- opcional (modo continuous)
unpaid_break_minutes integer DEFAULT 0
is_public_holiday boolean DEFAULT false
total_hours_worked numeric NOT NULL        -- horas já descontada a pausa
gross_pay_estimated numeric NOT NULL       -- valor bruto já com 8% se aplicável
notes text                                 -- Ex: "PH 1.5x Break 30m"
```

### Row Level Security (RLS)
**REGRA ABSOLUTA:** Todo dado é isolado por `user_id = auth.uid()`. Nenhum usuário pode ver dados de outro.
- `users`: select e update apenas do próprio perfil
- `companies`: todas as operações apenas para `user_id = auth.uid()`
- `work_sessions`: todas as operações para `user_id = auth.uid()` AND `company_id` deve pertencer ao mesmo usuário

---

## 6. Autenticação

### Fluxo
1. Usuário acessa `/login`
2. Pode fazer **Login** (email + senha) ou **Criar Conta** (nome + email + senha)
3. No signup: se `data.session === null`, Supabase enviou e-mail de verificação → mostra tela "Verifique seu e-mail"
4. Ao clicar no link do e-mail → redireciona para `/auth/callback?code=...` → troca o code por sessão → redireciona para `/dashboard`

### Middleware de Proteção (`src/middleware.ts`)
```
REGRA 1: Se URL tem ?code= → deixar passar (fluxo de callback OAuth)
REGRA 2: Rotas protegidas (/dashboard, /reports, /settings) sem sessão → redireciona para /login
REGRA 3: Usuário logado tentando acessar / ou /login → redireciona para /dashboard
```

**Matcher:** Exclui `_next/static`, `_next/image`, `favicon.ico`, `icons/`, `manifest.json` e arquivos de imagem estáticos.

---

## 7. Regras de Negócio (CRÍTICO)

### 7.1 Cálculo de Horas Pagas
```
paidHours = totalHours - (hasUnpaidBreak && totalHours > 0.5 ? 0.5 : 0)
```
A pausa deduzida é sempre **30 minutos fixos** (switch binário).

### 7.2 Cálculo de Pagamento Base
```
effectiveRate = isPublicHoliday ? hourlyRate × 1.5 : hourlyRate
basePay = paidHours × effectiveRate
```

### 7.3 Holiday Pay (8%)
Aplicado **sempre** sobre o `basePay` (incluindo o multiplicador de feriado):
```
holidayPay = basePay × 0.08
grossPay = basePay + holidayPay
```

### 7.4 Modo PAYG vs. Accrued (por empresa)
- **PAYG (`pays_holiday_pay_weekly = true`):** `gross_pay_estimated` já inclui os 8%. O total a receber na semana = `gross_pay_estimated`.
- **Accrued (`pays_holiday_pay_weekly = false`):** `gross_pay_estimated` ainda inclui os 8%, mas na tela de relatórios é **separado**:
  - `basePay` = horas × taxa = o que vai para a conta
  - `holidayPay` = os 8% ficam guardados no "cofre" com o empregador

### 7.5 Cálculo de Impostos (Estimativa — Relatórios)
Calculado apenas sobre `totalTaxableGross` (o que vai efetivamente para a conta):
```
accLevy = totalTaxableGross × 0.016         (ACC Earner Levy)

annualProj = totalTaxableGross × 52
if annualProj <= 15600: incomeTax = totalTaxableGross × 0.105
elif annualProj <= 53500: incomeTax = totalTaxableGross × 0.175
else: incomeTax = totalTaxableGross × 0.30

totalTax = incomeTax + accLevy
netPay = totalTaxableGross - totalTax
```

### 7.6 Salário Mínimo NZ
- Vigente desde **1 de Abril de 2026:** `NZD $23.95/h`
- Se o usuário cadastrar uma empresa com taxa abaixo desse valor → exibe modal de aviso (mas permite salvar)

### 7.7 Working Holiday Visa (WHV) — Alerta de 90 dias
- A partir do primeiro turno registrado em cada empresa, o app conta os dias corridos de vínculo
- **70+ dias:** alerta laranja "Aproximando-se do limite"
- **90+ dias:** alerta vermelho "Limite excedido"
- **AVISO LEGAL:** O app não substitui aconselhamento de imigração. Os avisos são lembretes genéricos.

### 7.8 Semana Laboral
- A semana vai de **Segunda-feira a Domingo**
- Definido em `src/lib/week.ts` via `getCurrentWeekRange()`

---

## 8. Páginas e Componentes

### 8.1 Landing Page (`/`) — `src/app/page.tsx`
**Pública.** Usuários logados são redirecionados para `/dashboard` pelo middleware.

**Estrutura:**
- Header sticky com glassmorphism (iOS): logo "KiwiTrack" + `LanguageSwitcher` + botão "Entrar"
- Hero section: H1 com "Assuma o controle das suas **horas** na Nova Zelândia" + CTA "Começar meu Teste Grátis"
- 3 cards de benefícios (grid 1→3 colunas):
  1. 🛡️ Proteção contra Wage Theft (azul)
  2. 💰 Feriados e Holiday Pay (verde)
  3. 📤 Relatórios à Prova de Balas (roxo)

**Problemas identificados na landing:**
- O botão "Entrar" no header está **hardcoded em português** ("Entrar") — o `LanguageSwitcher` existe mas **não traduz nada** (apenas muda o estado local visualmente)
- O CTA "Começar meu Teste Grátis" aponta para `/login` (correto)
- Não existe seção de preços, FAQ ou social proof
- Falta footer com links legais

### 8.2 Login (`/login`) — `src/app/(auth)/login/page.tsx`
**Pública.** Usuários logados são redirecionados pelo middleware.

- Toggle entre modo Login e Signup no mesmo componente
- Signup: captura `full_name` extra para `user_metadata`
- Tela de "Verifique seu e-mail" quando Supabase exige confirmação
- Toast de notificação de sucesso/erro

### 8.3 Dashboard (`/dashboard`) — `src/app/(dashboard)/dashboard/page.tsx`
**Protegida.**

- Exibe saudação com primeiro nome do usuário
- Card de ganhos da semana (bruto estimado + horas totais)
- Lista de ganhos por empresa na semana atual
- Botão "Lançar Novo Turno" → abre `QuickEntryModal`
- Clique em empresa → navega para `/reports`

### 8.4 Relatórios (`/reports`) — `src/app/(dashboard)/reports/page.tsx`
**Protegida.**

- Toggle Semanal / Mensal com navegação de período (← →)
- Card principal: Net Pay estimado (após impostos)
- Detalhamento: Bruto a Receber + Impostos (PAYE + ACC)
- "Cofre" (accrued): mostra holiday pay guardado com empregador quando `pays_holiday_pay_weekly = false`
- Lista de sessões clicáveis (accordion): mostra breakdown de horas × taxa + holiday pay
- Exclusão de turno individual (com confirmação via `confirm()` nativo)
- **Exportação PDF:** gera timesheet profissional com jsPDF

**Estrutura do PDF gerado:**
- Header: "KIWITRACK / TIMESHEET & EARNINGS SUMMARY"
- Dados: Employee name, Address, Pay Period
- Tabela: DATE | WORK/CLIENT | HOURS | RATE | ORDINARY TIME | HOLIDAY PAY | THIS PAY
- Rodapé da tabela: TAXABLE GROSS
- Tax Summary: PAYE + ACC + NET PAY
- Nota de accrued (se houver)

### 8.5 Perfil/Settings (`/settings`) — `src/app/(dashboard)/settings/page.tsx`
**Protegida.**

**Seções:**
1. **Header:** Avatar com inicial + nome + email + botão "Sair"
2. **Dados do Relatório:** Nome completo + Morada (usados no PDF exportado). Salvo via `supabase.auth.updateUser({ data: { full_name, address } })`
3. **Novo Trabalho:** Form para cadastrar empresa (nome + taxa/h + toggle Holiday Pay PAYG). Valida contra salário mínimo.
4. **Meus Vínculos:** Accordion interativo com:
   - Badge laranja pulsante se há alertas
   - Aviso de taxa abaixo do mínimo legal
   - Contador de dias de vínculo com cores de alerta (WHV)
   - Botão "Apagar Vínculo" com modal de confirmação

### 8.6 QuickEntryModal — `src/components/QuickEntryModal.tsx`
Modal de lançamento de turno. Abre a partir do Dashboard.

**Campos:**
- Seletor de empresa (dropdown com nome e taxa)
- Data (default: hoje)
- Horas Totais (step 0.5)
- Switch: "Tirei 30 min de pausa?" → desconta 0.5h
- Switch: "Feriado Público" → aplica 1.5x
- Preview em tempo real do valor estimado bruto

**Salva em `work_sessions`:**
- `total_hours_worked` = horas já descontada a pausa
- `gross_pay_estimated` = basePay × (1 + 0.08) [sempre com 8%]
- `notes` = string com flags: "PH 1.5x" e/ou "Break 30m"

### 8.7 LanguageSwitcher — `src/components/LanguageSwitcher.tsx`
Dropdown estilo iOS pill com bandeiras para PT 🇧🇷 / EN 🇺🇸 / ES 🇪🇸.

**ESTADO ATUAL:** É apenas **UI decorativa**. Muda o estado local mas **não traduz o app**. O comentário no código diz "Futuramente: disparar função que troca o dicionário inteiro do site".

**Idiomas suportados visualmente (sem implementação):**
- Português (padrão)
- English
- Español

---

## 9. Biblioteca de Cálculos (`src/lib/`)

### `payroll.ts`
```typescript
computePayBreakdown(hours, hourlyRate, { isPublicHoliday, includeHolidayPay }): PayBreakdown
// Retorna: basePay, holidayPayAmount, grossTotal

estimateGrossPay(hours, hourlyRate, includeHolidayPay, isPublicHoliday): number
// Wrapper simplificado

isBelowAdultMinimumWage(hourlyRate): boolean
// Compara com NZ_ADULT_MINIMUM_WAGE_2026_04
```

### `shift.ts`
```typescript
minutesFromMidnight(hhmm: string): number
calculateWorkedFromShift(startHHMM, endHHMM, unpaidBreakMinutes): { spanMinutes, workedMinutes, workedHours }
// Suporta turnos noturnos (ex: 22:00–06:00)

formatMinutesAsHoursAndMinutes(totalMinutes): string   // "7h 30min"
toPostgresTime(hhmm): string                            // "HH:MM" → "HH:MM:00"
describeNzBreakEntitlements(spanMinutes): string        // Orientação sobre pausas legais NZ
```

### `week.ts`
```typescript
getCurrentWeekRange(): { start: string, end: string }
// Retorna Seg–Dom da semana atual em formato "YYYY-MM-DD"
```

---

## 10. Constantes Legais (`src/constants/nz-employment.ts`)

```typescript
NZ_ADULT_MINIMUM_WAGE_2026_04 = 23.95       // NZD/h desde 1 abril 2026
NZ_PUBLIC_HOLIDAY_PAY_MULTIPLIER = 1.5      // Feriado público = 1.5×
NZ_WAGE_RECORDS_RETENTION_YEARS = 6         // Retenção de registros (anos)
```

> **ATENÇÃO para IAs:** Ao atualizar o salário mínimo, altere APENAS esta constante. Ela é importada em `payroll.ts` e `settings/page.tsx`.

---

## 11. Variáveis de Ambiente

Definidas em `.env.local` (não commitar):
```
NEXT_PUBLIC_SUPABASE_URL=https://xxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
```

---

## 12. PWA / Instalação Mobile

### Configuração (`src/app/layout.tsx`)
```typescript
viewport: {
  themeColor: "#09090b",
  width: "device-width",
  initialScale: 1,
  maximumScale: 1,
  userScalable: false,
  viewportFit: "cover",   // ocupa área do notch
}

metadata: {
  manifest: "/manifest.json",
  appleWebApp: {
    capable: true,
    statusBarStyle: "black-translucent",
    title: "KiwiTrack",
  },
  icons: { apple: "/icons/icon-192x192.png" }
}
```

O app é instalável via Safari no iPhone como PWA. O `viewportFit: "cover"` com classes `pb-safe` e `pt-safe` garante que o conteúdo não fique atrás do notch ou da home bar.

---

## 13. Análise da Landing Page

### ✅ O que está bom
- Design iOS limpo e consistente com o resto do app
- Hero copy direto e focado no problema ("Wage Theft", "Holiday Pay")
- 3 cards de benefícios bem estruturados
- Header sticky com glassmorphism correto
- Responsivo (1 coluna mobile → 3 colunas desktop)
- Middleware redireciona corretamente usuários logados

### ⚠️ Problemas e Gaps Identificados
1. **LanguageSwitcher não funciona:** O seletor existe mas não traduz nada. Usuários em EN/ES verão o app todo em PT.
2. **Texto hardcoded "Entrar" no header:** Não muda com o idioma.
3. **Sem footer:** Faltam links de Política de Privacidade, Termos de Uso e contato.
4. **Sem seção de preços:** O CTA diz "Teste Grátis" mas não há contexto de planos.
5. **Sem social proof:** Nenhum depoimento, número de usuários ou credibilidade.
6. **Sem FAQ:** Perguntas frequentes ajudariam a converter visitantes.
7. **SEO básico:** O `<h1>` existe mas não há structured data, og:image, ou descrição por página.
8. **CTA único:** Só um botão "Começar" — poderia ter um "Ver como funciona" secundário.

---

## 14. Regras para IAs ao Modificar o Projeto

### NUNCA fazer:
- Alterar a lógica de cálculo de `gross_pay_estimated` sem entender o modo PAYG vs. Accrued
- Remover o `on conflict (id) do update` do trigger `handle_new_user`
- Contornar o RLS do Supabase para "facilitar" queries
- Usar `supabase.from(...).select()` sem estar dentro de um contexto autenticado
- Quebrar o padrão de cores iOS (não trocar `#007AFF` por outro azul sem alinhamento)
- Usar `useRouter` em Server Components

### SEMPRE fazer:
- Verificar se `user` existe antes de fazer queries ao Supabase
- Usar `createClient()` de `@/utils/supabase/client` em Client Components
- Usar `createClient()` de `@/utils/supabase/server` em Server Components/Route Handlers
- Manter `"use client"` em todos os componentes que usam hooks ou eventos
- Seguir o padrão de toast de notificação (estado `notification` com tipo `success | error`)
- Qualquer nova constante legal NZ deve ir em `src/constants/nz-employment.ts`
- Novos cálculos financeiros devem ir em `src/lib/payroll.ts`

### Padrão de nova página protegida:
```tsx
"use client";
import { useEffect } from "react";
import { createClient } from "@/utils/supabase/client";
import { useRouter } from "next/navigation";

export default function NovaPage() {
  const supabase = createClient();
  const router = useRouter();

  useEffect(() => {
    async function load() {
      const { data: { user } } = await supabase.auth.getUser();
      if (!user) { router.push("/login"); return; }
      // ... lógica
    }
    load();
  }, [router, supabase]);
  // ...
}
```

---

## 15. Roadmap / Funcionalidades Futuras (Identificadas no Código)

- **Internacionalização real:** `LanguageSwitcher` tem comentário "Futuramente: disparar função que troca o dicionário inteiro"
- **Monetização:** Campos `stripe_customer_id` e `subscription_status` na tabela `users`
- **Modo Continuous:** `tracking_method: "continuous"` existe no DB mas a UI de lançamento usa apenas "bulk" (total de horas)
- **Modo Bulk:** `tracking_method: "bulk"` — lançamento por totais, já é o padrão atual
- **Arquivamento de empresas:** Campo `is_archived` existe mas não há UI para arquivar

---

*Gerado automaticamente em 2026-04-22. Atualizar sempre que houver mudanças de regras de negócio, rotas ou schema.*
