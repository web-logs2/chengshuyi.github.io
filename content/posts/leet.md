
notes1

A relation is an unordered set that contains the relationship of attributes that represent entities. Since the
relationships are unordered, the DBMS can store them in any way it wants, allowing for optimization.

A tuple is a set of attribute values (also known as its domain) in the relation. Originally, values had to be
atomic or scalar, but now values can also be lists or nested data structures. Every attribute can be a special
value, NULL, which means for a given tuple the attribute is undefined.

A relation with n attributes is called an n-ary relation.

Data Manipulation Languages (DMLs)
Relational Algebra

select type,primary_title,runtime_minutes from titles where runtime_minutes in (select runtime_minutes from titles order by runtime_minutes desc limit 1) order by type asc, primary_title asc;

select type, count(*) FROM titles GROUP BY type ORDER BY count(*) ASC

select cast(premiered/10*10 as varchar)||'s',count(*) from titles where premiered NOT NULL group by premiered/10;

select cast(premiered/10*10 as varchar)||'s',round(count(*)*100.0/(select count(*) from titles),4) from titles where premiered NOT NULL group by premiered/10;

select primary_title,b.num from titles as a, (select title_id,count(*) as num from akas group by title_id order by count(*) desc limit 10) as b where a.title_id = b.title_id; 
