# Database assignment 4

_Assignment:_ 
https://github.com/datsoftlyngby/soft2019spring-databases/blob/master/assignments/assignment5.md

## Get up and running

Following will download the data, and create a Docker container.

You will be put inside the docker container where you have to execute all of the following commands

**Start container and install dependencies to fetch the database**
```
sudo docker run --rm --name my_mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=pass1234 -d mysql
sudo docker exec -it my_mysql bash 
echo "Following is inside the container"
apt-get update
apt-get install wget p7zip-full -y
```
**Download the data** or copy the zip files into `docker cp ZIPFILES-PATH my_mysql:/workspace` and open the folder `cd /workspace` inside the container
```
cd /workspace
wget https://archive.org/download/stackexchange/askubuntu.com.7z
7z e askubuntu.com.7z 
```
**The rest will be executed inside of the mysql bash**

The flag is used to get access to the xml data when it has to be imported
```
mysql -u root -ppass1234  --local-infile
```

**Import data** 
```sql
create database stackoverflow DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;


use stackoverflow;

create table badges (
  Id INT NOT NULL PRIMARY KEY,
  UserId INT,
  Name VARCHAR(50),
  Date DATETIME
);

CREATE TABLE comments (
  Id INT NOT NULL PRIMARY KEY,
  PostId INT NOT NULL,
  Score INT NOT NULL DEFAULT 0,
  Text TEXT,
  CreationDate DATETIME,
  UserId INT NOT NULL
);

CREATE TABLE post_history (
  Id INT NOT NULL PRIMARY KEY,
  PostHistoryTypeId SMALLINT NOT NULL,
  PostId INT NOT NULL,
  RevisionGUID VARCHAR(36),
  CreationDate DATETIME,
  UserId INT NOT NULL,
  Text TEXT
);
CREATE TABLE post_links (
  Id INT NOT NULL PRIMARY KEY,
  CreationDate DATETIME DEFAULT NULL,
  PostId INT NOT NULL,
  RelatedPostId INT NOT NULL,
  LinkTypeId INT DEFAULT NULL
);


CREATE TABLE posts (
  Id INT NOT NULL PRIMARY KEY,
  PostTypeId SMALLINT,
  AcceptedAnswerId INT,
  ParentId INT,
  Score INT NULL,
  ViewCount INT NULL,
  Body text NULL,
  OwnerUserId INT NOT NULL,
  LastEditorUserId INT,
  LastEditDate DATETIME,
  LastActivityDate DATETIME,
  Title varchar(256) NOT NULL,
  Tags VARCHAR(256),
  AnswerCount INT NOT NULL DEFAULT 0,
  CommentCount INT NOT NULL DEFAULT 0,
  FavoriteCount INT NOT NULL DEFAULT 0,
  CreationDate DATETIME
);

CREATE TABLE tags (
  Id INT NOT NULL PRIMARY KEY,
  TagName VARCHAR(50) CHARACTER SET latin1 DEFAULT NULL,
  Count INT DEFAULT NULL,
  ExcerptPostId INT DEFAULT NULL,
  WikiPostId INT DEFAULT NULL
);


CREATE TABLE users (
  Id INT NOT NULL PRIMARY KEY,
  Reputation INT NOT NULL,
  CreationDate DATETIME,
  DisplayName VARCHAR(50) NULL,
  LastAccessDate  DATETIME,
  Views INT DEFAULT 0,
  WebsiteUrl VARCHAR(256) NULL,
  Location VARCHAR(256) NULL,
  AboutMe TEXT NULL,
  Age INT,
  UpVotes INT,
  DownVotes INT,
  EmailHash VARCHAR(32)
);

CREATE TABLE votes (
  Id INT NOT NULL PRIMARY KEY,
  PostId INT NOT NULL,
  VoteTypeId SMALLINT,
  CreationDate DATETIME
);

create index badges_idx_1 on badges(UserId);

create index comments_idx_1 on comments(PostId);
create index comments_idx_2 on comments(UserId);

create index post_history_idx_1 on post_history(PostId);
create index post_history_idx_2 on post_history(UserId);

create index posts_idx_1 on posts(AcceptedAnswerId);
create index posts_idx_2 on posts(ParentId);
create index posts_idx_3 on posts(OwnerUserId);
create index posts_idx_4 on posts(LastEditorUserId);

create index votes_idx_1 on votes(PostId);

SET GLOBAL local_infile = 1;
LOAD XML LOCAL INFILE 'Badges.xml' INTO TABLE badges;
LOAD XML LOCAL INFILE 'Comments.xml' INTO TABLE comments;
LOAD XML LOCAL INFILE 'PostHistory.xml' INTO TABLE post_history;
LOAD XML LOCAL INFILE 'PostLinks.xml' INTO TABLE post_links;
LOAD XML LOCAL INFILE 'Posts.xml' INTO TABLE posts;
LOAD XML LOCAL INFILE 'Tags.xml' INTO TABLE tags;
LOAD XML LOCAL INFILE 'Users.xml' INTO TABLE users;
LOAD XML LOCAL INFILE 'Votes.xml' INTO TABLE votes;

ALTER TABLE `stackoverflow`.`posts` 
ADD COLUMN `Comments` JSON NULL AFTER `CreationDate`;
alter table comments modify Id int auto_increment;
```

### Exercise 1
Write a stored procedure `denormalizeComments(postID)` that moves all comments for a post (the parameter) into a json array on the post. 

__Code for stored procedure__
```sql
DROP procedure IF EXISTS `denormalizeComments`;

DELIMITER $$
CREATE PROCEDURE `denormalizeComments` (p_postID INT)
BEGIN
update posts set Comments = (select json_arrayagg(Text) from comments where PostId = p_postID group by PostId) where Id = p_postID;
END$$

DELIMITER ;
```
And test it `call denormalizeComments(1);`, see the result with `select Id, Comments from posts where Id < 5;`

### Exercise 2
Create a trigger such that new adding new comments to a post triggers an insertion of that comment in the json array from exercise 1.

__Code for stored procedure__
```sql
DELIMITER $$
DROP TRIGGER if exists insert_post_comments$$
CREATE TRIGGER insert_post_comments
 AFTER INSERT ON comments
 FOR EACH ROW
BEGIN
 CALL denormalizeComments(NEW.PostId);
END $$
DELIMITER ;
```
Lets add it for update and delete as well to keep the data accurate
```sql
DELIMITER $$
DROP TRIGGER if exists update_post_comments$$
CREATE TRIGGER update_post_comments
 AFTER UPDATE ON comments
 FOR EACH ROW
BEGIN
 CALL denormalizeComments(NEW.PostId);
END $$

DROP TRIGGER if exists delete_post_comments$$
CREATE TRIGGER delete_post_comments
 AFTER DELETE ON comments
 FOR EACH ROW
BEGIN
 CALL denormalizeComments(OLD.PostId);
END $$
DELIMITER ;
```
And test it `insert into comments(PostId, Score, Text, CreationDate, UserId) VALUES (2, 0, "new comment", NOW(), 4);`, see the result with `select Id, Comments from posts where Id < 5;`

### Exercise 3
Rather than using a trigger, create a stored procedure to add a comment to a post - adding it both to the comment table and the json array

__Code for stored procedure__
```sql
DROP procedure IF EXISTS `addNewComment`;

DELIMITER $$
CREATE PROCEDURE `addNewComment` (p_postID INT, p_text TEXT, p_userid INT)
BEGIN
INSERT INTO comments(PostId, Score, Text, CreationDate, UserId) value (p_postID, 0, p_text, NOW(), p_userid);
END$$

DELIMITER ;
```

**no need to call denormalizeComments, it is handled by the trigger**

And test it `CALL addNewComment (3, "new comment using stored procedure", 4);`, see the result with `select Id, Comments from posts where Id < 5;`

### Exercise 4
Make a materialized view that has json objects with questions and its answeres, but no comments. Both the question and each of the answers must have the display name of the user, the text body, and the score.

todo

### Exercise 5
Using the materialized view from exercise 4, create a stored procedure with one parameter `keyword`, which returns all posts where the keyword appears at least once, and where at least two comments mention the keyword as well.

__Code for stored procedure__
```sql
DROP procedure IF EXISTS `keywordSearch`;
DELIMITER $$
create procedure keywordSearch(p_keyword TEXT)
begin
 select QuestionId, Detail
 from (
    SELECT QuestionId, Detail
    from q_and_a_cache
    where JSON_EXTRACT(Detail, "$.text") like concat("%", p_keyword, "%")
  ) as Questions
 left join (
  SELECT PostId, count(*) as noOfMentionsInComment FROM comments where Text like concat("%", p_keyword, "%") group by PostId
 ) as noOfComments on noOfComments.PostId = Questions.QuestionId
 where noOfComments.noOfMentionsInComment >= 2;
end$$

DELIMITER ;
```

And test it `call keywordSearch("bash");`



## Cleanup

Exit the shell with `exit` then `CTRL + d`

Remove the docker container with `sudo docker rm -f my_mysql`


