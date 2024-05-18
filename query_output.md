**Schema (PostgreSQL v9.4)**

    CREATE SCHEMA dannys_diner;
    SET search_path = dannys_diner;
    
    CREATE TABLE sales (
      "customer_id" VARCHAR(1),
      "order_date" DATE,
      "product_id" INTEGER
    );
    
    INSERT INTO sales
      ("customer_id", "order_date", "product_id")
    VALUES
      ('A', '2021-01-01', '1'),
      ('A', '2021-01-01', '2'),
      ('A', '2021-01-07', '2'),
      ('A', '2021-01-10', '3'),
      ('A', '2021-01-11', '3'),
      ('A', '2021-01-11', '3'),
      ('B', '2021-01-01', '2'),
      ('B', '2021-01-02', '2'),
      ('B', '2021-01-04', '1'),
      ('B', '2021-01-11', '1'),
      ('B', '2021-01-16', '3'),
      ('B', '2021-02-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-07', '3');
     
    
    CREATE TABLE menu (
      "product_id" INTEGER,
      "product_name" VARCHAR(5),
      "price" INTEGER
    );
    
    INSERT INTO menu
      ("product_id", "product_name", "price")
    VALUES
      ('1', 'sushi', '10'),
      ('2', 'curry', '15'),
      ('3', 'ramen', '12');
      
    
    CREATE TABLE members (
      "customer_id" VARCHAR(1),
      "join_date" DATE
    );
    
    INSERT INTO members
      ("customer_id", "join_date")
    VALUES
      ('A', '2021-01-07'),
      ('B', '2021-01-09');

---

**Query #1**

What is the total amount each customer spent at the restaurant?

    select 
    	s.customer_id,
        sum(price) as total_amount
    from
    	dannys_diner.sales s
    left join
    	dannys_diner.menu m
    on
    	s.product_id = m.product_id
    group by
    	s.customer_id
    ;

| customer_id | total_amount |
| ----------- | ------------ |
| B           | 74           |
| C           | 36           |
| A           | 76           |

---
**Query #2**

How many days has each customer visited the restaurant?

    select 
    	s.customer_id,
        count(distinct order_date) as num_days
    from
    	dannys_diner.sales s
    group by
    	s.customer_id
    ;

| customer_id | num_days |
| ----------- | -------- |
| A           | 4        |
| B           | 6        |
| C           | 2        |

---
**Query #3**

What was the first item from the menu purchased by each customer?

    select
    	s.customer_id,
        s.product_id,
        m.product_name
    from
    	dannys_diner.sales s
    inner join
    	dannys_diner.menu m
    on
    	s.product_id = m.product_id
    inner join
    	(
          select 
          	customer_id,
          	min(order_date) as first_date
          from 
          	dannys_diner.sales
          group by
          	customer_id
          ) first_purchase
    on
    	1 = 1
        and s.customer_id = first_purchase.customer_id
        and s.order_date = first_purchase.first_date
    order by
    	s.customer_id, s.product_id
    ;

| customer_id | product_id | product_name |
| ----------- | ---------- | ------------ |
| A           | 1          | sushi        |
| A           | 2          | curry        |
| B           | 2          | curry        |
| C           | 3          | ramen        |
| C           | 3          | ramen        |

---
**Query #4**

What is the most purchased item on the menu and how many times was it purchased by all customers?

    select
    	s.product_id,
        m.product_name,
        count(s.product_id) as purchase_freq
    from
    	dannys_diner.sales s
    inner join
    	dannys_diner.menu m
    on
    	s.product_id = m.product_id
    group by
    	s.product_id, m.product_name
    order by
    	count(s.product_id) desc
    limit 1
    ;

| product_id | product_name | purchase_freq |
| ---------- | ------------ | ------------- |
| 3          | ramen        | 8             |

---
**Query #5**

Which item was the most popular for each customer?

    select
    	p.customer_id,
        p.product_id,
        m.product_name
    from
    	(
          select
    		customer_id,
       		product_id,
        	dense_rank() over (partition by customer_id order by count(product_id) desc) as row_num
    	from
    		dannys_diner.sales
    	group by
    		customer_id, product_id
          ) p
    inner join
    	dannys_diner.menu m
    on
    	p.product_id = m.product_id
    where 
    	p.row_num = 1
    order by 
    	p.customer_id, p.product_id
    ;

| customer_id | product_id | product_name |
| ----------- | ---------- | ------------ |
| A           | 3          | ramen        |
| B           | 1          | sushi        |
| B           | 2          | curry        |
| B           | 3          | ramen        |
| C           | 3          | ramen        |

---
**Query #6**

Which item was purchased first by the customer after they became a member?

    with mem_pur as (
      	select
    		s.*
    	from
    		dannys_diner.sales s
    	inner join
    		dannys_diner.members mem
    	on
    		1 = 1
        	and s.customer_id = mem.customer_id
        	and s.order_date >= mem.join_date
      )
    select 
    	s.*,
        m.product_name
    from
    	mem_pur s
    inner join
    	(
          select
          	customer_id,
            min(order_date) as first_date
          from 
          	mem_pur
          group by
          	customer_id
          ) first_purchase
    on
    	1 = 1
        and s.customer_id = first_purchase.customer_id
        and s.order_date = first_purchase.first_date
    inner join
    	dannys_diner.menu m
    on
    	s.product_id = m.product_id
    order by
    	s.customer_id,
        s.product_id    
    ;

| customer_id | order_date               | product_id | product_name |
| ----------- | ------------------------ | ---------- | ------------ |
| A           | 2021-01-07T00:00:00.000Z | 2          | curry        |
| B           | 2021-01-11T00:00:00.000Z | 1          | sushi        |

---
**Query #7**

Which item was purchased just before the customer became a member?

    with nonmem_pur as (
      	select
    		s.*
    	from
    		dannys_diner.sales s
    	inner join
    		dannys_diner.members mem
    	on
    		1 = 1
        	and s.customer_id = mem.customer_id
        	and s.order_date < mem.join_date
      )
    select 
    	s.*,
        m.product_name
    from
    	nonmem_pur s
    inner join
    	(
          select
          	customer_id,
            max(order_date) as last_date
          from 
          	nonmem_pur
          group by
          	customer_id
          ) last_purchase
    on
    	1 = 1
        and s.customer_id = last_purchase.customer_id
        and s.order_date = last_purchase.last_date
    inner join
    	dannys_diner.menu m
    on
    	s.product_id = m.product_id
    order by
    	s.customer_id,
        s.product_id    
    ;

| customer_id | order_date               | product_id | product_name |
| ----------- | ------------------------ | ---------- | ------------ |
| A           | 2021-01-01T00:00:00.000Z | 1          | sushi        |
| A           | 2021-01-01T00:00:00.000Z | 2          | curry        |
| B           | 2021-01-04T00:00:00.000Z | 1          | sushi        |

---
**Query #8**

What is the total items and amount spent for each member before they became a member?

    with nonmem_pur as (
      	select
    		s.*
    	from
    		dannys_diner.sales s
    	inner join
    		dannys_diner.members mem
    	on
    		1 = 1
        	and s.customer_id = mem.customer_id
        	and s.order_date < mem.join_date
      )
    select
    	s.customer_id,
        count(s.product_id) as total_num,
        sum(m.price) as total_amount
    from 
    	nonmem_pur s
    inner join
    	dannys_diner.menu m
    on
    	s.product_id = m.product_id
    group by
    	s.customer_id
    order by
    	s.customer_id
    ;

| customer_id | total_num | total_amount |
| ----------- | --------- | ------------ |
| A           | 2         | 25           |
| B           | 3         | 40           |

---
**Query #9**

If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

    select
    	customer_id,
        sum(points) as points
    from (
    	select
    		s.customer_id,
        	m.product_name,
        	m.price,
        	case
        		when m.product_name = 'sushi' then 20*m.price
            	else 10*m.price
        	end as points
    	from
    		dannys_diner.sales s
    	inner join
    		dannys_diner.menu m
    	on
    		s.product_id = m.product_id
    ) p
    group by
    	p.customer_id
    order by
    	p.customer_id
    ;

| customer_id | points |
| ----------- | ------ |
| A           | 860    |
| B           | 940    |
| C           | 360    |

---
**Query #10**

In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

    select
    	customer_id,
        sum(points) as points
    from (
    	select
    		s.customer_id,
        	m.product_name,
        	m.price,
        	case
        		when m.product_name = 'sushi' or 
      				 s.mbr_fw_flag = 1
      				then 20*m.price
            	else 10*m.price
        	end as points
    	from
    		(
              select
              	s1.*,
              	case 
              		when (mbr.join_date is not null) and
              			 (s1.order_date between mbr.join_date and mbr.join_date+ INTERVAL '1 day')
              			then 1
             		else 0
              	end as mbr_fw_flag
              from
              	dannys_diner.sales s1
              left join
              	dannys_diner.members mbr
              on
              	s1.customer_id = mbr.customer_id          
             ) s
    	inner join
    		dannys_diner.menu m
    	on
    		s.product_id = m.product_id
    ) p
    group by
    	p.customer_id
    order by
    	p.customer_id
    ;

| customer_id | points |
| ----------- | ------ |
| A           | 1010   |
| B           | 940    |
| C           | 360    |

---
**Query #11**

Join All The Things: The following questions are related creating basic data tables that Danny and his team can use to quickly derive insights without needing to join the underlying tables using SQL.

    select
    	s.customer_id,
        s.order_date,
        m.product_name,
        m.price,
        case 
        	when mbr.join_date is null then 'N'
            else 'Y'
        end as member
    from
    	dannys_diner.sales s
    left join
    	dannys_diner.menu m
    on 
    	s.product_id = m.product_id
    left join
    	dannys_diner.members mbr
    on
    	1 = 1
        and s.customer_id = mbr.customer_id
        and s.order_date >= mbr.join_date
    order by
    	s.customer_id, s.order_date, s.product_id
    ;

| customer_id | order_date               | product_name | price | member |
| ----------- | ------------------------ | ------------ | ----- | ------ |
| A           | 2021-01-01T00:00:00.000Z | sushi        | 10    | N      |
| A           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |
| A           | 2021-01-07T00:00:00.000Z | curry        | 15    | Y      |
| A           | 2021-01-10T00:00:00.000Z | ramen        | 12    | Y      |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      |
| B           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |
| B           | 2021-01-02T00:00:00.000Z | curry        | 15    | N      |
| B           | 2021-01-04T00:00:00.000Z | sushi        | 10    | N      |
| B           | 2021-01-11T00:00:00.000Z | sushi        | 10    | Y      |
| B           | 2021-01-16T00:00:00.000Z | ramen        | 12    | Y      |
| B           | 2021-02-01T00:00:00.000Z | ramen        | 12    | Y      |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12    | N      |
| C           | 2021-01-07T00:00:00.000Z | ramen        | 12    | N      |

---
**Query #12**

Rank All The Things: Danny also requires further information about the ranking of customer products, but he purposely does not need the ranking for non-member purchases so he expects null ranking values for the records when customers are not yet part of the loyalty program.

    with full_data as (
    select
    	s.customer_id,
        s.order_date,
      	s.product_id,
        m.product_name,
        m.price,
        case 
        	when mbr.join_date is null then 'N'
            else 'Y'
        end as member
    from
    	dannys_diner.sales s
    left join
    	dannys_diner.menu m
    on 
    	s.product_id = m.product_id
    left join
    	dannys_diner.members mbr
    on
    	1 = 1
        and s.customer_id = mbr.customer_id
        and s.order_date >= mbr.join_date
    order by
    	s.customer_id, s.order_date, s.product_id
    )
    
    select
    	fd.*,
        r1.ranking
    from 
    	full_data fd
    left join
    	(
        select
          	customer_id,
          	order_date,
          	product_id,
          	dense_rank() over (partition by customer_id order by order_date, product_id) as ranking
    	from 
    		full_data
    	where
    		member = 'Y'
          ) r1
    on
    	fd.customer_id = r1.customer_id
        and fd.order_date = r1.order_date
        
    ;

| customer_id | order_date               | product_id | product_name | price | member | ranking |
| ----------- | ------------------------ | ---------- | ------------ | ----- | ------ | ------- |
| A           | 2021-01-01T00:00:00.000Z | 1          | sushi        | 10    | N      |         |
| A           | 2021-01-01T00:00:00.000Z | 2          | curry        | 15    | N      |         |
| A           | 2021-01-07T00:00:00.000Z | 2          | curry        | 15    | Y      | 1       |
| A           | 2021-01-10T00:00:00.000Z | 3          | ramen        | 12    | Y      | 2       |
| A           | 2021-01-11T00:00:00.000Z | 3          | ramen        | 12    | Y      | 3       |
| A           | 2021-01-11T00:00:00.000Z | 3          | ramen        | 12    | Y      | 3       |
| A           | 2021-01-11T00:00:00.000Z | 3          | ramen        | 12    | Y      | 3       |
| A           | 2021-01-11T00:00:00.000Z | 3          | ramen        | 12    | Y      | 3       |
| B           | 2021-01-01T00:00:00.000Z | 2          | curry        | 15    | N      |         |
| B           | 2021-01-02T00:00:00.000Z | 2          | curry        | 15    | N      |         |
| B           | 2021-01-04T00:00:00.000Z | 1          | sushi        | 10    | N      |         |
| B           | 2021-01-11T00:00:00.000Z | 1          | sushi        | 10    | Y      | 1       |
| B           | 2021-01-16T00:00:00.000Z | 3          | ramen        | 12    | Y      | 2       |
| B           | 2021-02-01T00:00:00.000Z | 3          | ramen        | 12    | Y      | 3       |
| C           | 2021-01-01T00:00:00.000Z | 3          | ramen        | 12    | N      |         |
| C           | 2021-01-01T00:00:00.000Z | 3          | ramen        | 12    | N      |         |
| C           | 2021-01-07T00:00:00.000Z | 3          | ramen        | 12    | N      |         |

---

[View on DB Fiddle](https://www.db-fiddle.com/f/2rM8RAnq7h5LLDTzZiRWcd/6965)
