## SQL Murder Mystery Solution

This is my solution and rationale to the [SQL Murder Mystery](https://mystery.knightlab.com/) that I wrote on March 19th while doing SQL training. I've decided to leave it as is to evaluate what my areas of improvement are and use it for further practice. To see how an experienced Data Scientist handles this problem, you can visit [this gist written by Mikhail Popov.](https://gist.github.com/bearloga/cfc8099223d1dace2604c8737dcbb4c3)


### Starting Instructions and Database Schema
> The crime was a ​murder​ that occurred sometime on ​Jan.15, 2018​ and that it took place in ​SQL City​. Start by retrieving the corresponding crime scene report from the police department’s database.

![](https://github.com/BrettRamsay/Data/blob/main/schema.png)

### STEP 1: Retrieve the crime scene report

The only thing needed here is to look over the schema and data formatting of `crime_scene_report` to ensure the WHERE clauses are formatted properly. 
```
SELECT * FROM crime_scene_report
WHERE type = 'murder'
AND date = 20180115
AND city = 'SQL City'
```
Query Results:
|date|type|description|city|
|:--:|:--:|:--:|:--:|
|20180115|murder|Security footage shows that there were 2 witnesses. The first witness lives at the last house on "Northwestern Dr". The second witness, named Annabel, lives somewhere on "Franklin Ave".|SQL City|

### STEP 2: Retrieve the two witness statements

The next step requires to identify the two witnesses from `person` and retreive their testimony from `witnesses`. Unlike the opening instructions, there aren't any discrete parameters given to use in the query, so a simple WHERE + OR clause is out. Instead, each person will require a subquery (or CTE) and then these two subqueries will be combined using the UNION operator. This will create a single table that can then be joined to `witnesses` using `person.id` as the foreign key.

```
WITH w1 AS (
  SELECT name, id
  FROM person
  WHERE address_street_name = 'Northwestern Dr'
  ORDER BY address_number DESC
  LIMIT 1
),
w2 AS (
  SELECT name, id
  FROM person
  WHERE name LIKE '%Annabel%'
  AND address_street_name = 'Franklin Ave'
),
combined AS (
  SELECT name, id
  FROM w1
  UNION
  SELECT name, id
  FROM w2)
  
SELECT name, interview.transcript
FROM combined
JOIN interview ON combined.id = interview.person_id;
```

Query Results:

|name|transcript|
|:--:|:--:|
|Morty Schapiro|I heard a gunshot and then saw a man run out. He had a "Get Fit Now Gym" bag. The membership number on the bag started with "48Z". Only gold members have those bags. The man got into a car with a plate that included "H42W".|
|Annabel Miller|I saw the murder happen, and I recognized the killer from my gym when I was working out last week on January the 9th.|

### Step 3: Use the two statements to find the killer

The witness statements reference information that can be found on three different tables: `get_fit_now_member`, `get_fit_now_check_in`, and `drivers_license`. Only the gym tables have a relationship, so we'll have to use `person` as a bridge table since it has relationships with both `get_fit_now_member` and `drivers_license`. A CTE that combines the gym tables produces the unique member ID needed to join `person`.
```
WITH gym AS 
(SELECT name, person_id 
FROM get_fit_now_member as g1
JOIN get_fit_now_check_in AS g2 ON g1.id = g2.membership_id
WHERE id LIKE '48Z%'
AND membership_status = 'gold'
AND g2.check_in_date = 20180109)

SELECT gym.person_id, gym.name, drivers_license.plate_number FROM gym
JOIN person ON gym.person_id = person.id 
JOIN drivers_license ON person.license_id = drivers_license.id
```
Query Results:

|person_id|name|plate_number|
|:--:|:--:|:--:|
|67318|Jeremy Bowers|0H42W2|

While we've caught the killer, using the killer's `person_ID` to retreive his statment from `interview` reveals there's a mastermind to be found.

>I was hired by a woman with a lot of money. I don't know her name but I know she's around 5'5" (65") or 5'7" (67"). She has red hair and she drives a Tesla Model S. I know that she attended the SQL Symphony Concert 3 times in December 2017.


### STEP 4: Use the killer's statement to find the mastermind

Similar to the previous step, we can use a CTE to generate a unique parameter that allows us to join queries from two unrelated tables (`drivers_license` and `facebook_event_check_in`) by using `person` as a bridge. 

```
WITH woman AS (
  SELECT id 
  FROM drivers_license
  WHERE car_make = 'Tesla'
  AND car_model = 'Model S'
  AND hair_color = 'red'
  AND height BETWEEN 65 AND 67
  )
  
  SELECT person.name, person.id from woman
  JOIN person ON person.license_id = woman.id
  JOIN facebook_event_checkin AS event ON person.id = event.person_id
  WHERE event_name = 'SQL Symphony Concert'
  AND date BETWEEN 20171201 AND 20171231
 ``` 
 
 Query Results:

 |name|id|
 |:--:|:--:|
|Miranda Priestly|99716
|Miranda Priestly|99716
|Miranda Priestly|99716

Gottem!
 

Things I could've done better on this query:  
- Use COUNT to aggregate and filter the `facebook_event_check_in` query results. My query worked out fine for this example, but it could potentially cause issues with a much larger table.
- Include a join to the `income` table. I didn't like building my queries using qualitative descriptors ("a lot of money") at the time, but it still should've been part of the query. 
- Include gender in the 'woman' CTE. I may have watched too many spy movies and was wary of of the mastermind potentially wearing a disguise. 

### Another mystery?

Although the murder is solved, I can't help but wonder if SQL City has some underlying issues that are causing its 2018 crime wave...
