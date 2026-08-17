# Data-Analytics-Portfolio
For this project, I dug into a hospital's operational data using T-SQl, to see what patterns show up across patients, appointments, doctors, and billings. I worked in SQl Server Management Studio, pulling from five linked tables; patients, appointments, doctors, treatments, and billing, to answer the kinds of questions a hospital admin team would actually care about, Who are our patients, and how has that changed over time?, which specialties have the most cancellations or the highest treatment costs?, where are payments failing and what is behind it?
Below, I have broken the analysis into individual queries. For each one, I state the question I was trying to answer, show the SQL, share the result, and explain what it means in plain terms

1. What is the total number of patients
SELECT COUNT(DISTINCT patient_id) AS total_patients
FROM   dbo.patients;
| total_patients |
| --- |
| 50 |

There are a total of 50 patients in the hospital

2. What is the lowest age, highest age, and average age among the patients
SELECT AVG(DATEDIFF(YEAR, date_of_birth, GETDATE()) - CASE WHEN MONTH(date_of_birth) > MONTH(GETDATE())
                                                                OR MONTH(date_of_birth) = MONTH(GETDATE())
                                                                   AND DAY(date_of_birth) > DAY(GETDATE()) THEN 1 ELSE 0 END) AS avg_age,
       MIN(DATEDIFF(YEAR, date_of_birth, GETDATE()) - CASE WHEN MONTH(date_of_birth) > MONTH(GETDATE())
                                                                OR MONTH(date_of_birth) = MONTH(GETDATE())
                                                                   AND DAY(date_of_birth) > DAY(GETDATE()) THEN 1 ELSE 0 END) AS min_age,
       MAX(DATEDIFF(YEAR, date_of_birth, GETDATE()) - CASE WHEN MONTH(date_of_birth) > MONTH(GETDATE())
                                                                OR MONTH(date_of_birth) = MONTH(GETDATE())
                                                                   AND DAY(date_of_birth) > DAY(GETDATE()) THEN 1 ELSE 0 END) AS max_age
   | avg_age | min_age | max_age |
| --- | --- | --- |
| 45 | 21 | 76 |

The youngest patient in the hospital is 21 years old, while the oldest is 76 years old. The mean age among the patients is 45 years.

3. Which age ranges do the patients fit into and what are the percentages

WITH     age_cte
AS       (SELECT DATEDIFF(YEAR, date_of_birth, GETDATE()) - CASE WHEN MONTH(date_of_birth) > MONTH(GETDATE())
                                                                      OR MONTH(date_of_birth) = MONTH(GETDATE())
                                                                         AND DAY(date_of_birth) > DAY(GETDATE()) THEN 1 ELSE 0 END AS Age,
                 patient_id
          FROM   dbo.patients)
SELECT   CASE WHEN age BETWEEN 20 AND 29 THEN '20-29' WHEN age BETWEEN 30 AND 39 THEN '30-39' WHEN age BETWEEN 40 AND 49 THEN '40-49' WHEN age BETWEEN 50 AND 59 THEN '50-59' WHEN age >= 60 THEN '60 and above' END AS age_groups,
         COUNT(DISTINCT patient_id) AS patient_count,
         CONCAT(CAST (COUNT(DISTINCT patient_id) * 100.0 / SUM(COUNT(DISTINCT patient_id)) OVER () AS DECIMAL (5, 2)), '%') AS percentage
FROM     age_cte
GROUP BY CASE WHEN age BETWEEN 20 AND 29 THEN '20-29' WHEN age BETWEEN 30 AND 39 THEN '30-39' WHEN age BETWEEN 40 AND 49 THEN '40-49' WHEN age BETWEEN 50 AND 59 THEN '50-59' WHEN age >= 60 THEN '60 and above' END;

| age_groups | patient_count | percentage |
| --- | --- | --- |
| 20-29 | 9 | 18.00% |
| 30-39 | 13 | 26.00% |
| 40-49 | 7 | 14.00% |
| 50-59 | 9 | 18.00% |
| 60 and above | 12 | 24.00% |

The age range 30 - 39 makes up the largest group representing 26% of the patients, followed by 60 and above which makes up 24%. 40 - 49 ism the smallest age group, making up 14% of the patient population

4. What is the breakdown of the patient population by gender with percentages
SELECT   gender,
         COUNT(DISTINCT patient_id) AS patient_count,
         CONCAT(CAST (COUNT(DISTINCT patient_id) * 100.0 / SUM(COUNT(DISTINCT patient_id)) OVER () AS DECIMAL (5, 2)), '%') AS percentage
FROM     dbo.patients
GROUP BY gender;

| gender | patient_count | percentage |
| --- | --- | --- |
| F | 19 | 38.00% |
| M | 31 | 62.00% |

Males make up the majority of the patient population at 62%, while Females constitute 38%
