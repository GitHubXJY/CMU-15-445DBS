
# 概述

[作业1](https://15445.courses.cs.cmu.edu/fall2022/homework1/)是构建一组SQL查询来分析将提供给你的[IMDB数据集](./IMDB.md)。

你可以：(1)学习基本的和某些高级的SQL功能,(2)熟悉使用功能齐全的DBMS [SQLite](https://www.sqlite.org/index.html)

# Placeholder Folder

Create the placeholder submission folder with the empty SQL files that you will use for each question:

```sql
$ mkdir placeholder
$ cd placeholder
$ touch \
  q1_sample.sql \
  q2_sci_fi.sql \
  q3_oldest_people.sql \
  q4_crew_appears_most.sql \
  q5_decade_ratings.sql \
  q6_cruiseing_altitude.sql \
  q7_year_of_thieves.sql \
  q8_kidman_colleagues.sql \
  q9_9th_decile_ratings.sql \
  q10_house_of_the_dragon.sql
$ cd ..
```

After filling in the queries, you can compress the folder by running the following command:
```$ zip -j submission.zip placeholder/*.sql```

The `-j` flag lets you compress all the SQL queries in the zip file without path information. The grading scripts will **not** work correctly unless you do this.
