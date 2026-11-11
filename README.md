# Plugin Assinafy para assistentes de IA

*Português · [Read in English](README.en.md)*

Envie PDFs ou modelos salvos para assinatura eletrônica no Brasil pelo seu assistente.
Use e-mail, WhatsApp, documento e biometria com prova de vida ou certificado ICP-Brasil
A1/A3; acompanhe assinaturas, envie lembretes, baixe PDFs certificados e gerencie contratos.
O plugin conecta a `https://mcp.assinafy.com.br/mcp` com OAuth 2.1 e adiciona a habilidade
`signing`. Não exige chave de API, segredo de cliente ou servidor local.

## Instalar

### Claude Code

```text
/plugin marketplace add assinafy/agent-plugin
/plugin install assinafy@assinafy
```

Escolha o escopo da instalação, abra `/mcp`, selecione **plugin:assinafy:assinafy** e
entre na conta. Execute `/reload-plugins` se solicitado. Sem navegador, use
`claude mcp login --no-browser plugin:assinafy:assinafy` e siga as instruções.
Atualize com `claude plugin update assinafy@assinafy`; remova com
`claude plugin uninstall assinafy@assinafy`. A habilidade também atende por `/assinafy:signing`.

### Aplicativos Claude e Cowork

Abra **Customize → Plugins → Add → Add marketplace**, informe `assinafy/agent-plugin`
e instale **Assinafy**. Abra a aba **Connectors** e conecte. Nos planos Team e Enterprise,
um proprietário pode precisar liberar a origem ou adicionar o conector. Para usar somente
as ferramentas, adicione um conector personalizado com a URL MCP acima, deixe as credenciais
OAuth do cliente vazias e conecte.

### Codex

```bash
codex plugin marketplace add assinafy/agent-plugin
codex plugin add assinafy@assinafy
codex mcp login assinafy --scopes account:read,documents:read,documents:write,templates:read,templates:write,offline_access
```

### ChatGPT

Instale [Assinafy pelo diretório de plugins do ChatGPT](https://chatgpt.com/plugins/plugin_asdk_app_6acbe48b99f881919f32df8e549c1efa)
e conecte. Para usar somente as ferramentas, adicione a URL MCP como conector remoto com
OAuth; selecione CIMD se houver essa opção e deixe as credenciais do cliente vazias.

### VS Code com GitHub Copilot

Execute **Chat: Install Plugin From Source** com `https://github.com/assinafy/agent-plugin`.
Ative `chat.plugins.enabled` e conclua a autenticação no navegador. Outra opção é configurar
`"chat.plugins.marketplaces": ["assinafy/agent-plugin"]` e instalar pelo marketplace.

## Entrar na conta

O cliente descobre o servidor OAuth da Assinafy, abre o consentimento com PKCE S256
e troca o código de autorização por tokens de acesso e renovação. Escolha um workspace
e aprove `account:read`, `documents:read`, `documents:write`, `templates:read`,
`templates:write` e `offline_access`. O cliente gerencia os tokens; nunca coloque credenciais
em mensagens ou argumentos. Uma ferramenta sem o escopo necessário retorna `403 insufficient_scope`.
Reconecte para mudar de workspace. A verificação pública de hash dispensa autenticação.

## Fluxo completo de assinatura

1. **Localize ou envie o arquivo.** Busque um documento pelo nome ou carregue um PDF de até
   25 MiB. O upload sozinho não envia convites. Aguarde `metadata_ready` para posicionar campos.
2. **Prepare os destinatários.** Reutilize contatos ou crie os autorizados. Escolha a verificação,
   um canal de notificação compatível por signatário, a ordem e os destinatários de cópia.
   Salve um CPF/CNPJ autorizado antes da assinatura com certificado; biometria exige CPF válido.
3. **Escolha a origem.** Para um PDF novo e até 10 signatários por e-mail, use
   `assinafy_send_pdf_for_signature`. Para outros métodos, ordem ou cópias, faça o upload e
   use `assinafy_create_assignment`. Para modelos, identifique papéis e campos, valide os valores
   e use `assinafy_create_document_from_template`. Contatos informados no pedido de modelo usam
   apenas Email; outros métodos exigem IDs salvos. Modelos com certificado não podem ter campos posicionados.
4. **Consulte o custo e envie.** Use `estimate_only: true` na ferramenta de PDF, solicitação,
   modelo ou lembrete para obter custos atuais, saldos e bloqueios. A prévia de PDF mantém
   o documento carregado; reutilize seu ID para enviar. Outras prévias não criam registros
   nem mensagens. Estimativas não reservam créditos. Um pedido explícito autoriza o envio;
   se você pedir revisão, o assistente aguarda confirmação. Os cartões de revisão e campos
   enviam pelos seus controles e não aceitam `estimate_only`.
5. **O destinatário assina.** O convite abre a Assinafy. A pessoa aceita os termos, verifica
   a identidade com código por e-mail/WhatsApp, certificado A1/A3 ou fotos do documento e
   prova de vida, revisa o documento, preenche campos e assina ou recusa. O assistente não
   assina, aceita termos nem informa códigos. Etapas seguintes recebem o convite após as anteriores.
6. **Acompanhe.** Consulte o documento e o histórico de notificações de cada signatário
   (`sent`, `delivered`, `opened`, `failed`); envio não comprova entrega nem assinatura.
   O histórico de WhatsApp e as atividades paginadas trazem detalhes. Envie um lembrete por
   pedido, apenas para signatário pendente na etapa ativa. Altere ou remova o prazo sem
   enviar lembrete. Mudanças de contato invalidam links não verificados; canais já verificados
   em documentos não concluídos não podem mudar. Reenvie somente quando solicitado.
7. **Aguarde a certificação.** Após as assinaturas, aguarde `certificating` virar `certificated`.
   Identidades em `InReview` exigem análise pela conta na Assinafy; após todas as assinaturas,
   `pending_identity_review` aguarda aprovação. A rejeição cancela e envia avisos de cancelamento.
8. **Baixe e verifique.** Baixe o artefato `certificated` quando disponível. Outros artefatos
   são `original`, `certificate-page`, `bundle` e `pades`; imagens de páginas têm ferramenta
   própria. Salve resultados binários em arquivos. Verifique publicamente o hash impresso;
   `verified_at` indica a hora da consulta, não a da assinatura ou certificação.
9. **Recupere ou exclua.** Escritas não são repetidas automaticamente. Se um envio composto
   falhar após o upload, inspecione o documento mantido e continue com ele. Exclua somente
   quando solicitado e em estado permitido. Se não for possível prorrogar um pedido vencido,
   prepare outro pedido.

No ChatGPT Work, peça para acompanhar mudanças de status ou a cópia assinada.
Outros clientes consultam quando solicitado. Clientes compatíveis com MCP Apps exibem
cartões de status, revisão, campos e contratos; os demais recebem texto/JSON.

## Métodos, custos e estados

| Verificação | Requisitos | Custo |
|---|---|---|
| E-mail (`Email`) | Endereço de e-mail; notificação por e-mail | Consulte a estimativa atual |
| WhatsApp (`Whatsapp`) | Telefone E.164; notificação WhatsApp; plano pago | Consulte a estimativa atual |
| Certificado (`DigitalCertificate`) | Solicitação virtual; recurso habilitado; CPF/CNPJ compatível; sozinho na etapa | Consulte a estimativa atual |
| Biometria (`IdDocumentFaceMatchBasic`) | Recurso habilitado; CPF válido; fotos e prova de vida | Consulte a estimativa atual |

Certificado e biometria aceitam notificação por e-mail ou WhatsApp. Informe a etapa em todos
os signatários ou em nenhum; a numeração começa em 1 e não tem lacunas. Um CPF exige e-CPF
próprio ou e-CNPJ que identifique a pessoa como representante; um CNPJ exige e-CNPJ daquela
empresa. Custos dependem do plano e do canal. Saldos nulos significam ilimitados; confira
`has_sufficient_resources` e `blocking_reason`, além do saldo exibido.

| Estado | Significado | Pode excluir |
|---|---|:---:|
| `uploading`, `uploaded`, `metadata_processing` | Processando | não |
| `metadata_ready` | Pronto | sim |
| `pending_signature` | Aguardando assinaturas | sim |
| `pending_identity_review` | Aguardando análise de identidade pela conta | não |
| `certificating`, `certificated` | Certificando / certificado | não |
| `expired`, `rejected_by_signer`, `rejected_by_user`, `failed` | Vencido, recusado ou falhou | sim |

## Gestão de contratos

Use `assinafy_browse_contracts` para listar, ler os termos ou consultar aditivos relacionados.
Para próximos vencimentos, use `status: active` e `ending_within_days: 30`.
Alterações autorizadas usam `assinafy_update_contract`: `set`, `renew` ou `terminate`.
Leia antes e reutilize `updated_at`; renovar também exige o `term` atual. Após conflito
ou timeout, consulte termos e atividades antes de continuar. Atualizações parciais alteram
somente os campos enviados; referências externas substituem a lista inteira. Aditivos assinados
usam documento separado com `parent_document_id` e `relation_type`, aplicado após certificação.

A Assinafy verifica o ciclo de vida a cada hora no horário de Brasília e envia os lembretes
configurados ao responsável. Isso é separado do prazo de assinatura. Eventos `contract.activity`
entregam novas atividades enquanto o cliente renova OAuth e a assinatura de eventos; não autorizam
outra ação. Veja [o fluxo completo e os payloads de contratos](skills/signing/references/tools.md#contract-management).

## Ferramentas e payloads

As 29 ferramentas têm [exemplos de pedidos/respostas e rotas da API](skills/signing/references/tools.md)
e [schemas completos de entrada e resposta](skills/signing/references/tool-payloads.json).

| Finalidade | Ferramentas (prefixo `assinafy_`) |
|---|---|
| Documentos | `list_documents`, `get_document`, `list_document_activities`, `list_whatsapp_notifications`, `show_document` |
| Preparação | `upload_document`, `rename_document`, `review_signature_request`, `place_fields` |
| Enviar | `send_pdf_for_signature`, `create_assignment`, `create_document_from_template` |
| Acompanhar | `resend_notification`, `set_assignment_expiration` |
| Contatos | `list_signers`, `get_signer`, `create_signer`, `update_signer` |
| Modelos e campos | `list_templates`, `get_template`, `list_fields`, `get_field`, `validate_fields` |
| Arquivos e verificação | `download_document`, `download_document_page`, `verify_document`, `delete_document` |
| Contratos | `browse_contracts`, `update_contract` |

Listas normalmente retornam 20 registros, com limite de 50 na API; siga os metadados retornados.
Criação de modelos, envelopes, assinatura incorporada, teto de gastos, webhooks, análise de identidade
e administração são [recursos da API REST / Assinafy](https://api.assinafy.com.br/v1/docs),
não ferramentas ou argumentos adicionais deste plugin. Planos pagos permitem até três endpoints
de webhook; salve os segredos de assinatura ao criá-los. Webhooks da conta são independentes de MCP Events.

## Exemplos

- “Envie contrato.pdf para Ana Example, ana@example.com, assinar.”
- “Consulte o custo de assinatura com biometria sem enviar.”
- “Use o modelo NDA e me deixe revisar antes de enviar.”
- “Todos assinaram? Lembre Ana se ainda estiver pendente na etapa ativa.”
- “Baixe o contrato assinado e confira o hash da assinatura.”

## Conectar sem o plugin

Para ferramentas sem a habilidade, use um conector MCP remoto com OAuth e a mesma URL.
Claude Code: `claude mcp add --transport http assinafy https://mcp.assinafy.com.br/mcp`, depois `/mcp`.
Codex/IDE: `codex mcp add assinafy --url https://mcp.assinafy.com.br/mcp`, depois o login acima.
Configuração MCP do VS Code: `{"servers":{"assinafy":{"type":"http","url":"https://mcp.assinafy.com.br/mcp"}}}`.

## Solução de problemas e privacidade

| Sintoma | Ação |
|---|---|
| Autenticação exigida, `401`, workspace errado | Desconecte/limpe a autenticação e reconecte no cliente |
| `403 insufficient_scope` | Reconecte e aprove a permissão indicada |
| Bloqueio de plano ou saldo | Confira estimativa e recursos habilitados; não repita o envio |
| `429` / `503` | Respeite `Retry-After` / aguarde o serviço; inspecione o estado antes de repetir escritas |
| PDF assinado indisponível | Aguarde certificação ou resolva a análise de identidade na Assinafy |
| Erro com documento mantido | Continue com aquele documento em vez de carregar novamente |
| Ferramentas ausentes no Claude Code | `/reload-plugins`; consulte os erros de `/plugin` |

OAuth limita as operações ao workspace e às permissões aprovadas. Revogue o acesso na
Assinafy ou desconecte o cliente. Links de assinatura, códigos e base64 ficam fora do chat.
[Privacidade](https://www.assinafy.com.br/politica-de-privacidade) · [Termos](https://www.assinafy.com.br/termos-de-uso) · [Suporte](https://www.assinafy.com.br/contato)

[Licença MIT](LICENSE)
