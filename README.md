🎓 University Database Analytics System

COMP9311 – Database Systems (UNSW, 24T1)

📌 Overview  
This project involved designing and implementing advanced SQL views and PL/pgSQL functions on a large-scale university database (MyMyUNSW) to support real-world academic and administrative analytics.
The system models students, staff, courses, programs, enrolments, and facilities, and enables complex queries such as student progression, course performance analysis, staff roles, and weighted average mark (WAM) computation.

🎯 Objectives
* Understand and navigate a large relational schema
* Implement complex SQL queries and views
* Develop PL/pgSQL functions with real business rules
* Ensure performance, correctness, and portability
* Produce results compatible with automated testing

🗂️ Dataset  
Database: MyMyUNSW  
Size: Large, production-style academic database  
Domain Entities:
    * Students, Staff, Courses, Programs
    * Enrolments, Grades, Semesters
    * Rooms, Faculties, Schools

🛠️ Technologies Used
* PostgreSQL
* Advanced SQL
* PL/pgSQL
* Window Functions (RANK)
* Aggregations & Joins
* Query Optimisation

🔍 Key Features Implemented

📊 SQL Views (Q1–Q8) - DONE Full Implemention
   * Identified Level-7 subjects offered by Information Schools
   * Filtered COMP courses based on class structure
   * Analysed student enrolment patterns across years
   * Computed course-level performance for high-achieving students
   * Identified high-demand courses and professor involvement
   * Analysed room utilisation frequency
   * Detected students completing multiple programs within time constraints
   * Ranked staff by distinction rates and organisational roles

🧮 PL/pgSQL Functions (Q9–Q10) - Partial Implementation  
🔹 Student Ranking Function
  * Ranks a student within a course based on marks
  * Handles ties using PostgreSQL window functions
  * Validates prerequisite relationships and edge cases
    
🔹 Program-wise WAM Calculation
  * Calculates WAM per program per student
  * Applies complex grading rules (pass/fail, excluded grades)
  * Handles missing data and edge conditions gracefully

⚙️ Performance & Constraints
* Queries designed to execute within 120 seconds
* No hardcoded values (schema-agnostic)
* Fully compatible with auto-marking scripts
* Loads and runs in a single SQL pass

🧪 Testing
* Verified using the provided check.sql auto-test framework
* Tested across multiple hidden datasets
* Ensured correct handling of:
     * Null values
     * Multiple results
     * Tied rankings
     * Invalid inputs
 
🚀 Skills Demonstrated
* Relational data modelling
* Complex SQL query design
* Stored procedure development
* Performance-aware database programming
* Translating business rules into database logic

📎 Notes  
This project simulates real-world challenges found in:
   * University administration systems
   * Enterprise databases
   * Data engineering & analytics pipelines

👤 Author
Chippy Susan Rajan
Master of Data Science & Decisions – UNSW Sydney
