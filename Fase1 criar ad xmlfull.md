# Fase 1 — Criar a AD_XMLFULL e o botão de import

**Objetivo:** ter a tabela de fila funcionando hoje na base de teste, com um botão que
traz **uma nota por clique** e uma coluna que mostra se a chave já está na `TGFIXN`.

---

## Antes de começar: onde é cada coisa

| Tela | Para quê |
|---|---|
| **Construtor de Telas** (Configurações > Avançado) | **Criar tabela nova.** É aqui que vamos trabalhar |
| Dicionário de Dados (Configurações > Avançado) | Adicionar campos a tabelas que **já existem** (TGFIXN, TGFPAR…) |
| Menu do Sistema | Publicar a tela para aparecer no menu |

É o mesmo caminho que gerou a `AD_TESTENOTA`.

---

## Passo 1 — Criar a tabela

**Construtor de Telas** → botão de inclusão (**+**) → **"Cadastrar Tela Adicional"**
→ selecione **"Tela Mestre"** → **Próximo**.

Preencha:

| Campo | Valor |
|---|---|
| Descrição da Tela | `Fila de XML do Full` |
| Nome da tabela no banco de dados | `XMLFULL` |

⚠️ Digite só `XMLFULL`. O sistema prefixa automaticamente e cria a tabela como
**`AD_XMLFULL`**. Se você digitar `AD_XMLFULL`, vira `AD_AD_XMLFULL`.

**Próximo.**

---

## Passo 2 — Chave primária

Na tela de chave primária, clique em incluir (**+**) e preencha:

| Campo | Valor |
|---|---|
| Nome do campo | `NUFILA` |
| Descrição do campo | `Nº` |
| Tipo de dados | **Número Inteiro** |
| Apresentação | Padrão |
| Permite pesquisa? | marcado |
| Visível no grid de pesquisa? | marcado |

**Próximo** → na tela seguinte, marque **`NUFILA` como auto-numerado** → **Concluir**.

A tabela nasce com a PK. Os outros campos entram na aba **Campos**.

---

## Passo 3 — Os campos

Na aba **Campos**, inclua um por um. A tabela abaixo já traduz os tipos para o que o
Sankhya oferece.

⚠️ **Atenção ao tipo Texto:** a apresentação define o tamanho no banco.
**Padrão** = `VARCHAR(100)` · **Caixa de texto** = `VARCHAR2(4000)` ·
**Lista de opções** = `VARCHAR(10)`

### Campos que a Denise vê

| Nome do campo | Descrição | Tipo de dados | Apresentação |
|---|---|---|---|
| `SITUACAO` | Situação | Texto | **Lista de opções** |
| `DHRECEBIDO` | Recebido em | Data e Hora | Padrão |
| `NOMECLIENTE` | Cliente | Texto | Padrão |
| `NUMNOTA` | Nota | Número Inteiro | Padrão |
| `SERIEDOC` | Série | Número Inteiro | Padrão |
| `VLRNOTA` | Valor | Número Decimal | Padrão |
| `DHEMISS` | Emissão | Data e Hora | Padrão |
| `NATUREZAOPER` | Natureza | Texto | Padrão |
| `TIPODOC` | Tipo do documento | Texto | Padrão |
| `DHTRANSFERIDO` | Enviado em | Data e Hora | Padrão |

### Campos técnicos

| Nome do campo | Descrição | Tipo de dados | Apresentação |
|---|---|---|---|
| `CHAVEACESSO` | Chave NF-e | Texto | Padrão |
| `IDARQUIVO` | ID Anymarket | Texto | Padrão |
| `SUBTIPODOC` | Subtipo | Texto | Padrão |
| `CNPJPARC` | Doc. do cliente | Texto | Padrão |
| `NUARQUIVO` | Nº na TGFIXN | Número Inteiro | Padrão |
| `TENTATIVAS` | Tentativas | Número Inteiro | Padrão |
| `URLORIGEM` | URL de origem | Texto | **Caixa de texto** |
| `MENSAGEM` | Observação | Texto | **Caixa de texto** |
| `XML` | XML | **Texto Longo (CLOB)** | Formatação HTML |

Notas:

- `URLORIGEM` e `MENSAGEM` passam de 100 caracteres, então **Caixa de texto**.
- `XML` como CLOB exige Apresentação **Formatação HTML** (é o que a documentação impõe).
- Nome de campo: até 32 caracteres, sem espaço nem acento.

---

## Passo 4 — As opções do campo `SITUACAO`

Selecione o campo `SITUACAO` na grade. Ao lado de **Apresentação** aparece o botão
**"Opções"**. Cadastre os pares:

| Valor | Descrição |
|---|---|
| `P` | Pendente |
| `T` | Transferido |
| `D` | Duplicado |
| `E` | Erro |

Depois, no botão **"Atributos"** do mesmo campo, marque **"Lista de opções"**.

Assim a Denise vê "Pendente" na grade, e o banco guarda `P`.

---

## Passo 5 — O campo calculado "Já está na TGFIXN?"

Este é o que você pediu. Ele **não existe no banco** — é resolvido por consulta a cada
leitura, então nunca fica desatualizado.

Inclua um campo novo:

| Campo | Valor |
|---|---|
| Nome do campo | `JANATGFIXN` |
| Descrição do campo | `Já está na TGFIXN?` |
| Tipo de dados | **Texto** |
| Apresentação | Padrão |
| **Campo calculado** | **marcado** |

No campo **Expressão**, cole (é **Java**, não JavaScript):

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

Como funciona:

- `$col_CHAVEACESSO` é o valor da coluna na linha atual (campos texto vêm como
  `java.lang.String`)
- `$sql` é o objeto de consulta; a assinatura é
  `select(colunas, tabelas, condicao)` e o placeholder é `{NOME}`
- O tipo devolvido pela expressão **precisa bater** com o Tipo de dados do campo. Como é
  Texto, retornamos String. Devolver número aqui daria
  *"Valor não conversível para campo numérico"*

⚠️ **Se der erro na expressão**, o mais provável é o `$sql.getString(1)`. A documentação
mostra essa forma no artigo do Construtor de Telas e uma variante em minúsculas
(`getstring`) em outro artigo. Se falhar, tente a minúscula.

### Alternativa mais simples, se a expressão der trabalho

Trocar o `Campo calculado` por um campo comum e preencher no botão 2. Perde a atualização
automática, mas funciona. Só que aí ele mostra o estado do momento da transferência, não o
atual.

---

## Passo 6 — Ordem das colunas na grade

Botão **"Outras Opções"** → **"Definir ordem dos campos"**.

Ordem sugerida, pensando na leitura da Denise:

```
NUFILA · SITUACAO · JANATGFIXN · DHRECEBIDO · NOMECLIENTE ·
NUMNOTA · SERIEDOC · VLRNOTA · DHEMISS · NATUREZAOPER ·
TIPODOC · DHTRANSFERIDO · MENSAGEM · (o resto)
```

Aproveite também **"Outras Opções" → "Avançado"** e defina **Campo para apresentação** =
`NOMECLIENTE`. A documentação diz que isso é necessário para tabelas criadas pelo usuário —
sem definir, pesquisas para esta tabela ficam sem descrição.

---

## Passo 7 — Publicar no menu

**Menu do Sistema** → selecione a pasta onde quer (a mesma da `AD_TESTENOTA`, por exemplo)
→ botão **"Criar Novo Item"**:

| Campo | Valor |
|---|---|
| Descrição | `Fila de XML do Full` |
| Tipo | Lançador |
| Tipo Lançador | Tela adicional |
| Tela | `AD_XMLFULL` |

---

## Passo 8 — Criar a ação do botão

Volte ao **Construtor de Telas**, na instância da `AD_XMLFULL`, aba **Ações** → incluir:

| Campo | Valor |
|---|---|
| Descrição | `Buscar na Anymarket` |
| Tipo | **Script (JavaScript)** |
| Controla Acesso | marcado (recomendado) |
| Depois de executar, recarregar | **Recarregar tudo** |

Cole o conteúdo de `botao1_buscar_anymarket.js`.

O **"Recarregar tudo"** importa: sem isso a linha nova não aparece sem F5.

---

## Passo 9 — Se algo não aparecer

Botão **"Outras Opções"** → **"Reiniciar esta unidade de dados"**. Ele descarta o cache
interno e relê a estrutura. É a primeira coisa a tentar quando um campo novo não aparece
na tela.

E **"Checar integridade de campos adicionais"** confirma se todos os campos do dicionário
existem de fato no banco, com o tipo certo.

---

## Passo 10 — Testar

1. Abra a tela **Fila de XML do Full**
2. Clique em **Buscar na Anymarket**
3. Deve aparecer **uma** linha, com situação Pendente
4. Confira: `NOMECLIENTE`, `NUMNOTA`, `VLRNOTA`, `DHEMISS` preenchidos
5. Olhe a coluna **Já está na TGFIXN?** — deve dizer SIM ou NÃO
6. Clique de novo. Deve vir **outra** nota, nunca a mesma

O passo 6 é o teste que importa. Se vier a mesma nota duas vezes, o dedup por `IDARQUIVO`
não está funcionando e vale parar para investigar.

---

## O que fica para a Fase 2

- O botão 2 ("Enviar para TGFIXN")
- A `UNIQUE` em `IDARQUIVO` (rede de segurança contra duplicata; precisa de DDL, então
  talvez do Paulo)
- Liberar acesso à Denise

---

## Anexo — Se preferir criar por DDL

Caso tenha permissão para executar DDL, é mais rápido criar a tabela direto e depois
registrar os campos no Construtor de Telas. Mas **atenção**: o Sankhya precisa conhecer os
campos no dicionário, então criar só por DDL não faz a tela aparecer. O caminho do
Construtor é o recomendado.

```sql
CREATE TABLE AD_XMLFULL (
    NUFILA         NUMBER(10)     NOT NULL,
    SITUACAO       VARCHAR2(10)   DEFAULT 'P' NOT NULL,
    DHRECEBIDO     DATE,
    NOMECLIENTE    VARCHAR2(100),
    NUMNOTA        NUMBER(10),
    SERIEDOC       NUMBER(10),
    VLRNOTA        FLOAT,
    DHEMISS        DATE,
    NATUREZAOPER   VARCHAR2(100),
    TIPODOC        VARCHAR2(100),
    DHTRANSFERIDO  DATE,
    MENSAGEM       VARCHAR2(4000),
    CHAVEACESSO    VARCHAR2(100),
    IDARQUIVO      VARCHAR2(100),
    SUBTIPODOC     VARCHAR2(100),
    CNPJPARC       VARCHAR2(100),
    NUARQUIVO      NUMBER(10),
    TENTATIVAS     NUMBER(10)     DEFAULT 0,
    XML            CLOB,
    CONSTRAINT PK_AD_XMLFULL PRIMARY KEY (NUFILA)
);

CREATE UNIQUE INDEX UK_AD_XMLFULL_IDARQ ON AD_XMLFULL (IDARQUIVO);
CREATE INDEX IX_AD_XMLFULL_SITUACAO ON AD_XMLFULL (SITUACAO, NUFILA);
```

Os tamanhos aqui seguem o que o Construtor de Telas geraria (`VARCHAR(100)` para Texto
Padrão), para as duas rotas ficarem equivalentes.
