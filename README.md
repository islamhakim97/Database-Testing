# Database-Testing
Database Basics StoredProcedures And Functions

--------------------------------------------Database Basics-------------------------------------------
use classicmodels ;
show tables ; 
describe INFORMATION_SCHEMA.Tables;
describe INFORMATION_SCHEMA.Columns ;
select column_name from INFORMATION_SCHEMA.Columns where table_name='customers';
select column_name,data_type,column_type from INFORMATION_SCHEMA.Columns where table_name='customers';
select column_name,is_nullable,column_key from INFORMATION_SCHEMA.Columns where table_name='customers';

select column_type from information_schema.columns where table_name='customers';
show procedure status where db = 'classicmodels';

show procedure status where db = 'classicmodels';
use classicmodels;
select country,case
when country ='USA' then '2-day Shipping'
when country ='Canda' then '3-day Shipping'
else '5-day Shipping'
end as shippingtime from customers where customerNumber=112;
select (select count(*)from orders where customerNumber=141 and status ='shipped')as shipped,(select count(*)from orders where customerNumber=141 and status ='Canceled')as Cancelled,(select count(*)from orders where customerNumber=141 and status ='Resolved')as Resolved,(select count(*)from orders where customerNumber=141 and status ='Disputed')as Disputed;
select country,case when country ='USA' then '2-day Shipping' when country ='Canda' then '3-day Shipping' else '5-day Shipping' end as shippingTime FROM customers where customerNumber=260;
select customerName,case when creditLimit >5000 then 'platinum' when creditLimit >10000  then 'Gold'when creditLimit <1000 then 'silver' END AS customerLevel from customers where customerNumber = 131; 

------------------------------------------ Stored Procedures --------------------------------------------

use classicmodels;
delimiter //
create procedure getAllCustomers()
begin
select * from customers;
end //
delimiter ;
call getAllCustomers();

delimiter //
create procedure getAllCustomersByCity(IN mycity varchar(50))
begin
   select * from customers where city = mycity;
end //
delimiter ;
call getAllCustomersByCity('singapore');

delimiter //
create procedure GetAllCutomersByCityandPin(IN mycity varchar(50), IN pcode varchar(15))
begin
select * from customers where city=mycity and postalCode=pcode; 
end //
delimiter ;
call GetAllCutomersByCityandPin('singapore','079903');

-- Make A procedure that take one param and return 4 output parap abount how many ordders are SHIPPED,Cancelled ,Resolved and Disputed.

delimiter //
create procedure Get_num_ofOrdersBy_custId(in cust_num int, out Shipped int,out Cancelled int , out Resolved int , out Disputed int)
begin
-- Shipped
select count(*) into shipped from orders where customerNumber=cust_num and status = 'Shipped';
-- Cancelled
select count(*) into Cancelled from orders where customerNumber=cust_num and status = 'Cancelled';
-- Resolved
select count(*) into Resolved from orders where customerNumber=cust_num and status = 'Resolved';
-- Disputed
select count(*) into Disputed from orders where customerNumber=cust_num and status = 'Disputed';
end //

delimiter ;

call Get_num_ofOrdersBy_custId(141,@shipped,@Cancelled,@Resolved,@Disputed);
select @shipped,@Cancelled,@Resolved,@Disputed;

-- create SP where u take cust_num as input and pshipping as output param based on ccode of customer u provide diffrent caases for shipping.

delimiter //
create procedure GetCustomerShipping(in cust_num int , out pShipping varchar(100))
begin
declare customerCountry varchar(100);
select country into customerCountry from customers where customerNumber=cust_num;
case customerCountry
when 'USA' then 
set pShipping = '2-day shipping';
when 'canada' then 
set pShipping ='3-day shipping';
else
set pShipping ='5-day shipping';
end case ;
end //
delimiter ;
call GetCustomerShipping(353,@pShipping);
select @pShipping;
-- Exception Handling in stored Procedure

delimiter //
create procedure insertOrderandcustomerNum(in orderid int,in cus_num int)
begin
-- exit if Dublicate Key Occurs
declare exit handler for 1062 select 'Dublicate keys error encounterd' message;
declare exit handler for SQLSTATE '23000' select 'sql state 2300' errorcode;
declare exit handler for sqlexception select 'sql exception error encounterd' message;
insert into orders (orderNumber,customerNumber) values (orderid,cus_num) ;
select count(*) from orders where orderNumber=orderid ;

end //
delimiter ;
call insertOrderandcustomerNum(1001,1100)

----------------------------------------- Stored Functions ----------------------------------

use classicmodels;
select * from customers;
delimiter //
create function customerLevel(credit decimal(10,2)) returns varchar(20)
deterministic
begin
    declare customerLevel varchar(20);
if credit>5000
then set customerLevel = 'Platinium';
elseif credit>10000 
then set customerLevel = 'silver';
end if ;
return customerLevel;
end //
delimiter ;
select customerLevel(creditLimit) from customers;
show function status where db='classicmodels';
delimiter //
create procedure GetCustomerLevel(in customerNO int,out customerLevel varchar(20))
begin
declare credit dec(10,2);
-- get credit limit of a customer 
select creditLimit into credit from customers where customerNumber = customerNo;
-- call the function
set customerLevel = customerLevel(credit);
end //
delimiter ;
call GetCustomerLevel(112,@customerLevel);
select @customerLevel;
select customerName,case when creditLimit >5000 then "platinum" when creditLimit >10000  then "Gold" END AS customerLevel from customers; 
select customerName,case when creditLimit >5000 then "platinum"when creditLimit >10000  then "Gold"when creditLimit <1000 then "silver"END AS customerLevel from customers where customerNumber = 131; 
call GetCustomerLevel(131,@customerLevel);
select @customerLevel;

---------------------------------------------- Triggers ------------------------------------------------------
use classicmodels;
show triggers;
drop table if exists workcenter;
drop table if exists workcenterstats;
create table workcenter(
id int auto_increment primary key,
name varchar(100) not null,
capacity varchar(100) not null
);
describe workcenter;
create table workcenterstats(
totalcapacity int not null);

-- ## create before_workcenters_update #trigger 1 ********************************************************************************************************
delimiter //
create trigger before_workcenters_update before insert on workcenter for each row 
begin
declare rowcount int;
select count(*) into rowcount from workcenterstats;
if rowcount>0 then
update workcenterstats set totalcapacity = totalcapacity+new.capacity ;
else 
insert into workcenterstats (totalcapacity) values (new.capacity);
end if; 
end //

delimiter ;
show triggers;
SET SQL_SAFE_UPDATES = 0;
select * from workcenterstats;
insert into workcenter (name,capacity) values ("Islam",100); 
select * from workcenterstats;
insert into workcenter (name,capacity) values ("Ahmed",200);
select * from workcenterstats;
insert into workcenter (name,capacity) values ("Rehap",300); 
select * from workcenterstats;

-- ### create a new After insert #Trigger2 ********************************************************************************************************
 -- members table
create table members (
id int auto_increment primary key,
name varchar (100) not null,
email varchar (100) not null,
birthDate date 
);
-- reminders table 
create table reminders(
id int auto_increment,
memberId int ,
message varchar(255) not null,
primary key (id,memberId)
);
-- create after_members_insert trigger 
delimiter //
create trigger after_members_insert after insert on members for each row 
begin 
if new.birthDate is null then 
insert into reminders (memberId,message) values (new.id,concat('Hi',' ',new.name,'please update your date of birth.'));
end if;
end //
delimiter ;
show triggers;
-- test triggers 
insert into members (name,email,birthDate) value ('islam','islam@mail.com',null);
select * from reminders;
insert into members (name,email,birthDate) value ('adnan','adnan@mail.com',null);
select * from reminders;

drop trigger if exists after_members_insert;
-- ## sales Table and #Trigger3  ****************************************************************************************************************************************************************************************************************
create table sales (
id int auto_increment,
product varchar(100) not null,
quantity int not null default 0 ,
fiscalyear smallint not null,
fiscalmonth smallint not null,
check(fiscalmonth>=1 and fiscalmonth<=12),
check(fiscalyear between 2000 and 2050),
check (quantity>=0),
unique(product,fiscalyear ,fiscalmonth),
primary key (id)
);
-- drop table if exists sales;

insert into sales (product,quantity,fiscalyear,fiscalmonth) values
('2003 Harley-Davidson Eagle Drag Bike',120,2020,1),
 ('1996 Corvair Monaz',150,2020,1),
('1970 Plymouth Hemi cuda Bike',200,2020,1);

delimiter //
create trigger before_sales_update before update on sales for each row 
begin
declare errorMessage varchar (400);
set errorMessage = concat('The new quantity ',new.quantity,'cannnot be 3 times greater than the current quantity',old.quantity);
if new.quantity>old.quantity *3 then 
signal sqlstate '45000' set message_text = errorMessage;
end if;
end //
delimiter ;
select * from sales;
-- test trigger 
update sales set quantity = 100 where id =1;
update sales set quantity = 500 where id =1;

-- ****************************************************************************************************************************************************************************************************************
-- #trigger #3 After_sales_update 
create table salesChange(
id int auto_increment primary key,
salesId int,
beforequantity int,
afterquantity int,
changeAt timestamp not null default current_timestamp
);
-- #trigger3 NotiCE : <> means not equl !=
delimiter //
create trigger after_sales_update after update on sales for each row
begin 
if old.quantity <> new.quantity then
insert into salesChange (salesId,beforequantity,afterquantity) values (old.id,old.quantity,new.quantity);
end if;
end//
delimiter ;
show triggers;
-- test trigger after_sales_update
update sales set quantity = 180 where id =1;
select * from salesChange;
update sales set quantity = cast(quantity*1.1 as unsigned) ;
select * from sales;


-- ## salariesTable and #Trigger4 berfore_salaries_delete  ****************************************************************************************************************************************************************************************************************

create table salaries(
employeeNumber int primary key ,
validForm date not null,
salary decimal(12,2) not null default 0
);
insert into salaries (employeeNumber,validForm,salary) values
(1002,'2000-01-01',40000),
(1003,'2010-02-21',40000),
(1004,'2011-05-22',40000);
select * from salaries;

create table salaryArchives(
id int primary key auto_increment,
validForm date not null,
salary decimal(12,2) not null default 0,
deletedAt timestamp default now()
);
select * from salaryArchives;
delimiter //
create trigger before_delete_trigger before delete on salaries for each row
begin
insert into salaryArchives (employeeNumber,validForm,salary) values(old.employeeNumber,old.validForm,old.salary);
end //
delimiter ;
show triggers;
-- test before_delete_trigger 
delete from salaries where employeeNumber = 1002;
select * from salaryArchives;
select * from salaries;
drop table salaries

