# DT - Dialog Text (SE61)

[![SAP ABAP](https://img.shields.io/badge/SAP-ABAP-0F6CBD?style=flat-square)](https://www.sap.com/) [![SE61](https://img.shields.io/badge/SE61-Document%20Maintenance-2F6FED?style=flat-square)](https://help.sap.com/) [![DT](https://img.shields.io/badge/DT-Dialog%20Text-6F42C1?style=flat-square)](https://help.sap.com/)

Referência técnica sobre o Document Class `DT` do SE61, com foco em uso prático em ABAP, comportamento do SAP, boas práticas e pontos que foram confirmados em sistema.

## Resumo executivo

- `DT` é um Document Class do SE61 usado para textos curtos de ajuda e confirmação.
- O objeto é armazenado em estruturas de documentação SAP, e não no mecanismo `SO10` / `READ_TEXT` / `SAVE_TEXT`.
- É consumido em cenários como `POPUP_TO_CONFIRM` usando `diagnose_object`.
- O mecanismo envolve `DOCU_GET_FOR_F1HELP` e `DOCU_GET`.
- O padrão é útil para tradução, parametrização e manutenção fora do código ABAP.

## O que é `DT`

`DT` é um dos `Document Class` disponíveis na transação **SE61** (Document Maintenance). Em termos práticos, ele representa um texto de diálogo ou ajuda associado a um objeto de documentação.

A identificação do objeto normalmente acontece por meio de `DOKHL-OBJECT` (`TYPE dokhl-object`), e o texto fica armazenado em estruturas de documentação do SAP, como `DOKHL` (header) e `DOKTL` / `TLINE` (linhas). Isso é diferente do mecanismo de textos longos usado pelo `SO10`, que trabalha com `STXH` / `READ_TEXT` / `SAVE_TEXT` e é outra infraestrutura.

### Diferenciação importante

- `DT` = texto de documento / diálogo dentro do ambiente SE61
- `SO10` = texto de objeto de texto (text object), com modelo diferente
- `READ_TEXT` / `SAVE_TEXT` = acesso ao mecanismo STXH, não ao Document Class DT

## Exemplo real de uso em ABAP

```abap
CONSTANTS gc_docu_object TYPE dokhl-object VALUE 'Z<AREA>_<ASSUNTO>' ##NO_TEXT.

METHOD my_method.
  DATA lt_param TYPE STANDARD TABLE OF spar.

  lt_param = VALUE #( ( param = 'COUNT' value = |{ lv_count }| ) ).

  CALL FUNCTION 'POPUP_TO_CONFIRM'
    EXPORTING
      titlebar              = TEXT-001
      diagnose_object       = gc_docu_object
      text_question         = space
      default_button        = '2'
      display_cancel_button = abap_false
    IMPORTING
      answer                = lv_answer
    TABLES
      parameter             = lt_param.

  rv_confirmed = xsdbool( lv_answer = '1' ).
ENDMETHOD.
```

Neste padrão, o campo `text_question` fica em branco porque o texto inteiro é carregado do objeto `DT`, e não montado em código ABAP.

## Como o mecanismo funciona

A validação abaixo foi feita a partir da leitura do comportamento dos FMs envolvidos e da análise dos pontos confirmados no sistema.

### Fluxo confirmado do `POPUP_TO_CONFIRM`

1. Quando `diagnose_object` não está vazio, o FM ativa o mecanismo de leitura de documentação.
2. Ele chama `DOCU_GET_FOR_F1HELP` com:
   - `id = docu_id_dialog_text`
   - `langu = sy-langu`
   - `object = diagnose_object`
3. `DOCU_GET_FOR_F1HELP` normaliza a informação e repassa para `DOCU_GET`.
4. Se a busca falhar e o idioma solicitado não for `EN`, o SAP tenta novamente em `LANGU = 'E'` antes de encerrar.
5. Se nada for encontrado, o popup cai em um texto genérico interno e não quebra por ausência do objeto DT.
6. Quando o texto existe, o SAP resolve:
   - includes
   - estruturas condicionais (`IF`, `ELSE`, `CASE`)
   - remoção de comandos de formatação (`tdformat = '/:'`)
7. Se a tabela `PARAMETER` estiver preenchida, ocorre a substituição de tokens como `&COUNT&` pelo valor correspondente.
8. Os botões padrão de confirmação são localizados pelo SAP, sem a necessidade de codificar textos fixos.

### Comportamento importante sobre parâmetros

A substituição de parâmetros é feita por processamento interno do FM. Em prática, isso significa que um texto no SE61 pode ser escrito como:

```text
&COUNT& registros foram encontrados.
```

E o código passa um parâmetro `COUNT = lv_count`, o que permite que o SAP substitua o token dinamicamente.

### F1 Help e objeto auxiliar

Existe também um parâmetro independente `userdefined_f1_help`, do tipo `dokhl-object`. Quando preenchido, ele pode abrir outro objeto de documentação para suporte adicional, sem sobrecarregar o texto principal.

## Vantagens desse padrão

Em comparação com hardcode em ABAP ou textos fixos em `TEXTLINE1-3`, o uso de `DT` oferece benefícios claros:

- Tradução via SE63 sem alteração no código ABAP
- Parametrização sem concatenação manual
- Texto fora do código-fonte, facilitando revisão funcional
- Reuso em diversos pontos do sistema
- Menor acoplamento entre lógica e texto de interface
- Melhor suporte para internacionalização

## Como criar e manter um objeto `DT`

1. Acesse **SE61**.
2. Selecione **Document Class = Dialog Text (DT)**.
3. Informe o nome, em convenção comum como `Z<AREA>_<ASSUNTO>`.
4. Crie o texto e salve.
5. Para preservar quebras e formatação de forma mais confiável, a importação via **XML ITF** costuma ser mais segura.
6. Verifique o status do objeto. Se ele ficar em `R` (Revision), pode afetar a disponibilidade para tradução.
7. Depois, use **SE63** para tradução, se necessário.
8. No transporte, lembre-se de que esse objeto é tratado como objeto de documentação, e não como texto clássico de programa.

## Traps e cuidados

### 1. `DT` não é o mesmo que `SO10`

Os objetos de documentação e os textos de texto objeto têm mecanismos diferentes. Isso é importante para evitar a confusão entre:

- `DT` / SE61 / `DOCU_GET`
- `SO10` / `READ_TEXT` / `SAVE_TEXT`

### 2. Status do objeto pode afetar a tradução

Se o objeto não está em status ativo, a tradução pode não aparecer como esperada no SE63.

### 3. Tokens precisam ser tratados com cuidado

Se um texto conter tokens como `&COUNT&`, o processamento de parâmetros precisa estar corretamente montado. Caso contrário, o texto pode aparecer com placeholder em vez do valor real.

### 4. Não confundir `text_question` com “texto do popup montado no código”

No padrão DT, o texto principal do popup normalmente vem do objeto de documentação, não do código fonte.

## Outros consumos confirmados

Além de `POPUP_TO_CONFIRM`, o mecanismo de documentação também é usado por funções que acessam `DOCU_GET_FOR_F1HELP` e `DOCU_GET`.

Há também casos em que outros FMs usam texto fixo em variáveis tipo `TEXTLINE1/2/3`; nesses casos, eles não têm relação direta com `DT`.

## Referências e documentação

- [ITF/OTF Format — SAP Help Portal](https://help.sap.com/doc/saphelp_gbt10/1.0/en-US/4e/16819fb84a1a27e10000000a42189e/content.htm?no_cache=true)
- [Long Text Editor — SAP Help Portal](https://help.sap.com/doc/saphelp_nw74/7.4.16/en-us/4d/100d48d6a5606be10000000a42189e/content.htm?no_cache=true)
- [Anatomy of a Function Module: POPUP_TO_CONFIRM (Part 1) — SAP Community](https://community.sap.com/t5/application-development-and-automation-blog-posts/anatomy-of-a-function-module-popup-to-confirm-part-1/ba-p/13562360)
- [Anatomy of a Function Module: POPUP_TO_CONFIRM — SAP Community](https://community.sap.com/t5/abap-blog-posts/anatomy-of-a-function-module-popup-to-confirm/ba-p/14240105)
- [POPUP_TO_CONFIRM — sapdatasheet.org](https://www.sapdatasheet.org/abap/func/popup_to_confirm.html)
- [DOCU_GET_FOR_F1HELP — sapdatasheet.org](https://www.sapdatasheet.org/abap/func/docu_get_for_f1help.html)
- [Create a Transport for SE61 Dialog Text — SAP Community](https://answers.sap.com/questions/7000870/create-a-transport-for-se61-dialog-text.html)
- [Dialog text — SAP Community Q&A](https://answers.sap.com/questions/1009487/dialog-text.html)
- [DOKHL — Documentation: Headers](https://community.sap.com/t5/application-development-discussions/dokhl-documentations-headers-table/td-p/3536094)

## Observação final

Este repositório tem valor como base de conhecimento técnico sobre um comportamento específico do SAP ABAP. O material foi organizado para servir como referência consultiva, com foco em clareza, uso prático e rastreabilidade de comportamento observado no sistema.

---

Se você quiser aprofundar o material, uma próxima etapa útil pode ser:

- adicionar um diagrama do fluxo de leitura do texto;
- separar “confirmado” e “inferido” em blocos específicos;
- criar uma seção de exemplos reais de uso em telas, popup e integração com tradução.
