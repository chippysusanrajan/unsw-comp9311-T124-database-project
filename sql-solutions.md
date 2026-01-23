**Q1. Level-7 Information School Subjects**  

**Problem**:  
Retrieve all Level-7 subjects offered by organisations classified as Schools whose name contains Information.  

**Approach**:
  * Filter subject codes matching level-7 pattern (XXXX7***)
  * Join subjects with organisational units and types
  * Ensure organisation type contains “School”
    
**Code**:      
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

**Q2. COMP Courses Offering Only Lecture and Laboratory Classes**  

**Problem**:  
Find COMP courses that offer only Lecture and Laboratory class types.  

**Approach**:
  * Filter subjects with codes starting with "COMP"
  * Join courses, classes, and class types
  * Group by course and count distinct class types
  * Retain courses with exactly two class types: Lecture and Laboratory
       
**Code**:      
```sql
CREATE VIEW Q2(course_id)
AS
SELECT DISTINCT co.id 
FROM Courses co
JOIN Subjects sub ON co.subject = sub.id
JOIN Classes cl ON co.id = cl.course
JOIN Class_types clt ON cl.ctype = clt.id
LEFT JOIN 
   (
    SELECT DISTINCT co.id
    FROM Courses co 
    JOIN Classes cl ON co.id = cl.course 
    JOIN Class_types clt ON cl.ctype = clt.id 
    WHERE clt.name NOT IN ('Lecture', 'Laboratory')
   ) AS subs ON co.id = subs.id
WHERE sub.code LIKE 'COMP%'
AND subs.id IS NULL
GROUP BY co.id
HAVING COUNT(distinct clt.name)=2;
```

**Q3. High-Enrolment Students with Professor-Taught Courses**  

**Problem**:  
Retrieve UNSW IDs of students who:
  * Enrolled in at least five courses
  * Between 2008 and 2012
  * Courses had at least two professors
  * UNSW ID starts with 320
    
**Approach**:
  * Filter semesters by year range
  * Identify staff with a title containing "Prof"
  * Ensure courses have two or more professors
  * Count qualifying enrolments per student
    
**Code**:      
```sql
CREATE VIEW Q3(unsw_id)
AS
SELECT p.unswid 
FROM people p
JOIN students std ON p.id = std.id
JOIN course_enrolments ce ON ce.student = std.id
JOIN courses co ON co.id = ce.course
JOIN semesters sem ON sem.id = co.semester
WHERE LEFT(CAST(p.unswid AS TEXT), 3) = '320'
AND sem.year BETWEEN 2008 AND 2012
AND co.id IN (
    SELECT co.id 
    FROM courses co 
    JOIN course_staff cs ON cs.course = co.id 
    JOIN staff sf ON cs.staff = sf.id
    JOIN people p ON p.id = sf.id
    WHERE p.title = 'Prof'
    GROUP BY co.id
    HAVING COUNT(p.id) >= 2
)
GROUP BY p.unswid
HAVING COUNT(ce.course) >= 5;
```

**Q4. Top Performing Courses by Faculty (Above Distinction)**  

**Problem**:  
For each faculty and semester in 2012, find courses with the highest average mark among students achieving DN or HD. 

**Approach**:
  * Filter course enrolments with grades DN or HD
  * Compute average marks per course
  * Compare course averages within the same faculty and semester
  * Select courses with the maximum average
  * Round results to two decimal places
    
**Code**:      
```sql
CREATE VIEW Q4(course_id, avg_mark)
AS
WITH Above_Distinction_Marks AS 
    (
     SELECT courses.id AS c_id, 
     ROUND(AVG(course_enrolments.mark)::numeric, 2) AS avg_mark,
     orgunits.name AS faculty,
     semesters.term AS semester,
     MAX(ROUND(AVG(course_enrolments.mark)::numeric, 2)) OVER (PARTITION BY orgunits.name, semesters.term) AS max_avg_mark
     FROM courses 
     JOIN  course_enrolments ON courses.id = course_enrolments.course
     JOIN semesters ON semesters.id = courses.semester
     JOIN subjects ON subjects.id = courses.subject
     JOIN orgunits ON orgunits.id = subjects.offeredby
     JOIN orgunit_types ON orgunits.utype = orgunit_types.id
     WHERE orgunit_types.name = 'Faculty'
    AND course_enrolments.grade IN ('DN', 'HD')
    AND semesters.year = '2012'
    GROUP BY  semesters.year, orgunits.name, semesters.term, courses.id
   )
SELECT c_id, avg_mark
FROM Above_Distinction_Marks
WHERE  avg_mark = max_avg_mark;
```

**Q5. Large Courses with Multiple Professors**  

**Problem**:  
Identify courses that:  
  * Enrolled more than 500 students
  * Had at least two professors
  * Occurred between 2005 and 2015
Display course ID and concatenated professor given names.
 
**Approach**:
  * Count enrolments per course
  * Filter by year range
  * Identify professors via title
  * Concatenate given names using "string_agg"
    
**Code**:      
```sql
CREATE VIEW Q5(course_id, staff_name)
AS
SELECT 
  courses.id AS course_id,
  STRING_AGG(DISTINCT CONCAT(people.given, '; '), '') AS staff_name
FROM courses c                                                               
  JOIN course_enrolments ce ON c.id = ce.course 
  JOIN semesters sem ON c.semester = sem.id
  JOIN course_staff cs ON c.id = cs.course
  JOIN people p ON cs.staff = p.id
  WHERE sem.year BETWEEN 2005 AND 2015
  AND people.title = 'Prof'                     
  GROUP BY c.id
  HAVING COUNT(DISTINCT ce.student) > 500
  AND COUNT(DISTINCT p.id) >= 2;
```

**Q6. Most Frequently Used Rooms in 2012**  

**Problem**:  
Find rooms used most frequently by classes in 2012 and list the subject(s) that used each room most often.  

**Approach**:
  * Filter classes held in 2012
  * Count class usage per room
  * Determine maximum usage count
  * Identify subjects with the highest room occupancy
    
**Code**:      
```sql
CREATE VIEW Q6(room_id, subject_code) 
AS
WITH Room_Subject_Occupied AS 
   (
    SELECT rms.id AS room_id,
           sb.code AS subject_code,
           COUNT(*) AS occupied_count 
    FROM classes cl
    JOIN courses c ON cl.course = c.id
    JOIN subjects sb ON c.subject = sb.id
    JOIN semesters sem ON c.semester = sem.id
    JOIN rooms rms ON cl.room = rms.id
    WHERE sem.year = 2012
    GROUP BY rms.id, sb.code
   ),
Max_Occupied AS 
   (
    SELECT room_id,
           MAX(occupied_count) AS max_occupied
    FROM Room_Subject_Occupied
    GROUP BY room_id
   ),
Most_Frequently_Occupied AS 
   (
    SELECT
          rso.room_id,
          rso.subject_code
    FROM Room_Subject_Occupied rso
    INNER JOIN Max_Occupied mo ON rso.room_id = mo.room_id AND rso.occupied_count = mo.max_occupied
   ),
Total_Room_Occupied AS 
  (
   SELECT room_id, 
          SUM(occupied_count) AS total_occupied
   FROM Room_Subject_Occupied
   GROUP BY room_id
  ),
Most_Occupied_Rooms AS 
  (
   SELECT room_id
   FROM Total_Room_Occupied
   WHERE total_occupied = (SELECT MAX(total_occupied) FROM Total_Room_Occupied)
  )
SELECT
    mfo.room_id,
    mfo.subject_code 
FROM Most_Frequently_Occupied mfo
WHERE mfo.room_id IN (SELECT room_id FROM Most_Occupied_Rooms)
ORDER BY mfo.room_id, mfo.subject_code;
```

**Q7. Students Completing Multiple Programs Within 1000 Days**  

**Problem**:  
Retrieve students who completed two or more programs from the same organisation within 1000 days.  

**Approach**:
  * Validate program completion using UOC requirements
  * Calculate program duration using semester start/end dates
  * Group programs by organisation
  * Filter students completing multiple programs within time constraint
    
**Code**:      
```sql
CREATE VIEW Q7(student_id, program_id) 
AS
WITH Courses_Passed AS 
   (
    SELECT ce.student,
           pe.program,
           SUM(sb.uoc) AS uoc_total,
           MIN(sem.starting) AS start_prgm,
           MAX(sem.ending) AS end_prgm
    FROM course_enrolments ce
    INNER JOIN courses c ON ce.course = c.id
    INNER JOIN subjects sb ON c.subject = sb.id AND ce.mark >= 50
    INNER JOIN semesters sem ON c.semester = sem.id
    INNER JOIN program_enrolments pe ON ce.student = pe.student AND c.semester = pe.semester
    GROUP BY ce.student, pe.program
   ), 
Programs_Completed AS 
   (
    SELECT cp.student,
           cp.program,
           cp.start_prgm,
           cp.end_prgm
    FROM Courses_Passed cp
    INNER JOIN programs p ON cp.program = p.id
    WHERE cp.uoc_total >= p.uoc
   ),
Organization_Completion AS 
   (
    SELECT pc.student,
           p.offeredby AS organization,
           ARRAY_AGG(pc.program) OVER (PARTITION BY pc.student, p.offeredby) AS programs,
           MIN(pc.start_prgm) OVER (PARTITION BY pc.student, p.offeredby) AS first_start_prgm,
           MAX(pc.end_prgm) OVER (PARTITION BY pc.student, p.offeredby) AS last_end_prgm
   FROM Programs_Completed pc
   INNER JOIN programs p ON pc.program = p.id
   ),
Students_Eligible AS 
   (
    SELECT DISTINCT oc.student,
                    unnest(oc.programs) AS program_id
    FROM Organization_Completion oc
    WHERE (oc.last_end_prgm - oc.first_start_prgm) <= 1000 AND array_length(oc.programs, 1) >= 2
   )
SELECT p.unswid AS student_id,
       se.program_id
FROM Students_Eligible se
JOIN people p ON se.student = p.id
ORDER BY student_id, program_id;
```

**Q8. Staff Roles and Above-Distinction Rates**  

**Problem**:  
For staff who held three or more roles in the same organisation, compute:  
  * Total number of roles
  * Above-distinction rate for courses they convened in 2012
Return the top 20 staff ordered by distinction rate.

**Approach**:
  * Count staff affiliations per organisation
  * Identify course convenors
  * Calculate proportion of marks ≥ 75
  * Rank and limit results
    
**Code**:      
```sql
CREATE VIEW Q8(staff_id, sum_roles, hdn_rate) 
AS
WITH Staff_Affiliations AS 
   (
    SELECT af.staff,
           af.orgunit,
           COUNT(*) AS count_roles
    FROM affiliations af
    GROUP BY af.staff, af.orgunit
   ),
Staff_Valid AS 
   (
    SELECT staff
    FROM Staff_Affiliations
    GROUP BY staff
    HAVING MAX(count_roles) >= 3
   ),
Roles_Combined AS 
   (
    SELECT sv.staff,
           SUM(saf.count_roles)::BIGINT AS sum_roles
    FROM Staff_Valid sv
    JOIN Staff_Affiliations saf ON sv.staff = saf.staff
    GROUP BY sv.staff
   ),
Course_Convenor_2012 AS 
   (
    SELECT s.id AS staff_id,
           csf.course
    FROM staff s
    JOIN course_staff csf ON s.id = csf.staff
    JOIN staff_roles sr ON csf.role = sr.id
    JOIN courses cs ON csf.course = cs.id
    JOIN semesters sem ON cs.semester = sem.id
    WHERE sr.name = 'Course Convenor' AND sem.year = 2012
    AND s.id IN (SELECT staff FROM Roles_Combined)
   ),
Above_Distinction_Rate AS 
   (
    SELECT ccv.staff_id,
           ROUND((SUM(CASE WHEN course_enrolments.mark >= 75 THEN 1 ELSE 0 END)::NUMERIC / COUNT(course_enrolments.mark))::NUMERIC, 2) AS hdn_rate
    FROM Course_Convenor_2012 ccv
    JOIN course_enrolments ON ccv.course = course_enrolments.course
    GROUP BY ccv.staff_id
   ),
Ranked_Results AS 
   (
    SELECT people.unswid AS staff_id,
           rc.sum_roles,
           COALESCE(Above_Distinction_Rate.hdn_rate, 0) AS hdn_rate,
           DENSE_RANK() OVER (ORDER BY COALESCE(Above_Distinction_Rate.hdn_rate, 0) DESC) AS dense_rank
    FROM Roles_Combined rc
    JOIN people ON rc.staff = people.id
    LEFT JOIN Above_Distinction_Rate ON rc.staff = Above_Distinction_Rate.staff_id
   ),
Cutoff_Top20 AS 
   (
    SELECT MIN(hdn_rate) AS cutoff_hdn_rate
    FROM 
       (
        SELECT hdn_rate
        FROM Ranked_Results
        ORDER BY hdn_rate DESC
        LIMIT 20
       ) AS Top20
   )
SELECT rst.staff_id,
       rst.sum_roles,
       rst.hdn_rate
FROM Ranked_Results rst, Cutoff_Top20
WHERE rst.hdn_rate >= Cutoff_Top20.cutoff_hdn_rate
ORDER BY rst.hdn_rate DESC, rst.staff_id;
```

**Q9. Student Rank Within Courses (PL/pgSQL)**  

**Problem**:  
Given a student UNSW ID, return:  
  * Subject codes of completed courses
  * Student rank within each course
Only consider courses with the same prefix prerequisites.

**Approach**:
  * Validate student input
  * Identify prerequisite relationships
  * Rank students using PostgreSQL "RANK()" window function
  * Handle invalid input with a warning message
    
**Code**:      
```sql
CREATE VIEW FUNCTION Q9(student_unswid integer)
RETURNS TABLE(subject_code text, rank integer) AS $$
BEGIN
    -- Check if student exists
    IF NOT EXISTS (SELECT 1 FROM people WHERE unswid = student_unswid) THEN
        RAISE NOTICE 'WARNING: Invalid Student Input [%]', student_unswid;
        RETURN;
    END IF;

    -- Return subject codes with ranking
    RETURN QUERY
    SELECT 
        s.code AS subject_code,
        RANK() OVER (
            PARTITION BY c.id
            ORDER BY ce.mark DESC
        ) AS rank
    FROM course_enrolments ce
    JOIN courses c ON ce.course = c.id
    JOIN subjects s ON c.subject = s.id
    WHERE ce.unswid = student_unswid
      AND ce.mark IS NOT NULL
      AND EXISTS (
          SELECT 1
          FROM prereq p
          JOIN courses pre ON p.prereq_course = pre.id
          WHERE pre.code LIKE LEFT(s.code, 4) || '%'
      )
    ORDER BY s.code;
    -- If no valid courses found, return warning
    IF NOT FOUND THEN
        RAISE NOTICE 'WARNING: Invalid Student Input [%]', student_unswid;
    END IF;
END;
$$ LANGUAGE plpgsql;
```


**Q10. Program-wise WAM Calculation**  

**Problem**:  
Compute WAM per program for a given student, applying complex grading and inclusion rules. 

**Approach**:
  * Identify enrolled programs (Only passed courses according to "setpass" and "setsy")
  * Filter courses based on grade rules (Exclude courses with NULL marks from calculation)
  * Apply the WAM formula using marks and UOC
  * Handle edge cases (no WAM, invalid student)
      
**Code**:      
```sql
CREATE OR REPLACE FUNCTION Q10(student_unswid integer)
RETURNS TABLE(unswid integer, program_name text, wam numeric) AS $$
DECLARE
    rec RECORD;
    total_weight numeric;
    total_uoc numeric;
BEGIN
    -- Check if student exists
    IF NOT EXISTS (SELECT 1 FROM people WHERE unswid = student_unswid) THEN
        RAISE NOTICE 'WARNING: Invalid Student Input [%]', student_unswid;
        RETURN;
    END IF;

    FOR rec IN
        SELECT p.id AS program_id, p.name AS program_name
        FROM programs p
        JOIN program_enrolments pe ON p.id = pe.program
        WHERE pe.unswid = student_unswid
    LOOP
        -- Compute total weighted marks and UOC
        SELECT 
            COALESCE(SUM(ce.mark * s.uoc),0),
            COALESCE(SUM(s.uoc),0)
        INTO total_weight, total_uoc
        FROM course_enrolments ce
        JOIN courses c ON ce.course = c.id
        JOIN subjects s ON c.subject = s.id
        WHERE ce.unswid = student_unswid
          AND ce.program = rec.program_id
          AND ce.grade IS NOT NULL
          AND ce.grade NOT IN ('SY','XE','T','PE');

        -- Return WAM or No WAM Available
        IF total_uoc = 0 THEN
            wam := NULL;
            unswid := student_unswid;
            program_name := rec.program_name;
            RAISE NOTICE 'No WAM Available for program %', rec.program_name;
            RETURN NEXT;
        ELSE
            wam := ROUND(total_weight / total_uoc, 2);
            unswid := student_unswid;
            program_name := rec.program_name;
            RETURN NEXT;
        END IF;
    END LOOP;

    -- If no programs found
    IF NOT FOUND THEN
        RAISE NOTICE 'WARNING: Invalid Student Input [%]', student_unswid;
    END IF;

END;
$$ LANGUAGE plpgsql;
```


