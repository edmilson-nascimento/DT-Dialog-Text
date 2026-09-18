# DT-Dialog-Text
# DT – Dialog Text (SE61)

Referência técnica sobre o Document Class **DT (Dialog Text)** do SE61: como é consumido em ABAP, vantagens, como criar/manter e outros pontos de uso confirmados no sistema.

## 1. O que é

`DT` é um dos "Document Class" (`DOKHL-ID`) da transação **SE61** (Document Maintenance). Um objeto DT é um texto de diálogo — pequeno texto de ajuda/pergunta — identificado por `DOKHL-OBJECT` (`TYPE dokhl-object`), armazenado nas tabelas `DOKHL` (header) e `DOKTL`/`TLINE` (linhas), **não** relacionado ao SO10 (que usa `STXH`/`READ_TEXT`/`SAVE_TEXT`, mecanismo totalmente separado).

## 2. Exemplo de uso: `POPUP_TO_CONFIRM` com objeto DT

```abap
CONSTANTS gc_docu_object TYPE dokhl-object VALUE 'Z<AREA>_<ASSUNTO>' ##NO_TEXT.

METHOD my_method.
  ...
  lt_param = VALUE #( ( param = 'COUNT' value = |{ lv_count }| ) ).

  CALL FUNCTION 'POPUP_TO_CONFIRM'
    EXPORTING
      titlebar              = TEXT-001
      diagnose_object        = gc_docu_object
      text_question           = space
      default_button         = '2'
      display_cancel_button = abap_false
    IMPORTING
      answer                 = lv_answer
    TABLES
      parameter              = lt_param.

  rv_confirmed = xsdbool( lv_answer = '1' ).
ENDMETHOD.
```

`text_question = space` porque o texto vem inteiro do objeto DT `Z<AREA>_<ASSUNTO>`, não de código.

### Mecanismo interno (confirmado lendo a fonte de `POPUP_TO_CONFIRM`, grupo de função `SPO1`)

1. `IF diagnose_object NE space.` → chama `DOCU_GET_FOR_F1HELP` com `id = docu_id_dialog_text` (constante interna do grupo = `'DT'`), `langu = sy-langu`, `object = diagnose_object`.
2. `DOCU_GET_FOR_F1HELP` (grupo `SDOH`) normaliza `TYP` e repassa para o FM genérico **`DOCU_GET`**. Se a busca falhar e o idioma pedido **não for EN**, ele automaticamente **tenta de novo em `LANGU = 'E'`** antes de desistir — inglês funciona como idioma de fallback para documentação SE61 sem tradução.
3. Se não achar nada (`sy-subrc = 4`), `POPUP_TO_CONFIRM` cai para um texto genérico interno (`text-101`) — nunca quebra por objeto DT inexistente.
4. Se achar, resolve textos incluídos (`TEXT_INCLUDE_REPLACE` — cobre o caso de um texto customizado por cliente, que vira um "extension document" enquanto o original só referencia) e elementos de controle IF/ELSE/CASE (`TEXT_CONTROL_REPLACE`), depois remove linhas de comando (`tdformat = '/:'`).
5. **Substituição de parâmetros**: se a tabela `PARAMETER` (`TYPE spar`, campos `PARAM`/`VALUE`) não estiver vazia, roda `insert_params`, que troca tokens `&PARAM&` no texto pelo `VALUE` correspondente — é assim que `&COUNT&` vira o número de registros prontos no nosso caso.
6. **Botões Sim/Não**: `TEXT_BUTTON_1`/`TEXT_BUTTON_2` têm default `'Ja'(001)`/`'Nein'(002)` — são os próprios text symbols do function group `SPO1`, já traduzidos pelo SAP padrão. Por isso `CONFIRM_UPLOAD` não passa esses parâmetros: ganha Sim/Não (ou Yes/No) localizado de graça.
7. **`userdefined_f1_help`** é um segundo parâmetro independente, também `LIKE dokhl-object`: se preenchido, adiciona um botão extra "Info" (`ICON_INFORMATION`) que abre OUTRO objeto DT — não é usado em `CONFIRM_UPLOAD` hoje, mas é uma capacidade disponível se um dia for preciso dar um help mais longo sem lotar o texto principal.

## 3. Vantagens desse padrão (vs. hardcode em ABAP / `TEXTLINE1-3`)

- **Tradução via SE63** sem tocar em código ABAP nem abrir transporte de classe — o texto é objeto próprio, traduzido como qualquer outro texto longo SAP.
- **Parametrização seletiva** (`&COUNT&` etc.) evita concatenação de string manual, que é frágil para i18n (ordem de palavras muda entre idiomas).
- **Botões padrão localizados automaticamente**, sem manter TEXT-nnn próprios para "Sim"/"Não".
- **Conteúdo mantido fora do código-fonte**, por quem tem acesso a SE61 mas não necessariamente a SE80/SE24 — útil para funcional revisar/ajustar texto sem passar por dev.
- **Reaproveitável**: o mesmo objeto DT pode ser lido por qualquer outro ponto do código via `DOCU_GET_FOR_F1HELP`/`DOCU_GET`, não fica preso ao `POPUP_TO_CONFIRM`.

## 4. Como criar e manter um objeto DT

1. **SE61** → Document Class = `Dialog Text` (`DT`) → informar o nome (convenção comum: `Z<área>_<assunto>`) → **Create**.
2. Editar o texto: dá para digitar direto no editor, mas para preservar quebras de linha/formatação de forma confiável é melhor colar via **XML ITF** (import). Nesse XML, um `&` literal (parte de um token `&COUNT&`) precisa ser escapado como `&amp;` — senão o parser de XML quebra ou o token não sobrevive ao roundtrip.
3. **Gotcha de status (`DOKHL-DOKSTATE`)**: um objeto pode ficar em `R` (Revision) em vez de `A` (Active) dependendo de como foi salvo/importado. **SE63 só traduz objetos com status Active** — se a tradução não aparecer disponível, voltar ao SE61, reabrir o objeto e salvar de novo até o status virar Active.
4. **Tradução (SE63)**: menu de tradução de textos longos, entrar com o objeto/idioma de destino. Segue o mesmo fallback para EN descrito no item 2 da seção anterior caso a tradução ainda não exista.
5. **Transporte**: objeto DT é **cross-client**, viaja como `R3TR DOCV DT<nome>` (confirmado via SAP Community, ver referências) — precisa estar em request próprio como qualquer outro objeto de transporte, é fácil esquecer porque não aparece junto da classe ABAP no mesmo request automaticamente.

## 5. Outras funções que usam esse mecanismo

Confirmado lendo a fonte diretamente (sem busca de código-fonte disponível neste sistema — `SAPSearch(searchType="source_code")` retorna erro `SADT_REST 020` aqui, então a verificação foi objeto a objeto):

- **`POPUP_TO_CONFIRM`** (grupo `SPO1`) — confirmado, é o consumidor documentado na seção 2.
- **Descartados** (lidos e confirmados que NÃO usam `DOKHL`/objeto DT — usam `TEXTLINE1/2/3` fixos):
  - `POPUP_TO_CONFIRM_STEP`
  - `POPUP_TO_DECIDE`
  - `POPUP_TO_DECIDE_LIST`
- **`DOCU_GET_FOR_F1HELP`** (grupo `SDOH`) e **`DOCU_GET`** — o mecanismo genérico por trás de `POPUP_TO_CONFIRM`. Não são exclusivos de DT: com `ID` diferente, leem qualquer Document Class (`RE` relatório, `DE` data element, `SD` palavra-chave ABAP etc.). Podem ser chamados diretamente de código customizado para reaproveitar o texto de um objeto DT fora de um popup (ex. exibir o mesmo texto numa tela, log, e-mail).

Se mais consumidores forem confirmados depois (ex. lendo outro FM específico), atualizar esta seção.

## 6. Referências / documentação SAP

Só links específicos sobre os objetos/funções tratados aqui (nada genérico):

- [ITF/OTF Format — SAP Help Portal](https://help.sap.com/doc/saphelp_gbt10/1.0/en-US/4e/16819fb84a1a27e10000000a42189e/content.htm?no_cache=true) — formato ITF usado pelos textos SE61/SAPscript, base do XML de import citado na seção 4.
- [Long Text Editor — SAP Help Portal](https://help.sap.com/doc/saphelp_nw74/7.4.16/en-us/4d/100d48d6a5606be10000000a42189e/content.htm?no_cache=true) — editor usado pelo SE61 para textos longos.
- [Anatomy of a Function Module: POPUP_TO_CONFIRM (Part 1) — SAP Community](https://community.sap.com/t5/application-development-and-automation-blog-posts/anatomy-of-a-function-module-popup-to-confirm-part-1/ba-p/13562360)
- [Anatomy of a Function Module: POPUP_TO_CONFIRM — SAP Community](https://community.sap.com/t5/abap-blog-posts/anatomy-of-a-function-module-popup-to-confirm/ba-p/14240105)
- [POPUP_TO_CONFIRM — referência técnica (sapdatasheet.org)](https://www.sapdatasheet.org/abap/func/popup_to_confirm.html)
- [DOCU_GET_FOR_F1HELP — discussão de uso (SAP Community)](https://community.sap.com/t5/application-development-discussions/regarding-use-of-fm-docu-get-for-f1help/td-p/1654756)
- [DOCU_GET_FOR_F1HELP — referência técnica (sapdatasheet.org)](https://www.sapdatasheet.org/abap/func/docu_get_for_f1help.html)
- [Create a Transport for SE61 Dialog Text — SAP Community](https://answers.sap.com/questions/7000870/create-a-transport-for-se61-dialog-text.html) — confirma `R3TR DOCV DT***` e o comportamento cross-client citado na seção 4.
- [Dialog text — SAP Community Q&A](https://answers.sap.com/questions/1009487/dialog-text.html)
- [DOKHL — Documentation: Headers — discussão (SAP Community)](https://community.sap.com/t5/application-development-discussions/dokhl-documentations-headers-table/td-p/3536094)
- [DOKHL — referência de tabela (leanx.eu)](https://leanx.eu/en/sap/table/dokhl.html)
