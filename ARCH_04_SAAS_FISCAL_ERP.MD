# Arquitetura 4: SaaS Fiscal & ERP Avançado (O Ápice)

Esta arquitetura representa o grau máximo de maturidade tecnológica e operacional da Software House. É a integração perfeita entre um SaaS seguro e performático e um motor de mensageria fiscal (Projeto ACBr), desenhada para suportar alto volume de emissões (NF-e, NFC-e, etc.) em ambiente *Multitenant* (múltiplas empresas) sem gargalos ou falhas de segurança.

## 1. Stack Tecnológica Distribuída
Diferente das arquiteturas anteriores, emissão fiscal requer processamento assíncrono e bibliotecas nativas do sistema operacional (DLLs/.so). Por isso, o sistema é dividido:

- **Core do SaaS (App Principal):** Next.js 15 (App Router) + Supabase (Herdado da ARCH_02). Responsável pela UI, regras de negócio tributárias e cadastro de clientes.
- **Motor Fiscal (Worker Isolado):** Um microsserviço Node.js (nativo) rodando em uma VPS ou Container (Docker). Este ambiente tem acesso ao sistema de arquivos local para executar a **ACBrLib (FFI)**.
- **Mensageria (Filas):** Redis (ex: BullMQ) ou RabbitMQ. Responsável por orquestrar os pedidos de emissão para que o ACBr processe um por vez ou em paralelo sem sobrecarregar a memória da VPS.
- **Armazenamento de Notas:** Supabase Storage para guardar os arquivos XML autorizados e os PDFs (DANFE).

## 2. Padrões Ouro (Segurança e Pentest)
Baseado nas vulnerabilidades mapeadas no `PENTEST.md`, o módulo fiscal adota defesas de nível militar:

- **Proteção TOCTOU (Idempotência e Locks):** Rotinas fiscais sofrem muito com "Double Clicks". A emissão de uma nota possui um `idempotency_key` (chave única da transação). Mesmo que o usuário aperte "Emitir" 10 vezes em um segundo, o banco de dados usa um *Lock* e a fila (Redis) garante que apenas UMA requisição chegará ao ACBr.
- **Isolamento de Certificados (Cofre Seguro):** O certificado A1 (PFX) do cliente NUNCA é retornado ao frontend (evitando *Data Exposure*). Ele é armazenado criptografado usando uma chave KMS no banco (ex: Supabase Vault). O Worker fiscal descriptografa o certificado em memória (RAM) ou em diretório volátil (TempFS) apenas pelos milissegundos necessários para a assinatura pelo ACBr, apagando-o em seguida.
- **Anti-Mass Assignment (Validação Zod):** O cliente envia dados brutos da venda. O backend exige um schema `Zod` rigoroso antes de transformar esses dados no padrão `.ini` do ACBr. Qualquer tentativa de enviar campos não autorizados (ex: adulterar a alíquota forçando uma tag XML indevida) é descartada.
- **Auditoria de Acessos (IDOR e RLS):** As rotas que retornam os XMLs/PDFs emitidos devem provar via RLS (Row Level Security) que o `tenant_id` que está baixando a NF-e é o mesmo que a emitiu.

## 3. O Fluxo Perfeito do SaaS Fiscal

A mágica desta arquitetura está na sua natureza **assíncrona**. O usuário não fica com a tela "congelada" esperando a SEFAZ responder.

1. **Validação Inteligente (Frontend -> Backend):** O cliente preenche a venda. Seu backend audita os CFOPs e tributações (seu diferencial de negócio). Se houver erro de legislação, a nota nem tenta ser emitida, poupando a infraestrutura.
2. **Enfileiramento:** Tudo validado, o backend cria um registro de NF com status `PROCESSING` e envia o JSON da nota para a Fila (Redis). O backend responde rapidamente ao frontend: *"Nota enviada para processamento!"*.
3. **Consumo e Conversão (Worker):** O Microsserviço na VPS pega a mensagem da fila, busca e descriptografa o Certificado A1. O Node.js converte o JSON seguro no padrão `Arquivo.ini`.
4. **Chamada DLL (ACBrLib):** O Worker dispara a função do ACBrLib. O ACBr monta o XML, assina com o certificado e transmite à SEFAZ.
5. **Retorno e Guarda:** O ACBrLib devolve o XML autorizado e gera o DANFE. O Worker salva esses arquivos no Supabase Storage.
6. **Sincronia Final (WebSockets/Polling):** O Worker atualiza o banco de dados para `AUTHORIZED`. O frontend do cliente, que estava escutando as mudanças via Supabase Realtime (WebSockets) ou TanStack Query, atualiza a tela automaticamente, liberando os botões "Baixar XML" e "Imprimir DANFE".

## 4. Estratégia de Deploy (VPS vs Serverless)
- O **Next.js** continua hospedado na Vercel (Edge/Serverless) para performance brutal na interface.
- O **Worker (ACBr)** e a **Fila (Redis)** rodam em uma **VPS robusta** (DigitalOcean, AWS EC2 ou Hetzner) com Linux Ubuntu. Isso garante o acesso direto aos arquivos nativos (`.so`) exigidos pela ACBrLib.

> [!IMPORTANT]
> **Gerenciamento de Erros da SEFAZ:** A SEFAZ frequentemente entra em contingência ou cai. Sua fila deve ser configurada para fazer tentativas automáticas (Retry Policies) ou enviar a nota para um status `REJECTED_SEFAZ`, guardando o "Motivo de Rejeição" (cStat) para o cliente corrigir o dado na tela sem precisar redigitar toda a nota.
