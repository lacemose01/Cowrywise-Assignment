SELECT 
    u.id AS owner_id,
    CONCAT(u.first_name, ' ', u.last_name) AS name,
    COUNT(CASE WHEN p.is_regular_savings = 1 THEN 1 END) AS savings_count,
    COUNT(CASE WHEN p.is_a_fund = 1 THEN 1 END) AS investment_count,
    SUM(s.confirmed_amount) / 100.0 AS total_deposits
FROM 
    users_customuser u
LEFT JOIN 
    savings_savingsaccount s ON s.owner_id = u.id
LEFT JOIN 
    plans_plan p ON p.owner_id = u.id
GROUP BY 
    u.id, u.first_name, u.last_name
ORDER BY 
    total_deposits DESC;

