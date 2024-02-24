DATA MODEL: 

![image](https://github.com/varsharamesh27/Univents/assets/58926214/51344ef2-6989-41c4-a6cd-311889607a21)


SSIS Workflow: 
 
![image](https://github.com/varsharamesh27/Univents/assets/58926214/123d9b73-7588-4df4-b2ba-ba35019a8554)

Execution: 

![image](https://github.com/varsharamesh27/Univents/assets/58926214/ddcec4e5-8f24-4250-be2e-2121a2468ccd)


SQL Queries:

SIMPLE QUERIES: 

-- Retrieve all events with their details
SELECT e.name, e.date, e.start_time, e.end_time
FROM event e 
INNER JOIN registers r ON e.eventid = r.eventid 
INNER JOIN user u ON u.userid = r.userid 
WHERE Date >= CURRENT_DATE
AND u.university = 'Northeastern University'
ORDER BY date asc

-- List active administrators and their roles
SELECT a.AdminID, a.AdminRole, u.First_Name, u.Last_Name
FROM univents.administrator a
INNER JOIN univents.user u ON a.AdminID = u.UserID;

AGGREGATED FUNCTIONS: 
-- Find the average rating for each event category with sentiment analysis
SELECT
    ec.CategoryID,
    ec.Name AS CategoryName,
    CAST(AVG(efa.AverageRating) AS DECIMAL(16,2)) AS AverageRating,
    COUNT(efa.EventID) AS EventCount
FROM univents.eventcategory ec
INNER JOIN event e ON e.CategoryID = ec.categoryid 
LEFT JOIN univents.eventfeedbacksentimentanalysis efa ON e.eventid = efa.eventid
GROUP BY ec.CategoryID, ec.Name
ORDER BY AverageRating DESC;

-- Calculate the total registrations and average rating for each event
SELECT
    e.EventID,
    e.Name AS EventName,
    COUNT(r.UserID) AS TotalRegistrations,
    CAST( AVG(f.Rating) AS DECIMAL(16,2)) AS AverageRating
FROM event e
LEFT JOIN registers r ON e.EventID = r.EventID
LEFT JOIN feedback f ON e.EventID = f.EventID
GROUP BY e.EventID, e.Name
ORDER BY TotalRegistrations DESC;

INNER / OUTER JOIN: 
-- List events along with their organizers, venues, and the earliest registration date
SELECT
    e.EventID,
    e.Name AS EventName,
    v.University,
    o.OrganizerID AS PrimaryOrganizerID,
    o.Bio AS PrimaryOrganizerBio,
    COALESCE(GROUP_CONCAT(co.OrganizerID), 'None') AS CoOrganizers,
    v.Name AS VenueName
FROM
    univents.event e
    INNER JOIN univents.organisedby ob ON e.EventID = ob.EventID
    INNER JOIN univents.organizer o ON ob.OrganizerID = o.OrganizerID
    LEFT JOIN univents.organisedby coob ON e.EventID = coob.EventID AND coob.OrganizerID != o.OrganizerID
    LEFT JOIN univents.organizer co ON coob.OrganizerID = co.OrganizerID
    LEFT JOIN univents.venue v ON e.VenueID = v.VenueID
GROUP BY
    e.EventID, e.Name, v.University, o.OrganizerID, o.Bio, v.Name
ORDER BY
    University, EventID;
    
-- Retrieve administrators for a specific event
SELECT
    a.AdminID,
    ar.AdminRole,
    u.First_Name,
    u.Last_Name,
    e.Date AS WorkDay
FROM
    administrator a
    INNER JOIN admin_role ar ON a.AdminRole = ar.AdminRoleID
    INNER JOIN user u ON a.AdminID = u.UserID
    INNER JOIN admin_manage am ON a.AdminID = am.AdminID
    INNER JOIN event e ON am.EventID = e.EventID
WHERE
    e.EventID = '1';

-- NESTED QUERIES 
-- Retrieve User Information for Attendees of Category 2 Events at Stanford University
SELECT 
    user.UserID,
    user.First_Name,
    user.Last_Name,
    user.University
FROM 
    user
WHERE 
    user.UserID = ANY (
        SELECT 
            attends.UserID
        FROM 
            attends
            JOIN event ON attends.EventID = event.EventID
        WHERE 
            event.CategoryID = 2 
    )
AND UNIVERSITY = 'Stanford University'
ORDER BY 
    user.University, user.UserID;
    
    
-- Students Who Organized Events at Massachusetts Institute of Technology
SELECT s.studentid, CONCAT(u.first_name, ' ', u.last_name) AS Name, s.major, s.Department 
FROM univents.student s
INNER JOIN univents.user u ON s.studentid = u.userid
WHERE studentid IN (
    SELECT ob.OrganizerID
    FROM univents.organisedby ob
    INNER JOIN univents.event e ON ob.EventID = e.EventID
    INNER JOIN univents.venue v ON e.VenueID = v.VenueID
    WHERE v.University = 'Massachusetts Institute of Technology'
);

-- CORRELATED QUERIES 

-- List of Organizers from Northeastern University for Research Symposium Events
SELECT DISTINCT CONCAT(u.First_Name, ' ', u.Last_Name) AS OrganizerName
FROM univents.user u
INNER JOIN univents.organizer o ON u.UserID = o.OrganizerID
WHERE EXISTS (
    SELECT *
    FROM univents.organisedby ob
    INNER JOIN univents.event e ON ob.EventID = e.EventID
    WHERE ob.OrganizerID = u.UserID
        AND e.CategoryID IN (
            SELECT ec.CategoryID
            FROM univents.eventcategory ec
            WHERE ec.Name = 'Research Symposium'
        )
)
AND University = 'Northeastern University'

-- List of MIT Students Attending High-Capacity Events
SELECT DISTINCT CONCAT(u.First_Name, ' ', u.Last_Name) as Name, u.university
FROM univents.user u
INNER JOIN univents.student s ON u.UserID = s.StudentID 
WHERE EXISTS (
    SELECT *
    FROM univents.attends a
    INNER JOIN univents.event e ON a.EventID = e.EventID
    INNER JOIN univents.venue v ON e.VenueID = v.VenueID
    WHERE a.UserID = u.UserID
        AND v.Capacity >= (SELECT MAX(CAPACITY) FROM venue)
)
AND u.university = 'Massachusetts Institute of Technology';
    
-- ALL/ ANY/ EXISTS/ NOT EXISTS
-- Retrieve the most commonly organized event by the same group across all organizers
SELECT
    ob.OrganizerID,
    MAX(pe.PopularEvent) AS PopularEvent,
    MAX(pe.EventCount) AS EventCount
FROM
    organisedby ob
JOIN (
    SELECT
        ob.OrganizerID,
        e.EventID,
        e.Name AS PopularEvent,
        COUNT(DISTINCT e.EventID) AS EventCount
    FROM
        event e
    INNER JOIN organisedby ob ON e.EventID = ob.EventID
    WHERE
        EXISTS (
            SELECT 1
            FROM organisedby otherOb
            WHERE otherOb.EventID = e.EventID
                AND otherOb.OrganizerID <> ob.OrganizerID
        )
    GROUP BY
        ob.OrganizerID, e.EventID, e.Name
) AS pe ON ob.OrganizerID = pe.OrganizerID
GROUP BY
    ob.OrganizerID
ORDER BY
    MAX(pe.EventCount) DESC
LIMIT 1;

-- List events that do not have any feedback
SELECT e.Name, e.Date, e.Start_Time,e.End_Time, v.Name
FROM event e 
INNER JOIN venue v ON e.venueid = v.venueid 
WHERE NOT EXISTS (
    SELECT 1
    FROM feedback f
    WHERE f.EventID = e.EventID
)
AND University = 'Northeastern University'

-- QUERIES WITH SET FUNCTION
-- Students who have not registered for any events
SELECT s.studentid
FROM student s
INNER JOIN user u ON s.studentid = u.userid 
WHERE u.university = 'Northeastern University'
EXCEPT
SELECT u.UserID
FROM registers r 
INNER JOIN user u ON r.userid = u.userid 
WHERE u.university = 'Northeastern University'

-- Events without any attendees (using INTERSECT)
SELECT e.EventID, e.Name AS EventName
FROM event e
INNER JOIN  venue v ON e.venueid = v.venueid 
WHERE EventID NOT IN (
    SELECT EventID
    FROM attends
    WHERE EventID IS NOT NULL
)
AND v.University = 'New York University'

INTERSECT

SELECT e.EventID, e.Name
FROM event e
INNER JOIN venue v ON e.venueid = v.venueid
WHERE v.University = 'New York University'
ORDER BY EventID;


    
-- SUBQUERIES IN SELECT & FROM 
-- Retrieve Most Attended User Details and Events Count by University
SELECT
    university.UniversityID,
    university.UniversityName,
    (
        SELECT CONCAT(user.First_Name, ' ', user.Last_Name) AS MostAttendedUserName
        FROM attends
        INNER JOIN user ON attends.UserID = user.UserID
        WHERE user.University = university.UniversityName
        GROUP BY user.UserID
        ORDER BY COUNT(DISTINCT attends.EventID) DESC
        LIMIT 1
    ) AS MostAttendedUserName,
    (
        SELECT COUNT(DISTINCT attends.EventID) AS AttendedEventsCount
        FROM attends
        INNER JOIN user ON attends.UserID = user.UserID
        WHERE user.University = university.UniversityName
        GROUP BY user.UserID
        ORDER BY AttendedEventsCount DESC
        LIMIT 1
    ) AS AttendedEventsCount
FROM
    university;
    
-- Organizers at Northeastern University with Average Ratings
SELECT 
    o.OrganizerID,
    CONCAT(u.first_name, ' ', u.Last_Name) AS OrganizerName,
    (
        SELECT CAST(AVG(f.Rating) AS DECIMAL(16,2)) 
        FROM organisedby ob 
        JOIN feedback f ON ob.EventID = f.EventID 
        JOIN event e ON ob.EventID = e.EventID
        JOIN venue v ON e.VenueID = v.VenueID
        WHERE ob.OrganizerID = o.OrganizerID AND v.University = 'Northeastern University' AND f.rating IS NOT NULL
    ) AS AverageRatingPerOrganizer
FROM 
    organizer o
INNER JOIN user u ON o.OrganizerID = u.UserID
HAVING 
    AverageRatingPerOrganizer IS NOT NULL;

Stored Procedures:

-- Update event and notify Organizer

DELIMITER $$
CREATE DEFINER=`root`@`localhost` PROCEDURE `UpdateEventAndNotifyAttendees`(
    IN eventID INT, 
    IN newDescription TEXT, 
    IN newDate VARCHAR(25),
    IN newStartTime VARCHAR(25), 
    IN newEndTime VARCHAR(25)
)
BEGIN
    -- Update the event
    UPDATE Event
    SET Description = newDescription, Date = newDate, Start_time = newStartTime, End_time = newEndTime
    WHERE EventID = eventID;
    
    -- Notify attendees (hypothetical)
    -- This would involve integration with a notification system
END$$
DELIMITER ;

-- Update Participant count: 
DELIMITER $$
CREATE DEFINER=`root`@`localhost` PROCEDURE `UpdateParticipantCount`(
    IN p_SessionID INT,
    IN p_NewParticipantCount INT
)
BEGIN
    DECLARE v_VenueCapacity INT;

    SELECT V.Capacity INTO v_VenueCapacity
    FROM NetworkingSession NS
    JOIN Event E ON NS.EventID = E.EventID
    JOIN Venue V ON E.VenueID = V.VenueID
    WHERE NS.SessionID = p_SessionID;

    IF p_NewParticipantCount <= v_VenueCapacity
    THEN
        UPDATE NetworkingSession
        SET Participants = p_NewParticipantCount
        WHERE SessionID = p_SessionID;

        SELECT 'Participant count updated successfully.' AS Result;
    ELSE
        SELECT 'New participant count exceeds venue capacity. Update failed.' AS Result;
    END IF;
END$$
DELIMITER ;

Views: 

-- Event View 

CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`localhost` SQL SECURITY DEFINER VIEW `univents`.`event_venue` AS 
select `e`.`EventID` AS `EventID`,
`e`.`Name` AS `EventName`,
`v`.`University` AS `University`,
`o`.`OrganizerID` AS `PrimaryOrganizerID`,
`o`.`Bio` AS `PrimaryOrganizerBio`,
coalesce(group_concat(`co`.`OrganizerID` separator ','),'None') AS `CoOrganizers`,
`v`.`Name` AS `VenueName` 
from (((((	`univents`.`event` `e` 
			join `univents`.`organisedby` `ob` on((`e`.`EventID` = `ob`.`EventID`))) 
            join `univents`.`organizer` `o` on((`ob`.`OrganizerID` = `o`.`OrganizerID`))) 
            left join `univents`.`organisedby` `coob` on(((`e`.`EventID` = `coob`.`EventID`) and (`coob`.`OrganizerID` <> `o`.`OrganizerID`)))) 
            left join `univents`.`organizer` `co` on((`coob`.`OrganizerID` = `co`.`OrganizerID`))) 
            left join `univents`.`venue` `v` on((`e`.`VenueID` = `v`.`VenueID`))) 
            group by `e`.`EventID`,`e`.`Name`,`v`.`University`,`o`.`OrganizerID`,`o`.`Bio`,`v`.`Name` 
            order by `v`.`University`,`e`.`EventID`;


-- Event popularity analysis

CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`localhost` SQL SECURITY DEFINER VIEW `univents`.`eventpopularityfeedbackanalysis` AS 
select `e`.`EventID` AS `EventID`,
`e`.`Name` AS `Name`,
count(`r`.`UserID`) AS `Registrations`,
cast(avg(`f`.`Rating`) as decimal(16,0)) AS `AverageRating`,
count(`f`.`FeedbackID`) AS `FeedbackCount` 
from ((`univents`.`event` `e` 
		left join `univents`.`registers` `r` on((`e`.`EventID` = `r`.`EventID`))) 
        left join `univents`.`feedback` `f` on((`e`.`EventID` = `f`.`EventID`))) 
        where (`f`.`EventID` is not null) group by `e`.`EventID`;

-- User details with event count 

CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`localhost` SQL SECURITY DEFINER VIEW `univents`.`userdetailswitheventcount` AS 
select `u`.`UserID` AS `UserID`,
`u`.`First_Name` AS `First_Name`,
`u`.`Last_Name` AS `Last_Name`,
`u`.`University` AS `University`,
`u`.`User_type` AS `User_type`,
`u`.`Email` AS `Email`,
count(distinct `e`.`EventID`) AS `EventCount`,
group_concat(distinct `e`.`Name` order by `e`.`Name` ASC separator ',') AS `AttendedEvents`,
max((case when (`u`.`User_type` = 'Student') then 1 else 0 end)) AS `IsStudent`,
max((case when (`u`.`User_type` = 'Faculty') then 1 else 0 end)) AS `IsFaculty`,
max((case when (`u`.`User_type` = 'Administrator') then 1 else 0 end)) AS `IsAdmin`,
max((case when (`u`.`User_type` = 'Organizer') then 1 else 0 end)) AS `IsOrganizer` 
from ((`univents`.`user` `u` 
		left join `univents`.`attends` `a` on((`u`.`UserID` = `a`.`UserID`)))
        left join `univents`.`event` `e` on((`a`.`EventID` = `e`.`EventID`))) 
        group by `u`.`UserID`,`u`.`First_Name`,`u`.`Last_Name`,`u`.`University`,`u`.`User_type`,`u`.`Email`;


-- Event feedback sentiment analysis 

CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`localhost` SQL SECURITY DEFINER VIEW `univents`.`eventfeedbacksentimentanalysis` AS 
select `f`.`EventID` AS `EventID`,
`e`.`Name` AS `EventName`,
cast(avg(`f`.`Rating`) as decimal(16,0)) AS `AverageRating`,
count(`f`.`FeedbackID`) AS `FeedbackCount`,
sum((case when (`f`.`Rating` >= 4) then 1 else 0 end)) AS `PositiveFeedbackCount`,
sum((case when ((`f`.`Rating` < 4) and (`f`.`Rating` >= 2)) then 1 else 0 end)) AS `NeutralFeedbackCount`,
sum((case when (`f`.`Rating` < 2) then 1 else 0 end)) AS `NegativeFeedbackCount`,
(case when (avg(`f`.`Rating`) >= 4) then 'Positive' when (avg(`f`.`Rating`) >= 2) then 'Neutral' else 'Negative' end) AS `Sentiment` 
from ( `univents`.`event` `e` 
		left join `univents`.`feedback` `f` on((`e`.`EventID` = `f`.`EventID`))) 
        where (`f`.`EventID` is not null) group by `f`.`EventID`,`e`.`Name`;


MONGODB code snippets: 

-- count of event venues per university 

from pymongo import MongoClient

# Requires the PyMongo package.
# https://api.mongodb.com/python/current

client = MongoClient('mongodb://localhost:27017/')
result = client['univents_nosql']['event_venue'].aggregate([
    {
        '$match': {
            'University': 'New York University'
        }
    }, {
        '$group': {
            '_id': '$University', 
            'eventCount': {
                '$sum': 1
            }
        }}
])

-- List of event venues and count of events happening at each venue at Stanford University 

from pymongo import MongoClient

# Requires the PyMongo package.
# https://api.mongodb.com/python/current

client = MongoClient('mongodb://localhost:27017/')
result = client['univents_nosql']['event_venue'].aggregate([
    {
        '$match': {
            'University': 'Stanford University'
        }
    }, {
        '$group': {
            '_id': '$VenueName', 
            'Number of Events': {
                '$sum': 1
            }
        }
    }
])  


-- Number of students per major at Northeastern University 

from pymongo import MongoClient

# Requires the PyMongo package.
# https://api.mongodb.com/python/current

client = MongoClient('mongodb://localhost:27017/')
result = client['univents_nosql']['user'].aggregate([
    {
        '$match': {
            'User_type': 'student'
        }
    }, {
        '$lookup': {
            'from': 'student', 
            'localField': 'UserID', 
            'foreignField': 'StudentID', 
            'as': 'student_data'
        }
    }, {
        '$unwind': '$student_data'
    }, {
        '$match': {
            'University': 'Northeastern University'
        }
    }, {
        '$group': {
            '_id': '$student_data.Major', 
            'StudentCount': {
                '$sum': 1
            }
        }
    }
])

-- Number of events attended by Mark 

from pymongo import MongoClient

# Requires the PyMongo package.
# https://api.mongodb.com/python/current

client = MongoClient('mongodb://localhost:27017/')
result = client['univents_nosql']['userdetailswitheventcount'].aggregate([
    {
        '$match': {
            'First_Name': 'Mark'
        }
    }, {
        '$group': {
            '_id': '$Last_Name', 
            'EventCount': {
                '$sum': '$EventCount'
            }
        }
    }
])


Python code for visualization: 

-- Python Code to plot a Bar graph between the event count from each university:

import pymysql import pandas AS pd import matplotlib.pyplot AS plt import seaborn AS sns host = 'localhost' user = 'root' password = 'Sribala22*' database = 'univents' connection = pymysql.connect(host=host,
user=user,
password=password,
database=database) cursor = connection.cursor() query = """SELECT v.University,
COUNT(DISTINCT e.EventID) AS EventCount
FROM event e
JOIN venue v
ON e.VenueID = v.VenueID
GROUP BY v.University; """ cursor.execute(query) result = cursor.fetchall() university_events = pd.DataFrame(result, columns=['University', 'EventCount']) colors = sns.color_palette("Blues_d", n_colors=len(university_events)) # Darker Blue plt.figure(figsize=(12, 6)) plt.rcParams['axes.facecolor'] = '#f0f0f0' # Light Grey ax = sns.barplot(x='EventCount', y='University', data=university_events, palette=colors) for p IN ax.patches: ax.annotate(f'{p.get_width():.0f}', (p.get_width() + 0.2, p.get_y() + p.get_height() / 2), ha='left', va='center') plt.title('University-wise Event Count') plt.xlabel('Event Count') plt.ylabel('University') plt.grid(False) plt.tight_layout() plt.show() cursor.close() connection.close()

-- Python Code to find user type distribution in different universities:

import pymysql import pandas AS pd import matplotlib.pyplot AS plt import seaborn AS sns host = 'localhost' user = 'root' password = 'Sribala22*' database = 'univents' connection = pymysql.connect(host=host,
user=user,
password=password,
database=database) cursor = connection.cursor() query = """SELECT University,
User_Type,
COUNT(UserID) AS UserCount
FROM userdetailswitheventcount
GROUP BY University, User_Type; """ cursor.execute(query) result = cursor.fetchall() user_type_distribution = pd.DataFrame(result, columns=['University', 'User_Type', 'UserCount']) plt.figure(figsize=(16, 8)) ax = sns.barplot(x='University', y='UserCount',
hue='User_Type', data=user_type_distribution) plt.title('User Type Distribution for Different Universities') plt.xlabel('University') plt.ylabel('User Count') plt.legend(title='User Type') for p IN ax.patches: ax.annotate(f'{int(p.get_height())}', (p.get_x() + p.get_width() / 2., p.get_height()), ha='center', va='center', xytext=(0, 10), textcoords='offset points') plt.xticks(rotation=45, ha='right') plt.show() cursor.close() connection.close()

-- Python Code to plot a line chart for monthly event count in the year 2024
import pymysql import pandas AS pd import matplotlib.pyplot AS plt import calendar host = 'localhost' user = 'root' password = 'Sribala22*' database = 'univents' connection = pymysql.connect(host=host,
user=user,
password=password,
database=database) sql_query = """SELECT DATE_FORMAT(e.Date,
'%Y-%m') AS Month, COUNT(e.EventID) AS EventCount
FROM event e
WHERE YEAR(e.Date) = 2024
GROUP BY Month
ORDER BY Month; """ df = pd.read_sql(sql_query, connection) df['Month'] = pd.to_datetime(df['Month']) df['Month'] = df['Month'].dt.month.map(lambda x: calendar.month_abbr[x]) plt.figure(figsize=(12, 6)) plt.plot(df['Month'], df['EventCount'], marker='o', markersize=10, linestyle='-', color='b') plt.title('Monthly Event Count in 2024') plt.xlabel('Month') plt.ylabel('Number of Events') plt.xticks(rotation=45, ha='right') # Rotate x-axis labels for better readability plt.grid(True) plt.tight_layout() plt.show() connection.close()

