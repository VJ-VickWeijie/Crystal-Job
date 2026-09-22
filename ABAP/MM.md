你是资深的SAP HANA 开发人员，
以下是代码片段，我调用了BAPI_PO_CREATE1，在lt_cond和lt_condx里面只传入了一个PB00的Condition。但是执行完之后只会出现一个P101的Condition TYpe，没有PB00的Conditon Type。

- 是不是Exporting里面的no_price_from_po 影响到了？如果是，具体是怎么影响的，原理是什么？
- 这个情况不是必然发生的，因为有些时候，会出现P101 和 PB00两个内容。

``` ABAP
CALL FUNCTION 'BAPI_PO_CREATE1'
    EXPORTING
      poheader          = ls_poheader
      poheaderx         = ls_poheaderx
      no_price_from_po  = 'X'
    IMPORTING
      exppurchaseorder  = lv_exppurchaseorder
      expheader         = ls_expheader
      exppoexpimpheader = ls_exppoexpimpheader
    TABLES
      return            = lt_return
      poitem            = lt_poitem
      poitemx           = lt_poitemx
      poschedule        = lt_sch
      poschedulex       = lt_schx
      pocond            = lt_cond
      pocondx           = lt_condx
      popartner         = lt_popartner
```

--这个PO的Condition Type在T683S查到的顺序是如下

| KAPPL | KALSM | STUNR | ZAEHK | KSCHL | STUNB | STUN2 | KAUTO | KOBED | KZWIW | KSTAT | KOFRM
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | 
| M |	ZRM201 | 50 |     | P101 | 		 | 	  |   | 	 | 	 | 	 |     | 
| M |	ZRM201 | 51 | 1   | PB00 | 		 | 	  | 	| 	 | 	 |   | 900 | 
| M |	ZRM201 | 51 | 2   | PBXX | 		 | 	  |	  | 5  | 	 | 	 | 900 | 
| M |	ZRM201 | 90 |     | 		 | 		 | 	  |   | 	 | 	 | 	 | 901 | 
| M |	ZRM201 | 100 |    | ZY98 | 90  | 90 |   | 	 | 	 | 	 | 900 | 
| M |	ZRM201 | 110 | 10 | FRB1 | 		 | 	  | X | 	 | 	 | 	 | 900 | 
| M |	ZRM201 | 110 | 20 | FRC1 | 		 | 	  | X | 	 | 	 | 	 | 900 | 
| M |	ZRM201 | 200 |    |  		 | 		 |    |   | 	 | 9 |   | 901 | 
| M |	ZRM201 | 301 |    |  		 | 100 |    |   | 	 | 	 | X | 901 | 
| M |	ZRM201 | 302 |    |  		 | 50	 | 	  |   | 	 | 	 | X | 901 | 
| M |	ZRM201 | 303 |    |  		 | 110 | 	  |	  | 	 | 	 | X | 901 | 
| M |	ZRM201 | 400 |    |  		 | 200 |    |   | 	 | S |   | 901 | 
| M |	ZRM201 | 500 |    | GRWR | 400 |    |   | 	 | C | 	 | 900 | 


``` abap
form frm_kondi_wert_900.                                         1
  check: ( xkomv-waers = 'EURC' and komk-waerk = 'EUR' )
      or ( xkomv-waers = 'USDC' and komk-waerk = 'USD' )
      or ( xkomv-waers = 'HKDC' and komk-waerk = 'HKD' )
      or ( xkomv-waers = 'CNYC' and komk-waerk = 'CNY' ).
  data: xkbetr like xkomv-kbetr.
  xkbetr = xkomv-kbetr.
  if komp-shkzg ne space.
    arbfeld = xkbetr * -1.
    xkbetr = arbfeld.
  endif.
  if xkomv-kpein ne 0.
    arbfeld = xkomv-kawrt * xkbetr / xkomv-kpein.
  else.
    arbfeld = xkomv-kawrt * xkbetr.
  endif.
  arbfeld = arbfeld / 100000.
  xkwert = arbfeld.
  xkomv-kkurs = komp-kursk.
endform.
```
