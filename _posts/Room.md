SELECT count(*) from TRIP_INFO  GROUP BY strftime('%Y',  END_TIME)


select  sum(TOTAL_CONSUM)  from TRIP_INFO where   strftime('%Y-%m',  END_TIME_TEXT)='2017'



select * from TRIP_INFO where strftime('%Y-%m', TRIP_INFO) = '2010-02'




//查询最近三个月  每天消耗，
SELECT sum(TOTAL_CONSUM), sum(TRIP_DISTANCE) from TRIP_INFO where  END_TIME_TEXT  between datetime('now','start of month','-2 month') and  datetime('now','start of month','1 month','-1 second')  GROUP BY strftime('%Y-%m-%d',  END_TIME_TEXT)


//查询最近三年，每个月消耗
SELECT  sum(TOTAL_CONSUM),sum(TOTAL_CONSUM) from TRIP_INFO where  END_TIME_TEXT  between datetime('now','start of year','-2 year') and  datetime('now','start of year','+1 year','-1 second')  GROUP BY strftime('%Y-%m',  END_TIME_TEXT)


//近三年数据的 开始时间和结束时间
SELECT min(START_TIME_TEXT),max(START_TIME_TEXT) from TRIP_INFO where  END_TIME_TEXT  between datetime('now','start of year','-2 year') and  datetime('now','start of year','+1 year','-1 second')


SELECT  sum(TOTAL_CONSUM),sum(TOTAL_CONSUM) from TRIP_INFO where  END_TIME_TEXT  between datetime('now','start of year','-2 year') and  datetime('now','start of year','+1 year','-1 second')  GROUP BY strftime('%Y-%m',  END_TIME_TEXT)


SELECT sum(TOTAL_CONSUM), sum(TRIP_DISTANCE), END_TIME_TEXT, START_TIME FROM TRIP_INFO where END_TIME_TEXT between datetime('2018-06-17','start of month') and datetime('2018-09-17','start of month','+1 month','-1 seconds') GROUP BY strftime('%Y-%m-%d',  END_TIME_TEXT )



SELECT sum(TOTAL_CONSUM), sum(TRIP_DISTANCE), END_TIME_TEXT, START_TIME FROM TRIP_INFO where END_TIME_TEXT between ? and ? GROUP BY ?

SELECT  sum(totalConsum),sum(tripDistance) from MonthTrip   where timestamp>= 1356970214 and timestamp <=1537860346 GROUP BY strftime('%Y',  '2018-09-20')


SELECT  sum(totalConsum),sum(tripDistance) from DayTrip where  date  between datetime('2017-09-20','start of year') and  datetime('2018-09-20','start of year','+1 year','-1 second')   GROUP BY strftime('%Y-%m',  '2018-09-20')


SELECT  sum(_totalConsum),sum(_tripDistance) from MonthTrip   where _timestamp>= 1356970214 and _timestamp <=1537860346 GROUP BY strftime('%Y',  '2018-09-20')






http://www.mamicode.com/info-detail-2225393.html
