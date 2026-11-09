# soliLang

Linguagem e processador para controle de doações: registrar entradas,
consultar estoque, distribuir itens e emitir relatórios.

Projeto acadêmico do Tema Integrador VIII — Projeto de Linguagens.

## Problema

Instituições comunitárias e ONGs que recebem e redistribuem doações
frequentemente controlam o estoque em cadernos ou planilhas manuais. Isso
gera perda de itens, distribuição desigual e falta de transparência.

A soliLang permite registrar as movimentações em uma linguagem simples, com
mensagens de erro claras e estoque sempre consistente.

## Relação com os ODS

- **ODS 10 — Redução das Desigualdades.** A instituição atende pessoas em
  situação de vulnerabilidade e distribui bens doados. Sem controle de
  estoque, itens se perdem, a distribuição fica desigual e não há registro
  do que entrou e do que saiu.
  - **Meta 10.2** (inclusão social de pessoas em vulnerabilidade): o
    registro confiável amplia o alcance real das doações.
  - **Meta 10.3** (igualdade de oportunidades e remoção de barreiras): a
    linguagem substitui o controle manual, que depende de memória e papel,
    por um registro verificável.

## Etapas do processamento

```
arquivo .soli -> Lexer -> Parser -> Análise semântica -> Interpretador
```

O projeto **não gera código de máquina**: é um processador de linguagem com
um interpretador.

## Front-end e back-end

O processador está dividido em duas fases:

| Fase | Módulo | Entrada | Saída | Erro |
|---|---|---|---|---|
| Léxica | `solilang/lexer.py` | texto | tokens | `ErroLexico` |
| Sintática | `solilang/parser.py` | tokens | AST | `ErroSintatico` |
| Semântica | `solilang/semantic.py` | AST | AST validada | `ErroSemantico` |
| Síntese | `solilang/interpreter.py` | AST | saída | — |

**Front-end** = análise (léxica + sintática + semântica). É a parte do
processador que **entende** o programa e é independente do alvo: poderia
alimentar tanto um interpretador quanto um gerador de código.

**Back-end** = síntese. Neste projeto é um **interpretador de árvore**: ele
percorre a AST já validada e executa as instruções, sem produzir artefato.

O `main.py` é o driver do front-end e serve como testador das fases:

```bash
python3 main.py --tokens examples/exemplo.soli   # revela a fase léxica
python3 main.py --ast examples/exemplo.soli      # revela a fase sintática
```

A geração de código não faz parte do escopo; é apontada como trabalho futuro
e entraria exatamente na saída do front-end (a AST), no lugar do
interpretador.

## Como executar

Requisito: Python 3.10 ou superior. Nenhuma dependência externa.

```bash
python3 main.py examples/exemplo.soli
```

Saída (com o estoque vazio, na primeira execução):

```
Consulta ARROZ: 50 unidades
Consulta ARROZ: 40 unidades

------------ Estoque atual ------------
ARROZ: 40 unidades
```

> A partir da segunda execução o estoque acumula: a mesma `DOACAO ARROZ, 50;`
> soma ao saldo anterior. Para repetir a saída acima, apague o arquivo:
> `rm -f estoque/estoque_atual.json`.

Outros modos (úteis para inspecionar as etapas):

```bash
python3 main.py --tokens examples/exemplo.soli   # lista de tokens
python3 main.py --ast examples/exemplo.soli      # árvore sintática
```

Sem argumentos, o programa abre o **modo interativo** (veja a seção abaixo).
O texto de uso aparece quando a chamada é inválida:

```bash
python3 main.py --tokens        # sem o arquivo: mostra o uso
```

### Persistência do estoque

No modo arquivo, o programa usa `estoque/estoque_atual.json` automaticamente. Ele lê o
arquivo antes de executar e grava o resultado depois:

```bash
python3 main.py examples/exemplo.soli
```

O arquivo fixo é `estoque/estoque_atual.json`, criado automaticamente na
pasta `estoque/`. Não existe opção para escolher outro arquivo.

O arquivo é JSON puro e legível:

```json
{
  "ARROZ": 40
}
```

Comportamento:

- **Arquivo inexistente** = estoque vazio (a primeira execução não é erro).
- **Só grava se o programa for válido.** Havendo erro léxico, sintático ou
  semântico, o arquivo não é tocado.
- **Gravação atômica**: escreve em `estoque/estoque_atual.json.tmp` e renomeia no final, então
  uma interrupção no meio não corrompe o estoque.
- **Acumula entre execuções**: uma `DOACAO` de 50 num estoque de 40 resulta em
  90 na execução seguinte.
- **Fora do Git**: `estoque/` está no `.gitignore` — é dado da
  instituição, não código.

Não há SGBD. A persistência em arquivo cobre o caso de uso da instituição sem
trazer a complexidade de banco de dados para dentro do escopo da linguagem.

### Modo interativo (REPL)

Para digitar os comandos diretamente no terminal, execute sem arquivo:

```bash
python3 main.py
```

Também é possível iniciar como módulo Python:

```bash
python3 -m solilang
```

Exemplo de sessão:

```text
soliLang> DOACAO ARROZ, 50;
soliLang> CONSULTA ARROZ;
Consulta ARROZ: 50 unidades
soliLang> DISTRIBUIR ARROZ, 10;
soliLang> RELATORIO;

------------ Estoque atual ------------
ARROZ: 40 unidades
soliLang> :sair
```

Cada linha é envolvida internamente por `INICIO` e `FIM`, então a gramática
original continua a mesma. O estado fica vivo entre as linhas; um erro mostra
a mensagem e não encerra a sessão.

No modo interativo, **`estoque/estoque_atual.json` é usado automaticamente**:
o arquivo é lido ao iniciar a sessão e atualizado depois de cada comando
válido. Use `:ajuda` para listar os comandos especiais e `:sair` para encerrar.

## A linguagem

| Comando | Forma | Efeito |
|---|---|---|
| `INICIO` | `INICIO` | abre o programa |
| `FIM` | `FIM` | fecha o programa |
| `DOACAO` | `DOACAO ITEM, QTD;` | soma unidades (cria o item se não existir) |
| `CONSULTA` | `CONSULTA ITEM;` | mostra o saldo (erro se o item não existir) |
| `DISTRIBUIR` | `DISTRIBUIR ITEM, QTD;` | subtrai unidades (erro se faltar saldo) |
| `RELATORIO` | `RELATORIO;` | mostra o estoque completo |

### Gramática (BNF)

```bnf
<programa>      ::= INICIO <instrucoes> FIM
<instrucoes>    ::= <instrucao> <instrucoes> | ε
<instrucao>     ::= <doacao> | <consulta> | <distribuicao> | <relatorio>
<doacao>        ::= DOACAO ID , NUM ;
<consulta>      ::= CONSULTA ID ;
<distribuicao>  ::= DISTRIBUIR ID , NUM ;
<relatorio>     ::= RELATORIO ;
ID              ::= [A-Z][A-Z0-9_]*
NUM             ::= [0-9]+
```

### Tokens e expressões regulares

| Token | Expressão | Exemplos |
|---|---|---|
| `ID` | `[A-Z][A-Z0-9_]*` | `ARROZ`, `FEIJAO_2`, `ITEM7` |
| `NUM` | `[0-9]+` | `1`, `50`, `2026` |
| `VIRGULA` | `,` | `,` |
| `PONTO_VIRGULA` | `;` | `;` |
| espaços | `[ \t\r\n]+` | ignorados |

### Autômato finito do identificador

```
        [A-Z]              [A-Z] / [0-9] / _
(q0) -----------> ((q1)) ---------------------> ((q1))
inicial           aceitação                     laço
```

O estado inicial só avança ao receber uma letra maiúscula; a partir daí o
autômato permanece em um estado de aceitação enquanto recebe letras
maiúsculas, dígitos ou sublinhado. Qualquer outro símbolo encerra o
reconhecimento — se for um caractere inválido fora desse padrão, o lexer
reporta erro léxico com a linha.

### Expressão EBNF equivalente

```ebnf
programa    = "INICIO" , { instrucao } , "FIM" ;
instrucao   = doacao | consulta | distribuicao | relatorio ;
doacao      = "DOACAO" , identificador , "," , numero , ";" ;
consulta    = "CONSULTA" , identificador , ";" ;
distribuicao= "DISTRIBUIR" , identificador , "," , numero , ";" ;
relatorio   = "RELATORIO" , ";" ;
identificador = letra_maiuscula , { letra_maiuscula | digito | "_" } ;
numero      = digito , { digito } ;
```

## Regras semânticas

1. Quantidades devem ser maiores que zero.
2. `DOACAO` cria o item automaticamente.
3. `CONSULTA` e `DISTRIBUIR` exigem item existente.
4. Não é permitido distribuir mais do que o saldo.
5. O programa inteiro é validado antes da execução: se houver erro, nada é
   aplicado ao estoque.

A validação roda sobre uma cópia do estoque. Assim, ou o programa inteiro é
válido e executa, ou nada executa — não existe estado parcial.

## Decisões de projeto

| # | Decisão | Escolha |
|---|---|---|
| 1 | Extensão dos fontes | `.soli` |
| 2 | Identificador | `[A-Z][A-Z0-9_]*` — só maiúsculas |
| 3 | Quantidade | inteiro positivo, validado `> 0` na semântica |
| 4 | `DOACAO` de item novo | cria o item automaticamente |
| 5 | `CONSULTA` de item inexistente | erro semântico com a linha |
| 6 | Programa vazio (`INICIO FIM`) | aceito (produção ε) |
| 7 | Erro no meio do programa | erro semântico; nada é executado |
| 8 | Saída | `DOACAO`/`DISTRIBUIR` silenciosos; `CONSULTA` e `RELATORIO` imprimem |
| 9 | Estoque | dicionário em memória; persistência fixa em `estoque/estoque_atual.json` |
| 10 | Comentários na linguagem | não existem (escopo mínimo) |

## Estrutura do projeto

```
main.py                 driver do front-end, modos de inspeção e persistência
solilang/tokens.py      [front-end] tipos de token e palavras reservadas
solilang/errors.py      [front-end] erros com número de linha
solilang/lexer.py       [front-end] texto -> tokens
solilang/ast_nodes.py   [front-end] nós da árvore sintática
solilang/parser.py      [front-end] tokens -> AST (descendente recursivo)
solilang/semantic.py    [front-end] regras de negócio
solilang/interpreter.py [back-end]  execução e saída
solilang/repl.py        interface interativa do terminal
solilang/__main__.py    entrada para `python3 -m solilang`
examples/               programas de exemplo (válidos e inválidos)
tests/test_lexer.py     tokens, linhas e erros léxicos
tests/test_parser.py    gramática, AST e erros sintáticos
tests/test_semantic.py  regras de estoque e erros semânticos
tests/test_interpreter.py  saída do interpretador
tests/test_estoque.py   persistência em JSON
tests/test_repl.py      modo interativo
```

## Testes

```bash
python3 -m unittest discover -v
```

São 41 testes: 7 de lexer, 9 de parser, 9 de semântica, 4 de integração,
7 de persistência em arquivo e 5 do modo interativo.

## Exemplos de erro

| Arquivo | Erro |
|---|---|
| `examples/erro_lexico.soli` | `Erro léxico na linha 2: identificador inválido começando com 'a'; use letra maiúscula (ex.: ARROZ)` |
| `examples/erro_sintatico.soli` | `Erro sintático na linha 2: esperado ',' após o identificador, mas encontrado '50' (NUM)` |
| `examples/erro_semantico.soli` | `Erro semântico na linha 3: estoque insuficiente para ARROZ: solicitado 10, disponível 4` |

Rode qualquer um deles para ver a mensagem completa:

```bash
python3 main.py examples/erro_semantico.soli
```

## Limitações

- Sem SGBD: a persistência é um arquivo JSON fixo (`estoque/estoque_atual.json`), não um
  banco de dados relacional.
- Sem funções, laços, variáveis ou comentários na linguagem.
- Sem interface gráfica, autenticação ou controle de usuários.
- Sem geração de código: o back-end é um interpretador de árvore.
- A linguagem é de domínio específico para gestão de doações; não é uma
  linguagem de Comunicação Alternativa e Aumentativa (CAA).

## Uso de IA generativa

O apoio de IA foi usado na organização do material, na revisão de mensagens
de erro e na explicação dos conceitos de análise léxica, sintática e
semântica. Os integrantes leram, executaram e explicaram cada módulo, e a
suíte de testes é executada localmente antes de cada commit.

A pesquisa de campo é realizada diretamente com a instituição parceira e não
é gerada por IA.

O registro completo — ferramenta, plano, modelo, prompts e evidências — está
em [`USO_DE_IA.md`](/docs/ia/USO_DE_IA.md).
