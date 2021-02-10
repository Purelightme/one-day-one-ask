1，左连接取副表最新一条记录

```sql
SELECT
	* 
FROM
	intent_student_info isi
	LEFT JOIN stu_followup_rec_data sfrd ON followup_time = ( SELECT max( followup_time ) FROM stu_followup_rec_data WHERE stu_id = isi.stu_id ) 
WHERE
	isi.school_visible_flag = '0'
```

2，特定值排序

```sql
select * from users order by field(id,4,5) desc,id;
```

