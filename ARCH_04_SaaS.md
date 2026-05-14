# Arquitetura 4: SaaS Soluções Lemos (O Dashboard Central)

Este documento define a arquitetura oficial do aplicativo principal **SaaS Soluções Lemos** (`app.solucoeslemos.com.br/dashboard`). Ele atua como o "Cérebro Operacional" onde os clientes interagem com a inteligência do sistema.

## 1. O Papel do Dashboard no Ecossistema
Enquanto a `ARCH_01` cuida das vendas (site institucional) e o *Worker Fiscal* (ACBr) cuida do processamento pesado, o **Dashboard (ARCH_04_SaaS)** é a interface de gestão diária. Ele consolida relatórios, captura os dados de faturamento, gerencia certificados digitais e exibe os resultados em tempo real.

## 2. Stack Tecnológica do Dashboard
A fundação segue as diretrizes de "Secure-Apps" do KIT 2.1:
- **Core Visual e Lógico:** Next.js 15 (App Router).
- **Interface (UI):** Shadcn/ui + Tailwind CSS (Design Premium, Responsivo e limpo).
- **Gestão de Estado & Cache:** TanStack Query (Para paginação ultra rápida de relatórios e *Optimistic UI* nas interações).
- **Autenticação & Multitenancy:** Supabase Auth com Row Level Security (RLS) severo por *tenant* (empresa).

## 3. Segurança Nível "Pentest" Implementada
Conforme as diretrizes de Cibersegurança (`PENTEST.md`), o Dashboard blinda o cliente e o sistema:

- **Autenticação e Força Bruta:** A tela de acesso possui **Rate Limiting** rigoroso. Bloqueia IPs automaticamente após tentativas repetidas de erro no login.
- **Defesa contra IDOR:** As URLs do dashboard exibem as entidades (ex: `/dashboard/notas/105`), mas o Supabase RLS intercepta no nível do banco. Se o `tenant_id` da nota 105 não pertencer à empresa do usuário logado (`auth.uid()`), o painel retorna "Not Found" ou "Unauthorized".
- **Blindagem de Formulários (Anti-Mass Assignment):** Quando o cliente preenche a tela de "Nova Nota/Venda", os dados vão para uma **Server Action** e passam por validação estrita de um *Schema Zod*. É impossível injetar campos maliciosos (como privilégios de admin) através da interface.
- **Prevenção de Data Exposure:** As rotas de dados (Server Components) retornam apenas as informações necessárias para renderizar a tela, ocultando metadados, tokens e chaves secretas.

## 4. Integração com o Motor Fiscal (O Diferencial de Negócio)
O Dashboard atua como o "Maestro" do faturamento:
1. **Auditoria em Tempo Real (UX):** O usuário digita a venda e a inteligência do Dashboard pré-valida NCM/CST e tributações *antes* de habilitar o envio.
2. **Delegação Assíncrona:** Ao confirmar, o Dashboard **não processa o XML diretamente**. Ele salva o status como `Processando` e despacha a ordem para a Fila do Worker do ACBr (VPS Isolada).
3. **Reatividade (Supabase Realtime):** O Dashboard "escuta" o banco de dados via WebSockets. Quando o *Worker* aprova a nota na SEFAZ, o status no banco muda e a tabela na tela do cliente se atualiza instantaneamente para "Autorizada", liberando o botão do DANFE, sem precisar recarregar a página (F5).

## 5. Módulos Core do Sistema (Escopo)
- **Overview (Painel Geral):** Métricas de faturamento e impostos processados de forma assíncrona.
- **Smart Forms (Emissor):** Formulários dinâmicos que guiam o cliente para evitar erros contábeis.
- **Cofre (Configurações):** Tela segura para o *upload* do Certificado A1. O arquivo viaja com criptografia TLS e é trancado no Supabase Vault.
