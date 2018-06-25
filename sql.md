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

#提取1L和2L患者

---基表左连接EC表，选择EC时间在基表时间之前
create table HCC_NOEC_AZ as
  select a.*,b.VISIT_TIME as EC_DATE from HCC_RESULT1_AZ a
  left join HCC_EC_DATE_AZ b
  on a.ID=b.ID and a.REGION=b.REGION and a.VISIT_TIME>=b.VISIT_TIME

---在上表基础上选择EC时间是空的患者
create table HCC_NOEC as
  select a.* from HCC_NOEC_AZ a where a.EC_DATE is null

---上表左连接出院记录
create table HCC_NOEC_LOCO_T as
  select a.*,b.ADMISSION_DATE_TIME as LOCO_DATE
  from HCC_NOEC a left join TEM_OUTP_DISCHARGE_STATUS b
  on a.ID=b.ID and a.REGION=b.REGION
  and regexp_like(b.TZL_PROCESS,'(介入)|(术后)|(TACE)|(射频消融)|(放疗)')
  and a.VISIT_TIME<=b.ADMISSION_DATE_TIME

---选择没有局部治疗信息或者这些信息在基表时间之后
create table HCC_NOEC_NOLOCO as
  select a.* from HCC_NOEC_LOCO_T a
  where a.LOCO_DATE is null

---选择那些有局部治疗信息的患者
create table HCC_NOEC_LOCOREC as
  select a.*,b.DISCHARGE_DATE_TIME as LOCO_DATE_ALL
  from HCC_NOEC_NOLOCO a inner join TEM_OUTP_DISCHARGE_STATUS b
  on a.REGION=b.REGION and a.ID=b.ID
  and regexp_like(b.TZL_PROCESS,'(介入)|(术后)|(TACE)|(射频消融)|(放疗)')

---选择进行过全身治疗没有局部治疗
create table HCC_ORDERS_ST as
  select c.* from
  (select a.*,b.LOCO_DATE_ALL from HCC_ORDERS_DATE_AZ a
  left join HCC_NOEC_LOCOREC b
  on a.ID=b.ID and a.REGION=b.REGION
  and a.START_DATE_TIME<=b.LOCO_DATE_ALL) c
  where c.LOCO_DATE_ALL is null

---选择满足要求且没有局部治疗信息的患者
create table HCC_1L_NEW_T as
  select a.*,b.START_DATE_TIME as ST_DATE
  from HCC_NOEC_NOLOCO a
  left join HCC_ORDERS_ST b
  on a.REGION=b.REGION and a.ID=b.ID
  and b.START_DATE_TIME<a.VISIT_TIME

---1L患者
create table HCC_1L_NEW as
  select a.* from HCC_1L_NEW_T a
  where a.ST_DATE is null

---2L患者
create table HCC_2L_NEW as
  select a.* from HCC_1L_NEW_T a
  where a.ST_DATE is not null

---1L和2L人数统计
select count(distinct a.ID),to_char(a.VISIT_TIME,'yyyy'),a.REGION
from HCC_1L_NEW a group by a.REGION,to_char(a.VISIT_TIME,'yyyy')

select count(distinct a.ID),to_char(a.VISIT_TIME,'yyyy'),a.REGION
from HCC_2L_NEW a group by a.REGION,to_char(a.VISIT_TIME,'yyyy')

---剔除不是原发性肝癌的人
create table HCC_1L_NEW_2 as
  select b.*,c.aID from HCC_1L_NEW b left join
  (select distinct a.id as aID from HCC_1L_NEW t,TEM_INP_ADMISSION_STATUS a
  where t.ID=a.ID and instr(a.PRELIMINARY_CLINICAL_DIAGNOSIS,'肝癌')<=0
  and instr(a.PRELIMINARY_CLINICAL_DIAGNOSIS,'肝细胞癌')<=0
  and instr(a.PRELIMINARY_CLINICAL_DIAGNOSIS,'肝占位')<=0
  and (instr(a.PRELIMINARY_CLINICAL_DIAGNOSIS,'癌')>0
       or instr(a.PRELIMINARY_CLINICAL_DIAGNOSIS,'恶性肿瘤')>0 )) c
  on c.aID=b.ID

---关联现病史
create table HCC_1L_NEW_2_HY as
  select a.ID,a.VISIT_TIME as INDEX_DATE,b.VISIT_TIME,b.HY_PRESENT
  from HCC_1L_NEW_2 a,TEM_INP_ADMISSION_STATUS b
  where a.ID=b.ID and a.aID is null

---按ID分组并选最后一条
create table HCC_1L_NEW_3_HY as
  select * from(
    select a.*,
      ROW_NUMBER() over(partition by a.ID,a.INDEX_DATE order by a.VISIT_TIME desc) rn
      from HCC_1L_NEW_2_HY a)
  where rn=1





```
