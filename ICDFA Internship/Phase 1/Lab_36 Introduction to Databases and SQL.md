# Introduction to Databases and SQL

## Overview

This lab introduced relational database concepts through hands-on work with MySQL and phpMyAdmin. I built a student management system from scratch, creating tables with proper relationships, inserting sample data, and running queries to retrieve and manipulate records. The goal was to understand how databases organize information, enforce data integrity through foreign keys, and respond to SQL commands that power web applications.

## Objectives

- Install and configure XAMPP to run MySQL and phpMyAdmin locally
- Design a relational database schema with multiple linked tables
- Define primary keys for unique identification and foreign keys for relationships
- Insert sample data while maintaining referential integrity across tables
- Write SQL queries to retrieve, update, and delete records
- Use JOIN operations to combine related data from multiple tables
- Understand ACID compliance and why databases matter for web applications

## Lab Environment

- **Operating System:** Kali Linux
- **Database Server:** MySQL (via XAMPP stack)
- **Database Management Tool:** phpMyAdmin (web-based GUI accessed at http://localhost/phpmyadmin/)
- **Target Database:** student_management_system

## Tools Used

- XAMPP (Apache, MySQL, PHP bundle)
- phpMyAdmin (MySQL administration interface)
- SQL (Structured Query Language for database operations)

## Methodology

### Exercise 1: XAMPP Installation and Database Creation

I started by installing XAMPP, which bundles Apache web server, MySQL database, and PHP into a single package. This eliminates the need to configure each component separately.

**Starting XAMPP services:**

```bash
sudo /opt/lampp/lampp start
```

![XAMPP startup showing Apache, MySQL, and ProFTPD services starting](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_01%20XAMPP%20startup%20showing%20Apache%2C%20MySQL%2C%20and%20ProFTPD%20services%20starting.jpg)

The output confirmed three services started: Apache (web server), MySQL (database), and ProFTPD (FTP server). MySQL runs on port 3306 by default, and phpMyAdmin connects to it through the Apache web server.

**Accessing phpMyAdmin:**

I navigated to http://localhost/phpmyadmin/ in Firefox. This interface provides a GUI for database operations instead of typing raw SQL commands in a terminal.

![phpMyAdmin homepage showing existing databases](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_02%20phpMyAdmin%20homepage%20showing%20existing%20databases.jpg)

**Creating the student_management_system database:**

![Creating new database in phpMyAdmin](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_03%20Creating%20new%20database%20in%20phpMyAdmin.jpg)

phpMyAdmin created the database and displayed it in the left sidebar. At this point, the database exists but contains no tables or data. It is an empty container waiting for structure.

**Why relational databases matter for web applications:**

A student management system stores information about students, courses, enrollments, grades, and attendance. Spreadsheets fail at this scale because:

- **No relationship enforcement:** Nothing prevents enrolling a student in a course that does not exist
- **Data duplication:** Every enrollment record would need to repeat the student's full name, email, and date of birth
- **Update anomalies:** Changing a student's email means finding and updating dozens of scattered records
- **Concurrent access failures:** Multiple users editing the same spreadsheet simultaneously causes conflicts and lost data

Relational databases solve these problems by:

- **Normalization:** Store each piece of information once. Student names live in the students table, course names in the courses table
- **Foreign key constraints:** The database refuses to enroll a student_id that does not exist in the students table
- **Atomic updates:** Changing a student's email in one place updates it everywhere the foreign key references it
- **ACID compliance:** Transactions either complete fully or roll back entirely, preventing partial saves during crashes

**MySQL-specific advantages:**

- **ACID compliance:** Atomicity, Consistency, Isolation, Durability. If the system crashes while saving a grade, the database rolls back the transaction instead of leaving corrupted half-saved data
- **High security:** Built-in user authentication, role-based permissions, and encryption protect sensitive student records from unauthorized access
- **Performance:** Optimized for concurrent read/write operations. Hundreds of students, parents, and teachers can query the database simultaneously without slowdowns

### Exercise 2: Table Creation and Schema Design

Designing tables requires understanding what entities exist and how they relate. A student management system has three core entities:

1. **Students** (people enrolled)
2. **Courses** (classes offered)
3. **Enrollments** (which students are taking which courses)

**Creating the students table:**

```sql
CREATE TABLE students (
    student_id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100) UNIQUE,
    date_of_birth DATE
);
```

![Students table structure in phpMyAdmin](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_04%20Students%20table%20structure%20in%20phpMyAdmin.jpg)

**Column design decisions:**

- `student_id`: INT (integer) with AUTO_INCREMENT generates unique IDs automatically (1, 2, 3...). PRIMARY KEY enforces uniqueness and creates an index for fast lookups
- `first_name` and `last_name`: VARCHAR(50) stores variable-length text up to 50 characters. This handles most names without wasting space on fixed-length columns
- `email`: VARCHAR(100) with UNIQUE constraint prevents duplicate email addresses. The database rejects INSERT attempts with existing emails
- `date_of_birth`: DATE stores dates in YYYY-MM-DD format for age calculations and validation

**Creating the courses table:**

```sql
CREATE TABLE courses (
    course_id INT AUTO_INCREMENT PRIMARY KEY,
    course_name VARCHAR(100),
    credits INT
);
```

![Courses table structure in phpMyAdmin](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_05%20Courses%20table%20structure%20in%20phpMyAdmin.jpg)

This table is simpler because courses have fewer attributes. The `credits` column stores how many credit hours the course is worth (3, 4, 6, etc.).

**Creating the enrollments table (junction table):**

```sql
CREATE TABLE enrollments (
    enrollment_id INT AUTO_INCREMENT PRIMARY KEY,
    student_id INT,
    course_id INT,
    enrollment_date DATE,
    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
);
```

![Enrollments table structure showing foreign key relationships](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_09%20Enrollments%20table%20showing%20student-course%20relationships.jpg)

![Enrollments table structure showing foreign key relationships](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_09B%20Enrollments%20table%20showing%20student-course%20relationships.jpg)

**Why foreign keys matter:**

The enrollments table links students to courses through a many-to-many relationship. One student can enroll in multiple courses, and one course can have multiple students. Foreign keys enforce referential integrity at the database level:

- **Prevents orphaned records:** Cannot insert enrollment_id=1, student_id=999 if student_id 999 does not exist in the students table
- **Cascading options:** When a student is deleted, the database can automatically delete all their enrollments (CASCADE) or prevent deletion if enrollments exist (RESTRICT)
- **Data consistency:** Ensures every student_id and course_id in enrollments points to a valid record in the parent tables

Without foreign keys, the application code would need to manually check existence before every insert. Mistakes would create broken links where enrollment records point to deleted students.

### Exercise 3: Data Insertion

Empty tables are useless. Inserting sample data demonstrates how the database stores and links information.

**Inserting students:**

```sql
INSERT INTO students (first_name, last_name, email, date_of_birth) VALUES
('John', 'Doe', 'john.doe@example.com', '1995-04-12'),
('Fatai', 'Asekun', 'fatai@mail.com', '2000-11-14'),
('Prime', 'Shell', 'prime@mail.com', '2001-02-09'),
('Kali', 'user', 'kali@mail.com', '1990-10-10');
```

![Students table populated with four records](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_07%20Students%20table%20populated%20with%20four%20records.jpg)


The database automatically assigned student_id values (1, 2, 3, 4) using AUTO_INCREMENT. I only provided the data, MySQL handled the unique identifiers.

**Inserting courses:**

```sql
INSERT INTO courses (course_name, credits) VALUES
('Introduction to Databases', 3),
('Web Development', 4),
('Cybersecurity Fundamentals', 3);
```

![Courses table showing three courses](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_08%20Courses%20table%20showing%20three%20courses.jpg)

Again, course_id values (1, 2, 3) were generated automatically.

**Inserting enrollments:**

```sql
INSERT INTO enrollments (student_id, course_id, enrollment_date) VALUES
(1, 1, '2024-01-15'),
(1, 2, '2024-01-16'),
(1, 3, '2024-01-17'),
(2, 1, '2024-01-16'),
(2, 2, '2024-02-17'),
(2, 3, '2024-02-18'),
(3, 1, '2024-03-17');
```

![Enrollments table showing student-course relationships](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_09%20Enrollments%20table%20showing%20student-course%20relationships.jpg)

![Enrollments table showing student-course relationships](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_09B%20Enrollments%20table%20showing%20student-course%20relationships.jpg)


**Reading the data:**

- Enrollment 1: student_id=1 (John Doe) enrolled in course_id=1 (Introduction to Databases) on 2024-01-15
- Enrollment 2: student_id=1 (John Doe) enrolled in course_id=2 (Web Development) on 2024-01-16
- Enrollment 4: student_id=2 (Fatai Asekun) enrolled in course_id=1 (Introduction to Databases) on 2024-01-16

Notice how student and course information is not duplicated in the enrollments table. It only stores the foreign keys (student_id, course_id) that point to the full records in their respective tables.

**Why data consistency matters:**

If I tried to insert:

```sql
INSERT INTO enrollments (student_id, course_id, enrollment_date) VALUES (999, 1, '2024-01-20');
```

The database would reject it with a foreign key constraint violation. Student_id 999 does not exist in the students table, so the enrollment cannot reference it. This prevents logic breaks where the application tries to display enrollment information for a non-existent student.

Without foreign keys, the INSERT would succeed, but queries would return NULL for the student's name because the JOIN would find no matching record. The application would crash or display broken data.

### Exercise 4: Querying and Data Manipulation

Databases become useful when you can retrieve specific information through queries.

**Retrieving all students:**

```sql
SELECT * FROM students;
```

![Query result showing all four students](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_10%20Query%20result%20showing%20all%20four%20students.jpg)


The asterisk (*) means "all columns". This query returned every field for every student: student_id, first_name, last_name, email, date_of_birth.

**Finding students enrolled in a specific course:**

This requires joining three tables:

- students (for names)
- enrollments (for the link)
- courses (for the course name filter)

```sql
SELECT s.first_name, s.last_name, c.course_name
FROM students s
JOIN enrollments e ON s.student_id = e.student_id
JOIN courses c ON e.course_id = c.course_id
WHERE c.course_name = 'Web Development';
```

![Query result showing students enrolled in Web Development](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_11A%20Query%20result%20showing%20students%20enrolled%20in%20Web%20Development.jpg)

![Query result showing students enrolled in Web Development](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_11B%20Query%20result%20showing%20students%20enrolled%20in%20Web%20Development.jpg)


**Breaking down the JOIN logic:**

1. Start with the students table (aliased as `s`)
2. JOIN enrollments table (aliased as `e`) ON the condition that `s.student_id = e.student_id`. This links each student to their enrollment records
3. JOIN courses table (aliased as `c`) ON the condition that `e.course_id = c.course_id`. This links each enrollment to its course details
4. Filter the results WHERE `c.course_name = 'Web Development'`

The result shows three students (John, Fatai, Prime) enrolled in Web Development. Without the JOIN, I would only see student_id values (1, 2, 3) instead of readable names.

**Updating a student's email address:**

```sql
UPDATE students
SET email = 'john.newemail@example.com'
WHERE first_name = 'John' AND last_name = 'Doe';
```

![Update query confirmation and updated record](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_12A%20Update%20query%20confirmation%20and%20updated%20record.jpg)

![Update query confirmation and updated record](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_12B%20Update%20query%20confirmation%20and%20updated%20record.jpg)

The WHERE clause is critical. Without it, the UPDATE would change every student's email to john.newemail@example.com. The query affected 1 row because only one student matched the WHERE condition.

**Deleting an enrollment record:**

```sql
DELETE FROM enrollments
WHERE student_id = 3 AND course_id = 3;
```

![Delete confirmation dialog in phpMyAdmin](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/36_13%20Delete%20confirmation%20dialog%20in%20phpMyAdmin.jpg)


This removed the record where student_id=3 (Prime Shell) was enrolled in course_id=3 (Cybersecurity Fundamentals). The deletion does not affect the students or courses tables because it only removed the link between them.

**How JOIN operations work:**

Relational databases normalize data by spreading it across multiple tables. A raw enrollment record looks like:

```
enrollment_id: 1
student_id: 1
course_id: 2
enrollment_date: 2024-01-16
```

This is efficient for storage but useless for display. Users need to see "John Doe enrolled in Web Development", not "student_id 1 enrolled in course_id 2".

JOIN instructs the database to:

1. Look up student_id=1 in the students table → find "John Doe"
2. Look up course_id=2 in the courses table → find "Web Development"
3. Combine the results into a single row with first_name, last_name, and course_name

This happens automatically in milliseconds. Without JOIN, the application would need to:

1. Query enrollments to get student_id and course_id
2. Query students separately using student_id
3. Query courses separately using course_id
4. Manually combine the results in code

This is slower, error-prone, and requires multiple round trips to the database.

## Findings

- **Relational databases organize information into normalized tables linked by foreign keys.** The student management system uses three tables (students, courses, enrollments) instead of one giant table with duplicated student and course information in every row. This prevents update anomalies and saves storage space.

- **Primary keys provide unique identification, foreign keys enforce relationships.** Every table has a PRIMARY KEY column (student_id, course_id, enrollment_id) that uniquely identifies each record. Foreign keys in the enrollments table (student_id, course_id) must reference valid PRIMARY KEY values in their parent tables. The database rejects invalid references automatically.

- **ACID compliance prevents data corruption during failures.** If MySQL crashes while updating a student's email, the transaction rolls back entirely instead of leaving the email field partially modified. This guarantees the database is always in a consistent state, even after power failures or system crashes.

- **Data insertion requires respecting foreign key constraints.** Cannot enroll a student_id that does not exist in the students table. Cannot reference a course_id that has been deleted. The database enforces these rules at the engine level, making it impossible for application bugs to create orphaned records.

- **SQL provides structured commands for data manipulation.** SELECT retrieves records, INSERT adds new data, UPDATE modifies existing records, DELETE removes data. WHERE clauses filter operations to specific rows. All operations are atomic and logged for recovery.

- **JOIN operations combine related data from multiple tables into coherent results.** Instead of displaying raw student_id and course_id values, JOINs follow foreign key relationships to retrieve human-readable names. This transforms normalized storage into useful output for web applications.

- **Unique constraints prevent duplicate entries at the database level.** The email column has a UNIQUE constraint, so attempting to INSERT a second student with john.doe@example.com fails with a constraint violation error. This prevents duplicate accounts without requiring application-level validation.

## Challenges Faced

- **Understanding foreign key directionality:** Initially, I was confused about which table should contain the foreign key. The rule is: the "many" side of a relationship stores the foreign key pointing to the "one" side. In a one-to-many relationship (one course has many enrollments), the enrollments table stores course_id as a foreign key. In a many-to-many relationship (students can take many courses, courses can have many students), a junction table (enrollments) stores both foreign keys.

**JOIN syntax confusion:** The difference between INNER JOIN, LEFT JOIN, and RIGHT JOIN was unclear at first. Running test queries helped:

- INNER JOIN returns only rows where matches exist in both tables
- LEFT JOIN returns all rows from the left table, with NULL for unmatched right table columns
- RIGHT JOIN is the reverse of LEFT JOIN

For finding students enrolled in Web Development, INNER JOIN was correct because I only wanted students who actually have enrollment records.

**WHERE clause importance in UPDATE and DELETE:** Forgetting the WHERE clause in an UPDATE or DELETE statement affects every row in the table. I tested this in a backup database first and saw the entire students table get the same email address. Always include WHERE clauses to limit scope.

## Key Takeaways

- **Databases are not just storage, they are logic enforcers.** Foreign keys, UNIQUE constraints, and data types prevent invalid data from entering the system. The database engine validates every INSERT, UPDATE, and DELETE against defined rules. This is more reliable than trusting application code to always validate correctly.

- **Normalization trades storage space for data integrity.** Storing student names once in the students table instead of repeating them in every enrollment record uses less disk space and prevents inconsistencies. If a student changes their name, one UPDATE fixes it everywhere. Denormalized databases (like spreadsheets) require finding and updating dozens of scattered records.

- **AUTO_INCREMENT eliminates manual ID management.** Letting the database generate primary key values prevents collisions where two students get the same student_id. The application only provides the data (name, email, date of birth), and MySQL handles unique identification automatically.

- **JOINs transform foreign key IDs into readable information.** Raw database tables store integer IDs (student_id=1, course_id=2) for efficiency. JOIN operations convert these into human-readable output (John Doe, Web Development) by following foreign key relationships. This is what makes relational databases usable for web applications.

- **SQL is declarative, not procedural.** You specify what data you want (SELECT first_name, last_name WHERE course_name = 'Web Development'), not how to retrieve it. The database query optimizer determines the fastest execution plan automatically. This is different from writing loops and conditionals in application code.

- **Transaction safety prevents partial failures.** ACID compliance guarantees that multi-step operations (like enrolling a student, updating their account balance, and logging the transaction) either complete fully or roll back entirely. There is no "student enrolled but balance not updated" intermediate state.

## Disclaimer

This lab was performed in a local development environment using XAMPP and phpMyAdmin for educational purposes. The student management database contains fictional sample data created specifically for this exercise. No production systems or real student information were accessed.
