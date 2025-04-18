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

## Value Set: MACE_KIT_DURATION_CHECK

![vs_1_2](/images/vs_1_2.png)





![vs_1_2](/images/vs_1_2.png)


