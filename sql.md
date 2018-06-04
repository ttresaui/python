```sql
#提取医嘱相关信息
create table ORDERS1 as
select a.ID,a.REGION,b.ORDER_TEXT,b.ENTER_DATE_TIME 
from TABLEA a, INP_ORDERS b where a.ID=b.ID and a.REGION=b.REGION

#建立子表HCC_ORDERS_AZ
create table HCC_ORDERS_AZ as
select * from ORDERS1 a where 
a.ORDER_TEXT in (select b.DRUG1 from ,DRUG_DICT_HC b ) 
or a.ORDER_TEXT in (select b.DRUG2 from ,DRUG_DICT_HC b ) 
or a.ORDER_TEXT in (select b.DRUG3 from ,DRUG_DICT_HC b ) 
or a.ORDER_TEXT in (select b.DRUG4 from ,DRUG_DICT_HC b ) 
```
