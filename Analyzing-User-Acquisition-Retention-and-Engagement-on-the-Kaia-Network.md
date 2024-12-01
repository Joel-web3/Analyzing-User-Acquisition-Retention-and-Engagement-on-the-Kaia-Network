# Analyzing-User-Acquisition-Retention-and-Engagement-on-the-Kaia-Network

The blockchain ecosystem is advancing at a remarkable pace, and success in this space demands more than just innovative technology—it requires a deep understanding of the individuals powering the network. Kaia has emerged as a prominent player, blending state-of-the-art technology with a vibrant and diverse user base. But what sets Kaia’s ecosystem apart? Who are the users behind its growth? How and when do they interact with the platform?

In this article, we’ll explore three key areas that define Kaia’s growth and user behavior:

User Acquisition – How new users are joining the network and what drives their onboarding.
User Retention – Understanding how well Kaia keeps users engaged and active over time.
User Engagement Trends – Analyzing when and how users interact with the platform, including transaction patterns and peak activity times.
By examining these metrics and leveraging the insights from the attached charts, we’ll uncover the trends, strengths, and opportunities shaping Kaia’s ecosystem. 

The data will speak for itself and we'll also break down the methodology used to derive the results and explain how you can leverage **Flipside** analytics to replicate and dig deeper into this kind of analysis.

Let’s dive in!
---

### **Methodology: Understanding the Data**

To gather insights on Kaia’s network, we used the **Flipside** platform to extract and analyze blockchain transaction data. The main dataset we’re working with consists of transaction logs that contain crucial information about user activity on the platform. Below, we explain the key columns from this dataset and how they were used to generate the findings.

#### **Key Columns in the Dataset:**

1. **from_address**: This column identifies the user (or wallet) initiating the transaction. It's essential for determining unique users and calculating user engagement.
2. **tx_hash**: This represents a unique transaction identifier. It helps in counting the total number of transactions per user.
3. **block_timestamp**: The timestamp of when the transaction was mined, which is used for aggregating activity by time (e.g., hour, day, week, month).
4. **dayofweek(block_timestamp)**: This function extracts the day of the week from the block_timestamp. It's useful for understanding weekly activity trends and user engagement patterns across different days.
5. **user_first_tx**: This subquery identifies the first transaction date for each user. This is crucial for calculating new users and tracking retention over time.
6. **DISTINCT**: Used to ensure that each user or transaction is only counted once in aggregations like total users and transaction counts.


### **Key Metrics Analyzed**

#### **1. User Acquisition and Growth**

Understanding how new users are joining the Kaia network is vital for measuring growth. Over the last 90 days, Kaia’s user base expanded significantly, with 6.2 million total users interacting on the platform. During the same period, 2.9 million new users joined the network, signaling strong onboarding efforts and adoption.

<img width="1055" alt="K1" src="https://github.com/user-attachments/assets/55017d13-b3ee-41ce-a8a6-08733abe3143">

Weekly Trends
From the “Weekly Trend of New Users” chart, we observe steady growth, with spikes in new user acquisition around marketing campaigns or platform events. Cumulative new users surpassed 18 million by the end of the period, a testament to Kaia’s appeal and expanding reach.

Key Insight: The platform's ability to sustain growth while onboarding large numbers of users indicates an effective acquisition strategy.

**Example Query for New Users:**

```
WITH user_first_tx AS (
    SELECT 
        from_address,
        MIN(TO_DATE(block_timestamp)) AS first_tx_date
    FROM kaia.core.fact_transactions
    GROUP BY from_address
),
weekly_data AS (
    SELECT 
        DATE_TRUNC('WEEK', TO_DATE(block_timestamp)) AS week_start_date,
        COUNT(DISTINCT t.from_address) AS total_users_week,
        COUNT(DISTINCT CASE 
            WHEN u.first_tx_date >= DATE_TRUNC('WEEK', TO_DATE(t.block_timestamp)) 
            AND u.first_tx_date < DATE_TRUNC('WEEK', TO_DATE(t.block_timestamp)) + INTERVAL '1 WEEK'
            THEN t.from_address 
        END) AS new_users_week
    FROM kaia.core.fact_transactions t
    JOIN user_first_tx u ON t.from_address = u.from_address
    WHERE t.block_timestamp >= CURRENT_DATE - INTERVAL '365 days'  -- To fetch data for the past year
    GROUP BY DATE_TRUNC('WEEK', TO_DATE(block_timestamp))
)
SELECT 
    week_start_date,
    total_users_week,
    SUM(total_users_week) OVER (ORDER BY week_start_date) AS cumulative_total_users,
    new_users_week,
    SUM(new_users_week) OVER (ORDER BY week_start_date) AS cumulative_new_users
FROM weekly_data
ORDER BY week_start_date;
```

#### **2. Retention: Assessing Kaia Users Over Time**

Retention is where the long-term health of a platform is truly tested. From the User Cohort Retention Analysis, we see varying retention rates across months. For instance, in May 2024, 30% of users returned after one month—a significant improvement compared to earlier months like January, where only 9% of users returned.

<img width="1057" alt="k6" src="https://github.com/user-attachments/assets/0487c831-3eb2-4f54-b826-c6d4b0123eb9">

Retention rates tend to drop after the first month, but we observe remarkable 39% retention rates at the six-month mark for users acquired in June 2024. This indicates targeted efforts to engage users over time, especially during June’s campaigns or updates.

Key Insight: Kaia’s retention strategies are improving, but there’s room for optimization in retaining users long-term across all cohorts.

**Example Query for Cohort Retention Analysis:**

```
WITH transfer_data AS (
    SELECT DISTINCT
        block_timestamp,
        tx_hash,
        to_address,
        from_address
    FROM kaia.core.fact_transactions
),
user_activity AS (
    SELECT
        from_address AS user_address,
        DATE_TRUNC('month', block_timestamp) AS activity_month,
        MIN(DATE_TRUNC('month', block_timestamp)) OVER (PARTITION BY from_address) AS first_activity_month,
        DATEDIFF(
            'month',
            MIN(DATE_TRUNC('month', block_timestamp)) OVER (PARTITION BY from_address),
            DATE_TRUNC('month', block_timestamp)
        ) AS month_difference
    FROM transfer_data
    WHERE block_timestamp >= '2024-01-01'
),
new_users AS (
    SELECT
        first_activity_month,
        COUNT(DISTINCT user_address) AS new_user_count
    FROM user_activity
    GROUP BY 1
),
returning_users AS (
    SELECT
        first_activity_month,
        month_difference,
        COUNT(DISTINCT user_address) AS returning_user_count
    FROM user_activity
    WHERE month_difference <> 0
    GROUP BY 1, 2
),
retention_data AS (
    SELECT
        new_users.first_activity_month,
        returning_users.month_difference,
        new_users.new_user_count,
        returning_users.returning_user_count,
        ROUND(returning_users.returning_user_count::NUMERIC / new_users.new_user_count, 2) AS retention_percentage
    FROM new_users
    LEFT JOIN returning_users
    ON new_users.first_activity_month = returning_users.first_activity_month
),
retention_pivot AS (
    SELECT 
        first_activity_month,
        new_user_count,
        MAX(CASE WHEN month_difference = 1 THEN retention_percentage ELSE NULL END) AS retention_one_month,
        MAX(CASE WHEN month_difference = 2 THEN retention_percentage ELSE NULL END) AS retention_two_months,
        MAX(CASE WHEN month_difference = 3 THEN retention_percentage ELSE NULL END) AS retention_three_months,
        MAX(CASE WHEN month_difference = 4 THEN retention_percentage ELSE NULL END) AS retention_four_months,
        MAX(CASE WHEN month_difference = 5 THEN retention_percentage ELSE NULL END) AS retention_five_months,
        MAX(CASE WHEN month_difference = 6 THEN retention_percentage ELSE NULL END) AS retention_six_months
    FROM retention_data
    GROUP BY first_activity_month, new_user_count
),
final_aggregate AS (
    SELECT 
        TO_VARCHAR(first_activity_month, 'yyyy-MM') AS start_month,
        TO_CHAR(new_user_count, '999,999,999,999') AS new_users,
        CONCAT(MAX(retention_one_month) * 100, '%') AS one_month_later,
        CONCAT(MAX(retention_two_months) * 100, '%') AS two_months_later,
        CONCAT(MAX(retention_three_months) * 100, '%') AS three_months_later,
        CONCAT(MAX(retention_four_months) * 100, '%') AS four_months_later,
        CONCAT(MAX(retention_five_months) * 100, '%') AS five_months_later,
        CONCAT(MAX(retention_six_months) * 100, '%') AS six_months_later
    FROM retention_pivot 
    GROUP BY start_month, new_users
)
SELECT * FROM final_aggregate
ORDER BY start_month;
```

#### **3. User Engagement: How and When Do Users Interact?**

Time of Day and Day of Week
From the “User Activity by Time of Day” chart, Kaia experiences the highest activity during Fridays and Saturdays, particularly between 12 PM and 6 PM UTC. This aligns with common patterns in blockchain ecosystems, where weekends provide users more time to participate in activities like staking, trading, or governance.

<img width="1055" alt="k5" src="https://github.com/user-attachments/assets/dc8621db-78b6-44fc-bf30-6505b496210c">

Interestingly, Monday activity is also significant, reflecting users catching up on platform events or interactions after the weekend.

<img width="528" alt="k3" src="https://github.com/user-attachments/assets/daf4d8c9-883d-49a2-a3e6-856e45eb091b">

Key Insight: Timing events, promotions, or governance proposals around peak activity windows can maximize engagement and participation.

**Example Query for User Activity by Time of Day:**

```
WITH user_activity AS (
    SELECT 
        DATE_PART('hour', block_timestamp) AS activity_hour,
        CASE 
            WHEN DAYOFWEEK(block_timestamp) = 0 THEN '7 - Sunday'
            WHEN DAYOFWEEK(block_timestamp) = 1 THEN '1 - Monday'
            WHEN DAYOFWEEK(block_timestamp) = 2 THEN '2 - Tuesday'
            WHEN DAYOFWEEK(block_timestamp) = 3 THEN '3 - Wednesday'
            WHEN DAYOFWEEK(block_timestamp) = 4 THEN '4 - Thursday'
            WHEN DAYOFWEEK(block_timestamp) = 5 THEN '5 - Friday'
            WHEN DAYOFWEEK(block_timestamp) = 6 THEN '6 - Saturday'
        END AS activity_day,
        COUNT(tx_hash) AS total_transactions,
        COUNT(DISTINCT from_address) AS unique_users
    FROM kaia.core.fact_transactions
    WHERE block_timestamp >= CURRENT_DATE - INTERVAL '30 days'  -- Past 30 days
    GROUP BY 
        DATE_PART('hour', block_timestamp), 
        CASE 
            WHEN DAYOFWEEK(block_timestamp) = 0 THEN '7 - Sunday'
            WHEN DAYOFWEEK(block_timestamp) = 1 THEN '1 - Monday'
            WHEN DAYOFWEEK(block_timestamp) = 2 THEN '2 - Tuesday'
            WHEN DAYOFWEEK(block_timestamp) = 3 THEN '3 - Wednesday'
            WHEN DAYOFWEEK(block_timestamp) = 4 THEN '4 - Thursday'
            WHEN DAYOFWEEK(block_timestamp) = 5 THEN '5 - Friday'
            WHEN DAYOFWEEK(block_timestamp) = 6 THEN '6 - Saturday'
        END
)
SELECT 
    activity_day,
    activity_hour,
    total_transactions,
    unique_users
FROM user_activity
ORDER BY 
    activity_day, 
    activity_hour;
```

#### **4. User Distribution by Transaction Count**

The “User Distribution by Transaction Count” chart shows that most users are low-frequency participants, with a large proportion completing 1-5 transactions. However, high-frequency traders (users completing over 1,000 transactions) are critical to Kaia’s ecosystem, contributing a significant portion of transaction volume.

<img width="524" alt="k4" src="https://github.com/user-attachments/assets/29700168-d071-4f58-bd20-3bb8506571c0">

This dual-user dynamic highlights the importance of retaining retail users while continuing to support and incentivize power users or "whales."

Key Insight: Diversified strategies are needed to cater to both retail users and high-frequency traders, as they represent different but complementary aspects of the network.

**Example Query for User Distribution by Transaction Count:**

```
WITH user_transaction_counts AS (
    SELECT 
        from_address,
        COUNT(tx_hash) AS transaction_count
    FROM kaia.core.fact_transactions
    GROUP BY from_address
),
user_distribution AS (
    SELECT 
        CASE 
            WHEN transaction_count = 1 THEN '1 Transaction'
            WHEN transaction_count BETWEEN 2 AND 5 THEN '2-5 Transactions'
            WHEN transaction_count BETWEEN 6 AND 10 THEN '6-10 Transactions'
            WHEN transaction_count BETWEEN 11 AND 50 THEN '11-50 Transactions'
            WHEN transaction_count BETWEEN 51 AND 100 THEN '51-100 Transactions'
            WHEN transaction_count BETWEEN 101 AND 500 THEN '101-500 Transactions'
            WHEN transaction_count BETWEEN 501 AND 1000 THEN '501-1,000 Transactions'
            WHEN transaction_count BETWEEN 1001 AND 10000 THEN '1,001-10,000 Transactions'
            WHEN transaction_count > 10000 THEN '>10,000 Transactions'
        END AS transaction_range,
        COUNT(from_address) AS user_count
    FROM user_transaction_counts
    GROUP BY 
        CASE 
            WHEN transaction_count = 1 THEN '1 Transaction'
            WHEN transaction_count BETWEEN 2 AND 5 THEN '2-5 Transactions'
            WHEN transaction_count BETWEEN 6 AND 10 THEN '6-10 Transactions'
            WHEN transaction_count BETWEEN 11 AND 50 THEN '11-50 Transactions'
            WHEN transaction_count BETWEEN 51 AND 100 THEN '51-100 Transactions'
            WHEN transaction_count BETWEEN 101 AND 500 THEN '101-500 Transactions'
            WHEN transaction_count BETWEEN 501 AND 1000 THEN '501-1,000 Transactions'
            WHEN transaction_count BETWEEN 1001 AND 10000 THEN '1,001-10,000 Transactions'
            WHEN transaction_count > 10000 THEN '>10,000 Transactions'
        END
)
SELECT 
    transaction_range,
    user_count
FROM user_distribution
ORDER BY 
    CASE 
        WHEN transaction_range = '1 Transaction' THEN 1
        WHEN transaction_range = '2-5 Transactions' THEN 2
        WHEN transaction_range = '6-10 Transactions' THEN 3
        WHEN transaction_range = '11-50 Transactions' THEN 4
        WHEN transaction_range = '51-100 Transactions' THEN 5
        WHEN transaction_range = '101-500 Transactions' THEN 6
        WHEN transaction_range = '501-1,000 Transactions' THEN 7
        WHEN transaction_range = '1,001-10,000 Transactions' THEN 8
        WHEN transaction_range = '>10,000 Transactions' THEN 9
    END;
```

### **Cumulative Growth: The Bigger Picture**

The Cumulative Total Users and Total New Users by Week charts reveal Kaia’s overall momentum. The network is not just onboarding users at a fast rate—it’s doing so consistently, with cumulative totals climbing steadily over time. This sustained growth positions Kaia as a leading player in the blockchain space.

<img width="1058" alt="k2" src="https://github.com/user-attachments/assets/d8dbca17-c258-494e-9839-61243acd2189">

Meanwhile, average transactions per user hover around 6.8, showing active engagement among the user base and reflecting consistent utility across the platform’s offerings.

Key Insight: Kaia’s ability to scale while maintaining strong user engagement showcases the robustness of its infrastructure and community.

## **6. Strategic Takeaways and Opportunities**

### **Strengths**

* **Explosive user acquisition**: Over **2.9 million new users** in 90 days highlights Kaia’s successful outreach and onboarding strategies.
* **Strong engagement patterns**: High activity during weekends and peak hours provides clear opportunities for event timing.
* **Retention improvements**: The jump from **9% retention in January to 30% in May** showcases the platform’s ability to adapt and enhance user retention efforts.


### **Challenges**

* **Retention consistency**: While recent months show improvement, earlier cohorts demonstrate the need for sustained engagement across all timeframes.
* **Balancing user types**: Strategies are needed to support both low-frequency users and high-value participants, ensuring inclusivity while driving high transaction volumes.

---

## **Conclusion: Kaia's Path Forward**

Kaia’s impressive growth and engagement metrics demonstrate its potential as a dominant force in the blockchain ecosystem. By leveraging user insights—like peak activity times, transaction distributions, and retention trends—Kaia can refine its strategies to further strengthen its user base and expand its influence.

The numbers tell a powerful story: **6.2 million total users**, **2.9 million new users**, and a retention strategy that continues to evolve. As Kaia grows, the focus should remain on creating a platform that caters to both casual users and power players, ensuring long-term success in the blockchain landscape.
