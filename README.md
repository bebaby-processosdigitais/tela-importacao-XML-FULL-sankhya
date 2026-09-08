# Fila de XML do Full — Tela `AD_XMLFULL` com dois botões

**O que é:** importação dos XMLs do Mercado Livre Full para o Sankhya, operada por dois
botões numa tela adicional. Substitui a integração Tem Api, que parou de trazer as notas do
Full em **29/06/2026**.

**Status:** ✅ **funcionando de ponta a ponta**, validado em homologação em 08/09/2026.
**Ambiente:** base de teste (homologação).

---

## Índice

1. [Como funciona](#1-como-funciona)
2. [O que está construído](#2-o-que-está-construído)
3. [A tabela AD_XMLFULL](#3-a-tabela-ad_xmlfull)
4. [Botão 1 — Buscar na Anymarket](#4-botão-1--buscar-na-anymarket)
5. [Botão 2 — Enviar para TGFIXN](#5-botão-2--enviar-para-tgfixn)
6. [Rotina de operação](#6-rotina-de-operação)
7. [Resultados dos testes](#7-resultados-dos-testes)
8. [Queries de verificação](#8-queries-de-verificação)
9. [Limitações do estado atual](#9-limitações-do-estado-atual)
10. [Decisões de projeto](#10-decisões-de-projeto)
11. [Solução de problemas](#11-solução-de-problemas)
12. [Pendências](#12-pendências)

---

## 1. Como funciona

```
[Botão 1: Buscar na Anymarket]
        ↓
   Anymarket API → filtra tipo → dedup por IDARQUIVO → baixa XML
        ↓
   AD_XMLFULL  (Situação = Pendente)
        ↓
   Denise confere na grade: cliente, nota, valor, emissão,
   e a coluna "Já está na TGFIXN?"
        ↓
[Botão 2: Enviar para TGFIXN]  ← com as linhas SELECIONADAS
        ↓
   TGFIXN (STATUS 0)  +  fila vira Transferido
        ↓
   Motor de importação do Sankhya (regras do Full do Paulo)
        ↓
   Nota fiscal gerada
```

O robô **não** monta a nota, **não** define a TOP e **não** cadastra parceiro. Isso é
trabalho do motor de importação.

---

## 2. O que está construído

| Componente | Onde | Status |
|---|---|---|
| Tabela `AD_XMLFULL` (20 campos + 1 calculado) | Construtor de Telas | ✅ criada |
| Tela publicada no menu | Menu do Sistema | ✅ |
| Campo calculado `JANATGFIXN` | expressão Java | ✅ validado |
| Domínio da `SITUACAO` (P/T/D/E) | Opções do campo | ✅ |
| Botão "Buscar na Anymarket" | ação Script | ✅ validado |
| Botão "Enviar para TGFIXN" | ação Script | ✅ validado |
| Parâmetro do token | `ANYMARKET_TOKEN` | ⚠️ ver §9 |

### Arquivos

| Arquivo | Conteúdo |
|---|---|
| `botao1_buscar_anymarket.js` | Script do botão 1 |
| `botao2_enviar_tgfixn.js` | Script do botão 2 |
| `fase1_criar_ad_xmlfull.md` | Passo a passo de criação da tela |
| `plano_arquitetura_agendamento.md` | Plano A (agendado) — para levar ao Paulo |
| `estado_projeto_tgfixn_2026-08-27.md` | Histórico da investigação e engenharia reversa |

---

## 3. A tabela `AD_XMLFULL`

### Campos que a Denise vê

| Campo | Rótulo | Tipo |
|---|---|---|
| `NUFILA` | Nº | Inteiro (PK auto) |
| `SITUACAO` | Situação | Texto / Lista de Opções |
| `JANATGFIXN` | Já está na TGFIXN? | **Calculado** |
| `DHRECEBIDO` | Recebido em | Data e Hora |
| `NOMECLIENTE` | Cliente | Texto |
| `NUMNOTA` | Nota | Inteiro |
| `SERIEDOC` | Série | Inteiro |
| `VLRNOTA` | Valor | Decimal |
| `DHEMISS` | Emissão | Data e Hora |
| `NATUREZAOPER` | Natureza | Texto |
| `TIPODOC` | Tipo do documento | Texto |
| `DHTRANSFERIDO` | Enviado em | Data e Hora |
| `MENSAGEM` | Observação | Texto / Caixa de Texto |

### Campos técnicos

`CHAVEACESSO` · `IDARQUIVO` · `SUBTIPODOC` · `CNPJPARC` · `NUARQUIVO` · `TENTATIVAS` ·
`URLORIGEM` (Caixa de Texto) · `XML` (CLOB, Formatação HTML)

### Domínio da `SITUACAO`

| Valor | Rótulo | Significado |
|---|---|---|
| `P` | Pendente | Chegou da Anymarket, ainda não enviado |
| `T` | Transferido | Já foi para a TGFIXN |
| `D` | Duplicado | A chave já existia na TGFIXN — **não é erro** |
| `E` | Erro | Falhou. Ver a coluna Observação |

### O campo calculado `JANATGFIXN`

**Não existe coluna no banco.** É resolvido por consulta a cada leitura da grade, então
nunca fica desatualizado. Expressão (Java):

```java
if ($col_CHAVEACESSO == null) {
    return "-";
}
$sql.setParam("CH", $col_CHAVEACESSO);
$sql.select("COUNT(*)", "TGFIXN", "CHAVEACESSO = {CH}");
String resultado = "NAO";
if ($sql.next()) {
    if (!"0".equals($sql.getString(1))) {
        resultado = "SIM";
    }
}
return resultado;
```

Requisitos: Tipo de dados **Texto**, Apresentação **Padrão**, e o toggle
**"Campo calculado?"** ligado. Com o toggle desligado, o Sankhya trata a expressão como
expressão de UPDATE e cria coluna real — comportamento diferente.

---

## 4. Botão 1 — Buscar na Anymarket

**Ação Script** na `AD_XMLFULL`. "Depois de executar, recarregar" = **Recarregar tudo**.

### O que faz

1. Carrega em memória os `IDARQUIVO` já na fila (1 query)
2. Pagina a listagem da Anymarket, 50 por página
3. Filtra pelos tipos habilitados
4. Descarta o que já está na fila — **antes de baixar o XML**
5. Baixa o XML dos novos (do S3, sem token)
6. Extrai os campos legíveis: cliente, nota, série, valor, natureza, emissão
7. Insere com `SITUACAO = 'P'`

Não toca na `TGFIXN`.

### Configuração

| Variável | Valor atual | O que faz |
|---|---|---|
| `LIMITE_LOTE` | **1** | Notas por clique. Fase de teste |
| `FILA_FOLGA` | 5 | Multiplicador de candidatas |
| `MAX_PAGINAS` | 30 | Páginas varridas por clique |
| `PAGE_SIZE` | 50 | **Mínimo aceito pela API é 5** |
| `TIPOS` | `devolution`, `sale_return` | Tipos habilitados |
| `TOKEN_FALLBACK` | preenchido | ⚠️ ver §9 |

### Tipos de documento

| Tipo | Total na conta | Habilitado |
|---|---|---|
| `symbolic_inbound_return` | 522 | ⚪ não |
| `sale` | 476 | 🔴 não — exige aval |
| `symbolic_inbound` | 32 | ⚪ não |
| `devolution` | 17 | ✅ **sim** |
| `inbound` | 13 | ⚪ não |
| `inbound_return` | 11 | ⚪ não |
| `sale_return` | 7 | ✅ **sim** |

Total de documentos na conta: **1078**.

Para habilitar, descomente a linha em `TIPOS`. Diagnóstico de 27/08 mostrou que os quatro
tipos de entrada e simbólicos têm 100% dos parceiros já cadastrados e contraparte CNPJ.

### Leitura do resumo

Exemplo real:

```
1 nota trazida para a fila (Pendente). || devolution | NF 3/43323 |
R$ 119.00 | Talita Nascimento | emissao 2026-08-27 ||
(fila antes: 3 | verificados: 533 | outros tipos: 525 | ja tinha: 3)
```

| Número | Significado |
|---|---|
| `fila antes` | Linhas na `AD_XMLFULL` antes do clique |
| `verificados` | Documentos **lidos da API** nesta rodada |
| `outros tipos` | Descartados pelo filtro `TIPOS` |
| `ja tinha` | Descartados pelo dedup por `IDARQUIVO` |

O funil: `verificados` − `outros tipos` − `ja tinha` = candidatas. Traz até `LIMITE_LOTE`.

---

## 5. Botão 2 — Enviar para TGFIXN

**Ação Script** na `AD_XMLFULL`.

### ⚠️ Sempre selecione as linhas antes de clicar

Ver §9 — o modo sem seleção não atualiza a fila corretamente nesta instalação.

### O que faz

1. Lê os `NUFILA` das linhas **selecionadas** na grade
2. Pula as que já estão `T` ou `D`
3. Verifica se a `CHAVEACESSO` já existe na `TGFIXN`
   - existe → marca `SITUACAO = 'D'`, **não grava**
   - não existe → segue
4. Lê o XML da fila (com fallback de rebaixar do S3 se o CLOB vier truncado)
5. Extrai os 20 campos e grava na `TGFIXN` com `STATUS = 0`
6. Recupera o `NUARQUIVO` gerado e grava na fila
7. Marca `SITUACAO = 'T'`

Cada linha é isolada em `try/catch`: erro numa não impede as outras.

### Campos gravados na `TGFIXN`

| Campo | Origem |
|---|---|
| `XML` | fila (ou rebaixado do S3) |
| `CHAVEACESSO` | `<chNFe>` |
| `NOMEARQUIVO` | `INVOICE-{tipo}-{IDARQUIVO}.XML` |
| `TIPO` | `'N'` |
| `STATUS` | `0` |
| `CODEMP` | `1` |
| `DHIMPORT` | `new Date()` |
| `CODUSUIMP` | `0` |
| `TIPONFE` | derivado do `<tpNF>`: `0 → 'E'`, `1 → 'V'` |
| `NUMNOTA` | chave, posições 26–34 |
| `SERIEDOC` | chave, posições 23–25 |
| `NATUREZAOPER` | `<natOp>` |
| `CFOPXML` | `<CFOP>` do 1º item |
| `ENTSAINFE` | `<tpNF>` **como String** (coluna é VARCHAR2(1)) |
| `VLRNOTA` | `<vNF>` |
| `DHEMISS` | `<dhEmi>` |
| `DTAUTORIZACAO` | `<dhRecbto>` do protNFe |
| `XNOMEEMIT` / `XNOMEDEST` | `<xNome>` de cada bloco, truncado em 60 |
| `CNPJDEST` | `<dest>` CNPJ ou CPF |
| `CNPJPARC` | o lado que **não** é a BeBaby |
| `DOCSREF` | `<docsRef><chaveAcesso>{refNFe}</chaveAcesso></docsRef>` |

**Deliberadamente NÃO preenchidos:** `NUARQUIVO` (PK auto), `CODTIPOPER` (o motor deduz),
`CODPARC` (o motor cadastra), `CONFIG` (o motor gera), `NUNOTA`, `DHPROCESS`,
`CODUSUPROC`, `DETALHESIMPORTACAO`.

---

## 6. Rotina de operação

Texto para passar à Denise:

> **Importar devoluções do Mercado Livre Full**
>
> 1. Abra a tela **Fila de XML do Full**
> 2. Clique em **Buscar na Anymarket**. Ele traz nota nova com situação *Pendente*
> 3. Confira na lista: cliente, número da nota, valor e data de emissão
> 4. Olhe a coluna **Já está na TGFIXN?** — se disser SIM, a nota já entrou no sistema
>    por outro caminho e não precisa ser enviada
> 5. **Selecione** as linhas que quer importar
> 6. Clique em **Enviar para TGFIXN**
> 7. As linhas viram *Transferido*. Se aparecer *Duplicado*, a nota já estava no sistema —
>    não é erro. Se aparecer *Erro*, veja a coluna Observação
>
> Nada é enviado sem você selecionar e clicar. Pode buscar quantas vezes quiser: o sistema
> não traz a mesma nota duas vezes.

---

## 7. Resultados dos testes

Homologação, 08/09/2026.

### Botão 1 — quatro cliques

| Clique | Nota trazida | `fila antes` | `verificados` | `ja tinha` |
|---|---|---|---|---|
| 1 | Josimara Cristina — NF 3/43049 — R$ 25,90 | 0 | 501 | 0 |
| 2 | Vera Lucia de Alc. — NF 3/43122 — R$ 119,00 | 1 | — | 1 |
| 3 | Leticia Francielle — NF 3/43297 — R$ 339,90 | 2 | — | 2 |
| 4 | Talita Nascimento — NF 3/43323 — R$ 119,00 | 3 | 533 | 3 |

**Quatro notas diferentes, zero duplicata.** O dedup por `IDARQUIVO` funciona.

### Campo calculado

A coluna mostrou `NAO`, `NAO`, **`SIM`**, `NAO`. O `SIM` caiu na NF 43297 — exatamente a
nota que havia sido importada manualmente nos testes anteriores (NUARQUIVO 112625). A
consulta bate com a `TGFIXN` real.

### Botão 2 — caminho Duplicado

Selecionada a linha 3 (Leticia). Resultado: `0 enviadas | 1 ja existia`. A fila registrou
`SITUACAO = 'D'`, `DHTRANSFERIDO` preenchido, mensagem "Chave ja existia na TGFIXN". Nada
foi gravado na `TGFIXN`.

### Botão 2 — caminho de gravação

Selecionada a linha 1 (Josimara). Resultado: `1 nota enviada`. A fila registrou
`SITUACAO = 'T'` e `NUARQUIVO = 112632`.

Verificação na `TGFIXN` (NUARQUIVO 112632):

| Campo | Valor | |
|---|---|---|
| `LENGTH(XML)` | **8803** | ✅ CLOB veio inteiro, sem truncar |
| `NOMEARQUIVO` | `INVOICE-DEVOLUTION-259061706.6751513459.XML` | ✅ |
| `STATUS` | 0 | ✅ |
| `TIPONFE` | E | ✅ |
| `ENTSAINFE` | 0 | ✅ |
| `CODTIPOPER` | vazio | ✅ o motor deduz |
| `CODPARC` | vazio | ✅ o motor cadastra |
| `NATUREZAOPER` | Devolucao de mercadorias | ✅ |
| `CFOPXML` | 1202 | ✅ |
| `NUMNOTA` / `SERIEDOC` | 43049 / 3 | ✅ |
| `VLRNOTA` | 25,90 | ✅ |
| `DHEMISS` / `DTAUTORIZACAO` | 14/08/2026 09:52 | ✅ |
| `XNOMEEMIT` | BEBABY GROUP IMPORTACAO LTDA | ✅ |
| `XNOMEDEST` | Josimara Cristina da Silva | ✅ |
| `CNPJPARC` = `CNPJDEST` | 34704259811 (CPF) | ✅ não é o CNPJ da BeBaby |
| `DOCSREF` | formato idêntico ao histórico | ✅ |

**Paridade completa** com o gabarito da importação interna (112625) e com as 121 notas que
a Tem Api processou com sucesso.

---

## 8. Queries de verificação

### Estado da fila

```sql
SELECT NUFILA, SITUACAO, NOMECLIENTE, NUMNOTA, VLRNOTA,
       CHAVEACESSO, NUARQUIVO, TENTATIVAS, DHTRANSFERIDO, MENSAGEM
FROM AD_XMLFULL
ORDER BY NUFILA;
```

### Cruzamento fila × TGFIXN — a verificação principal

```sql
SELECT F.NUFILA, F.SITUACAO, F.NOMECLIENTE, F.NUMNOTA,
       F.NUARQUIVO AS NUARQ_NA_FILA,
       X.NUARQUIVO AS NUARQ_REAL,
       X.STATUS AS STATUS_TGFIXN,
       X.TIPONFE, X.ENTSAINFE, X.CODTIPOPER,
       X.VLRNOTA, X.CNPJPARC, X.CODPARC, X.NOMEARQUIVO,
       F.MENSAGEM
FROM AD_XMLFULL F
LEFT JOIN TGFIXN X ON X.CHAVEACESSO = F.CHAVEACESSO
ORDER BY F.NUFILA;
```

Se `NUARQ_REAL` vier preenchido e `NUARQ_NA_FILA` vazio: a gravação funcionou e o UPDATE
da fila falhou.

### Conferir uma nota gravada, campo a campo

```sql
SELECT NUARQUIVO, NOMEARQUIVO, STATUS, TIPO, TIPONFE, CODEMP, CODUSUIMP,
       CODTIPOPER, NATUREZAOPER, CFOPXML, ENTSAINFE,
       NUMNOTA, SERIEDOC, VLRNOTA, DHEMISS, DTAUTORIZACAO,
       XNOMEEMIT, XNOMEDEST, CNPJDEST, CNPJPARC, CODPARC,
       TO_CHAR(SUBSTR(DOCSREF, 1, 200)) AS DOCSREF_TXT,
       LENGTH(XML) AS TAM_XML
FROM TGFIXN
WHERE NUARQUIVO = 112632;
```

O `LENGTH(XML)` confirma se o CLOB veio completo. Um XML de NF-e do Full fica na casa dos
8–9 mil caracteres.

### Erros na fila

```sql
SELECT NUFILA, SITUACAO, TENTATIVAS, MENSAGEM, DHTRANSFERIDO
FROM AD_XMLFULL
WHERE SITUACAO = 'E'
ORDER BY NUFILA;
```

Linha com `SITUACAO = 'E'` e `TENTATIVAS < 3` volta a ser tentada — não precisa mexer na mão.

### Contagem por tipo na TGFIXN

⚠️ O Sankhya grava o `NOMEARQUIVO` em maiúsculas, então precisa de `UPPER()`:

```sql
SELECT SUBSTR(NOMEARQUIVO, 9, INSTR(NOMEARQUIVO, '-', 9) - 9) AS TIPO,
       STATUS, COUNT(*) AS QTD
FROM TGFIXN
WHERE UPPER(NOMEARQUIVO) LIKE 'INVOICE-%-%'
GROUP BY SUBSTR(NOMEARQUIVO, 9, INSTR(NOMEARQUIVO, '-', 9) - 9), STATUS
ORDER BY TIPO, STATUS;
```

O `LIKE 'INVOICE-%-%'` com dois hífens isola só o formato novo, separando das 133 notas
antigas da Tem Api.

---

## 9. Limitações do estado atual

### O botão 2 exige linha selecionada

**Descoberto no teste de 08/09.** Nenhum dos métodos de UPDATE via script
(`nativeUpdate`, `executeUpdate`, `executeNative`) existe nesta instalação. A função
`atualizaFila` só funciona pelo caminho alternativo, que grava pelo objeto `linhas` — e
esse objeto só existe para linhas selecionadas na grade.

**Como se detectou:** o SQL de UPDATE inclui `TENTATIVAS = TENTATIVAS + 1`, e o contador
permaneceu `0` após a atualização. Como o caminho alternativo não incrementa tentativas,
ficou provado qual dos dois rodou.

**Consequência:** clicar sem selecionar grava na `TGFIXN` mas deixa a fila dizendo
"Pendente". No clique seguinte o dedup por chave evita a duplicata, mas a linha vira `D`
em vez de `T` — confuso, embora não destrutivo.

**Mitigação atual:** regra de operação — sempre selecionar antes de clicar.
**Correção pendente:** fazer o script recusar o clique sem seleção, com mensagem clara.

### Uma nota por clique

`LIMITE_LOTE = 1` no botão 1, por escolha, para não poluir a fila durante os testes.

### O custo da varredura cresce

Primeiro clique: 501 documentos verificados. Quarto: 533. Como a listagem da Anymarket vem
**do mais antigo para o mais novo**, e as devoluções conhecidas vão sendo puladas, cada
clique precisa varrer mais fundo. Com a fila completa, um clique varreria a conta quase
inteira para não achar nada.

Não é problema no volume atual. É o argumento a favor da janela de dias (§12).

### Token no `TOKEN_FALLBACK`

O script lê de `getParametroSistema('ANYMARKET_TOKEN')` e cai no `TOKEN_FALLBACK` se o
parâmetro não existir. Hoje está no fallback, ou seja, **o token está no código**.

⚠️ Criar o parâmetro `ANYMARKET_TOKEN` e esvaziar o `TOKEN_FALLBACK` antes de qualquer
uso em produção. Um token já foi exposto anteriormente e deve estar rotacionado.

### O motor não processa

As notas ficam em `STATUS 0`. Não é limitação do robô — as notas que o **próprio Paulo**
subiu pela importação interna (112628, 112629, 112630) também estão paradas. Ver §12.

### Não é possível rodar DELETE

Afeta a limpeza da fila e das linhas de teste na `TGFIXN`. Não impede a operação.

---

## 10. Decisões de projeto

### Por que dois botões, e não uma Ação Agendada

Ações Agendadas aceitam apenas **Java** (exige .jar com `ScheduledAction` cadastrado em
Módulo Java) ou **Proc. Banco de Dados**. Não aceitam ação de tela em Script.

Como o robô precisa falar com a internet (a API da Anymarket e o S3), a rota PL/SQL
exigiria `UTL_HTTP` com ACL de rede e Oracle Wallet — projeto de infra. E a rota Java
depende do Paulo.

Os dois botões usam a mesma tecnologia já validada, ficam de pé em horas, e colocam a
Denise no controle: nada entra na tabela fiscal sem alguém olhar. Enquanto o processo não
está validado ponta a ponta, isso é vantagem.

**A tabela e a lógica são as mesmas do Plano A.** Migrar depois é trocar o gatilho: o botão
1 vira a Ação Agendada em Java, o botão 2 vira a procedure. Nada aqui é descartável.

### `IDARQUIVO` = nome do arquivo na URL do S3

A listagem da API retorna só `id`, `url`, `marketplace`, `type` e `subType` — e **nenhum
identificador existe em todos os tipos**: o `id` falta nos `sale`, o `idOrder` falta nos
`symbolic_inbound_return`.

Sonda de 27/08 varreu a conta inteira (1078 documentos) testando o nome do arquivo na URL:
**0 repetidos, 0 vazios**. Único em 100% dos casos.

```
https://s3.../transactionType-devolution/259061706.6606158899.xml
                                          └──────────┬─────────┘
                                              IDARQUIVO
```

### Dedup em dois estágios

| Estágio | Critério | Quando | Por quê |
|---|---|---|---|
| 1 | `IDARQUIVO` | botão 1, antes do download | Barato. Evita baixar XML conhecido |
| 2 | `CHAVEACESSO` | botão 2, antes de gravar | Pega as 133 notas antigas (formato de nome anterior), DF-e e importações manuais |

### Por que não é preciso consultar a `TGFCAB`

Verificado em 27/08:

```sql
SELECT COUNT(*) FROM TGFCAB C
WHERE C.CODTIPOPER IN (1234, 1766) AND C.CHAVENFE IS NOT NULL
  AND NOT EXISTS (SELECT 1 FROM TGFIXN X WHERE X.CHAVEACESSO = C.CHAVENFE);
-- resultado: 0
```

Nota do Full sempre chega como XML, e XML sempre entra pela `TGFIXN`. Paulo, 27/08:
*"só a IXN, e o resto o motor faz."* A `TGFIXN` é o universo completo dessas notas.

### `CODTIPOPER` não é enviado

Paulo, 27/08: *"Não precisa enviar, porque pode ser mais de uma. Ela pega pelo modelo do
que está no XML — se for dev, venda ou remessa."*

Confirmado pelos dados: a TOP `1766` cobre 2 naturezas × 2 CFOPs (1202 dentro do estado,
2202 fora). Não há mapeamento 1:1 possível — enviar limitaria a dedução do motor.

### `TIPONFE` derivado do `tpNF` — divergência documentada

⚠️ **Paulo disse:** D = Devolução, E = Remessa, V = Venda.
⚠️ **Os dados dizem:** não existe **nenhuma** nota com `'D'` em 121 notas processadas.

| `ENTSAINFE` | `TIPONFE` | Qtd em `STATUS 5` |
|---|---|---|
| 1 (saída) | `V` | 93 vendas |
| 0 (entrada) | `E` | 27 devoluções / retornos |

Correlação perfeita, sem exceção. O campo espelha o `tpNF` do XML (entrada/saída), não a
natureza da operação. Isso também explica o `'E'` na importação interna do Paulo — não foi
descuido, foi o comportamento correto.

Se ele confirmar um caso de `'D'`, a função `tipoNfeDe()` é o **único** ponto a alterar.

### `CNPJPARC` = o lado que não é a BeBaby

O CNPJ da BeBaby sai das posições 7–20 da própria chave de acesso, e o script escolhe o
lado oposto. Vale para qualquer tipo de documento.

⚠️ **Não validado empiricamente:** nos 98 documentos diagnosticados a BeBaby era **sempre**
a emitente, inclusive nos tipos de entrada. O ramo que escolheria o emitente nunca
executou. A lógica é mais segura que fixar no destinatário, mas não tem prova de campo.

### O prefixo `INVOICE-` foi mantido

O identificador mudou, mas o prefixo não — preserva as consultas de investigação que usam
`LIKE 'INVOICE-%'` e a convenção da Tem Api. O `type` entrou no nome porque a coluna tem
200 caracteres e o espaço é gratuito: permite contar por tipo sem abrir XML.

### `getErrorStream()` no `baixa()`

Quando o HTTP não é 2xx, `getInputStream()` estoura antes de ler o corpo — e a mensagem de
erro da API se perde. Foi assim que o limite mínimo de 5 itens por página ficou invisível
por uma rodada inteira.

### Campos legíveis extraídos ao enfileirar

Cliente, nota, série, valor, natureza e emissão são extraídos no botão 1, não no botão 2.
Custa nada (o XML já está em memória) e é o que torna a grade utilizável pela Denise. Sem
isso, ela veria uma lista de chaves de 44 dígitos.

O `NOMECLIENTE` usa a mesma regra do `CNPJPARC` — o lado que não é a BeBaby — para que a
coluna continue correta se um dia entrar tipo em que a BeBaby seja destinatária.

### `SITUACAO = 'D'` não é erro

Estado normal: a nota já estava na `TGFIXN`. Separar de `'E'` importa para a Denise não ver
alarme onde não há problema.

---

## 11. Solução de problemas

### `Token da Anymarket nao configurado`

O parâmetro `ANYMARKET_TOKEN` não existe e o `TOKEN_FALLBACK` está vazio.

### `HTTP 400: O limite mínimo permitido por página é de 5 recursos`

`PAGE_SIZE` menor que 5. A API não aceita.

### `HTTP 401` / `HTTP 403`

Token inválido ou expirado.

### `atualizaFila is not a function`

Já corrigido. Era declaração de função dentro de um bloco `{}` — o Rhino não faz hoisting
para fora de bloco. A função está no topo do arquivo agora.

### `Propriedade 'X' com largura acima do limite: (126 > 100)`

O Sankhya valida largura além de tipo, e a mensagem dá o número exato. Larguras conhecidas:
`AD_TESTENOTA.STATUS` = 100 · Texto/Padrão no Construtor = `VARCHAR(100)` ·
Texto/Caixa de Texto = `VARCHAR2(4000)` · `TGFIXN.NOMEARQUIVO` = 200 ·
`TGFIXN.XNOMEEMIT`/`XNOMEDEST` = 60 · `TGFIXN.CNPJPARC`/`CNPJDEST` = 14.

### `Tipo esperado 'String', tipo recebido 'java.math.BigDecimal'`

Um `setCampo` mandando número onde a coluna é texto. **O valor exibido num export não
revela o tipo** — `CFOPXML` mostra `1202` e é `VARCHAR2`; `ENTSAINFE` mostra `0` e é
`VARCHAR2(1)`.

```sql
SELECT COLUMN_NAME, DATA_TYPE, DATA_LENGTH
FROM USER_TAB_COLUMNS
WHERE TABLE_NAME = 'TGFIXN' AND COLUMN_NAME = 'NOME_DO_CAMPO';
```

Regra: `VARCHAR2`/`CHAR`/`CLOB` → `String(x)` · `NUMBER`/`FLOAT` → `Number(x)` ·
`DATE` → objeto `Date`

### `unterminated string literal`

Aspa curva vinda de copiar/colar, ou `\n` dentro de string. Usar aspas retas.

### Campo novo não aparece na tela

Construtor de Telas → **"Outras Opções"** → **"Reiniciar esta unidade de dados"**.

### Notas ficam em `STATUS = 0` para sempre

O motor não está processando. Verifique se as notas do próprio Paulo (112628–112630)
também estão em `STATUS 0` — se sim, o problema não é do robô. Ver §12.

### Convenções de script nesta base

- **Sucesso:** `mensagem = "texto";` → NÃO cancela a transação. Caixa amarela
- **Erro:** `throw "texto";` → **CANCELA a transação** (rollback, desfaz `save()`
  anteriores). Caixa vermelha. **Nunca usar `throw` depois de gravar**
- SELECT: `getQuery("native")`, placeholder é `{x}`, **não** `?`
- UPDATE: não disponível via script nesta instalação (ver §9)

---

## 12. Pendências

### Bloqueio real

| # | Pendência | Com quem |
|---|---|---|
| 1 | **O motor não processa.** Notas em `STATUS 0`, incluindo as do Paulo (112628–112630). Pendências de `TGFLOCOPER` e `TGFTOP->NFSETIPOPER` | Paulo |

Sem isso não é possível validar que a nota nasce correta — e sem essa validação não se
habilita tipo novo.

### Melhorias discutidas, não implementadas

**Janela de dias (30–40 dias).** A API **não** filtra por data — os únicos campos por
documento são `id`, `url`, `marketplace`, `type` e `subType`, e o parâmetro `createdAfter`
é aceito mas ignorado (testado). Porém a listagem vem **do mais antigo para o mais novo**,
então a solução é **paginar de trás para frente** e cortar pelo `dhEmi` do XML.

Ganhos: notas recentes no primeiro clique, custo por clique deixa de crescer, histórico
antigo não entra sem pedir.

Falta uma sonda para saber se a resposta da API traz `totalElements` ou link `rel:"last"` —
se não trouxer, dá para achar o fim dobrando o offset até vir vazio (~10 chamadas de
listagem, sem baixar XML).

**Lote de verdade.** Subir `LIMITE_LOTE` no botão 1 e selecionar várias linhas no botão 2.
A mecânica já suporta; é só mudar a constante. Faz mais sentido depois da janela de dias.

**Cadastro automático de parceiro.** Levantado o gabarito (parceiros 55500 e 56811, criados
pela Tem Api) e os 68 campos `NOT NULL` da `TGFPAR`. Descoberta importante: **`CODEND` é
obrigatório e está preenchido** — criar parceiro implica criar ou localizar cidade
(`TSICID`, e o XML traz código IBGE, não o `CODCID` do Sankhya), bairro (`TSIBAI`) e
endereço (`TSIEND`). São quatro cadastros em cascata.

Também: `CODPARCMATRIZ` = o próprio `CODPARC`, o que exige gravar, ler o código gerado e
atualizar a própria linha.

**Recomendação:** não construir agora. Paulo afirmou que o motor cadastra o parceiro, e
isso não foi testado porque o motor não roda. Construir um cadastrador para contornar um
problema que talvez não exista é trabalho arriscado — cadastro errado em `TGFPAR` é sujeira
permanente, e não é possível rodar `DELETE`.

Alternativa barata: uma coluna calculada `PARCEXISTE` (mesmo mecanismo do `JANATGFIXN`,
consultando a `TGFPAR`), para a Denise ver quais notas têm parceiro faltando **antes** de
enviar. Para os poucos que faltarem, cadastrar pela tela de Parceiros do Sankhya — que já
sabe criar cidade, bairro e endereço corretamente.

Diagnóstico de 27/08, para dimensionar: dos 98 documentos analisados, os 74 de entrada e
simbólicos tinham **100%** dos parceiros cadastrados. O problema se concentra em
`devolution` e `sale_return` (contraparte CPF, consumidor final).

**Recusar clique sem seleção no botão 2.** Ver §9.

### Perguntas em aberto com o Paulo

| # | Pergunta |
|---|---|
| 2 | `TIPONFE`: existe caso em que o `'D'` é usado? Os dados só mostram `E` e `V` |
| 3 | Se o motor cadastra o parceiro, em que situação ocorre o "CNPJ não encontrado"? |
| 4 | `NOMEARQUIVO` é apenas rótulo? Nada no motor depende do formato? |
| 5 | Permissão de `DELETE` no DBExplorer: é do usuário ou a ferramenta é somente-leitura? |
| 6 | A ação "Importar por Local" (`STP_ALTERALOCALPADRAO_ABC`) altera o local padrão de **todos** os produtos ativos, dá `COMMIT` sozinha, ignora `P_QTDLINHAS` e está **sem controle de acesso**. Ligar o controle ou renomear? |

### Pergunta em aberto com a Denise

| # | Pergunta |
|---|---|
| 7 | O Full é *sempre* importado e *nunca* faturado internamente — é regra formal ou prática observada? Necessário antes de habilitar o tipo `sale` (476 documentos) |

---

## Referência: a chave de acesso da NF-e

44 dígitos. Posições em base 1 (SQL) — em JavaScript, subtraia 1.

| Posições | Conteúdo | Uso |
|---|---|---|
| 1–2 | cUF | — |
| 3–6 | AAMM da emissão | — |
| 7–20 | **CNPJ do emitente** | identificar a BeBaby → `CNPJPARC` |
| 21–22 | modelo (55 = NF-e) | — |
| 23–25 | **série** | → `SERIEDOC` |
| 26–34 | **número da nota** | → `NUMNOTA` |
| 35 | tipo de emissão | — |
| 36–43 | código numérico | — |
| 44 | dígito verificador | — |

Em JavaScript: `ch.substring(6,20)` = CNPJ, `ch.substring(22,25)` = série,
`ch.substring(25,34)` = número.

---

## Contatos

| Pessoa | Papel | Assuntos |
|---|---|---|
| **Paulo Vieira** | Dev terceirizado Sankhya | Motor de importação, TOPs, estrutura fiscal, infra da homologação |
| **Denise** | Operações | Operação da tela; aval sobre importação de notas de venda |
