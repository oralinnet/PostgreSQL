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

- Delete data from primary table By using on delete cascade 
In this example when you delete data from primary table it is also delete value from references tables
```sql
DROP TABLE photos;

-- This will result in an error
SELECT * FROM photos;

CREATE TABLE photos (
  id SERIAL PRIMARY KEY,
  url VARCHAR(200),
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE
);

INSERT INTO photos (url, user_id)
VALUES
	('http://one.jpg', 4),
	('http://two.jpg', 1),
  ('http://25.jpg', 1),
  ('http://36.jpg', 1),
  ('http://754.jpg', 2),
  ('http://35.jpg', 3),
  ('http://256.jpg', 4);

DELETE FROM users
WHERE id = 1;

SELECT * FROM photos;
```
- Delete data from primary table by using on delete set null
In this example when you delete data from primary table it is set null value in reference tables value. 

```sql
DROP TABLE photos;

CREATE TABLE photos (
  id SERIAL PRIMARY KEY,
  url VARCHAR(200),
  user_id INTEGER REFERENCES users(id) ON DELETE SET NULL
);

INSERT INTO photos (url, user_id)
VALUES
  ('http:/one.jpg', 4),
  ('http:/754.jpg', 2),
  ('http:/35.jpg', 3),
  ('http:/256.jpg', 4);

DELETE FROM users
WHERE id = 4;

SELECT * FROM photos;
```