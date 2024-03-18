# Big-Data

Running Postgres Three Node Cluster
Open virtual machine and terminal
bd
cd docker
ls
​
If the defined cluster is not found then:
And you have changes in your local machine
git status
git add.
git commit -m "comment"
​
If you don’t have any changes in your local machine then:
git pull origin main
git merge origin/main
​
Then again check for repository
ls
cd PostgresCluster
ls -la
cat docker-compose.yaml
docker compose up -d
​
Open browser and connect to localhost:6050
username :admin@admin.com
Password  :admin
Adding Servers
Click on add servers or right click on server>register>servers
Under general tab fill Name
Under connection tab fill host name, username and password.(check the yaml file in terminal)
Simple Sharding
bd
cd docker
ls
cd PostgresCluster

#To check if any node is running
docker ps

#To stop running nodes
docker compose stop

#To start nodes
docker compose up
​
Open browser and connect to localhost:6050
username :admin@admin.com
Password  :admin
Check there are three servers node1, node2, node3
Creating table node2
-- create a schema for our project
DROP SCHEMA node2_schema CASCADE;
CREATE SCHEMA node2_schema;

-- for information purposes only, show all schemas
SHOW search_path;

-- set the search path to node2_schema 
SET search_path TO node2_schema;

-- create table foo. This table will be created in the node2_schema
CREATE TABLE foo (
    a int PRIMARY KEY,
    b VARCHAR(100)
);

-- The following will drop the schema. You may want to do
-- this after testing to 'clean things up'
-- DROP SCHEMA node2_schema CASCADE;
​
Create wrapper node 1
-- this is run on postgres_node1
--Uncomment the following line to enable postgres_fdw. you only need to do this once
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

DROP SCHEMA IF EXISTS node1_schema CASCADE;
CREATE SCHEMA node1_schema;

CREATE SERVER IF NOT EXISTS node2 FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'postgres_node2', dbname 'shard2');

CREATE USER MAPPING IF NOT EXISTS FOR user1 SERVER node2
    OPTIONS (user 'user2', password 'password2');
	
IMPORT FOREIGN SCHEMA node2_schema LIMIT TO (foo)
    FROM SERVER node2 INTO node1_schema;
​
Insert data 
-- this is run on postgres_node1
-- set the search path to node1_schema 
SET search_path TO node1_schema;

-- run the following on postgres_node1
INSERT INTO foo (a) VALUES (1);
INSERT INTO foo (a) VALUES (2);
INSERT INTO foo (a) VALUES (3);
INSERT INTO foo (a) VALUES (4);
INSERT INTO foo (a) VALUES (5);
INSERT INTO foo (a) VALUES (6);

-- run the following on postgres_node1
SELECT * FROM node1_schema.foo;

-- run the following on postgres_node2
--SELECT * FROM node2_schema.foo;
​
Sharding with Partitioning
Node 2
-- NOTE: If you are rerunning this SQL script, and node1_schema exists on node one you must first drop that
-- schema on node1. Failure to do this will result in the following script seemingly runs OK, but the
-- tables will not be created. 


-- create a schema for our project
DROP SCHEMA IF EXISTS sharding_schema CASCADE;
CREATE SCHEMA sharding_schema;
SET search_path = sharding_schema;

CREATE TABLE temps_2023 (
    reading_date date,
    city VARCHAR(100),
    temperature DECIMAL
);

CREATE TABLE temps_2021 (
    reading_date date,
    city VARCHAR(100),
    temperature DECIMAL
);
​
Node 3
-- NOTE: If you are rerunning this SQL script, and node1_schema exists on node one you must first drop that
-- schema on node1. Failure to do this will result in the following script seemingly runs OK, but the
-- tables will not be created. 

-- create a schema for our project
DROP SCHEMA IF EXISTS sharding_schema CASCADE;
CREATE SCHEMA sharding_schema;
SET search_path = sharding_schema;

CREATE TABLE temps_2022 (
    reading_date date,
    city VARCHAR(100),
    temperature DECIMAL
);
​
Node 1
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

DROP SCHEMA IF EXISTS sharding_schema CASCADE;
CREATE SCHEMA sharding_schema;
SET search_path = sharding_schema;


-- create master partitioned table
CREATE TABLE IF NOT EXISTS temperatures (
    reading_date date,
    city VARCHAR(100),
    temperature DECIMAL
) PARTITION BY RANGE (reading_date);

CREATE TABLE temps_default 
   PARTITION OF temperatures DEFAULT;
   
-- create servers 
CREATE SERVER IF NOT EXISTS node2 FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'postgres_node2', dbname 'shard2');

CREATE SERVER IF NOT EXISTS node3 FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'postgres_node3', dbname 'shard3');
	
-- create user mappings
CREATE USER MAPPING IF NOT EXISTS FOR user1 SERVER node2
    OPTIONS (user 'user2', password 'password2');

CREATE USER MAPPING IF NOT EXISTS FOR user1 SERVER node3
    OPTIONS (user 'user3', password 'password3');


CREATE FOREIGN TABLE temps_2021
    PARTITION OF temperatures
    FOR VALUES FROM ('2021-01-01') TO ('2022-01-01')
    SERVER node2;

CREATE FOREIGN TABLE temps_2022
    PARTITION OF temperatures
    FOR VALUES FROM ('2022-01-01') TO ('2023-01-01')
    SERVER node3;

CREATE FOREIGN TABLE temps_2023
    PARTITION OF temperatures
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01')
    SERVER node2;
​
Insert
INSERT INTO temperatures (reading_date, city, temperature) VALUES ('2023-06-15', 'Tampa', 82);
INSERT INTO temperatures (reading_date, city, temperature) VALUES ('2021-06-15', 'Tampa', 80);
INSERT INTO temperatures (reading_date, city, temperature) VALUES ('2022-06-15', 'Tampa', 85);
INSERT INTO temperatures (reading_date, city, temperature) VALUES ('2023-07-15', 'Tampa', 83);
INSERT INTO temperatures (reading_date, city, temperature) VALUES ('2021-07-15', 'Tampa', 90);
INSERT INTO temperatures (reading_date, city, temperature) VALUES ('2022-07-15', 'Tampa', 88);
INSERT INTO temperatures (reading_date, city, temperature) VALUES ('2019-06-15', 'Tampa', 82);
INSERT INTO temperatures (reading_date, city, temperature) VALUES ('2019-07-15', 'Tampa', 83);
​
Query
-- uncomment one, as later selects will overwrite the screen output
SELECT * FROM temperatures WHERE reading_date BETWEEN '2019-01-01' AND '2023-12-31';

--UPDATE temperatures SET temp = 90 WHERE reading_date='2019-06-15';

--SELECT * FROM temperatures WHERE reading_date BETWEEN '2019-01-01' AND '2023-12-31';
