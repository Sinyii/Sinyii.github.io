---
layout: post
title:  "NoSQL: Neo4j & Redis"
date:   2019-05-05 20:58:36 -0400
thumbnail: "/image/graphdb/comment2.PNG"
categories: projects
---

# W4111: Introduction to Databases - NoSQL: Neo4j(GraphDB) & Redis(in-memory DB)
### Part 1- Neo4j

### 1. FanGraph Class Implementation


```python
from py2neo import data, Graph, NodeMatcher, Node, Relationship, RelationshipMatcher

import json
import uuid


class FanGraph(object):
    """
    This object provides a set of helper methods for creating and retrieving nodes and relationships from
    a Neo4j database holding information about players, teams, fans, comments and their relationships.
    """

    # Note:
    # I tend to avoid object mapping frameworks. Object mapping frameworks are fun in the beginning
    # but tend to be annoying after a while. So, I did not create types Player, Team, etc.
    #

    # Connects to the DB and sets a Graph instance variable.
    # Also creates a NodeMatcher and RelationshipMatcher, which are a py2neo framework classes.
    def __init__(self, auth=('neo4j', 'sh3907'), host='localhost', port=7687, secure=False, ):
        self._graph = Graph(secure=secure,
                            bolt=True,
                            auth=auth,
                            host=host,
                            port=port)
        self._node_matcher = NodeMatcher(self._graph)
        self._relationship_matcher = RelationshipMatcher(self._graph)

    def run_q(self, qs, args):
        """

        :param qs: Query string that may have {} slots for parameters.
        :param args: Dictionary of parameters to insert into query string.
        :return:  Result of the query, which executes as a single, standalone transaction.
        """
        try:
            tx = self._graph.begin(autocommit=False)
            result = self._graph.run(qs, args)
            return result
        except Exception as e:
            print("Run exaception = ", e)

    def run_match(self, labels=None, properties=None):
        """
        Uses a NodeMatcher to find a node matching a "template."
        :param labels: A list of labels that the node must have.
        :param properties: A dictionary of {property_name: property_value} defining the template that the
            node must match.
        :return: An array of Node objects matching the pattern.
        """
        # ut.debug_message("Labels = ", labels)
        # ut.debug_message("Properties = ", json.dumps(properties))

        if labels is not None and properties is not None:
            result = self._node_matcher.match(labels, **properties)
        elif labels is not None and properties is None:
            result = self._node_matcher.match(labels)
        elif labels is None and properties is not None:
            result = self._node_matcher.match(**properties)
        else:
            raise ValueError("Invalid request. Labels and properties cannot both be None.")

        # Convert NodeMatch data into a simple list of Nodes.
        full_result = []
        for r in result:
            full_result.append(r)

        return full_result

    def find_nodes_by_template(self, tmp):
        """

        :param tmp: A template defining the label and properties for Nodes to return. An
         example is { "label": "Fan", "template" { "last_name": "Ferguson", "first_name": "Donald" }}
        :return: A list of Nodes matching the template.
        """
        labels = tmp.get('label', None)
        props = tmp.get("template", None)
        result = self.run_match(labels=labels, properties=props)
        return result

    # Create and save a new node for  a 'Fan.'
    def create_fan(self, uni, last_name, first_name):
        """

        :param uni: uni
        :param last_name: Obvious
        :param first_name: Obvious
        :return: Node created.

        NOTE: This does not check uni uniqueness. We could do using transactions or setting a constraint
        on the database.
        """
        n = Node("Fan", uni=uni, last_name=last_name, first_name=first_name)
        tx = self._graph.begin(autocommit=True)
        tx.create(n)
        return n

    # Given a UNI, return the node for the Fan.
    def get_fan(self, uni):
        n = self.find_nodes_by_template({"label": "Fan", "template": {"uni": uni}})
        if n is not None and len(n) > 0:
            # I should throw an exception here if there is more than 1.
            n = n[0]
        else:
            n = None

        return n

    def create_player(self, player_id, last_name, first_name):
        n = Node("Player", player_id=player_id, last_name=last_name, first_name=first_name)
        tx = self._graph.begin(autocommit=True)
        tx.create(n)
        return n

    def get_player(self, player_id):
        n = self.find_nodes_by_template({"label": "Player", "template": {"player_id": player_id}})
        if n is not None and len(n) > 0:
            n = n[0]
        else:
            n = None

        return n

    def create_team(self, team_id, team_name):
        n = Node("Team", team_id=team_id, team_name=team_name)
        tx = self._graph.begin(autocommit=True)
        tx.create(n)
        return n

    def get_team(self, team_id):
        n = self.find_nodes_by_template({"label": "Team", "template": {"team_id": team_id}})
        if n is not None and len(n) > 0:
            n = n[0]
        else:
            n = None

        return n

    def create_supports(self, uni, team_id):
        """
        Create a SUPPORTS relationship from a Fan to a Team.
        :param uni: The UNI for a fan.
        :param team_id: An ID for a team.
        :return: The created SUPPORTS relationship from the Fan to the Team
        """
        f = self.get_fan(uni)
        t = self.get_team(team_id)
        r = Relationship(f, "SUPPORTS", t)
        tx = self._graph.begin(autocommit=True)
        tx.create(r)
        return r

    def get_appearance(self, player_id, team_id, year_id):
        """
        Get the information about appearances for a player and team.
        :param player_id: player_id
        :param team_id: team_id
        :param year_id: The year for getting appearances.
        :return:
        """
        try:
            # Get the Nodes at the ends of the relationship representing appearances.
            p = self.get_player(player_id)
            t = self.get_team(team_id)

            # Run a match looking for relationships of a specific type linking the nodes.
            rm = self._graph.match(nodes=[p, t], r_type="APPEARED")
            result = []

            # If there is a list of relationships.
            if rm is not None:
                for r in rm:

                    # The type will be a class APPEARED() because of the OO mapping.
                    node_type = type(r).__name__
                    year = r['year']

                    # If the type and year are correct, add to result
                    if node_type == "APPEARED" and (year == year_id or year_id is None):
                        result.append(r)

                return result
            else:
                return None
        except Exception as e:
            print("get_appearance: Exception e = ", e)
            raise e

    # Create an APPEARED relationship from a player to a Team
    def create_appearance_all(self, player_id, team_id, year, games):
        """

        :param player_id: O
        :param team_id:
        :param year:
        :param games:
        :return:
        """
        try:
            tx = self._graph.begin(autocommit=False)
            q = "match (n:Player {player_id: '" + player_id + "'}), " + \
                "(t:Team {team_id: '" + team_id + "'}) " + \
                "create (n)-[r:APPEARED { games: " + str(games) + ", year : " + str(year) + \
                "}]->(t)"
            result = self._graph.run(q)
            tx.commit()
        except Exception as e:
            print("create_appearances: exception = ", e)

    # Create a FOLLOWS relationship from a Fan to another Fan.
    def create_follows(self, follower, followed):
        f = self.get_fan(follower)
        t = self.get_fan(followed)
        r = Relationship(f, "FOLLOWS", t)
        tx = self._graph.begin(autocommit=True)
        tx.create(r)

    # ----- sh3907 implementation begins----- #
    def get_comment(self, comment_id):
        """

        :param comment_id: Comment ID
        :return: Comment
        """
        # create a matcher
        matcher = NodeMatcher(self._graph)

        # connect fan node to comment node
        comment_n = matcher.match("Comment", comment_id=comment_id).first()

        # return type: Node
        return comment_n

    def create_comment(self, uni, comment, team_id=None, player_id=None):
        """
        Creates a comment
        :param uni: The UNI for the Fan making the comment.
        :param comment: A simple string.
        :param team_id: A valid team ID or None. team_id and player_id cannot BOTH be None.
        :param player_id: A valid player ID or None
        :return: The Node representing the comment.
        """
        try:
            tx = self._graph.begin(autocommit=False)
            # create a random id for comment
            cid = uuid.uuid4()

            # create comment node
            comment_n = Node("Comment", comment=comment, comment_id=str(cid))
            tx.create(comment_n)
            print(comment_n)

            # create a matcher
            matcher = NodeMatcher(self._graph)

            # connect fan node to comment node
            fan_n = matcher.match("Fan", uni=uni).first()
            rela = Relationship(fan_n, "COMMENT_BY", comment_n)
            tx.create(rela)

            # connect comment node to team node
            if team_id:
                team_n = matcher.match("Team", team_id=team_id).first()
                rela = Relationship(comment_n, "COMMENT_ON", team_n)
                tx.create(rela)

            # connect comment node to player node
            if player_id:
                player_n = matcher.match("Player", player_id=player_id).first()
                rela = Relationship(comment_n, "COMMENT_ON", player_n)
                tx.create(rela)

            tx.commit()

        except Exception as e:
            print("create_comment: exception = ", e)

        # return type: Node
        return comment_n

    def create_sub_comment(self, uni, origin_comment_id, comment):
        """
        Create a sub-comment (response to a comment or response) and links with parent in thread.
        :param uni: ID of the Fan making the comment.
        :param origin_comment_id: Id of the comment to which this is a response.
        :param comment: Comment string
        :return: Created comment.
        """
        try:
            tx = self._graph.begin(autocommit=False)

            # create a random id for comment
            cid = uuid.uuid4()

            # create comment node
            comment_n = Node("Comment", comment=comment, comment_id=str(cid))
            tx.create(comment_n)

            # create a matcher
            matcher = NodeMatcher(self._graph)

            # connect fan node to comment node
            fan_n = matcher.match("Fan", uni=uni).first()
            rela = Relationship(fan_n, "COMMENT_BY", comment_n)
            tx.create(rela)

            # connect comment node to original comment node
            origin_comment_n = matcher.match("Comment", comment_id=origin_comment_id).first()
            rela = Relationship(comment_n, "COMMENT_ON", origin_comment_n)
            tx.create(rela)

            tx.commit()

        except Exception as e:
            print("create_comment: exception = ", e)

        # return type: Node
        return comment_n

    def get_sub_comments(self, comment_id):
        """

        :param comment_id: The unique ID of the comment.
        :return: The sub-comments.
        """
        # create a matcher
        matcher = NodeMatcher(self._graph)

        comment_n = matcher.match("Comment", comment_id=comment_id).first()

        sub_comments = RelationshipMatcher(self._graph).match((None, comment_n), "COMMENT_ON")

        # return type: RelationshipMatch
        return sub_comments

    def get_player_comments(self, player_id):
        """
        Gets all of the comments associated with a player, Also returns the Nodes for people making the comments.
        :param player_id: ID of the player.
        :return: Graph containing comment, comment streams and commenters.
        """
        tx = self._graph.begin(autocommit=False)

        q = "MATCH (n:Player {player_id:'" + player_id + "'}) MATCH (c:Comment)-[r:COMMENT_ON]->(n) RETURN c;"

        result = self._graph.run(q)

        tx.commit()

        # return type: Cursor
        return result

    def get_team_comments(self, team_id):
        """
        Gets all of the comments associated with a team.  Also returns the Nodes for people making the comments.
        :param player_id: ID of the team.
        :return: Graph containing comment, comment streams and commenters.
        """
        # create a matcher
        matcher = NodeMatcher(self._graph)

        team_n = matcher.match("Team", team_id=team_id).first()

        comments = RelationshipMatcher(self._graph).match((None, team_n), "COMMENT_ON")

        # return type: RelationshipMatch
        return comments

    def get_players_by_team(self, team_id, yearid):
        """

        :param team_id: The ID of a team.
        :param yearid: A year.
        :return: Returns the players who played for the team in the year.
        """
        tx = self._graph.begin(autocommit=False)

        q = "MATCH (t:Team {team_id:'" + team_id + "'}) MATCH (p:Player)-[r:APPEARED]->(n) WHERE r.year = " + yearid + " RETURN p;"

        result = self._graph.run(q)

        tx.commit()

        # return type: Cursor
        return result
    # ----- sh3907 implementation ends----- #

```

### 2. Create a Fan graph and connecting with mysql db


```python
import fan_graph as f
import json
import pymysql
import pandas as pd

pd.set_option('display.width', 132)

# connect with mysql database
cnx = pymysql.connect(host='localhost',
                             user='dbuser',
                             password='dbuserdbuser',
                             db='lahman2017',
                             charset='utf8mb4',
                             cursorclass=pymysql.cursors.DictCursor)

# connect with neo4j database
fg = f.FanGraph(auth=('neo4j', 'neo4j'),
                  host="localhost",
                  port=7687,
                  secure=False)
print("fg = ", fg)
```

    fg =  <fan_graph.FanGraph object at 0x0000025CDFDFBF98>
    

### 3-1. Load players from laman2017.people


```python
def load_players():

    """
    Load a subset of the player data. We only load a subset of the columns and the rows.
    """

    q = "SELECT playerID, nameLast, nameFirst FROM People where  " + \
        "exists (select * from appearances where appearances.playerID = " + \
        " people.playerID and yearID >= 2010)"


    curs = cnx.cursor()
    curs.execute(q)
    
    r = curs.fetchone()
    cnt = 0

    # Loop until we are out of rows.
    while r is not None:
        print(r)
        cnt += 1
        if r is not None:
            p = fg.create_player(player_id=r['playerID'], last_name=r['nameLast'], first_name=r['nameFirst'])
            print("Created player = ", p)

        r = curs.fetchone()

    print("Loaded ", cnt, "records.")

```

### 3-2. Load teams from laman2017.teams


```python
def load_teams():
    q = "SELECT distinct teamid, name from teams where yearid >= 2000"

    curs = cnx.cursor()
    curs.execute(q)

    cnt = 0
    r = curs.fetchone()
    while r is not None:

        print(r)
        cnt += 1

        p = fg.create_team(team_id=r['teamid'], team_name=r['name'])
        print("Created team = ", p)

        r = curs.fetchone()

    print("Loaded ", cnt, "records.")
```

### 3-3. Load appearances from laman2017.appearances


```python
def load_appearances():

    q = \
    "SELECT distinct playerid, teamid, yearid, g_all as games " + \
    " from appearances where yearid >= 2010"

    curs = cnx.cursor()
    curs.execute(q)

    r = curs.fetchone()
    cnt = 0
    while r is not None:
        print(r)
        cnt += 1

        if r is not None:
            try:
                p = fg.create_appearance_all(team_id=r['teamid'], player_id=r['playerid'], \
                                        games=r['games'], year=r['yearid'])
                print("Created appearances = ", json.dumps(p))
            except Exception as e:
                print("Could not create.")
        r = curs.fetchone()

    print("Loaded ", cnt, "records.")
```

#### Execute.


```python
load_players()
load_teams()
load_appearances()
```

![player_team](/image/graphdb/player_team.PNG)

### 4. Load Fans data


```python
def load_fans():
    fg.create_fan(uni="js1", last_name="Smith", first_name="John")
    fg.create_fan(uni="ja1", last_name="Adams", first_name="John")
    fg.create_fan(uni="tj1", last_name="Jefferson", first_name="Thomas")
    fg.create_fan(uni="gw1", last_name="Washing", first_name="George")
    fg.create_fan(uni="jm1", last_name="Monroe", first_name="James")
    fg.create_fan(uni="al1", last_name="Lincoln", first_name="Abraham")


def load_follows_fans():
    fg.create_follows(follower="gw1", followed="js1")
    fg.create_follows(follower="tj1", followed="gw1")
    fg.create_follows(follower="ja1", followed="gw1")
    fg.create_follows(follower="jm1", followed="gw1")
    fg.create_follows(follower="al1", followed="jm1")


def create_supports():
    fg.create_supports("gw1", "WAS")
    fg.create_supports("ja1", "BOS")
    fg.create_supports("tj1", "WAS")
    fg.create_supports("jm1", "NYA")
    fg.create_supports("al1", "CHA")
    fg.create_supports("al1", "CHN")
```

#### Execute.


```python
load_fans()
load_follows_fans()
create_supports()
```

![fans1](/image/graphdb/fans1.PNG)

#### Change display style

![fans2](/image/graphdb/fans2.PNG)

### 5. Testing function implementations of FanGraph class


#### Connection setting


```python
import fan_graph as f
import json
import pymysql

# connect with mysql database
cnx = pymysql.connect(host='localhost',
                             user='dbuser',
                             password='dbuserdbuser',
                             db='lahman2017',
                             charset='utf8mb4',
                             cursorclass=pymysql.cursors.DictCursor)

# connect with neo4j database
fg = f.FanGraph(auth=('neo4j', 'neo4j'),
                  host="localhost",
                  port=7687,
                  secure=False)
print("fg = ", fg)
```

    fg =  <fan_graph.FanGraph object at 0x0000025CDFC93898>
    

#### Helper function for displaying comment node.


```python
import pandas as pd

pd.set_option('display.width', 132)

def betterDisplay(c):
    result = {}
    for r in c:
        s = r.start_node
        e = r.end_node
        cid = s['comment_id']
        this_c = result.get(cid, None)
        if this_c is None:
            this_c = {}
            this_c['comment'] = s['comment']

        if "Team" in e.labels:
            this_c['team'] = e['team_name']
        elif "Player" in e.labels:
            this_c['player'] = e['last_name'] + "," + e['first_name']

        result[cid] = this_c

    final_result = []
    for k, v in result.items():
        new_r = {}
        new_r["id"] = k
        new_r = {**new_r, **v}
        final_result.append(new_r)

    df2 = pd.DataFrame(final_result)
    print(df2)
```

### 5-1. create_comment(self, uni, comment, team_id=None, player_id=None)


```python
comment_n = fg.create_comment("ja1", "Awesome", team_id="BOS", player_id="pedrodu01")
cid = comment_n['comment_id']
```

    (_42133:Comment {comment: 'Awesome', comment_id: 'f8074b43-271f-4fde-ae29-716207f76fde'})
    

![comment](/image/graphdb/comment1.PNG)

### 5-2. get_comment(self, comment_id)


```python
comment_n = fg.get_comment(cid)
print(comment_n)
print(comment_n['comment'])
```

    (_42133:Comment {comment: 'Awesome', comment_id: 'f8074b43-271f-4fde-ae29-716207f76fde'})
    Awesome
    

### 5-3. create_sub_comment(self, uni, origin_comment_id, comment)


```python
sub_comment_n = fg.create_sub_comment("tj1", cid, "I don't agree, they are so so.")
sub_cid = sub_comment_n['comment_id']
print(sub_comment_n)
print(sub_comment_n['comment'])
```

    (_29811:Comment {comment: "I don't agree, they are so so.", comment_id: 'cbf5ca6c-2202-4d87-ad0e-140379b1ed4e'})
    I don't agree, they are so so.
    

![sub-comment](/image/graphdb/sub_comment.PNG)

### 5-4. get_sub_comments(self, comment_id)


```python
comment_rm = fg.get_sub_comments(cid) # get cid's sub comments, return type: RelationshipMatch
print(comment_rm)
print(betterDisplay(comment_rm))
```

    <py2neo.matching.RelationshipMatch object at 0x0000025CF1DB4710>
                              comment                                    id
    0  I don't agree, they are so so.  cbf5ca6c-2202-4d87-ad0e-140379b1ed4e
    None
    

### 5-5. get_player_comments(self, player_id)


```python
# create another comment
fg.create_comment("gw1", "Totally", "BOS", "pedrodu01")

comment_ns = fg.get_player_comments('pedrodu01')
for comment in comment_ns:
    print(comment)
```

    (_29812:Comment {comment: 'Totally', comment_id: 'db8d18db-0d5c-43ff-8a29-100b5e777658'})
    <Record c=(_29812:Comment {comment: 'Totally', comment_id: 'db8d18db-0d5c-43ff-8a29-100b5e777658'})>
    <Record c=(_42133:Comment {comment: 'Awesome', comment_id: 'f8074b43-271f-4fde-ae29-716207f76fde'})>
    

![comment2](/image/graphdb/comment2.PNG)

### 5-6. get_team_comments(self, team_id)


```python
comment_rm = fg.get_team_comments('BOS')
print(comment_ns)
print(betterDisplay(comment_rm))
```

    <py2neo.database.Cursor object at 0x0000025CEFE2B320>
       comment                                    id            team
    0  Totally  db8d18db-0d5c-43ff-8a29-100b5e777658  Boston Red Sox
    1  Awesome  f8074b43-271f-4fde-ae29-716207f76fde  Boston Red Sox
    None
    

### 5-7. get_players_by_team(self, team_id, yearid)


```python
players = fg.get_players_by_team('BOS', '2013') # return a cursor instance
for player in players:
    print(player)
```
    

#### NOTE
#### delete all nodes in neo4j

MATCH (n) DETACH DELETE n;

### Part 2- Redis

#### Main functions


```python
import redis
import json
from redis_helper import MysqlHelpers
from urllib.parse import urlencode
"""
Connect to local Redis server. StrictRedis complies more closely with standard than
simple Redis client in this package. decode_responses specifies whether or not to convert
bytes to UTF8 character string or treat as raw bytes, e.g. an image, audio stream, etc.
"""


class RedisFunctions:

    def __init__(self):
        self.r = redis.StrictRedis(
            host='localhost',
            port=6379,
            charset="utf-8", decode_responses=True)

        if self.r:
            print("Connected with Redis.")

        self.db = MysqlHelpers()

    # ----- sh3907 implementation begins----- #
    # requirement function 1   
    def retrieve_by_template(self, table, template, fields=None, limit=None, offset=None, order_by=None, use_cache=False):

        in_cache = None

        if use_cache:

            in_cache = self.check_cache(table, template, fields, limit, offset, order_by)

            if in_cache:

                print("Check cache returned: ", json.dumps(json.loads(in_cache), indent=2))
                print("CACHE HIT")

        if not use_cache or not in_cache:
            q_result = self.db.find_by_template(table, template, fields, limit, offset, order_by)

            if q_result:
                print("Retrieve data from mysql database: ", json.dumps(q_result, indent=2))
                self.add_to_cache(table, template, fields, limit, offset, order_by, q_result)
                
                if use_cache:
                    print("CACHE MISS")
            else:
                print("Data not found from the database. Please modify your input")
    
    
    # requirement function 2      
    def retrieve_from_cache(self, key):
        """
        :param key: A valid Redis key.
        :return: The "map object" associated with the key.
        """
        result = self.r.get(key)
        return result
    
    
    # requirement function 3
    def add_to_cache(self, table, tmp, fields, limit, offset, order_by, q_result):
        key = self.generate_key(table, tmp, fields, limit, offset, order_by)
        v = json.dumps(q_result)
        save_result = self.r.set(key, v)

        if save_result:
            print("Successful add key={} data into cache".format(key))
        else:
            print("Fail to add into cache.")

            
    def generate_key(self, table, template, fields, limit, offset, order_by):
        tmp = template.copy()
        if fields:
            tmp['field'] = ",".join(fields)
        if limit:
            tmp['limit'] = limit
        if offset:
            tmp['offset'] = offset
        if order_by:
            tmp['order_by'] = order_by
        k = urlencode(tmp)
        k = table + "/" + k

        return k


    def check_cache(self, table, tmp, fields, limit, offset, order_by):
        key = self.generate_key(table, tmp, fields, limit, offset, order_by)
        result = self.retrieve_from_cache(key)
        if result:
            return result
        else:
            return None


    def delete_keys(self):
        keys = self.get_keys()
        self.r.delete(*keys)
# ----- sh3907 implementation ends----- #

    def get_keys(self):
        result = self.r.keys()
        return result

```

#### Helper functions


```python
'''
HW4 will add checking the Redis cache to a simple, standalone  𝑟𝑒𝑡𝑟𝑖𝑒𝑣𝑒_𝑏𝑦_𝑡𝑒𝑚𝑝𝑙𝑎𝑡𝑒()  function.
Write a function  𝑎𝑑𝑑_𝑡𝑜_𝑐𝑎𝑐ℎ𝑒()  that adds a retrieve result to the cache.
Write a function  𝑟𝑒𝑡𝑟𝑖𝑒𝑣𝑒_𝑓𝑟𝑜𝑚_𝑐𝑎𝑐ℎ𝑒()  that checks the cache and returns the value if present.
Modify  𝑟𝑒𝑡𝑟𝑖𝑒𝑣𝑒_𝑏𝑦_𝑡𝑒𝑚𝑝𝑙𝑎𝑡𝑒()  to use the cache.
Check and return if cached.
Call DB and add to cache if not cached.
'''

import pymysql.cursors

class MysqlHelpers:

    def __init__(self):
        self.db_schema = None                                # Schema containing accessed data
        self.cnx = None                                      # DB connection to use for accessing the data.
        self.key_delimiter = '_'                             # This should probably be a config option.
        self.set_config()


    def get_new_connection(self, params=None):
        if not params:
            params = self.default_db_params

        cnx = pymysql.connect(
            host=params["dbhost"],
            port=params["port"],
            user=params["dbuser"],
            password=params["dbpw"],
            db=params["dbname"],
            charset=params["charset"],
            cursorclass=params["cursorClass"])
        return cnx

    def set_config(self):
        """
        Creates the DB connection and sets the global variables.

        :param cfg: Application configuration data.
        :return: None
        """



        db_params = {
            "dbhost": "localhost",
            "port": 3306,
            "dbname": "lahman2017",
            "dbuser": "dbuser",
            "dbpw": "dbuserdbuser",
            "cursorClass": pymysql.cursors.DictCursor,
            "charset": 'utf8mb4'
        }

        self.db_schema = "lahman2017"

        self.cnx = self.get_new_connection(db_params)

        print("Mysql Connection Object: ", self.cnx)
        return self.cnx


    # Given one of our magic templates, forms a WHERE clause.
    # { a: b, c: d } --> WHERE a=b and c=d. Currently treats everything as a string.
    # We can fix this by using PyMySQL connector query templates.
    def templateToWhereClause(self, t):
        s = ""
        for k,v in t.items():
            if s != "":
                s += " AND "
            s += k + "=\'" + v + "\'"

        if s != "":
            s = "WHERE " + s + ";"

        return s


    def run_q(self, cnx, q, args, fetch=False, commit=True):
        """
        :param cnx: The database connection to use.
        :param q: The query string to run.
        :param args: Parameters to insert into query template if q is a template.
        :param fetch: True if this query produces a result and the function should perform and return fetchall()
        :return:
        """
        #debug_message("run_q: q = " + q)
        #ut.debug_message("Q = " + q)
        #ut.debug_message("Args = ", args)

        result = None

        try:
            cursor = cnx.cursor()
            result = cursor.execute(q, args)
            if fetch:
                result = cursor.fetchall()
            if commit:
                cnx.commit()
        except Exception as original_e:
            #print("dffutils.run_q got exception = ", original_e)
            raise(original_e)

        return result

    def find_by_template(self, table, t, fields=None, limit=None, offset=None, orderBy=None):

        if t is not None:
            w = self.templateToWhereClause(t)
        else:
            w = ""

        if orderBy is not None:
            o = "order by " + ",".join(orderBy['fields']) + " " + orderBy['direction'] + " "
        else:
            o = ""

        if limit is not None:
            w += " LIMIT " + str(limit)
        if offset is not None:
            w += " OFFSET " + str(offset)

        if fields is None:
            fields = " * "
        else:
            fields = " " + ",".join(fields) + " "

        #cursor = self.cnx.cursor()
        q = "SELECT " + fields + " FROM " + table + " " + w + ";"

        r = self.run_q(self.cnx, q, None, fetch=True, commit=True)

        return r
```

### Testing


```python
r = RedisFunctions()
```

    Connected with Redis.
    Mysql Connection Object:  <pymysql.connections.Connection object at 0x0000025CF28D8550>
    

##### Retrieve_by_template



```python
tmp = { "nameLast": "Williams", "birthState": "CA"}
fields = ["playerID", "nameLast", "nameFirst", "throws"]
resource = "People"

r.retrieve_by_template(table=resource, template=tmp, fields=fields, use_cache=True)
```

    Retrieve data from mysql database:  [
      {
        "playerID": "willibe01",
        "nameLast": "Williams",
        "nameFirst": "Bernie",
        "throws": "R"
      },
      {
        "playerID": "willido02",
        "nameLast": "Williams",
        "nameFirst": "Don",
        "throws": "R"
      },
      {
        "playerID": "williji03",
        "nameLast": "Williams",
        "nameFirst": "Jimy",
        "throws": "R"
      },
      {
        "playerID": "willike02",
        "nameLast": "Williams",
        "nameFirst": "Ken",
        "throws": "R"
      },
      {
        "playerID": "willima04",
        "nameLast": "Williams",
        "nameFirst": "Matt",
        "throws": "R"
      },
      {
        "playerID": "willimi02",
        "nameLast": "Williams",
        "nameFirst": "Mitch",
        "throws": "L"
      },
      {
        "playerID": "williri02",
        "nameLast": "Williams",
        "nameFirst": "Rinaldo",
        "throws": "R"
      },
      {
        "playerID": "williri03",
        "nameLast": "Williams",
        "nameFirst": "Rick",
        "throws": "R"
      },
      {
        "playerID": "willish01",
        "nameLast": "Williams",
        "nameFirst": "Shad",
        "throws": "R"
      },
      {
        "playerID": "willite01",
        "nameLast": "Williams",
        "nameFirst": "Ted",
        "throws": "R"
      },
      {
        "playerID": "willitr01",
        "nameLast": "Williams",
        "nameFirst": "Trevor",
        "throws": "R"
      }
    ]
    Successful add key=People/nameLast=Williams&birthState=CA&field=playerID%2CnameLast%2CnameFirst%2Cthrows data into cache
    CACHE MISS
    

##### Retrieve same template again


```python
r.retrieve_by_template(table=resource, template=tmp, fields=fields, use_cache=True)
```

    Check cache returned:  [
      {
        "playerID": "willibe01",
        "nameLast": "Williams",
        "nameFirst": "Bernie",
        "throws": "R"
      },
      {
        "playerID": "willido02",
        "nameLast": "Williams",
        "nameFirst": "Don",
        "throws": "R"
      },
      {
        "playerID": "williji03",
        "nameLast": "Williams",
        "nameFirst": "Jimy",
        "throws": "R"
      },
      {
        "playerID": "willike02",
        "nameLast": "Williams",
        "nameFirst": "Ken",
        "throws": "R"
      },
      {
        "playerID": "willima04",
        "nameLast": "Williams",
        "nameFirst": "Matt",
        "throws": "R"
      },
      {
        "playerID": "willimi02",
        "nameLast": "Williams",
        "nameFirst": "Mitch",
        "throws": "L"
      },
      {
        "playerID": "williri02",
        "nameLast": "Williams",
        "nameFirst": "Rinaldo",
        "throws": "R"
      },
      {
        "playerID": "williri03",
        "nameLast": "Williams",
        "nameFirst": "Rick",
        "throws": "R"
      },
      {
        "playerID": "willish01",
        "nameLast": "Williams",
        "nameFirst": "Shad",
        "throws": "R"
      },
      {
        "playerID": "willite01",
        "nameLast": "Williams",
        "nameFirst": "Ted",
        "throws": "R"
      },
      {
        "playerID": "willitr01",
        "nameLast": "Williams",
        "nameFirst": "Trevor",
        "throws": "R"
      }
    ]
    CACHE HIT
    

##### Retrieve other template & not using cache


```python
tmp = { "nameLast": "Avery", "birthState": "GA"}
fields = ["playerID", "nameLast", "nameFirst", "throws"]
resource = "People"

r.retrieve_by_template(table=resource, template=tmp, fields=fields, use_cache=False)
```

    Retrieve data from mysql database:  [
      {
        "playerID": "averyxa01",
        "nameLast": "Avery",
        "nameFirst": "Xavier",
        "throws": "L"
      }
    ]
    Successful add key=People/nameLast=Avery&birthState=GA&field=playerID%2CnameLast%2CnameFirst%2Cthrows data into cache
    

##### Get all keys in Redis DB


```python
keys = r.get_keys()
print(keys)
```

    ['People/nameLast=Avery&birthState=GA&field=playerID%2CnameLast%2CnameFirst%2Cthrows', 'People/nameLast=Williams&birthState=CA&field=playerID%2CnameLast%2CnameFirst%2Cthrows']
    

##### Use key to retrieve from cache


```python
result = r.retrieve_from_cache(keys[1])
print(json.dumps(json.loads(result), indent=2))
```

    [
      {
        "playerID": "willibe01",
        "nameLast": "Williams",
        "nameFirst": "Bernie",
        "throws": "R"
      },
      {
        "playerID": "willido02",
        "nameLast": "Williams",
        "nameFirst": "Don",
        "throws": "R"
      },
      {
        "playerID": "williji03",
        "nameLast": "Williams",
        "nameFirst": "Jimy",
        "throws": "R"
      },
      {
        "playerID": "willike02",
        "nameLast": "Williams",
        "nameFirst": "Ken",
        "throws": "R"
      },
      {
        "playerID": "willima04",
        "nameLast": "Williams",
        "nameFirst": "Matt",
        "throws": "R"
      },
      {
        "playerID": "willimi02",
        "nameLast": "Williams",
        "nameFirst": "Mitch",
        "throws": "L"
      },
      {
        "playerID": "williri02",
        "nameLast": "Williams",
        "nameFirst": "Rinaldo",
        "throws": "R"
      },
      {
        "playerID": "williri03",
        "nameLast": "Williams",
        "nameFirst": "Rick",
        "throws": "R"
      },
      {
        "playerID": "willish01",
        "nameLast": "Williams",
        "nameFirst": "Shad",
        "throws": "R"
      },
      {
        "playerID": "willite01",
        "nameLast": "Williams",
        "nameFirst": "Ted",
        "throws": "R"
      },
      {
        "playerID": "willitr01",
        "nameLast": "Williams",
        "nameFirst": "Trevor",
        "throws": "R"
      }
    ]
    

##### Delete all keys


```python
r.delete_keys()
```


```python
print(r.get_keys())
```

    []
    
