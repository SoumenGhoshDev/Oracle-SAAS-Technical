# Input Validation Fast Formula

### Requirement: 

Due to some urgent work , Manager can request their subordinates to work even if they are on Parental Leave. As a benefit for the same Manager can reward Kit Days in return. There are below restrictions on the same.

1. The Kit days should be fully within Parental leave Duration.
2. The maximum Kit Days can be awarded upto 10 days for a Parental Leave Duration.
3. The Kit Days should not overlap with each other.


### Solution:

To achieve this we have come up with below objects:

1. One element with Element Input Values as `KIT Start Date` and `KIT End Date`
2. One Input Validation Fast Formula
3. Three Value Sets to get below information
    - Duration check wheher it is falling within Parental Leave Duration.
    - Overlapping check
    - Sum of existing Kit Days

Please find below the Fast Formula :
```SQL
inputs are KIT_Start_Date(date), KIT_End_Date(date)
DEFAULT FOR l_person_num is 'XX'

l_person_id = GET_CONTEXT(PERSON_ID, -1)

if(DAYS_BETWEEN( KIT_End_Date,KIT_Start_Date ) < 0 ) then
(
formula_message = 'KIT Start date should be less than KIT End date'
formula_status = 'E'
)
else if(DAYS_BETWEEN( KIT_End_Date,KIT_Start_Date ) > 9 ) then
(
formula_message = 'You can not apply more than 10 days for KIT days.'
formula_status = 'E'
)
else if(DAYS_BETWEEN( KIT_End_Date,KIT_Start_Date ) <= 9 ) then
(
       l_person_num = GET_VALUE_SET('MACE_KIT_DURATION_CHECK',
                                     '|=PERSON_ID=' || to_char(l_person_id)
                                     || '|KIT_START_DATE=' || to_char(KIT_Start_Date,'YYYYMMDD')
                                     || '|KIT_END_DATE=' || to_char(KIT_End_Date,'YYYYMMDD')
                                    )
         if(l_person_num||'XX' = 'XX') then
         (
          formula_message = 'Kit Day Duration should fall within your Parental Leave Duration' 
          formula_status = 'E'
         )
         else
         (
			  l_person_num_2 = GET_VALUE_SET('MACE_KIT_OVERLAP_CHECK',
                                        '|=PERSON_ID=' || to_char(l_person_id)
                                        || '|KIT_START_DATE=' || to_char(KIT_Start_Date,'YYYYMMDD')
                                        || '|KIT_END_DATE=' || to_char(KIT_End_Date,'YYYYMMDD')
                                    )						
			  if(l_person_num_2||'XX' <> 'XX') then
              (
                 formula_message = 'Entered Kit Days should not overlap with existing KIT Days in System' 
                 formula_status = 'E'
              )
			  else
		      (
				 l_kit_db_sum = GET_VALUE_SET('MACE_KIT_DAYS_SUM',
                                        '|=PERSON_ID=' || to_char(l_person_id)
                                    )			 
			     if(DAYS_BETWEEN( KIT_End_Date,KIT_Start_Date ) > 9- to_number(l_kit_db_sum) ) then
				 (
				    formula_message = 'Your existing Kit days count : '|| l_kit_db_sum || ' and you are applying for '|| to_char(DAYS_BETWEEN( KIT_End_Date,KIT_Start_Date )+1) ||' day/s which is exceeding 10 day limit'
                    formula_status = 'E' 
				 )
				 ELSE
				 (
				    formula_message = 'You have successfully saved KIT days entry'
                    formula_status = 'S'
				 )
			  )
          )
)
else
(
formula_message = 'You have successfully saved KIT days entry'
formula_status = 'S'
)

return formula_message, formula_status
```

### Value Set: MACE_KIT_DURATION_CHECK

![vs_1_1](/images/vs_1_1.png)

From Clause: `ANC_PER_ABS_ENTRIES apae, per_all_people_f papf, ANC_ABSENCE_TYPES_F_TL aatft`

Value Column Name: `papf.person_number`

ID Column Name: `papf.person_number`

WHERE Clause: 
``` SQL
       papf.person_id = apae.person_id
   and apae.ABSENCE_TYPE_ID = aatft.ABSENCE_TYPE_ID
   and aatft.language = 'US'
   and trunc(sysdate) between papf.effective_start_date and papf.effective_end_date
   and to_char(papf.person_id) = :{PARAMETER.PERSON_ID}
   and aatft.name IN ('Maternity Leave','Paternity Leave','Adoption Leave','Paternity Adoption Leave')
   and apae.APPROVAL_STATUS_CD = 'APPROVED'
   and processing_status = 'P'
   and to_char(apae.START_DATE,'YYYYMMDD') <= :{PARAMETER.KIT_START_DATE}
   and to_char(apae.END_DATE,'YYYYMMDD') >= :{PARAMETER.KIT_END_DATE}
and ROWNUM=1
```

### Value Set: MACE_KIT_OVERLAP_CHECK

![vs_2_1](/images/vs_2_1.png)

From Clause: `per_all_people_f papf , pay_element_entries_f peef, pay_element_types_f petf, pay_input_values_f pivf_st, pay_input_values_f pivf_end, pay_element_entry_values_f peevf_st, pay_element_entry_values_f peevf_end`

Value Column Name: `papf.person_number`

ID Column Name: `papf.person_number`

WHERE Clause: 
``` SQL
1=1
and papf.person_id=:{PARAMETER.PERSON_ID}
and papf.person_id = peef.person_id
AND peef.element_type_id = petf.element_type_id
AND TRUNC(peef.effective_start_date) BETWEEN petf.effective_start_date AND
                                   petf.effective_end_date
AND TRUNC(peef.effective_start_date) BETWEEN papf.effective_start_date AND  papf.effective_end_date
AND TRUNC(sysdate) BETWEEN papf.effective_start_date AND papf.effective_end_date
AND petf.base_element_name = 'MaceKITDay'
AND pivf_st.element_type_id =  petf.element_type_id
AND pivf_end.element_type_id =  petf.element_type_id
AND pivf_st.base_name in ('KIT_Start_date')
AND pivf_end.base_name in ('KIT_End_date')
AND peevf_st.input_value_id = pivf_st.input_value_id
AND peevf_end.input_value_id = pivf_end.input_value_id
AND peevf_st.ELEMENT_ENTRY_ID =peef.ELEMENT_ENTRY_ID
AND peevf_end.ELEMENT_ENTRY_ID =peef.ELEMENT_ENTRY_ID
AND (to_date(:{PARAMETER.KIT_START_DATE},'YYYYMMDD') BETWEEN TO_DATE(peevf_st.SCREEN_ENTRY_VALUE,'YYYY-MM-DD HH24:MI:SS')
                                    AND TO_DATE(peevf_end.SCREEN_ENTRY_VALUE,'YYYY-MM-DD HH24:MI:SS')
    OR  
    to_date(:{PARAMETER.KIT_END_DATE},'YYYYMMDD') BETWEEN TO_DATE(peevf_st.SCREEN_ENTRY_VALUE,'YYYY-MM-DD HH24:MI:SS')
                                    AND TO_DATE(peevf_end.SCREEN_ENTRY_VALUE,'YYYY-MM-DD HH24:MI:SS')
									)
 AND ROWNUM = 1
```

### Value Set: MACE_KIT_DAYS_SUM

![vs_3_1](/images/vs_3_1.png)

From Clause: `(SELECT to_number(LEVEL) as num FROM dual CONNECT BY LEVEL<=300) Num_Tab`

Value Column Name: `to_char(Num_Tab.num)`

ID Column Name: `to_char(Num_Tab.num)`

WHERE Clause: 
``` SQL
Num_Tab.num = (
SELECT SUM((TO_DATE(peevf_end.SCREEN_ENTRY_VALUE,'YYYY-MM-DD HH24:MI:SS')- TO_DATE(peevf_st.SCREEN_ENTRY_VALUE,'YYYY-MM-DD HH24:MI:SS'))+1)
FROM per_all_people_f papf ,
pay_element_entries_f peef,
pay_element_types_f petf,
pay_input_values_f pivf_st,
pay_input_values_f pivf_end,
pay_element_entry_values_f peevf_st,
pay_element_entry_values_f peevf_end
where papf.person_id=:{PARAMETER.PERSON_ID}
AND papf.person_id = peef.person_id
AND peef.element_type_id = petf.element_type_id
AND TRUNC(peef.effective_start_date) BETWEEN petf.effective_start_date AND
                                   petf.effective_end_date
AND TRUNC(peef.effective_start_date) BETWEEN papf.effective_start_date AND  papf.effective_end_date
AND TRUNC(SYSDATE) BETWEEN papf.effective_start_date AND papf.effective_end_date
AND petf.base_element_name = 'MaceKITDay'
AND pivf_st.element_type_id =  petf.element_type_id
AND pivf_end.element_type_id =  petf.element_type_id
AND pivf_st.base_name in ('KIT_Start_date')
AND pivf_end.base_name in ('KIT_End_date')
AND peevf_st.input_value_id = pivf_st.input_value_id
AND peevf_end.input_value_id = pivf_end.input_value_id
AND peevf_st.ELEMENT_ENTRY_ID =peef.ELEMENT_ENTRY_ID
AND peevf_end.ELEMENT_ENTRY_ID =peef.ELEMENT_ENTRY_ID)
```

### Additional Info

#### Query to find out existing KIT Days 

``` SQL
select papf.person_number,petf.base_element_name,  peef.effective_start_date, 
       peevf_st.SCREEN_ENTRY_VALUE st_date, peevf_end.SCREEN_ENTRY_VALUE end_date
  from per_all_people_f papf , 
       pay_element_entries_f peef, 
	   pay_element_types_f petf, 
	   pay_input_values_f pivf_st, 
	   pay_input_values_f pivf_end, 
	   pay_element_entry_values_f peevf_st, 
       pay_element_entry_values_f peevf_end
 where 1=1
and papf.person_number = '20030670'
and papf.person_id = peef.person_id
AND peef.element_type_id = petf.element_type_id
AND TRUNC(peef.effective_start_date) BETWEEN petf.effective_start_date AND
                                   petf.effective_end_date
AND TRUNC(peef.effective_start_date) BETWEEN papf.effective_start_date AND  papf.effective_end_date
AND TRUNC(sysdate) BETWEEN papf.effective_start_date AND papf.effective_end_date
AND petf.base_element_name = 'MaceKITDay'
AND pivf_st.element_type_id =  petf.element_type_id
AND pivf_end.element_type_id =  petf.element_type_id
AND pivf_st.base_name in ('KIT_Start_date')
AND pivf_end.base_name in ('KIT_End_date')
AND peevf_st.input_value_id = pivf_st.input_value_id
AND peevf_end.input_value_id = pivf_end.input_value_id
AND peevf_st.ELEMENT_ENTRY_ID =peef.ELEMENT_ENTRY_ID
AND peevf_end.ELEMENT_ENTRY_ID =peef.ELEMENT_ENTRY_ID
```
### Some Example Error Mesaages:

![Error1](/images/Error1.jpg)

![Error2](/images/Error2.jpg)

![Error3(/images/Error3.jpg)

![Error4](/images/Error4.jpg)
