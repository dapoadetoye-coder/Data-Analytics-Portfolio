# Data-Analytics-Portfolio
For this project, I dug into a hospital's operational data using T-SQl, to see what patterns show up across patients, appointments, doctors, and billings. I worked in SQl Server Management Studio, pulling from five linked tables; patients, appointments, doctors, treatments, and billing, to answer the kinds of questions a hospital admin team would actually care about, Who are our patients, and how has that changed over time?, which specialties have the most cancellations or the highest treatment costs?, where are payments failing and what is behind it?
Below, I have broken the analysis into individual queries. For each one, I state the question I was trying to answer, show the SQL, share the result, and explain what it means in plain terms

1. What is the total number of patients


```SQL
SELECT COUNT(DISTINCT patient_id) AS total_patients
FROM   dbo.patients;
```
| total_patients |
| --- |
| 50 |

There are a total of 50 patients in the hospital

2. What is the lowest age, highest age, and average age among the patients
 
```SQL
SELECT AVG(DATEDIFF(YEAR, date_of_birth, GETDATE()) - CASE WHEN MONTH(date_of_birth) > MONTH(GETDATE())
                                                                OR MONTH(date_of_birth) = MONTH(GETDATE())
                                                                   AND DAY(date_of_birth) > DAY(GETDATE()) THEN 1 ELSE 0 END) AS avg_age,
       MIN(DATEDIFF(YEAR, date_of_birth, GETDATE()) - CASE WHEN MONTH(date_of_birth) > MONTH(GETDATE())
                                                                OR MONTH(date_of_birth) = MONTH(GETDATE())
                                                                   AND DAY(date_of_birth) > DAY(GETDATE()) THEN 1 ELSE 0 END) AS min_age,
       MAX(DATEDIFF(YEAR, date_of_birth, GETDATE()) - CASE WHEN MONTH(date_of_birth) > MONTH(GETDATE())
                                                                OR MONTH(date_of_birth) = MONTH(GETDATE())
                                                                   AND DAY(date_of_birth) > DAY(GETDATE()) THEN 1 ELSE 0 END) AS max_age
```
   
   | avg_age | min_age | max_age |
| --- | --- | --- |
| 45 | 21 | 76 |

The youngest patient in the hospital is 21 years old, while the oldest is 76 years old. The mean age among the patients is 45 years.

3. Which age ranges do the patients fit into and what are the percentages

```SQL
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
```
| age_groups | patient_count | percentage |
| --- | --- | --- |
| 20-29 | 9 | 18.00% |
| 30-39 | 13 | 26.00% |
| 40-49 | 7 | 14.00% |
| 50-59 | 9 | 18.00% |
| 60 and above | 12 | 24.00% |

The age range 30 - 39 makes up the largest group representing 26% of the patients, followed by 60 and above which makes up 24%. 40 - 49 ism the smallest age group, making up 14% of the patient population

4. What is the breakdown of the patient population by gender with percentages
   
```SQL
SELECT   gender,
         COUNT(DISTINCT patient_id) AS patient_count,
         CONCAT(CAST (COUNT(DISTINCT patient_id) * 100.0 / SUM(COUNT(DISTINCT patient_id)) OVER () AS DECIMAL (5, 2)), '%') AS percentage
FROM     dbo.patients
GROUP BY gender;
```
| gender | patient_count | percentage |
| --- | --- | --- |
| F | 19 | 38.00% |
| M | 31 | 62.00% |

Males make up the majority of the patient population at 62%, while Females constitute 38%

5. Rank the Insurance providers in Descending Order
```SQL
   SELECT   insurance_provider,
         COUNT(DISTINCT patient_id) AS patient_count,
         ROW_NUMBER() OVER (ORDER BY COUNT(DISTINCT patient_id) DESC) AS rank
FROM     dbo.patients
GROUP BY insurance_provider;
```

| insurance_provider | patient_count | rank |
| --- | --- | --- |
| MedCare Plus | 18 | 1 |
| WellnessCorp | 16 | 2 |
| PulseSecure | 10 | 3 |
| HealthIndia | 6 | 4 |

Med care plus is the most popular Insurance provider, Providing insurance coverage for 18 of the 50 patients, WelnessCorp comes at a close second with 16 patients, HealthIndia covers only 6 patients

6. How many patients registered each year, and what is the year over year change in registrations
```SQL
SELECT   YEAR(registration_date) AS year,
         COUNT(patient_id) AS patient_count,
         LAG(COUNT(patient_id)) OVER (ORDER BY YEAR(registration_date)) AS previous_year,
         COUNT(patient_id) - LAG(COUNT(patient_id)) OVER (ORDER BY YEAR(registration_date)) AS previous_year_change
FROM     dbo.patients
GROUP BY YEAR(registration_date);
```
| year | patient_count | previous_year | previous_year_change |
| --- | --- | --- | --- |
| 2021 | 21 | NULL | NULL |
| 2022 | 17 | 21 | -4 |
| 2023 | 12 | 17 | -5 |

Patient registration has been on a downward trend, It started at 21 patients registering in the year 2021, followed by 17 in 2022, a decline by 4 patients. 12 patients registered in 2023, which is a reduction by 5 patients compared to the previous year.

7. What is the monthly trend in patient registrations and what is the cumulative(running) total of patients registered over time

```SQL
SELECT   FORMAT(registration_date, 'MMM-yyyy') AS month,
         COUNT(patient_id) AS patient_count,
         SUM(COUNT(patient_id)) OVER (ORDER BY MIN(registration_date) ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM     dbo.patients
GROUP BY FORMAT(registration_date, 'MMM-yyyy')
ORDER BY MIN(registration_date);
```
| month | patient_count | running_total |
| --- | --- | --- |
| Jan-2021 | 1 | 1 |
| Mar-2021 | 2 | 3 |
| Apr-2021 | 1 | 4 |
| May-2021 | 3 | 7 |
| Jul-2021 | 2 | 9 |
| Aug-2021 | 2 | 11 |
| Sep-2021 | 5 | 16 |
| Oct-2021 | 2 | 18 |
| Dec-2021 | 3 | 21 |
| Jan-2022 | 2 | 23 |
| Feb-2022 | 1 | 24 |
| Mar-2022 | 1 | 25 |
| Apr-2022 | 1 | 26 |
| May-2022 | 1 | 27 |
| Jun-2022 | 2 | 29 |
| Jul-2022 | 2 | 31 |
| Aug-2022 | 1 | 32 |
| Sep-2022 | 4 | 36 |
| Oct-2022 | 2 | 38 |
| Jan-2023 | 1 | 39 |
| Apr-2023 | 3 | 42 |
| May-2023 | 1 | 43 |
| Jun-2023 | 4 | 47 |
| Jul-2023 | 1 | 48 |
| Sep-2023 | 1 | 49 |
| Dec-2023 | 1 | 50 |

The table above shows a running total of the patient by registration month. September 2021 had the highest number of registrations with 5 new patients, September of 2022 had the second highest with 4 new patients. However, September of 2023 did not follow this pattern, having only 1 new registration.

8. What are the reasons for the Patient visits

```SQL
SELECT
COUNT(appointment_id) AS Total
FROM dbo.appointments;

SELECT   reason_for_visit,
         COUNT(appointment_id) AS patient_count
FROM     dbo.appointments
GROUP BY reason_for_visit
ORDER BY patient_count DESC;
```
| Total |
| --- |
| 200 |

| reason_for_visit | patient_count |
| --- | --- |
| Checkup | 45 |
| Consultation | 43 |
| Therapy | 42 |
| Follow-up | 41 |
| Emergency | 29 |

There were a total of 200 appointments, 45 of them were for Checkup, Consultation, Therapy and Follow up were also common reasons for visit, with 43, 42 and 41 appointments respectively. Emergencies were the lowest with only 29 appointments.

9. How many appointments ended up completed vs cancelled vs no-show vs re-scheduled
```SQL
SELECT   status,
         COUNT(appointment_id) AS patient_count
FROM     dbo.appointments
GROUP BY status
ORDER BY patient_count DESC;
```
| status | patient_count |
| --- | --- |
| No-show | 52 |
| Scheduled | 51 |
| Cancelled | 51 |
| Completed | 46 |

10. Which Doctors have the most appointments with patients
(Involved Joining the Appointment and Doctors table)
```SQL
WITH     doctor_appointment_cte
AS       (SELECT   'Dr' + ' ' + CONCAT(D.first_name, ' ', D.last_name) AS doctor_name,
                   A.appointment_id,
                   A.status
          FROM     dbo.appointments AS A
                   LEFT JOIN
                   dbo.doctors AS D
                   ON A.doctor_id = D.doctor_id
          GROUP BY CONCAT(D.first_name, ' ', D.last_name), A.appointment_id, A.status)
SELECT   doctor_name,
         COUNT(appointment_id) AS appointment_count
FROM     doctor_appointment_cte
GROUP BY doctor_name;
```
| doctor_name | appointment_count |
| --- | --- |
| Dr Sarah Taylor | 29 |
| Dr David Taylor | 25 |
| Dr Alex Davis | 24 |
| Dr Jane Smith | 22 |
| Dr Jane Davis | 21 |
| Dr Linda Wilson | 19 |
| Dr Sarah Smith | 17 |
| Dr Linda Brown | 16 |
| Dr David Jones | 14 |
| Dr Robert Davis | 13 |

Dr Sarah Taylor had the most appointments with 29 appointments, Dr David Taylor had the second highest, while Dr Alex Davis had the third highest with 25 and 29 appointments respectively. Dr Robert Davis had the lowest number of appointments, with 13.

11. Which of these Doctors have the most cancelled appointments
```SQL
WITH     doctor_appointment_cte
AS       (SELECT   'Dr' + ' ' + CONCAT(D.first_name, ' ', D.last_name) AS doctor_name,
                   A.appointment_id,
                   A.status
          FROM     dbo.appointments AS A
                   LEFT OUTER JOIN
                   dbo.doctors AS D
                   ON A.doctor_id = D.doctor_id
          GROUP BY CONCAT(D.first_name, ' ', D.last_name), A.appointment_id, A.status)
SELECT   doctor_name,
         COUNT(appointment_id) AS appointment_count
FROM     doctor_appointment_cte
WHERE    status = 'cancelled'
GROUP BY doctor_name
ORDER BY appointment_count DESC; 
```
| doctor_name | appointment_count |
| --- | --- |
| Dr Jane Davis | 8 |
| Dr David Taylor | 7 |
| Dr Sarah Taylor | 6 |
| Dr Alex Davis | 6 |
| Dr Linda Brown | 5 |
| Dr Robert Davis | 5 |
| Dr Sarah Smith | 4 |
| Dr Jane Smith | 4 |
| Dr David Jones | 3 |
| Dr Linda Wilson | 3 |

Dr Jane Davis had the most cancelled appointments with 8, Dr David Taylor comes at a close second, having 7 cancelled appointments. Dr Sarah Taylor and Alex Davis both have 6 cancelled appointments.
At the bottom of the cancelled appointment list is Dr David Jone and Dr Linda Wilson

12. What are the preferred payment methods?
```SQL
SELECT   payment_method,
         COUNT(patient_id) AS count,
         CONCAT(CAST(COUNT(patient_id)*100/SUM(COUNT(patient_id)) OVER() AS DECIMAL(5,2)), '%') AS percentage
FROM dbo.billing
GROUP BY payment_method
```
| payment_method | count | percentage |
| --- | --- | --- |
| Cash | 61 | 30.00% |
| Credit Card | 75 | 37.00% |
| Insurance | 64 | 32.00% |

Credit card is the most preferred payment method, being used by 37% of the patients. Insurance in used by 32% of the patients. Cash is the least preferred payment method, utilized in only 30% of payments.

13. What is the status of the payment made?
```SQL
SELECT   payment_status,
         COUNT(patient_id) AS count,
         CONCAT(CAST(COUNT(patient_id)*100/SUM(COUNT(patient_id)) OVER() AS decimal(5,2)), '%') AS percentage
         FROM     dbo.billing
GROUP BY payment_status;
```
| payment_status | count | percentage |
| --- | --- | --- |
| Failed | 67 | 33.00% |
| Paid | 64 | 32.00% |
| Pending | 69 | 34.00% |

Payment status is almost evenly splited into 3 thirds, a third (33%) failed, 32% went through, while 34% showed pending.

14. Which payment method has the most failed transactions
```SQL
SELECT   payment_method,
         COUNT(patient_id) AS count
FROM     dbo.billing
WHERE    payment_status = 'failed'
GROUP BY payment_method;
```
| payment_method | count |
| --- | --- |
| Cash | 23 |
| Credit Card | 23 |
| Insurance | 21 |

The number of failed transactions was similar across the 3 payment methods, Cash and Credit Card had 23 while Insurance had 21.

15. What is Lowest, Highest and Average amount billed to the patients
```SQL
SELECT 
ROUND(AVG(amount), 2) AS avg_amount,
ROUND(MIN(amount), 2) AS min_amount,
ROUND(MAX(amount), 2) AS max_amount
FROM    dbo.billing;
```

| avg_amount | min_amount | max_amount |
| --- | --- | --- |
| 2756.25 | 534.03 | 4973.63 |

The average amount billed is 2,756.25. The highest amount is 4973.63 while the lowest is 534.03

17. Which Doctor and Branch of the Hospital costs the most
(Involved Joining the Appointment, Treatment and Doctor Tables)
```SQL
WITH cost_cte AS (
SELECT 
'Dr' + ' ' + CONCAT(D.first_name, ' ', D.last_name) doctor_name,
D.specialization,
D.hospital_branch,
T.cost
FROM dbo.treatments AS T
JOIN dbo.appointments as A
ON T.appointment_id = A.appointment_id
JOIN dbo.doctors AS D
ON A.doctor_id = D.doctor_id)

SELECT
doctor_name,
specialization,
hospital_branch,
ROUND(AVG(cost), 2) avg_cost_per_doctor
FROM cost_cte
GROUP BY doctor_name,
specialization,
hospital_branch
ORDER BY avg_cost_per_doctor DESC
```
| doctor_name | specialization | hospital_branch | avg_cost_per_doctor |
| --- | --- | --- | --- |
| Dr Linda Brown | Dermatology | Westside Clinic | 3339.21 |
| Dr Robert Davis | Oncology | Westside Clinic | 3089.73 |
| Dr Alex Davis | Pediatrics | Central Hospital | 2899.42 |
| Dr Sarah Taylor | Dermatology | Central Hospital | 2851.6 |
| Dr Jane Davis | Pediatrics | Eastside Clinic | 2847.78 |
| Dr David Jones | Pediatrics | Central Hospital | 2808.28 |
| Dr David Taylor | Dermatology | Westside Clinic | 2663.42 |
| Dr Linda Wilson | Oncology | Eastside Clinic | 2601.91 |
| Dr Jane Smith | Pediatrics | Eastside Clinic | 2399.61 |
| Dr Sarah Smith | Pediatrics | Central Hospital | 2202.41 |

Dr Linda Brown a Dermatologist at Westside clinic, Her appointment costs the most at 3,339.21, Dr Robert Davis at the same hospital costs 3089.73. The cheapest service can be found at Central Hospital with Dr Sarah Smith who is a Pediatrician and whose appointment costs 2202.42

18. What is the average cost of treatment based on the treatment type
```SQL
SELECT
treatment_type,
AVG(cost) avg_cost
FROM 
dbo.treatments
GROUP BY treatment_type
ORDER BY avg_cost DESC
```
| treatment_type | avg_cost |
| --- | --- |
| MRI | 3224.95 |
| Physiotherapy | 2761.61 |
| X-Ray | 2698.87 |
| Chemotherapy | 2629.71 |
| ECG | 2532.22 |

The most expensive treatment type is the MRI, which costs on an average, 3224.95, followed by Physiotherapy which costs 2761.61. ECG is the cheapest treatment type, with the average cost being 2532.22

20. What is the duration between appointment day and treatment day (treatment delay)
```SQL
SELECT TOP 10
A.patient_id,
A.appointment_date,
T.treatment_date,
DATEDIFF(DAY, appointment_date, treatment_date) treatment_delay
FROM dbo.appointments AS A
LEFT JOIN dbo.treatments AS T
ON A.appointment_id = T.appointment_id
ORDER BY treatment_delay DESC
```

| patient_id | appointment_date | treatment_date | treatment_delay |
| --- | --- | --- | --- |
| 22.00 | 2023-11-12 | 2023-11-12 | 0 |
| 5.00 | 2023-01-13 | 2023-01-13 | 0 |
| 39.00 | 2023-03-05 | 2023-03-05 | 0 |
| 16.00 | 2023-05-24 | 2023-05-24 | 0 |
| 1.00 | 2023-04-09 | 2023-04-09 | 0 |
| 45.00 | 2023-06-19 | 2023-06-19 | 0 |
| 40.00 | 2023-07-06 | 2023-07-06 | 0 |
| 25.00 | 2023-09-01 | 2023-09-01 | 0 |
| 48.00 | 2023-06-28 | 2023-06-28 | 0 |
| 32.00 | 2023-06-09 | 2023-06-09 | 0 |

There is no treatment delay for any patient. All patients were treated on their appointment dates. 
Note: Top 10 was shown to prevent an ambiguous table when there is no treatment delay in any patient
