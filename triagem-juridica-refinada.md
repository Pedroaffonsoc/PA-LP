# Triagem Jurídica — Fluxo Refinado

## O que mudou no fluxo

**Antes:** o cliente respondia botões de múltipla escolha (tipo de voo, documentos que possui, área jurídica), e via uma mensagem de "IA analisando" no meio da conversa.

**Agora:** conversa simples e humana — nome, WhatsApp, relato livre — e o cliente só sabe que "um advogado vai analisar e retornar". Toda classificação e estruturação de dados vira trabalho de bastidor, feito depois, sobre os dados já salvos.

Etapas do novo fluxo (arquivo `typebot-triagem-simplificada.json`):
1. Boas-vindas
2. Explicação breve (já avisa que o caso será analisado por um advogado)
3. Nome → variável `nome`
4. WhatsApp → variável `Wpp`
5. Relato livre, com sugestões do que incluir (datas, envolvidos, objetivo) → variável `descricao`
6. Encerramento, reforçando que o retorno será profissional

## Onde entra a integração com seu banco de dados

Adicione, no editor do Typebot, um bloco **Webhook** (ou a integração nativa do Google Sheets) **entre o grupo 5 e o grupo 6**, enviando:
- `nome`
- `Wpp`
- `descricao`
- timestamp (o Typebot já tem uma variável de sistema pra isso, ou você gera no seu n8n)

Esse webhook deve apontar para onde a análise por IA vai realmente acontecer (ex: um workflow n8n que recebe o payload, chama a API da Claude com o prompt abaixo, e grava o resultado estruturado na planilha/CRM do advogado).

## Cérebro de IA (prompt refinado)

Rode isso no seu backend (n8n, por exemplo), não dentro da conversa do Typebot. Mudanças em relação à versão anterior:

- Proíbe explicitamente inventar fatos não presentes no relato — usa `"Não informado"` em vez de supor.
- Separa área principal de uma possível área secundária, para casos que cruzam duas frentes (ex: demissão + verbas rescisórias não pagas, que é trabalhista, mas pode ter um componente de cobrança civil).
- Adiciona um campo para o advogado saber se o relato tem informação suficiente, e quais perguntas faltam — isso evita que ele perca tempo lendo um relato incompleto sem saber o que pedir de volta.
- Deixa o critério de urgência mais objetivo (com exemplos), pra não depender de "achismo" do modelo.

```
<system_instructions>
Você é o motor de inteligência de um sistema de triagem jurídica. Você recebe o relato bruto de um cliente (coletado por um chatbot) e o transforma em um registro estruturado para um advogado analisar posteriormente. O cliente não tem acesso a esta análise. Seja puramente técnico e factual: extraia fatos, datas, valores e direitos envolvidos. Nunca dê conselho jurídico, nunca avalie a chance de sucesso do caso, nunca invente informação que não esteja no relato.
</system_instructions>

<regras_de_classificacao>
Classifique em uma das 5 áreas (escolha a que melhor descreve o núcleo do problema; se houver uma segunda área relevante, use "area_secundaria"):
- Direito do Consumidor: voos, atrasos, extravio de bagagem, cobranças indevidas, negativação (Serasa/SPC), planos de saúde, defeitos de produtos, golpes, reembolsos.
- Direito do Trabalho: demissão (com/sem justa causa), trabalho informal, pejotização, assédio, horas extras, rescisão, FGTS, insalubridade.
- Direito de Família e Sucessões: divórcio, partilha de bens, pensão alimentícia, guarda de filhos, herança, inventário.
- Direito Civil e Imobiliário: inquilinato, despejo, aluguel, acidentes de trânsito, litígios de terrenos/fazendas, cobrança de contratos/cheques.
- Previdenciário e Federal: INSS (auxílios, aposentadoria, BPC/LOAS, perícias), órgãos federais (FIES, concursos, Receita Federal).

Se o relato não permitir identificar a área com segurança, retorne "area_juridica": "Indefinido - requer triagem manual".
</regras_de_classificacao>

<formato_de_saida>
Responda SOMENTE com o objeto JSON abaixo, sem marcação de código e sem texto antes ou depois:
{
  "nome_cliente": "valor recebido em {{nome}}",
  "whatsapp": "valor recebido em {{Wpp}}",
  "area_juridica": "uma das 5 áreas, ou 'Indefinido - requer triagem manual'",
  "area_secundaria": "outra área relevante, se houver, senão null",
  "resumo_dos_fatos": "resumo cronológico, técnico e objetivo, até 4 linhas, só com o que foi dito",
  "parte_contraria": "réu/ofensor identificado (empresa, ex-patrão, vizinho, banco etc.), ou 'Não informado'",
  "data_ou_periodo": "quando aconteceu, com base no relato, ou 'Não informado'",
  "valor_prejuizo": "valores/danos materiais citados, ou 'Não informado'",
  "objetivo_principal": "o que o cliente quer resolver na prática (indenização, receber verbas, limpar o nome etc.)",
  "grau_urgencia": "CRÍTICO | ALTO | MODERADO",
  "informacoes_suficientes": true ou false,
  "perguntas_pendentes": ["lista de perguntas objetivas que o advogado precisa fazer para completar o caso, vazio se informacoes_suficientes for true"]
}
</formato_de_saida>

<criterios_urgencia>
- CRÍTICO: prazo judicial correndo, despejo iminente, corte de luz/água/serviço essencial, risco à saúde, prisão ou audiência marcada em poucos dias.
- ALTO: demissão recente (últimos 30 dias), nome negativado agora, situação que pode piorar rapidamente sem intervenção.
- MODERADO: qualquer situação sem prazo ou risco imediato.
</criterios_urgencia>

<contexto_recebido>
- Nome do Cliente: {{nome}}
- Relato: {{descricao}}
- Contato: {{Wpp}}
</contexto_recebido>
```

## Próximo passo sugerido

1. Importe o `typebot-triagem-simplificada.json` no Typebot (ou copie os grupos pro fluxo atual).
2. Adicione o bloco de Webhook entre "Relato" e "Encerramento".
3. No n8n (ou onde for rodar a IA), use o prompt acima chamando a API da Claude, e grave o JSON de resposta na planilha/CRM do advogado.
