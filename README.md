# 🥝 KiwiTrack NZ (Showcase)
**PWA de Rastreio de Turnos, Impostos e Vistos para Trabalhadores na Nova Zelândia**

> ⚠️ **Nota:** Este é um repositório de portfólio (Showcase). Como o KiwiTrack possui um modelo de monetização (SaaS B2C) focado no mercado neozelandês, o código-fonte original é mantido privado. Abaixo, detalho a lógica de negócios, arquitetura e cálculos fiscais implementados. Estou disponível para demonstrar o código em entrevistas técnicas.

---

## 📖 Resumo Executivo
O KiwiTrack é um Progressive Web App (PWA) projetado especificamente para trabalhadores horistas e imigrantes na Nova Zelândia (ex: setores de hospitalidade, limpeza, construção). 

Seu objetivo principal é rastrear turnos de trabalho, calcular automaticamente o salário líquido de acordo com as complexas leis fiscais da Nova Zelândia e garantir a conformidade com as regras de imigração e vistos.

## 💻 Stack Tecnológico
- **Front-end:** Next.js
- **Interface:** Mobile-first PWA (experiência de app nativo) com Dark Mode UI.
- **Back-end & Banco de Dados:** Supabase / PostgreSQL.
- **Monetização:** Fluxo de Trial de 7 dias integrado.

## 🏗️ Lógica de Negócios e Funcionalidades Core

### 1. Motor de Folha de Pagamento e Impostos (NZ Tax Engine)
O "cérebro" do app replica a matemática fiscal do governo neozelandês:
- Calcula o Salário Bruto (Gross Pay) baseado na taxa por hora.
- Dedução automática do imposto de renda padrão da NZ (**PAYE** - Pay As You Earn).
- Dedução automática da taxa de acidentes (**ACC** levy).
- Cálculo e separação dos **8% de Holiday Pay** (padrão para trabalhadores casuais/part-time).
- Cálculo final exato do Salário Líquido (Net Pay).

### 2. Hub de Imigração e Alertas de Visto
- Hub centralizado com diretrizes oficiais de emprego e vistos da Nova Zelândia.
- **Sistema de Alerta para WHV (Working Holiday Visa):** Lógica específica que rastreia a duração do emprego em uma mesma empresa e emite alertas antes que o usuário exceda o limite legal de 90 dias de trabalho por empregador.

### 3. Registro de Turnos (Dual Method)
- **Entrada Detalhada:** O usuário insere Hora de Início, Hora de Término e duração do intervalo não remunerado.
- **Entrada Rápida:** O usuário insere diretamente o "Total de Horas" trabalhadas.

### 4. Relatórios e Auditoria de Holerites
Um dashboard de extratos dedicado que agrega os ganhos semanais/mensais, permitindo que os usuários auditem os holerites oficiais (payslips) gerados por seus empregadores para evitar fraudes ou erros contábeis.

---

## 📸 Demonstração do Produto
*

https://github.com/user-attachments/assets/f00af34b-68f7-4e0f-b2db-c400aedeff6a

*

---
**Desenvolvido por:** [Patrick Fiuza Alves](https://github.com/codetrick95)
