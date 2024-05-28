- Create Table with primary key with auto generate serial number
```sql
create table user(
id serial primary key,
name varchar(50),
dept varchar(20)
);
```

- Insert data user table 
```sql
insert into user (name,dept)
values ('Rakibul Islam Raju','IT'),
		('Rasel Dewan','Marketing'),
		('Hayder Hossain','IT'),
		('Nitu Akter','HR');
		
select * from user;
```

- Foreign key 
```sql
create table photo
(
	id serial primary key,
	url varchar(250),
	user_id integer references user(id)
);
```

- Data insert into photo table 
```sql

insert into photo (url,user_id)
values ('https://xyz.bd/03.png',1),
		('https://xyz.bd/02.png',2),
		('https://xyz.bd/04.png',2),
		('https://xyz.bd/05.png',2),
		('https://xyz.bd/06.png',3),
		('https://xyz.bd/07.png',3),
		('https://xyz.bd/08.png',4);
	

select * from PHOTO;
```
- join table user and photo
```sql
select name,url,user_id from photo
join user on user.id=photo.user_id;
```


