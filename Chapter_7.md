
Acrroding to most books and web sties, they state that MySQL is known for not providing a window funciton.However, the version of 8
does support some including ROW_NUMBER,RANK(),DESE_RANK() and so on. 



## CHAPTER 7 MANUPULATION OF DATA TABLE 
### 1) GROUPING 

GROUP BY function is very often used for grouping rows that have the same values into summary rows and accompanied 
with aggregate function(MAX,MIN,AVG etc) to group the result set by one or more columns. 
  

method 1) preparing the 'raw'data
```MySQL
DROP TABLE IF EXISTS review;
CREATE TABLE review (
    user_id    varchar(255)
  , product_id varchar(255)
  , score      numeric
);

INSERT INTO review
VALUES
    ('U001', 'A001', 4.0)
  , ('U001', 'A002', 5.0)
  , ('U001', 'A003', 5.0)
  , ('U002', 'A001', 3.0)
  , ('U002', 'A002', 3.0)
  , ('U002', 'A003', 4.0)
  , ('U003', 'A001', 5.0)
  , ('U003', 'A002', 4.0)
  , ('U003', 'A003', 4.0)
;
```
method 2) computing 
```MySQL
SELECT user_id,
	   COUNT(*) AS total_count,
	   COUNT(DISTINCT product_id) AS product_count,
	   SUM(score) as sum,
       ROUND(AVG(score),2) as avg,
       MAX(score) as max,
       MIN(score) as min 
	   FROM review
       GROUP  BY user_id;
```
### 2) THE Syntax of OVER CLUASE 
#### * OVER() VS OVER(PARTITION BY)

In order to present data before and after aggreate functions  are implemented all together, OVER is the most suitable function. 
we could take.Unless OVER funcion is used with specifying any options,the aggreating function will be applied to the whole table(see the column named avg_score in our code where we are actully averaging all the scores on the table). However,  PARTITION BY caluse in the parenthesis of Over will determine which rows will be applied to the given functions. Let's look at how these concepts are actually 
implemented in our case stduy. 


```MySQL
SELECT user_id,
       product_id, 
       score,
       ROUND(AVG(score) OVER(),2)  AS 'avg_score', #since no specification, the AVG function is applied to 9 rows. 
       ROUND(AVG(score) OVER(PARTITION BY user_id),2) AS 'user_avg_score', #return averge values based on the particular usr_id
       ROUND(score-AVG(score) OVER(PARTITION BY user_id),2) AS 'use_avg+diff'
       FROM review;
```

For more detail about OVER cluase, visit [https://www.sqlservercentral.com/articles/understanding-the-over-clause]

#### * OVER(ORDER BY) and Ranking The Rows 
First,we want to arrange rows in either descending or ascedning order  with a help of  ORDER BY clause. In addition, 
there are three separate window functions to label the ranks of rows of a result-set, each of which has a distinct feature. 

- ROW_NUMBER() : ranks the rows with no overlap 
- RANK() : allows for overlap , but 'leaping' or 'skiping' in the rank 
- DENSE_LANK()  : allows for overlap and no 'leaping' or 'skiping' in the rank

Furthermore, it is a worthy of remembering other window functions such as "LEAD","LAG". 

- LEAD() : to return a value from  the next row
- LAG() :  to return a value from the previous row 

To see how these all window functions are acually implmented int the case of popular_products

```MySql
DROP TABLE IF EXISTS popular_products;
CREATE TABLE popular_products (
    product_id varchar(255)
  , category   varchar(255)
  , score      numeric
);

INSERT INTO popular_products
VALUES
    ('A001', 'action', 94)
  , ('A002', 'action', 81)
  , ('A003', 'action', 78)
  , ('A004', 'action', 64)
  , ('D001', 'drama' , 90)
  , ('D002', 'drama' , 82)
  , ('D003', 'drama' , 78)
  , ('D004', 'drama' , 58)
;

SELECT product_id,
       score,
       ROW_NUMBER() OVER(ORDER BY score DESC) AS 'row_number',
	   RANK() OVER(ORDER BY score DESC) AS 'rank',
       DENSE_RANK() OVER(ORDER BY score DESC) AS 'density_rank',
       LAG(product_id) OVER(ORDER BY score DESC) AS 'lag1',
       LAG(product_id,2) OVER(ORDER BY score DESC) AS 'lag2', # a row before the previous raw 
	   LEAD(product_id) OVER(ORDER BY score DESC) AS lead1,
       LEAD(product_id,2) OVER(ORDER BY score DESC) as lead2 # a raw after the next raw
	   FROM popular_products;
```
#### * OVER() clause and Aggregating Functions (1)

<Descrition of Each Label>

- row : rank the data on score withouth any overlap
- cum_sum : compute cumulative sum up to k th index 
- local_avg : return an average value of three 'local' values
             (the preceding value right before the current index, the current index,and the follwing value)
- first_value : find the product_id whose socre is the least 
- last_value :  Return the product_id whose score is the highest 


```MySQL
SELECT product_id,
       score, 
	   ROW_NUMBER() OVER(ORDER BY score DESC) AS 'row', 
       SUM(score) OVER(ORDER BY score DESC ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cum_sum,
       AVG(score) OVER(ORDER BY score DESC ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING) AS loca_avg,
       FIRST_VALUE(product_id) OVER(ORDER BY score DESC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS 'first_value',
       LAST_VALUE(product_id) OVER(ORDER BY score DESC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS 'last_value'
       FROM popular_products
       ORDER BY ROW_NUMBER() OVER(ORDER BY score DESC) ; 
	   
       ;
 ```
 
 #### * Using PARTITION BY AND ORDER BY in THE OVER() function at The same time

Like what we have done on the previous section, we are still interested in ranking the rows based on the given scores. Howeever, there
is one more prerequisite at this time - ranking the orders based on scores with the rows being partitioned by category. 

```MySQL
SELECT category,product_id,score,
       ROW_NUMBER() OVER(PARTITION BY category ORDER BY score DESC) AS 'row',
       RANK() OVER(PARTITION BY category ORDER BY score DESC) AS 'rank',
       DENSE_RANK() OVER(PARTITION BY category ORDER BY score DESC) AS 'dense_rank'
       FROM popular_products
       ORDER BY category,'row'
       ;
 ```
 
 #### * K-th higheset Rnaks 
 
 Now, let's extract k-th highest ranks from the sorted rows with all the rows still being partitioned by category. 
 Then, WHERE is an essential clause to meet the task but the problem beings as fllowing 
 
 _" Unser SQL implementing a window funciton is prohibited in WHERE clause"_
 
 To handle this issue, we should use a sub-query,indstead. 
 
```MySQL
SELECT * 
	   FROM(
            SELECT 
                 category,
		 score,
		 ROW_NUMBER() OVER(PARTITION BY category ORDER BY score DESC) AS ranks
                 FROM popular_products
                 )AS popluar_proudcts_k_st_rank
                 WHERE ranks<=2
                 ORDER BY category,ranks;
```

#### * Find The First Rank and Least Rank

```MySQL

SELECT DISTINCT category,
       FIRST_VALUE(product_id) OVER(PARTITION BY category ORDER BY score DESC
       ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS first_rank,
       LAST_VALUE(product_id) OVER(PARTITION BY category ORDER BY score DESC 
       ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) as rank_last
       FROM popular_products
       ORDER BY category;

```
#### * GROUP_CONCAT CALUSE 

Just image a vast of consumers logged on Amazone with thier own user id and puchased a variety of products on display. You may present 
all the purchaseses and the total amount each user spent as two separate fields on the table. This can be done through use of GROUP_CONCAT. 

```MySQL
DROP TABLE IF EXISTS purchase_detail_log;
CREATE TABLE purchase_detail_log (
    purchase_id integer
  , product_id  varchar(255)
  , price       integer
);

INSERT INTO purchase_detail_log
VALUES
    (100001, 'A001', 3000)
  , (100001, 'A002', 4000)
  , (100001, 'A003', 2000)
  , (100002, 'D001', 5000)
  , (100002, 'D002', 3000)
  , (100003, 'A001', 3000)
;


SELECT DISTINCT purchase_id,
       GROUP_CONCAT(product_id),
       SUM(PRICE)
       FROM purchase_detail_log
       GROUP BY purchase_id
       ;
```
As a default, the separtor of GROUP_CONCAT is ','. If you want to change it, put whatever you want to the second part of the clause. 
GROUP_CONCAT(EXP1,'|') 

### 3) Transfomration of Rows into Columns

### _This special 'transposition' is conditional on the fact that you know exactly the number of rows and thier type._

First, select the columns of interset. In losely term, we take y-values and x-values
       In our case, dt provides y-values and indicator provides x-values.
       
Second,extend the 'base'table(ie table with only y-values) with extra columns,one for each x-value
       We add one colum per x-value. Be aware that our x-values is 'indicator'

Third, group and aggregate the exteded table.
       We need to group by dt since it provides the y-values. 
       
       
Lastly, use MAX/MIN function to extract each value for newly created fields.       
        On every dt there is the only one TRUE value in response to CASE cluase available for each of newly created                             fileds;'impressions','sessions',and'users'. Use MAX/MIN to extract the value for every dt


 __"Note that the resulting output from CASE clause is a list. Even for only one single scalar, the resulting foramt is still
  list not a scalar.In order to extract the elment from the list contining the only one value, we often use MAX/MIN function"__
       

```MySQL
DROP TABLE IF EXISTS daily_kpi;
CREATE TABLE daily_kpi (
    dt        varchar(255)
  , indicator varchar(255)
  , val       integer
);

INSERT INTO daily_kpi
VALUES
    ('2017-01-01', 'impressions', 1800)
  , ('2017-01-01', 'sessions'   ,  500)
  , ('2017-01-01', 'users'      ,  200)
  , ('2017-01-02', 'impressions', 2000)
  , ('2017-01-02', 'sessions'   ,  700)
  , ('2017-01-02', 'users'      ,  250)
;


SELECT dt,
       MAX(CASE WHEN indicator='impressions' THEN val END) AS impressions,
	   MAX(CASE WHEN indicator='sessions' THEN val END) AS sessions,
       MAX(CASE WHEN indicator='users' THEN val END) AS users
       FROM daily_kpi
       GROUP BY dt
       ORDER BY dt;
```    
### 4) Splitting comm_Separted Values in MySQL

Now take a look at purchase_log table and you will notice a comma-separated list of product_ids that one of costumers purcahsed with
thier own ideas. You want to add two separte columns which indicate the index of each element in the specific produc_ids
and its corresponding product_id.


- step1) Createa a Separate Table containg numbers at least as big as the length of our longest comma-separted list. 
         In our data, each slot of product_ids conatin up to 3 elements and this makes a temporary table with numbers 
	 equal to 3.
```MySQL
create temporary table numbers as (
  select 1 as n
  union select 2 as n
  union select 3 as n
);

SELECT * FROM numbers;
```

- step 2) Selecting Each Item in the Table
We use a SUBSTRING_INDEX with n-th comma and select the entire list before that comma. We call it again with thrid arguament being 
-1 which selects everything to the left of the commna. 

```MySQL
SELECT purchase_id,product_ids,
	   SUBSTRING_INDEX(SUBSTRING_INDEX(product_ids,',',n),',',-1) AS product_id
       FROM purchase_log
       CROSS JOIN numbers
       ORDER BY purchase_id,product_id
       ;
```

- step 3) A Column Indicatiing a Length of Each list

Now, we need to prepare a sepearte procedure to restirct the number of rows to the length of each list and make it appear on 
the column named as idx'.


Let's take a look at the # code in peices. 
First is char_length, which returns the number of characters in a string. replace(product_ids, ',', '') removes commas from email_recipients. So char_length(email_recipients) - char_length(replace(product_ids, ',', '')) counts the commas.

By joining on the number of commas <= , we get exactly the number of rows as there.

```MySQL

SELECT purchase_id,product_ids,
	   SUBSTRING_INDEX(SUBSTRING_INDEX(product_ids,',',n),',',-1) AS product_id,
       p.n
       FROM purchase_log
       CROSS JOIN numbers AS p
             ON p.n<=(CHAR_LENGTH(product_ids)-CHAR_LENGTH(REPLACE(product_ids,',',''))+1)
       ORDER BY purchase_id,product_id
       ;
 ```











       
   
 
 
 
 
 





 
 
       
    






	 
	     



	




 