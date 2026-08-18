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

Dr Sarah Smith had the most appointments with 29 appointments, Dr David Taylor had the second highest, while Dr Alex Davis had the third highest with 25 and 29 appointments respectively. Dr Robert Davis had the lowest number of appointments, with 13.
