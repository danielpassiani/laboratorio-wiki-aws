# Wiki Inteligente de Conhecimento da Empresa — Proposta de Arquitetura AWS

## Qual problema o projeto resolve

Toda empresa acumula registros importantes espalhados em formatos que ninguém mais consulta: atas em PDF, papéis digitalizados com anotação à mão, exportações de CRM em planilha. A informação existe, mas está presa em arquivos soltos, sem organização e sem busca. Este projeto propõe uma arquitetura na AWS que transforma esse acervo bagunçado em uma base consultável em linguagem natural — alguém pergunta "qual foi a decisão sobre o projeto X" e recebe uma resposta que **cita o documento de onde a informação veio**.

## Como a arquitetura funciona, do arquivo bruto até a resposta

1. Os três arquivos chegam em `raw/` (S3) e permanecem intocados — a pasta original nunca é modificada.
2. Um evento de criação de objeto no S3 dispara uma **Step Functions state machine** que roteia cada arquivo conforme seu tipo.
3. PDFs e imagens digitalizadas passam pelo **Amazon Textract** (que também reconhece escrita à mão); o CSV é interpretado diretamente por uma **Lambda**, sem OCR.
4. O texto extraído (com o respectivo score de confiança, quando aplicável) é salvo em um prefixo `processed/`, separado do bruto.
5. O **Amazon Comprehend** enriquece o texto não estruturado com entidades e frases-chave; tudo é padronizado em um esquema comum e registrado em uma tabela **DynamoDB** (`document-registry`), que também controla quais trechos precisam de revisão humana por baixa confiança de OCR.
6. As unidades normalizadas alimentam o **Amazon Bedrock Knowledge Bases**, que faz chunking, gera embeddings (Titan) e indexa tudo em uma coleção **OpenSearch Serverless**.
7. Perguntas em linguagem natural são respondidas pela API `RetrieveAndGenerate` do Bedrock, que busca os trechos mais relevantes e gera a resposta citando o documento de origem.

Detalhes completos do raciocínio, etapa por etapa, estão em [`resposta.md`](./resposta.md).

## Quais serviços foram escolhidos e por quê

- **Step Functions**: orquestra três caminhos de processamento distintos com retry e rastreabilidade — necessário porque os três formatos não podem ser tratados da mesma forma.
- **Amazon Textract**: única peça capaz de tornar pesquisável o que hoje é só imagem, incluindo texto manuscrito.
- **Lambda**: parser direto para o CSV, evitando gastar OCR em algo que já é texto estruturado.
- **Amazon Comprehend**: padroniza metadados (entidades, datas, projetos) a partir de texto livre.
- **DynamoDB**: registro central de metadados e controle de qualidade (flag de revisão para OCR de baixa confiança).
- **Amazon Bedrock Knowledge Bases + OpenSearch Serverless**: indexação vetorial e geração de resposta em linguagem natural com citação nativa da fonte, sem precisar montar essa lógica do zero.

## Como cada um dos três formatos é tratado

- **Ata em PDF**: extração de texto via Textract, focando em decisões, responsáveis e datas.
- **Folha digitalizada**: Textract com reconhecimento de escrita à mão, com score de confiança guardado para sinalizar trechos que precisam de revisão.
- **Exportação do CRM em CSV**: sem OCR — cada linha vira um registro estruturado, aproveitando as próprias colunas como metadados.
