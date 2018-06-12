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

#提取基表中无医嘱的患者
create table HCC_1L_AZ as
  select d.* from HCC_RESULT1_AZ d,
  (select distinct a.ID,a.REGION from HCC_RESULT1_AZ a
  minus
  select distinct b.ID,b.REGION from HCC_ORDERS_AZ b) c
  where d.ID=c.ID and d.REGION=c.REGION

#提取基表中无医嘱但包含EC的患者
select distinct c.ID,c.REGION from
(select a.ID,a.REGION,min(a.VISIT_TIME) as mina,min(b.VISIT_TIME) as minb
from HCC_1L_AZ a,HCC_EC_AZ b where a.ID=b.ID and a.REGION=b.REGION
group by a.ID,a.REGION) c
where c.mina>c.minb
上述结果为0，证明HCC_1L_AZ表中患者均可划分为1L中，即HCC_1L_AZ可对应分析步骤8中1

#提取基表中有医嘱的患者
create table HCC_RESULT2_AZ as
  select * from HCC_RESULT1_AZ
  minus
  select * from HCC_1L_AZ

#统计基表中有医嘱无PD无EC的患者
select count(d.ID),d.REGION from
(
  (select distinct a.ID,a.REGION from HCC_RESULT2_AZ a
  minus
  select distinct b.ID,b.REGION from HCC_PD_AZ b)
  minus
  select distinct c.ID,c.REGION from HCC_EC_AZ
) d group by d.REGION
此结果对应分析步骤8中2

#提取基表中有医嘱无EC的患者
create table HCC_RESULT2_ECNO_AZ as
  select d.* from HCC_RESULT2_AZ d,
  (select distinct a.ID,a.REGION from HCC_RESULT2_AZ a
  minus
  select distinct b.ID,b.REGION from HCC_EC_AZ b) c
  where d.ID=c.ID and d.REGION=c.REGION

#提取基表中有医嘱，有PD，无EC且PD时间在医嘱时间之后，医嘱时间在基表时间之后
select * from
(select a.ID,a.REGION,min(a.VISIT_TIME) as mina,min(b.START_DATE_TIME)
as minb,min(c.ADMISSION_DATE_TIME)as minc
from HCC_RESULT2_ECNO_AZ a,HCC_ORDERS_AZ b,HCC_PD_AZ c
where a.ID=b.ID and b.ID=c.ID and a.REGION=b.REGION and b.REGION=c.REGION
group by a.ID,a.REGION) e
where e.mina<e.minb and e.minb<e.minc
此结果对应分析步骤8中3，也对应分析步骤9种1

#提取基表中有医嘱，有PD，无EC,且医嘱时间在PD和基表时间之后  
select * from
(select a.ID,a.REGION,min(a.VISIT_TIME) as mina,min(b.START_DATE_TIME)
as minb,min(c.ADMISSION_DATE_TIME)as minc
from HCC_RESULT2_ECNO_AZ a,HCC_ORDERS_AZ b,HCC_PD_AZ c
where a.ID=b.ID and b.ID=c.ID and a.REGION=b.REGION and b.REGION=c.REGION
group by a.ID,a.REGION) e
where e.minb<e.mina and e.minb<e.minc
此结果对应分析步骤9中2

#提取基表中有医嘱，有PD，有EC，且PD时间在医嘱时间之后，且EC时间在最后
select * from
(select a.ID,a.REGION,min(a.VISIT_TIME) as mina,min(b.START_DATE_TIME)
as minb,min(c.ADMISSION_DATE_TIME) as minc,min(d.VISIT_TIME) as mind
from HCC_RESULT2_ECNO_AZ a,HCC_ORDERS_AZ b,HCC_PD_AZ c,HCC_EC_AZ d
where a.ID=b.ID and b.ID=c.ID and c.ID=d.ID
and a.REGION=b.REGION and b.REGION=c.REGION and c.REGION=d.REGION
group by a.ID,a.REGION) e
where e.minc>e.minb and e.mind>e.minc and e.mind>e.mina
此结果对应于分析步骤9中3
```
