# Models_and_Toys
## SQL queries from a mock database on a models and toys company

This is the result of a 3 week project developed with 3 other colleagues from the Wild Code School.
It was a project designed to improve our SQL skills while providing some visualization of our results.
We were asked to produce a Tableau or Power BI dashboard with a number of mandatory KPI and propose some others, as well as improvement suggestions for the company's database. We decided to use Tableau for our dashboard for compatibility.
The project constraint was to not use Tableau designed filters, but to provide all filters with our SQL queries. Tableau was to be used only for the visual aspect of our work.

Within this project, I was designed Product Owner in addition to the development of my queries. This meant I had an overarching view of all queries performed, I chose what queries were to be included in the dashboard, and I also supervised the correct coding of all queries. Sometimes this meant I had to correct my colleague's code.


## Presentation of the company:

### Turnover per year

~~~~sql
SELECT YEAR(pay.paymentDate) AS year, SUM(pay.amount) AS year_turnover
            FROM payments AS pay
            GROUP BY year;
~~~~

<p align="center">
<img src="https://github.com/pablomc87/Models_and_Toys/blob/main/turnover_year.png" />
</p>

### Where are our clients from?

~~~~sql
SELECT country FROM customers
GROUP BY country;
~~~~

<p align="center">
<img src="https://github.com/pablomc87/Models_and_Toys/blob/33b359e11b13fb73eab08bb42ead2bdeeeddb19a/customers_countries.png" />
</p>

### Where do our employees work?

~~~~sql
SELECT count(*) NbrEmployees, city
FROM employees em
INNER JOIN offices ofc ON em.officeCode= ofc.officeCode
GROUP BY city;
~~~~

<p align="center">
<img src="https://github.com/pablomc87/Models_and_Toys/blob/main/employees_offices.png" />
</p>

### Turnover per agency each year

~~~~sql
SELECT offi.city, YEAR(o.orderDate), y1.turnover
FROM offices offi
CROSS JOIN orders o

LEFT JOIN (SELECT offi.city, YEAR(pay.paymentDate) AS year1, SUM(pay.amount) AS turnover
			FROM offices AS offi
			INNER JOIN employees AS emp ON emp.officeCode = offi.officeCode
			INNER JOIN customers AS cust ON cust.salesRepEmployeeNumber = emp.employeeNumber 
			INNER JOIN payments AS pay ON pay.customerNumber = cust.customerNumber
			GROUP BY offi.city, year1
			ORDER BY year1, SUM(pay.amount) DESC) y1 ON y1.year1 = YEAR(o.orderDate) AND offi.city = y1.city

GROUP BY offi.city, YEAR(o.orderDate);
~~~~

<p align="center">
<img src="https://github.com/pablomc87/Models_and_Toys/blob/main/turnover_agency_year.png" />
</p>


## Finance

### Turnover per clients' country (last 2 months)

~~~~sql
SELECT cu.country, SUM(quantityOrdered*priceEach) quantity_price
FROM customers cu

LEFT JOIN orders odr ON cu.customerNumber = odr.customerNumber
LEFT join orderdetails ods ON ods.orderNumber=odr.orderNumber

WHERE odr.orderDate > DATE_SUB(NOW(), INTERVAL 2 MONTH)
GROUP BY cu.country
ORDER BY cu.country ASC
~~~~

<p align="center">
<img src="https://github.com/pablomc87/Models_and_Toys/blob/main/turnover_customers_countries.png" />
</p>

### Comparing stock value with turnover in 2021

~~~~sql
SELECT SUM((p.buyPrice)*(p.quantityInStock)) valeur_du_stock
FROM products AS p
UNION
SELECT SUM(p.amount) chiffre_daffaires_2021
FROM payments p
WHERE YEAR(p.paymentDate) = "2021";
~~~~

<p align="center">
<img src="https://github.com/pablomc87/Models_and_Toys/blob/main/stock_vs_turnover_last_year.png" />
</p>

### Unpayed orders per client (total amount, in $)

~~~~sql
SELECT c.customerName, c.contactFirstName, c.contactLastName, c.phone, result.difference, 
    ROUND(result.difference/c.creditLimit*100,2) as pct_credit_used 

FROM (SELECT t1.eachCustomer, t1.payed, t2.toPay, t2.toPay-t1.payed difference 
        FROM (SELECT customerNumber, o.orderNumber, sum(quantityOrdered*priceEach) toPay
                FROM orderdetails od
                LEFT JOIN orders o ON od.orderNumber = o.orderNumber
                GROUP BY customerNumber) t2, 
             (SELECT temp.eachCustomer, sum(amount) payed 
                FROM (SELECT DISTINCT o.customerNumber eachCustomer, amount
                        FROM payments p
                        LEFT JOIN orders o ON o.customerNumber = p.customerNumber) temp
                GROUP BY temp.eachCustomer) t1

        WHERE t1.eachCustomer = t2.customerNumber AND t2.toPay-t1.payed != 0) result

LEFT JOIN customers c ON c.customerNumber = result.eachCustomer

ORDER BY result.difference DESC;
~~~~

<p align="center">
<img src="https://github.com/pablomc87/Models_and_Toys/blob/main/unpayed_orders_per_client_and_credit_limit.png" />
</p>

## Logistics

### What could our stock problems be in the next months?

~~~~sql
SELECT f.productName,f.quantityInStock, round(temp.average,0) MonthlyOrders, f.quantityInStock-round(temp.average,0)*3 delta 

FROM    (SELECT p.productName , p.quantityInStock 
            FROM products p
            LEFT JOIN orderdetails ods ON p.productCode = ods.productCode
            GROUP BY p.productName
            ORDER BY sum(ods.quantityOrdered) DESC) f
LEFT JOIN (SELECT e.productName, avg(d) average 
            FROM(SELECT orderDate, ods.productCode, sum(quantityOrdered) d , p.productName
                    FROM orders od
                    INNER JOIN orderdetails ods ON ods.orderNumber = od.orderNumber
                    INNER JOIN products p ON p.productCode = ods.productCode
                    GROUP BY year(orderDate),month(orderDate),productCode
                    ORDER BY   productName, orderDate) e
            GROUP BY e.productName) temp ON temp.productName = f.productName

WHERE quantityInStock < round(temp.moyenne,0)*3
ORDER BY dif ASC;
~~~~

<p align="center">
<img src="https://github.com/pablomc87/Models_and_Toys/blob/main/possible_out-of-stock_problems.png" />
</p>

### Our current stock for our top5

~~~~sql
SELECT p.productName , p.quantityInStock , sum(ods.quantityOrdered)
FROM products p
LEFT JOIN orderdetails ods ON p.productCode = ods.productCode
GROUP BY p.productName
ORDER BY sum(ods.quantityOrdered) DESC
LIMIT 5;
~~~~

<p align="center">
<img src="https://github.com/pablomc87/Models_and_Toys/blob/main/stock_top_5.png" />
</p>


## Sales

### Average margin of 20 products most sold in the last 3 months

~~~~sql
SELECT productName, SUM(quantityOrdered), ROUND(SUM(priceEach * quantityOrdered)/SUM(quantityOrdered),2)-buyPrice avgMargin
FROM products p

LEFT JOIN orderdetails od ON p.productCode = od.productCode
LEFT JOIN orders o ON o.orderNumber = od.orderNumber

WHERE o.orderDate > DATE_SUB(CURDATE(), INTERVAL 3 MONTH)

GROUP BY p.productCode

ORDER BY SUM(quantityOrdered) DESC

LIMIT 20
~~~~

<p align="center">
<img src="https://github.com/pablomc87/Models_and_Toys/blob/33b359e11b13fb73eab08bb42ead2bdeeeddb19a/avg_margin_bestsellers.png" />
</p>

### Difference in sales per type of product, compared to same month previous year

~~~~sql
SELECT pl.productLine, COALESCE(temp1.sold,0) sold1, COALESCE(temp2.sold,0) sold2, 
        DATE_FORMAT(DATE(TIMESTAMP(CONCAT(YEAR(o.orderDate)+1,'-', MONTH(o.orderDate),'-01'))), '%Y-%m') as LatestPeriod, 
        ROUND((COALESCE(temp2.sold,0)-temp1.sold)/temp1.sold*100, 2) change_rate
FROM productlines pl

CROSS JOIN orders o

LEFT JOIN (SELECT p.productLine, SUM(od.quantityOrdered) as sold, DATE_FORMAT(DATE (o.orderDate), '%Y-%m') as orderPeriod, 
                  MONTH(DATE(o.orderDate)) month_ref, YEAR(DATE(o.orderDate)) year_ref
		    FROM orderdetails od
		    LEFT JOIN products p ON p.productCode = od.productCode
		    LEFT JOIN orders o ON o.orderNumber = od.orderNumber
            GROUP BY p.productLine, month_ref, year_ref
            ORDER BY orderPeriod) temp1
ON temp1.month_ref = MONTH(o.orderDate) AND temp1.year_ref = YEAR(o.orderDate) AND temp1.productLine = pl.productLine

LEFT JOIN (SELECT p.productLine, SUM(od.quantityOrdered) as sold, DATE_FORMAT(DATE (o.orderDate), '%Y-%m') as orderPeriod, 
                  MONTH(DATE(o.orderDate)) month_ref, YEAR(DATE(o.orderDate)) year_ref
		    FROM orderdetails od
		    LEFT JOIN products p ON p.productCode = od.productCode
		    LEFT JOIN orders o ON o.orderNumber = od.orderNumber
            GROUP BY p.productLine, month_ref, year_ref
            ORDER BY orderPeriod) temp2
ON temp2.month_ref = MONTH(o.orderDate) AND temp2.year_ref = YEAR(o.orderDate) + 1 AND temp2.productLine = pl.productLine

WHERE DATEDIFF(DATE(o.orderDate), DATE_FORMAT(DATE(CURRENT_TIMESTAMP),'%Y-%m-%d')) <= -365

GROUP BY YEAR(o.orderDate), MONTH(o.orderDate), pl.productLine

ORDER BY change_rate ASC;
~~~~

<p align="center">
<img src="https://github.com/pablomc87/Models_and_Toys/blob/main/year_variation_category.png" />
</p>

### In our bestsellers, where could we have earned more last 3 months?

~~~~sql
SELECT temp.productName, temp.soldProducts, temp.pctMarginLost, temp.totalLost 

FROM (SELECT productName, SUM(quantityOrdered) soldProducts, 
                ROUND(SUM(priceEach * quantityOrdered)/SUM(quantityOrdered),2) avgSellPrice, 
                ROUND(SUM(priceEach * quantityOrdered)/SUM(quantityOrdered),2)-buyPrice avgMargin, 
                MSRP-buyPrice possibleMargin, (MSRP - ROUND(SUM(priceEach * quantityOrdered)/SUM(quantityOrdered),2)) lostMargin, 
                ROUND((MSRP - SUM(priceEach * quantityOrdered)/SUM(quantityOrdered))/(MSRP-buyPrice)*100, 2) pctMarginLost, 
                ROUND((MSRP - SUM(priceEach * quantityOrdered)/SUM(quantityOrdered))*SUM(quantityOrdered),2) totalLost 
        FROM products p

        LEFT JOIN orderdetails od ON p.productCode = od.productCode
        LEFT JOIN orders o ON o.orderNumber=od.orderNumber
        
        WHERE o.orderDate > DATE_SUB(CURDATE(), INTERVAL 3 MONTH)
        GROUP BY p.productCode
        ORDER BY SUM(quantityOrdered) DESC
        LIMIT 10) temp
        
ORDER BY temp.pctMarginLost DESC
~~~~

<p align="center">
<img src="https://github.com/pablomc87/Models_and_Toys/blob/main/margin_loss_bestsellers.png" />
</p>


## Human Resources

### What margin per agency in the last quarter?

~~~~sql
SELECT offi.city, YEAR(o.orderDate), QUARTER(o.orderDate), temp.margin
FROM offices offi
CROSS JOIN orders o

LEFT JOIN (SELECT offi2.officeCode, ROUND(AVG((((od2.priceEach - p2.buyPrice) / (p2.buyPrice))*100))) margin, 
                QUARTER(o2.orderDate) quarter, YEAR(o2.orderDate) ref_year  
                FROM orderdetails od2
                LEFT JOIN orders o2 ON od2.orderNumber = o2.orderNumber
                LEFT JOIN customers c2 ON c2.customerNumber = o2.customerNumber
                LEFT JOIN employees e2 ON e2.employeeNumber=c2.salesRepEmployeeNumber
                LEFT JOIN offices offi2 ON offi2.officeCode=e2.officeCode
                LEFT JOIN products p2 ON p2.productCode = od2.productCode
                GROUP BY  offi2.officeCode, QUARTER(o2.orderDate), YEAR(o2.orderDate)) temp 
                
    ON temp.officeCode = offi.officeCode AND temp.trimestre = QUARTER(o.orderDate) AND YEAR(o.orderDate) = temp.annee

WHERE QUARTER (o.orderDate) = QUARTER(CURDATE())
GROUP BY offi.city, QUARTER(o.orderDate), YEAR(o.orderDate)
ORDER BY offi.officeCode ASC, YEAR(o.orderDate) DESC, QUARTER(o.orderDate) DESC;
~~~~

<p align="center">
<img src="https://github.com/pablomc87/Models_and_Toys/blob/main/margin_per_agency_last_quarter.png" />
</p>

### Two best sales representatives last 6 months

~~~~sql
WITH f AS (SELECT SUM(p.amount) turnover_employee, e.firstName employee_firstName, e.lastName employee_lastName, 
                o.city employee_office, DATE_FORMAT(DATE (p.paymentDate), '%Y-%M') orderPeriod1,
	              YEAR(p.paymentDate) year_payment, MONTH(p.paymentDate) month_payment,
	              ROW_NUMBER() OVER(PARTITION BY year_payment, month_payment 
                  ORDER BY turnover_employee DESC) rang
                  FROM payments p

                INNER JOIN customers c ON p.customerNumber=c.customerNumber
                INNER JOIN employees e ON e.employeeNumber=c.salesRepEmployeeNumber
                INNER JOIN offices o ON o.officeCode=e.officeCode

                WHERE p.paymentDate >= DATE_SUB(CURDATE(), INTERVAL 29 WEEK)
                GROUP BY year_payment, month_payment, e.employeeNumber, orderPeriod1 ASC
                ORDER BY orderPeriod1
                )
SELECT * FROM f 
WHERE (rang = 1 OR rang=2) 
ORDER BY year_payment, month_payment, rang ASC;
~~~~

<p align="center">
<img src="https://github.com/pablomc87/Models_and_Toys/blob/main/two_top_sellers_month.png" />
</p>

### Turnover per team each quarter

~~~~sql
SELECT e.reportsTo, m.lastName as manager_lastName, o.city as manager_city, m.firstName as manager_firstName, m.jobTitle,
	   SUM(p.amount) turnover_team, DATE_FORMAT(DATE (p.paymentDate), '%Y-%M') orderPeriod,
       DATE_FORMAT(DATE (p.paymentDate), '%Y') year_payment, QUARTER(p.paymentDate) quarter_payment
FROM employees m

LEFT JOIN employees e ON m.employeeNumber=e.reportsTo
LEFT JOIN customers c ON e.employeeNumber=c.salesRepEmployeeNumber
LEFT JOIN payments p ON p.customerNumber=c.customerNumber
LEFT JOIN offices o ON o.officeCode=m.officeCode

WHERE e.reportsTo IS NOT NULL AND p.amount IS NOT NULL AND QUARTER(p.paymentDate)= QUARTER(CURDATE())

GROUP BY year_payment, QUARTER(p.paymentDate),e.reportsTo
~~~~

<p align="center">
<img src="https://github.com/pablomc87/Models_and_Toys/blob/main/turnover_team_quarter.png" />
</p>





