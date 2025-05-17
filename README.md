DataAnalytics-Assessment/
│
├── Assessment_Q1.sql

SELECT 
    u.id AS owner_id,
    CONCAT(u.first_name, ' ', u.last_name) AS name,
    COUNT(DISTINCT s.id) AS savings_count,
    COUNT(DISTINCT p.id) AS investment_count,
    COALESCE(SUM(s.confirmed_amount), 0) / 100 AS total_deposits
FROM 
    users_customuser u
JOIN 
    savings_savingsaccount s ON u.id = s.owner_id
JOIN 
    plans_plan p ON u.id = p.owner_id AND p.is_a_fund = 1
WHERE 
    s.confirmed_amount > 0  -- Funded savings account
GROUP BY 
    u.id, u.first_name, u.last_name
HAVING 
    COUNT(DISTINCT s.id) >= 1 AND COUNT(DISTINCT p.id) >= 1
ORDER BY 
    total_deposits DESC;

├── Assessment_Q2.sql

WITH monthly_counts AS (
    SELECT 
        owner_id,
        COUNT(*) AS transactions,
        YEAR(transaction_date) AS year,
        MONTH(transaction_date) AS month
    FROM 
        savings_savingsaccount
    WHERE 
        confirmed_amount > 0  
    GROUP BY 
        owner_id, YEAR(transaction_date), MONTH(transaction_date)
),
customer_avg AS (
    SELECT 
        owner_id,
        AVG(transactions) AS avg_monthly_transactions
    FROM 
        monthly_counts
    GROUP BY 
        owner_id
)
SELECT 
    CASE 
        WHEN avg_monthly_transactions >= 10 THEN 'High Frequency'
        WHEN avg_monthly_transactions >= 3 THEN 'Medium Frequency'
        ELSE 'Low Frequency'
    END AS frequency_category,
    COUNT(owner_id) AS customer_count,
    ROUND(AVG(avg_monthly_transactions), 1) AS avg_transactions_per_month
FROM 
    customer_avg
GROUP BY 
    frequency_category
ORDER BY 
    CASE 
        WHEN frequency_category = 'High Frequency' THEN 1
        WHEN frequency_category = 'Medium Frequency' THEN 2
        ELSE 3
    END;

├── Assessment_Q3.sql

SELECT 
    s.id AS plan_id,
    s.owner_id,
    'Savings' AS type,
    MAX(s.created_on) AS last_transaction_date,
    DATEDIFF(CURDATE(), MAX(s.created_on)) AS inactivity_days
FROM 
    savings_savingsaccount s
WHERE 
    s.created_on IS NOT NULL
GROUP BY 
    s.id, s.owner_id
HAVING 
    MAX(s.created_on) < CURDATE() - INTERVAL 365 DAY

UNION ALL

SELECT 
    p.id AS plan_id,
    p.owner_id,
    'Investment' AS type,
    MAX(s.created_on) AS last_transaction_date,
    DATEDIFF(CURDATE(), MAX(s.created_on)) AS inactivity_days
FROM 
    plans_plan p
LEFT JOIN 
    savings_savingsaccount s ON p.owner_id = s.owner_id
WHERE 
    p.is_a_fund = 1
    AND p.is_archived = 0 
    AND p.is_deleted = 0
GROUP BY 
    p.id, p.owner_id
HAVING 
    MAX(s.created_on) IS NULL OR MAX(s.created_on) < CURDATE() - INTERVAL 365 DAY;


├── Assessment_Q4.sql

WITH transaction_stats AS (
    SELECT 
        owner_id AS customer_id,
        COUNT(id) AS total_transactions,
        SUM(confirmed_amount) / 100 AS total_amount
    FROM 
        savings_savingsaccount
    WHERE 
        confirmed_amount > 0
    GROUP BY 
        owner_id
)
SELECT 
    u.id AS customer_id,
    CONCAT(u.first_name, ' ', u.last_name) AS name,
    TIMESTAMPDIFF(MONTH, u.date_joined, CURRENT_DATE()) AS tenure_months,
    COALESCE(ts.total_transactions, 0) AS total_transactions,
    ROUND(
        CASE 
            WHEN TIMESTAMPDIFF(MONTH, u.date_joined, CURRENT_DATE()) > 0
            THEN (COALESCE(ts.total_transactions, 0) / 
                 TIMESTAMPDIFF(MONTH, u.date_joined, CURRENT_DATE())) * 12 * 
                 (0.001 * COALESCE(ts.total_amount, 0) / 
                 NULLIF(COALESCE(ts.total_transactions, 1), 0))
            ELSE 0
        END, 2
    ) AS estimated_clv
FROM 
    users_customuser u
LEFT JOIN 
    transaction_stats ts ON u.id = ts.customer_id
WHERE 
    TIMESTAMPDIFF(MONTH, u.date_joined, CURRENT_DATE()) > 0
ORDER BY 
    estimated_clv DESC;
│
└── README.md
