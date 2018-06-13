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
  select * from HCC_ID_AZ a,INP_ORDERS_CQ b
  where a.ID=b.ID and a.REGION='重庆'
#根据药物字典表，提取包含里面药物的医嘱信息
create table HCC_ORDERS_AZ as
  select b.*,a.* from HCC_DRUGS a,HCC_ORDERS_AZ_CQ b
  where instr(b.ORDER_TEXT,a.DRUGS_FOR_HCC)>0


#提取满足分期信息和评分标准的患者建立子表
create table HCC_STAGING_BC_AZ as
  select * from HCC_STAGING_20180608 a
  where regexp_like(a.BCLC,'B|C')

create table HCC_ASSESSMENT_DATE_AZ as
  select * from HCC_ASSESSMENT_20180608 a
  where regexp_like(a.KPS,'(90)|(100)') or regexp_like(a.ECOG,'0|1')

#将STAGING表、ASSESSMENT表和DATE表通过ID和REGION字段关联一起，选择年龄大于18岁的患者
create table HCC_RESULT_STAGING_ID_AZ as
  select * from HCC_STAGING_BC_AZ a,HCC_ID_AZ b
  where a.ID=b.ID and a.REGION=b.REGION and b.BIRTH>=18

create table HCC_RESULT1_AZ as
  select from HCC_RESULT_STAGING_ID_AZ a,HCC_ASSESSMENT_DATE_AZ b
  where a.ID=b.ID and a.REGION=b.REGION and a.VISIT_TIME=b.VISIT_TIME

#提取1L患者

---基表左连接EC表，选择EC时间在基表时间之前
create table HCC_NOEC_AZ as
  select a.*,b.VISIT_TIME as EC_DATE from HCC_RESULT1_AZ a
  left join HCC_EC_DATE_AZ b
  on a.ID=b.ID and a.REGION=b.REGION and a.VISIT_TIME>=b.VISIT_TIME

---在上表基础上选择EC时间是空的患者
create table HCC_NOEC as
  select a.* from HCC_NOEC_AZ a where a.EC_DATE is null

---用上表左连接医嘱表选择遗嘱时间在基表时间之前
create table HCC_1L_T_AZ as
  select a.*,b.START_DATE_TIME as ST_DATE from HCC_NOEC a
  left join HCC_ORDERS_DATE_AZ b
  on a.ID=b.ID and a.REGION=b.REGION and a.VISIT_TIME>b.START_DATE_TIME

---在上表基础上选择医嘱时间是空的患者
create table HCC_1L as
  select a.* from HCC_1L_T_AZ a where a.ST_DATE is null


#提取2L患者

---用HCC_1L_T_AZ表中选择医嘱时间非空的患者
create table HCC_2L_T as
  select a.* from HCC_1L_T_AZ a where a.ST_DATE is not null

---用上表左连接PD表且选择医嘱时间小于PD时间小于基表时间的患者
create table HCC_2L_TT as
  select a.*,b.ADMISSION_DATE_TIME as PD_DATE from HCC_2L_T a
  left join HCC_PD_AZ b  
  on a.ID=b.ID and a.REGION=b.region
  and a.ST_DATE<=b.ADMISSION_DATE_TIME and b.ADMISSION_DATE_TIME<a.VISIT_TIME

---用上表选择PD时间非空的患者
create table HCC_2L as
  select a.* from HCC_2L_TT a where a.PD_DATE is not null

#统计患者（1L和2L）
---按年份和地区
select to_char(a.VISIT_TIME,'yyyy') as INDEX_TIME,
a.REGION,COUNT(DISTINCT a.ID) as PAT_COUNT from HCC_1L a
group by a.REGION,to_char(a.VISIT_TIME,'yyyy')

---按年份
select to_char(a.VISIT_TIME,'yyyy') as INDEX_TIME,
COUNT(DISTINCT a.ID) as PAT_COUNT from HCC_1L a
group by to_char(a.VISIT_TIME,'yyyy')


```
