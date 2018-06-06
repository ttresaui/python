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


```sql
merge into HCC_ID_AZ a
using
(select * from
  (select row_column() over(partition by b.id order by b.date_of_birth)rn,b.*
   from TEM_PAT_MASTER_INDEX_L_CQ b)
 where rn=1) c
 on (a.id=c.id and a.region='重庆')
 when matched then
  update set a.date_of_birth=b.date_of_birth

```
```sql
#根据各省医嘱提取药物信息
create table HCC_ORDERS_AZ_CQ as
  select from HCC_ID_AZ a,INP_ORDERS_CQ b
  where a.ID=b.ID and a.REGION='重庆'
#根据药物字典表，提取包含里面药物的医嘱信息
create table HCC_ORDERS_AZ as
  select b.*,a.* from HCC_DRUGS a,HCC_ORDERS_AZ_CQ b
  where instr(b.ORDER_TEXT,a.DRUGS_FOR_HCC)>0
```
