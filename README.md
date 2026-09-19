# DT - Dialog Text (SE61)

[![SAP ABAP](https://img.shields.io/badge/SAP-ABAP-0F6CBD?style=flat-square)](https://www.sap.com/) [![SE61](https://img.shields.io/badge/SE61-Document%20Maintenance-2F6FED?style=flat-square)](https://help.sap.com/) [![DT](https://img.shields.io/badge/DT-Dialog%20Text-6F42C1?style=flat-square)](https://help.sap.com/)

Esse repositório reúne anotações e observações sobre o `DT` do SE61, com foco em como ele funciona na prática em ABAP e em situações reais de uso.

## Por que esse repositório existe

A ideia aqui é guardar uma referência prática sobre um padrão que costuma aparecer em projetos SAP, mas nem sempre é documentado de forma clara: usar textos de documentação do SE61 como parte do fluxo de interface e confirmação em ABAP.

Ele serve como material de consulta rápida para:

- entender o conceito de `DT`
- revisar como o SAP busca esse texto em runtime
- lembrar como isso se relaciona com `POPUP_TO_CONFIRM`
- evitar confusão com `SO10` / `READ_TEXT` / `SAVE_TEXT`

## Resumo rápido

- `DT` é um `Document Class` do SE61
- normalmente é usado para textos de ajuda, confirmação e diálogo
- o texto é armazenado em estruturas de documentação do SAP
- ele não é o mesmo mecanismo do `SO10`
- em runtime, o SAP o busca por meio de `DOCU_GET_FOR_F1HELP` e `DOCU_GET`

## O que é `DT`

`DT` é um dos `Document Class` do SE61. Em termos simples, é um texto de diálogo/ajuda armazenado como objeto de documentação do SAP.

O objeto fica identificado por `DOKHL-OBJECT` (`TYPE dokhl-object`) e o conteúdo fica em estruturas como `DOKHL` e `DOKTL`/`TLINE`. Isso é diferente do `SO10`, que usa `STXH` / `READ_TEXT` / `SAVE_TEXT` e tem outro mecanismo.

### Diferença rápida

- `DT` = texto de documentação/dialog do SE61
- `SO10` = texto de objeto de texto, com outro modelo
- `READ_TEXT` / `SAVE_TEXT` = acesso ao mecanismo STXH

## Exemplo prático em ABAP

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

## Como isso funciona na prática

O ponto mais interessante aqui é que o SAP não precisa “montar” o texto no código. Ele busca o texto em um objeto `DT` e aplica a lógica de apresentação por conta própria.

### Fluxo que costuma acontecer

1. `POPUP_TO_CONFIRM` recebe `diagnose_object`.
2. Ele chama `DOCU_GET_FOR_F1HELP`.
3. Esse FM repassa a busca para `DOCU_GET`.
4. Se não encontrar no idioma atual, tenta fallback em inglês.
5. Se encontrar, resolve includes, condicionais e parâmetros.
6. O texto final aparece no popup.

### O papel dos parâmetros

O texto pode ter tokens como:

```text
&COUNT& registros foram encontrados.
```

E no código, você passa o valor do parâmetro `COUNT`. Isso é bem útil porque evita concatenação manual e facilita tradução.

### Um detalhe útil

Existe também o parâmetro `userdefined_f1_help`, que pode abrir outro objeto de documentação. Não é o caso do uso mais simples, mas é uma possibilidade real quando o texto principal precisa de apoio extra.

## Por que usar esse padrão?

Em comparação com misturar texto direto no código ou usar `TEXTLINE1-3`, esse modelo tem algumas vantagens bem claras:

- tradução fica mais simples
- o texto vive fora do código
- menos concatenação manual
- melhor manutenção por funcional e por tradução
- menos risco de quebrar i18n por ordem de palavras diferentes entre idiomas

## Como criar um objeto `DT`

1. Abrir **SE61**.
2. Escolher **Document Class = Dialog Text (DT)**.
3. Inserir nome, normalmente algo como `Z<AREA>_<ASSUNTO>`.
4. Criar o texto.
5. Revisar o status do objeto, porque isso pode interferir em tradução.
6. Usar **SE63** quando a tradução for necessária.
7. Verificar transporte e integração como qualquer outra documentação SAP.

## Coisas que costumam confundir

### `DT` não é `SO10`

Esse é o ponto mais comum. O mecanismo é diferente, e a forma de armazenamento também.

### Texto do popup não precisa vir do código

No padrão `DT`, o texto principal vem do objeto de documentação, não de `TEXT-001` ou concatenação dentro do método.

### Tokens precisam ser montados corretamente

Se você usar `&COUNT&`, por exemplo, precisa passar o parâmetro correspondente para que o SAP faça a substituição.

## Outros casos que vale conhecer

Além do `POPUP_TO_CONFIRM`, o mecanismo de documentação é usado por funções que acessam `DOCU_GET_FOR_F1HELP` e `DOCU_GET`.

Também existe o caso de FMs que usam `TEXTLINE1/2/3` fixos, mas isso é outro padrão e não é o mesmo que `DT`.

## Referências

- [ITF/OTF Format — SAP Help Portal](https://help.sap.com/doc/saphelp_gbt10/1.0/en-US/4e/16819fb84a1a27e10000000a42189e/content.htm?no_cache=true)
- [Long Text Editor — SAP Help Portal](https://help.sap.com/doc/saphelp_nw74/7.4.16/en-us/4d/100d48d6a5606be10000000a42189e/content.htm?no_cache=true)
- [Anatomy of a Function Module: POPUP_TO_CONFIRM (Part 1) — SAP Community](https://community.sap.com/t5/application-development-and-automation-blog-posts/anatomy-of-a-function-module-popup-to-confirm-part-1/ba-p/13562360)
- [Anatomy of a Function Module: POPUP_TO_CONFIRM — SAP Community](https://community.sap.com/t5/abap-blog-posts/anatomy-of-a-function-module-popup-to-confirm/ba-p/14240105)
- [POPUP_TO_CONFIRM — sapdatasheet.org](https://www.sapdatasheet.org/abap/func/popup_to_confirm.html)
- [DOCU_GET_FOR_F1HELP — sapdatasheet.org](https://www.sapdatasheet.org/abap/func/docu_get_for_f1help.html)
- [Create a Transport for SE61 Dialog Text — SAP Community](https://answers.sap.com/questions/7000870/create-a-transport-for-se61-dialog-text.html)
- [Dialog text — SAP Community Q&A](https://answers.sap.com/questions/1009487/dialog-text.html)
- [DOKHL — Documentation: Headers](https://community.sap.com/t5/application-development-discussions/dokhl-documentations-headers-table/td-p/3536094)

## Fechamento

Esse repositório funciona bem como uma referência técnica leve: não é um manual oficial da SAP, mas é um material útil para entender o comportamento real do `DT` em ABAP e como ele se encaixa na prática.

A ideia aqui é simples: guardar conhecimento de forma clara, acessível e reaproveitável.

---

Se quiser continuar, os próximos passos mais úteis seriam:

- criar um pequeno diagrama do fluxo de leitura do texto;
- separar melhor o que foi confirmado do que foi inferido;
- adicionar uma seção de exemplos reais de uso em popup e tradução.
