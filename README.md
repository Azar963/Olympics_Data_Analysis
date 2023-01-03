# Olympics_Data_Analysis

Olympics Data Analysis Project with SQL

The Olympic Games are a global event that brings together athletes from around the world to compete in various sports. With the Summer and Winter Olympics held every four years, there is a wealth of data that can be analyzed to understand the performance of different countries, sports, and athletes.

For this data analysis project, we will use SQL to analyze data from the past 20 years (2000-2020) of the Summer Olympics.
To begin, we will need to gather the data and import it into a SQL database. The official Olympic website (www.olympic.org) has a list of all medalists for each Olympic Games, which can be downloaded as a CSV file and imported into a table in our database.

Once the data is in the database, we can use SQL queries to clean and organize it. This may include removing any irrelevant columns, renaming columns for clarity, and handling any missing or incorrect data.

-- 1. How many olympics games have been held?


    SELECT 
        COUNT(DISTINCT games) AS total_olympic_game
    FROM
        olympics_history;
        
-- -- 2.List down all olympics games held so far.


    SELECT DISTINCT
        oh.year, oh.season, oh.city
    FROM
        olympics_history oh
    ORDER BY year;
    
 -- 3. Mention the total no of nations who participated in each olympics game?
 
 
    with all_countries as 
      (select games, nr.region 
        from olympics_history oh 
        join olympics_history_noc_regions nr on nr.noc = oh.noc
      group by games, nr.region)
    select games, count(1) as total_countries
    from all_countries
    group by games
    order by games;
    
-- 4. Which year saw the highest and lowest no of countries participating in olympics


    with  all_countries as
        (select games, nr.region
        from olympics_history oh
        join olympics_history_noc_regions nr on nr.noc = oh.noc
        group by games, nr.region),
      tot_countries as
        (select games, count(1) as total_countries
            from all_countries 
            group by games)
      select distinct
        concat(first_value(games)  over(order by total_countries), '-',
         first_value(total_countries) over(order by total_countries)) as Lowest_Countries,
         concat(first_value(games) over(order by total_countries desc), '-',
         first_value(total_countries) over(order by total_countries desc)) as Highest_Countries
         from tot_countries
         order by 1;
         
-- 5. Which nation has participated in all of the olympic games


        with tot_games as
          (select count(distinct games) as total_games
          from olympics_history),
        countries as 
          (select games, nr.region as country
                from olympics_history oh 
                join olympics_history_noc_regions nr on nr.noc = oh.noc
                group by games, nr.region),
        countries_participated as
          (select country, count(1) as total_participated_games
                from countries
                group by country)
      select cp.*
        from countries_participated cp
        join tot_games tg on tg.total_games = cp.total_participated_games
        order by 1;
        

-- 6. Identify the sport which was played in all summer olympics.


      with t1 as
        (select count(distinct games) as total_games
            from olympics_history where season = 'Summer'),
            t2 as
            (select distinct games, sport
            from olympics_history where season ='Summer'),
            t3 as
            (select sport, count(1) as no_of_games
            from t2
            group by sport)
      select * 
        from t3
        join t1 on t1.total_games = t3.no_of_games;
        
        
-- 7. Which Sports were just played only once in the olypics.


      with t1 as
        (select distinct games, sport
            from olympics_history),
            t2 as
            (select sport, count(1) as no_of_games
            from t1
            group by sport)
      select t2.*, t1.games
        from t2
        join t1 on t1.sport = t2.sport
        where t2.no_of_games = 1
        order by t1.sport;
        
        
-- 8. Fetch the total no of sports played in each olympic games.


      With t1 as
        (select distinct games, sport
            from olympics_history),
            t2 as
            (select games, count(1) as no_of_sports
            from t1
            group by games)
      select * from t2
        order by no_of_sports desc;
        
        
-- 9. Fetch oldest athletes to win a gold medal


      with temp as
          (select name, sex, cast(case when age = 'NA' then '0' else age end as int) as age,
          team, games, city, sport, event, medal
          from olympics_history),
        ranking as
          (select *, rank() over(order by age desc) as rnk
                from temp
                where medal = 'Gold')
      select *
        from ranking
        where rnk = 1;
        
        
-- 10. Find the Ratio of male and female athletes participated in all olympic games.


      with t1 as 
        (select sex, count(1) as cnt
            from olympics_history
            group by sex),
        t2 as
          (select *, row_number() over(order by cnt) as rn
                from t1),
         min_cnt as
          (select cnt from t2 where rn = 1),
         max_cnt as
          (select cnt from t2 where rn =2)
      select concat('1 : ', round(cast(max_cnt.cnt as decimal) /min_cnt.cnt, 2)) as ratio
        from min_cnt,max_cnt;
        
        
-- 11. Fetch the top 5 athletes who have won the most gold medals.


      with t1 as
        (select name, team, count(1) as total_gold_medals
            from olympics_history
            where medal = 'Gold'
            group by name, team
            order by total_gold_medals desc),
            t2 as
          (select *, dense_rank() over (order by total_gold_medals desc) as rnk
          from t1)
      select name, team, total_gold_medals
        from t2
        where rnk <= 5;
        
        
-- 12. Fetch the top 5 athletes who have won the most medals(gold/silver/bronze).


      with t1 as
        (select name, team, count(1) as total_medals
            from olympics_history
            where medal in ('Gold', 'Silver', 'Bronze')
            group by name, team
            order by total_medals desc),
          t2 as
                (select *, dense_rank() over(order by total_medals desc) as rnk
                from t1)
      select name, team, total_medals
        from t2 
        where rnk <=5;
        
        
-- 13. Fetch the top 5 most successful countries in olympics. Success is defined by no of medals won.


      with t1 as
        (select nr.region, count(1) as total_medals
            from olympics_history oh 
            join olympics_history_noc_regions nr on nr.noc = oh.noc
            where medal <> 'NA'
            group by nr.region 
            order by total_medals desc),
            t2 as
            (select *, dense_rank() over(order by total_medals desc) as rnk
            from t1)
         select * 
         from t2
         where rnk <= 5;
         
-- 14. List down total gold, silver and bronze medals won by each country.


    SELECT nr.region as country,
           SUM(CASE WHEN medal = 'Gold' THEN 1 ELSE 0 END) AS gold,
           SUM(CASE WHEN medal = 'Silver' THEN 1 ELSE 0 END) AS silver,
           SUM(CASE WHEN medal = 'Bronze' THEN 1 ELSE 0 END) AS bronze
    FROM olympics_history oh
    JOIN olympics_history_noc_regions nr ON nr.noc = oh.noc 
    WHERE medal <> 'NA' 
    GROUP BY nr.region
    ORDER BY gold DESC, silver DESC, bronze DESC;
    
    
-- 15. List down total gold, silver and bronze medals won by each country corresponding to each olympics games.


    SELECT LEFT(games, LOCATE(' - ', games) - 1) AS games,
           RIGHT(games, LENGTH(games) - LOCATE(' - ', games) - 2) AS country,
           SUM(CASE WHEN medal = 'Gold' THEN 1 ELSE 0 END) AS gold,
           SUM(CASE WHEN medal = 'Silver' THEN 1 ELSE 0 END) AS silver,
           SUM(CASE WHEN medal = 'Bronze' THEN 1 ELSE 0 END) AS bronze
    FROM olympics_history oh
    JOIN olympics_history_noc_regions nr ON nr.noc = oh.noc
    WHERE medal <> 'NA'
    GROUP BY games + ' - ' + nr.region
    ORDER BY games, gold DESC, silver DESC, bronze DESC;
    
    
  -- 16. Identify country won the most gold,most silver and most bronze medals in each olympic games.
  
  
      WITH temp as
    	(SELECT substring(games, 1, position(' - ' in games) - 1) as games
    	 	, substring(games, position(' - ' in games) + 3) as country
            , coalesce(gold, 0) as gold
            , coalesce(silver, 0) as silver
            , coalesce(bronze, 0) as bronze
    	FROM CROSSTAB('SELECT concat(games, '' - '', nr.region) as games
    					, medal
    				  	, count(1) as total_medals
    				  FROM olympics_history oh
    				  JOIN olympics_history_noc_regions nr ON nr.noc = oh.noc
    				  where medal <> ''NA''
    				  GROUP BY games,nr.region,medal
    				  order BY games,medal',
                  'values (''Bronze''), (''Gold''), (''Silver'')')
    			   AS FINAL_RESULT(games text, bronze bigint, gold bigint, silver bigint))
    select distinct games
    	, concat(first_value(country) over(partition by games order by gold desc)
    			, ' - '
    			, first_value(gold) over(partition by games order by gold desc)) as Max_Gold
    	, concat(first_value(country) over(partition by games order by silver desc)
    			, ' - '
    			, first_value(silver) over(partition by games order by silver desc)) as Max_Silver
    	, concat(first_value(country) over(partition by games order by bronze desc)
    			, ' - '
    			, first_value(bronze) over(partition by games order by bronze desc)) as Max_Bronze
    from temp
    order by games;
    
    
-- 17. In which Sport/event, India has won highest medals.
    
    
    with t1 as
        	(select sport, count(1) as total_medals
        	from olympics_history
        	where medal <> 'NA'
        	and team = 'India'
        	group by sport
        	order by total_medals desc),
        t2 as
        	(select *, rank() over(order by total_medals desc) as rnk
        	from t1)
    select sport, total_medals
    
    
-- 18. Break down all olympic games where india won medal for Hockey and how many medals in each olympic games


        select team, sport, games, count(1) as total_medals
        from olympics_history
        where medal <> 'NA'
        and team = 'India' and sport = 'Hockey'
        group by team, sport, games
        order by total_medals desc;
        from t2
        where rnk = 1;
