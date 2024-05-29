- Join Three tables 
```sql
SELECT * FROM USERS;

SELECT * FROM COMMENTS;

SELECT * FROM PHOTOS;

SELECT USERNAME,CONTENTS,URL
FROM COMMENTS
JOIN USERS ON USERS.ID = COMMENTS.USER_ID
JOIN PHOTOS ON USERS.ID = PHOTOS.USER_ID;
```
- Alternate forms of syntax 

```sql
--- join two tables
select comments.id, contents, url from comments
join photos on photos.id=comments.photo_id;

--- join tables and alias table column
select comments.id as comment_id, contents as story, url as "web site link"
from comments 
join photos on photos.id=comments.photo_id;

--- join three tables and alias table column
select username as "user Name", contents as story, url as "web site link"
from comments 
join photos on photos.id=comments.photo_id
join users on users.id=photos.user_id;

--- join three tables, alias table and table column
select c.id, username as "user Name", contents as story, url as "web site link"
from comments as c 
join photos on photos.id=c.photo_id
join users on users.id=photos.user_id;
```
50