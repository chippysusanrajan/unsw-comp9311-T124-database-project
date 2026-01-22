**Q1. Level-7 Information School Subjects**  

Problem:
Retrieve all Level-7 subjects offered by organisations classified as Schools whose name contains Information.  
Approach:
  * Filter subject codes matching level-7 pattern (XXXX7***)
  * Join subjects with organisational units and types
  * Ensure organisation type contains “School”
    
```sql
CREATE OR REPLACE VIEW Q1(subject_code) AS
SELECT s.code
FROM subjects s
JOIN orgunits o ON s.offeredby = o.id
JOIN orgunit_types ot ON o.utype = ot.id
WHERE ot.name LIKE '%School%'
  AND o.longname LIKE '%Information%'
  AND s.code ~ '^[A-Z]{4}7[0-9]{3}$';
```

