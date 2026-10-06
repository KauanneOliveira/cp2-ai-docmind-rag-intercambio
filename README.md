# DocMind RAG: Assistente de Editais de Intercâmbio

CKP02 da disciplina Prompt Engineering and Artificial Intelligence (FIAP, 2º semestre).
Pipeline RAG completo sobre editais reais de intercâmbio acadêmico, com comparação de
duas estratégias de chunking medida pelo RAGAS.

**Notebook no Colab (somente leitura):** [link do Colab](https://colab.research.google.com/drive/1njceGTgiASneVQxW0JMCnN9Oe-E64tDi?usp=sharing)

## Integrantes

| Nome | RM |
|---|---|
| Kauanne Oliveira | 574191 |
| Nayhely Estela| 571416 |

**Turma:** 1CCPI 

## O que o projeto faz

`load → split → embed → store → retrieve → generate`

1. **Load:** PyMuPDF lê os PDFs página a página e anexa metadados (instituição, nível, tipo, link de origem).
2. **Split:** `RecursiveCharacterTextSplitter` com separadores hierárquicos e overlap de 12,5%.
3. **Embed:** `nomic-embed-text`.
4. **Store:** ChromaDB (coleção `intercambio_editais`, distância cosseno), uma por `chunk_size`.
5. **Retrieve:** função `buscar(consulta)` com filtros opcionais por metadados e, como diferencial, reranking com cross-encoder.
6. **Generate:** `gemma4:cloud` com `temperature=0` e prompt de grounding. A resposta cita arquivo e página, e responde "Não encontrei essa informação nos documentos fornecidos." quando a informação não está nos trechos.

## Tecnologias

| Etapa | Ferramenta |
|---|---|
| Embeddings | `nomic-embed-text` (Ollama local no Colab) |
| Geração | `gemma4:cloud` (Ollama Cloud) |
| Vector store | ChromaDB |
| Orquestração | LangChain |
| Avaliação | RAGAS (faithfulness e answer_relevancy) |
| Reranking | `cross-encoder/ms-marco-MiniLM-L-6-v2` |

**Nota sobre os embeddings:** o endpoint de embeddings do Ollama Cloud retornou
`401 unauthorized` com a nossa chave, enquanto o chat funcionou. Por orientação do
professor, os embeddings usam o mesmo modelo aprovado, `nomic-embed-text`, servido por um
Ollama local no Colab, e o chat continua no Ollama Cloud.

## Base de conhecimento

Seis editais reais, em PDF, de fontes oficiais (acessados em [data]):

| Arquivo | Instituição | Nível | Tipo | Fonte |
|---|---|---|---|---|
| `unifesp_002_2026.pdf` | UNIFESP | graduação | mobilidade | [link](https://site.unifesp.br/sri/images/Valquiria%20-%20Mobilidade/Edital%20Conjunto%202026/Edital_Conjunto_2026.pdf) |
| `unifafibe_prri_01_2026.pdf` | UNIFAFIBE | graduação | mobilidade | [link](https://unifafibe.com.br/controlesite/intercambio/anexo_inscricao/1/1765386719_edital_intercambio_01_2026.pdf) |
| `ufc_prointer_20_2026.pdf` | UFC | graduação | mobilidade | [link](https://prointer.ufc.br/wp-content/uploads/2026/08/sei-6560983-edital-20-2026-1.pdf) |
| `uff_pdse_02_2026.pdf` | UFF | pós-graduação | PDSE | [link](https://editais.uff.br/sites/default/files/arquivos/Edital_02_PDSE_2026_.pdf) |
| `capes_22_2026.pdf` | CAPES | pós-graduação | PDSE | [link](https://www.gov.br/capes/pt-br/centrais-de-conteudo/editais/sei_2849220_edital_n__22_2026.pdf/@@display-file/file) |
| `usp_internacionalizacao.pdf` | USP | graduação | mobilidade | [link](https://prip.usp.br/wp-content/uploads/sites/1632/2026/08/Edital-2336-%E2%80%93-Internacionalizacao-com-Inclusao.pdf) |

Os PDFs também estão na pasta `docs/`.

## Como executar

1. Abra o notebook no Google Colab: [link do Colab](https://colab.research.google.com/drive/1njceGTgiASneVQxW0JMCnN9Oe-E64tDi?usp=sharing) (ou importe `CKP02_intercambio.ipynb`).
2. Ative uma GPU (Ambiente de execução, Alterar tipo de ambiente de execução, T4 GPU). Recomendado: a geração de embeddings é bem mais lenta na CPU.
3. Em **Secrets** (ícone de chave), crie `OLLAMA_API_KEY` com a sua chave do Ollama e ative o acesso ao notebook.
4. Rode **Ambiente de execução, Executar tudo**. Se algum PDF não for baixado automaticamente, o notebook pede o upload dele.

A indexação das duas coleções levou cerca de 12 minutos na nossa execução.

## Resultados

Seis perguntas avaliadas com RAGAS (juiz: `gemma4:cloud`):

| Versão | chunk_size | faithfulness (média) | answer_relevancy (média) | notas zero |
|---|---|---|---|---|
| sem filtro | 512 | 0,694 | 0,597 | 2 |
| sem filtro | 1024 | 1,000 | 0,391 | 3 |
| com filtro | 512 | 0,792 | 0,856 | 0 |
| com filtro | 1024 | 0,944 | 0,422 | 3 |
| filtro + rerank | 512 | 0,837 | 0,788 | 0 |
| filtro + rerank | 1024 | 0,952 | 0,357 | 3 |

**Configuração escolhida: `chunk_size=512` com filtro de metadados.** É a que tem menos
respostas "Não encontrei" e faithfulness média acima da meta de 0,7. A faithfulness alta do
1024 vem de respostas que não afirmam nada. Detalhes e análise de falha no notebook.

**Limitações:** apenas 6 perguntas, e o LLM usado como juiz varia entre execuções, então
diferenças pequenas não são conclusivas.

## Como adicionar novos documentos à base

1. Coloque o PDF na pasta de documentos (`/content/docs` no Colab). O PDF precisa ter texto selecionável (PDF escaneado precisa de OCR).
2. No notebook, adicione o arquivo ao dicionário `LINKS_PDF` (célula de download), com o link direto do PDF ou vazio para upload manual.
3. Adicione uma entrada ao dicionário `CATALOGO` (célula de carregamento):
```python
   "novo_edital.pdf": {
       "instituicao": "SIGLA", "nivel": "graduacao", "tipo": "mobilidade",
       "url_origem": "https://link-do-edital",
   },
```
4. Se for uma instituição nova, inclua a sigla na lista `INSTITUICOES` (célula `detectar_filtros`) para o filtro automático reconhecê-la.
5. Execute novamente a partir da célula de carregamento dos documentos: o split e a indexação são refeitos com a nova base.

## Diferenciais implementados

- **Metadata filtering:** busca com `where` por instituição, nível e tipo, detectada automaticamente na pergunta.
- **Reranking:** cross-encoder reordena 10 candidatos e envia os 3 melhores ao LLM.
