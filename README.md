# SQL--GoodThought-NGO-Data-Analysis

## Project Overview
GoodThought NGO has been a driving force in education, healthcare, and sustainable development, aiming to create meaningful change worldwide. The organization manages various assignments designed to uplift underprivileged communities, and this project provides a hands-on opportunity to explore how data-driven insights can optimize these humanitarian efforts.

### Dataset Overview
The GoodThought PostgreSQL database (2010-2023) contains information about:
- **Assignments**: Project details such as name, duration, budget, region, and impact score.
- **Donations**: Financial contributions linked to donors and assignments.
- **Donors**: Information on individuals and organizations funding GoodThought’s initiatives.

### Objectives
The project focuses on two key SQL exercises:
1. **Identifying the top five assignments based on total donation value, categorized by donor type.**
2. **Finding the highest impact assignment per region, ensuring they received at least one donation.**

The goal is to compare different SQL approaches to achieving the same results, analyzing performance and readability.

---

## Task 1: Top Five Assignments by Donation Value
### **Objective:**
Find the five assignments that received the highest total donation amounts, categorized by donor type.

### **Approach 1: Using Direct Aggregation in a Subquery**
```sql
SELECT a.assignment_name,
       a.region,
       ROUND(SUM(d.amount), 2) AS rounded_total_donation_amount,
       d.donor_type
FROM assignments AS a
JOIN (SELECT assignment_id,
            amount,
            donor_type
      FROM donations
      JOIN donors USING (donor_id)) AS d
USING (assignment_id)
GROUP BY a.assignment_name, a.region, d.donor_type
ORDER BY rounded_total_donation_amount DESC
LIMIT 5;
```
**Pros:**
- Simple and direct.
- Easily understandable for smaller datasets.

**Cons:**
- Repeated aggregations in case of larger datasets.
- Performance may degrade with extensive joins.

### **Approach 2: Using a CTE for Pre-aggregation**
```sql
WITH donor_donations AS (
    SELECT assignment_id,
           ROUND(SUM(amount), 2) AS rounded_total_donation_amount,
           donor_type
    FROM donations
    JOIN donors USING (donor_id)
    GROUP BY assignment_id, donor_type
)
SELECT a.assignment_name,
       a.region,
       d.rounded_total_donation_amount,
       d.donor_type
FROM assignments AS a
JOIN donor_donations AS d USING (assignment_id)
ORDER BY rounded_total_donation_amount DESC
LIMIT 5;
```
**Pros:**
- Optimized performance due to pre-aggregated calculations.
- Easier to read and debug.

**Cons:**
- Requires familiarity with Common Table Expressions (CTEs).

---

## Task 2: Highest Impact Assignment per Region
### **Objective:**
Find the highest-impact assignment in each region that has received at least one donation.

### **Approach 1: Using ROW_NUMBER() for Ranking**
```sql
WITH donation_counts AS (
    SELECT assignment_id,
           COUNT(donation_id) AS num_total_donations
    FROM donations
    GROUP BY assignment_id
),
ranked_assignments AS (
    SELECT a.assignment_name,
           a.region,
           a.impact_score,
           dc.num_total_donations,
           ROW_NUMBER() OVER (PARTITION BY a.region ORDER BY a.impact_score DESC) AS rank_in_region
    FROM assignments a
    JOIN donation_counts dc ON a.assignment_id = dc.assignment_id
    WHERE dc.num_total_donations > 0
)
SELECT assignment_name,
       region,
       impact_score,
       num_total_donations
FROM ranked_assignments
WHERE rank_in_region = 1
ORDER BY region ASC;
```
**Pros:**
- Allows for handling ties or ranking multiple results.
- Easily extendable if we want second or third best assignments per region.

**Cons:**
- ROW_NUMBER() can be computationally expensive for large datasets.

### **Approach 2: Using DISTINCT ON()**
```sql
WITH donation_counts AS (
    SELECT assignment_id, COUNT(donation_id) AS num_total_donations
    FROM donations
    GROUP BY assignment_id
    HAVING COUNT(donation_id) > 0
)
SELECT DISTINCT ON (a.region) 
       a.assignment_name,
       a.region,
       a.impact_score,
       dc.num_total_donations
FROM assignments a
JOIN donation_counts dc ON a.assignment_id = dc.assignment_id
ORDER BY a.region, a.impact_score DESC;
```
**Pros:**
- More efficient for selecting one row per group.
- Simpler to read compared to window functions.

**Cons:**
- Not standard SQL (PostgreSQL-specific feature).
- Less flexible for handling ties.

---

## Results
### Task 1: Highest Donation Assignments

| index | assignment_name  | region | rounded_total_donation_amount | donor_type  |
|-------|-----------------|--------|------------------------------|-------------|
| 0     | Assignment_3033 | East   | 3840.66                      | Individual  |
| 1     | Assignment_300  | West   | 3133.98                      | Organization |
| 2     | Assignment_4114 | North  | 2778.57                      | Organization |
| 3     | Assignment_1765 | West   | 2626.98                      | Organization |
| 4     | Assignment_268  | East   | 2488.69                      | Individual  |

The results reveal that the highest total donations were received by Assignment_3033 (East region), with €3,840.66 from individual donors. This indicates that individual contributions can sometimes outweigh organizational ones. However, organizations dominate the list, contributing significantly to assignments in the West and North regions, suggesting that certain regions may attract more structured funding sources.

The West region appears twice, implying a strong presence of well-funded assignments in this area. Meanwhile, the South region is absent, potentially highlighting a funding gap in that area.

### Task 2: Top Regional Impact Assignments

| index | assignment_name  | region | impact_score | num_total_donations |
|-------|-----------------|--------|--------------|----------------------|
| 0     | Assignment_316  | East   | 10.00        | 2                    |
| 1     | Assignment_2253 | North  | 9.99         | 1                    |
| 2     | Assignment_3547 | South  | 10.00        | 1                    |
| 3     | Assignment_2794 | West   | 9.99         | 2                    |

The impact score rankings show that the highest-rated assignments per region all have scores close to 10, indicating strong outcomes. The East and South regions have assignments with a perfect impact score of 10, while North and West assignments follow closely at 9.99.

Interestingly, despite Assignment_316 (East) and Assignment_2794 (West) having multiple donations, Assignment_2253 (North) and Assignment_3547 (South) still achieved top impact scores with just a single donation. This suggests that impact is not solely dependent on the number of donations but also on how effectively the funds are utilized.

Additionally, the South region, which was absent in the donation ranking, appears in the impact ranking, suggesting that while funding may be lower there, it does not necessarily hinder achieving high-impact outcomes.

---

## Further Analysis and Visualizations
![image](https://github.com/user-attachments/assets/085c06ae-1925-4719-b325-92702674c4f8)
As a general trend, we can see that the higher the donation amount, the higher the impact score.

Invidual donations on South region were the most impactful and significant, and the Corporate North the least. It's interesing to note that individual donations seem to be the most impactful (more on that next).


![image](https://github.com/user-attachments/assets/15da6187-ea82-444d-882d-ba07b6117ce0)
Continuing with the observation made before, here it's even more clear that the Individual South donations were the more significant.
Corporate donations as a hole were the least significant and Organization donations had a high impact as a hole, showing good scores in all regions.

---

## Conclusion
This project demonstrates how different SQL techniques can achieve the same results, with trade-offs in efficiency, readability, and flexibility. By exploring CTEs, subqueries, aggregation, and PostgreSQL-specific functions, we can optimize queries based on the dataset size and use case.

On the other hand, related to the data used, we can conclude that the higher the donation, the bigger the impact, and that individual donations tend to show a bigger impact.

