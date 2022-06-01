# SQL, Pandas or Both - Analysing the UK Electoral System
## Pandas is great for analysing and plotting data but should you store your data in a database and select it with SQL. Let's take a look at some common operations using Pandas and SQL and see how they compare

![The elections table](https://github.com/alanjones2/Alan-Jones-article-code/raw/master/sqlpandas/images/20211215_123011.jpg)

I'm not an SQL expert. In fact, to be perfectly honest, it's one of the two things that that I have tried to avoid for most of my professional life (the other one is Visual Basic). But there are those who scoff at the idea of using R or Python for data analysis because... well, that's what SQL was designed for.

Certainly SQL has been around for a while - 50 years or so - and is still the de facto query language in the relational database world. Popular products like SQLite, MySQL, MariaDB and Postgresql, are all SQL based, as are the high end enterprise systems, Oracle and Microsoft SQL Server.

So, if you are a Pythonista doing data analysis, which should you use, Pandas or SQL?

I wanted to do a little analysis on some UK election data which required me to do a number of fairly common and straightforward analysis tasks. Initially, I wanted to work out how it is that the UK government has a large majority in Parliament and yet only secured a relatively small percentage of the popular vote.

 I decided to use both Pandas and SQL to see which I was happier with. I'll describe what I did and how and you can decide for yourself which approach you think is better (or, indeed, how they could be improved).

To do the analysis we'll use data from the UK 2019 General Election (the last one), and while we compare SQL and Pandas, we can learn a little bit about how democracy works in Britain, too.

There are two files, a CSV file (to be used by Pandas) and a SQLite database - both contain exactly the same data. The data are a matter of public record and were derived from election results held by the House of Commons Library. This data may be used freely under the Open Parliament Licence 3.0 and my abridged version of it is freely downloadable from my Github repo.

My interest here is to see just how representative the UK Parliament is. Given that the current Conservative government received less than 50% of the popular vote, how come they have a majority of around 80 in the House of Commons?

## First past the post

Actually, anyone who is familiar with Britain's _first past the post_ voting system may already knows the answer. For voting purposes, the UK is split into 650 contituencies each of which vote for the candidate of their choice (these candidates normally represent a political party) and it is the candidate with the most votes that wins.  If there are only two candidates then it follows that the winner must have got more than 50% of the vote. But this is not normally the case. 

Most constituencies do have more than two candidates and so if the vote is close with the winning candidate having only a small advantage over her rivals, that candidate is likely to have less than 50% of the popular vote. We can see this with a simple example: say there are 50,000 votes cast in a constituency - 5,000 go to the Green Party, 10,000 go to the Liberal Democrats, 15,000 go to the Labour Party and the remaining 20,000 got to the Conservatives. The Conservatives win with a majority of 5,000 and are awarded the parliamentary seat for that constituency. But they only secured 40% of the vote! 

When this is repeated over the 650 contituencies, you can see how the party with the most seats in the House of Commons may not have the support of the majority of voters.

Critics of this system regard this as a fundamental weakness and would prefer a more proportional system as found in other European countries. Proponents say that _first past the post_ avoids the inevitable weak coalition governments.

But let's not worry about the politics, let's make it clear what is going on and do the analysis anyway.

### The raw data
As I said there are two identical data sets, one an SQLite database and the other a CSV file. I've anonymised the data to a certain extent in that I have removed the names of the MPs who were elected and the names of the constituencies that they represent - this is not about personalities or areas of the country but just about how the numbers add up.

The tables have the following columns:

- ons_id: the identifier for the constituency
- result: The party that won and whether it was a new win or a 'hold'
- first_party: the party that won
- second_party: the party that came second
- electorate: the number of voters
- valid_votes: the number of valid votes 
- invalid_votes: the number of invalid votes
- majority: the difference in votes between the winner and the runner-up
- con, lab, ld, ,brexit, green, snp, pc, dup, sf, sdlp, uup, alliance, other: the various parties and their share of the votes

To load the data we use the following code. 

For the CSV file:

    import pandas as pd
    election_df = pd.read_csv('elections.csv')

and for the database:

    import sqlite3 as sql
    conn = sql.connect('elections.db')

This gives us the two forms of the data set.

And if we do this

    election_df.head(4)

we get to see what it looks like. Here is a partial view:

![The elections table](https://github.com/alanjones2/Alan-Jones-article-code/raw/master/sqlpandas/images/elections_head.png)

In order to do our analysis, we want to find out just how many votes were cast for each political party.
So the first thing to do is to identify the names all of the individual parties that have won a seat in the House of Commons.
We start by getting a list of all the winners from the column _first_party_.

Using Pandas we simply do this:

    election_df['first_party']

We can assign that expression to a variable and we have a list of all the winners.

How do we do that with SQL?

The first thing is to construct an SQL query and then execute it.
You can see the query below as a string. We first use the ```SELECT```
keyword to declare the field that we are interested in (```first_party```) and then the table that contains the field (```elections```) in the ```FROM``` clause. We execute it with the connection that we create from the database file, earlier. That returns a cursor that we can use to retrieve the rows that contain our data. 

    query = """
        SELECT first_party 
        FROM elections
    """
    cur = conn.execute(query)
    rows = cur.fetchall()

The list is now in ```rows```. Not quite as concise as Pandas.

Of course, the list we have contains many duplicates as the same party may have won many seats. So, what we want are the unique values from this list of winners and in Pandas this is straightforward:

    partiesdf = election_df['first_party'].unique()

We just use the ```unique()``` method to filter the result and this is what we assign to the variable ```partiesdf```.

In SQL we use the ```DISTINCT``` keyword in the query to produce the same result.

    query = """
        SELECT DISTINCT first_party 
        FROM elections
    """
    cur = conn.execute(query)
    rows = cur.fetchall()
    partiesdb = rows

And, in this code, we have assigned the result to the variable ```partiesdb```.

The results that we get with the two techniques are, in fact, not quite the same. With the Pandas version the result is a list, whereas with the SQL query, the result is a list of tuples. The reason for this is that while we will only get a list of single values from Pandas, we could have specified more than one value in the SQL ```SELECT``` statement. This is not a big deal as long as we are aware of the difference, i.e. we will use each individual element of the Pandas list, whereas in the SQL version, we are intersted in the first value of the tuple in each list element.

Here's the list from the Pandas version:

    ['Lab', 'Con', 'SNP', 'PC', 'LD', 'DUP', 'SF', 'SDLP', 'Green','Spk', 'Alliance']

And from the SQL we get:

    [('Lab',), ('Con',), ('SNP',), ('PC',), ('LD',), ('DUP',), ('SF',), ('SDLP',), ('Green',), ('Spk',), ('Alliance',)]

All the names in the lists are those of political parties that have at least one set in the British Parliament, with the exception of ```Spk``` which represents the Speaker of the House of Commons. This particular Member of Parliament manages the House, is deemed to be neutral, and is not normally required to vote. Thus he or she does not count as a party member.

We will remove the Speaker from our calculations later.

The next thing we want to do is to find the number of wins that each party has. Each win represents a seat in the House of Commons and they are recorded in the ```'first_party'``` column. So, with Pandas we could write:

    election_df['first_party'] == 'lab'

to get the list of results for the Labour Party. This is it:

    0       True
    1      False
    2      False
    3      False
    4      False
        ...  
    645     True
    646    False
    647    False
    648     True
    649    False
    Name: first_party, Length: 650, dtype: bool

So if we ran this code:

    election_df[election_df['first_party']=='lab']

we would get a list of all of the wins for the Labour Party and the length of this list would represent the number of seats that they gained. Do this for reach of the parties in the list we created earlier and we have the total number of seats won by each party.

We could do this by appending the count to an empty list like this:

    partywinsdf = []

    for i in partiesdf:
        partywinsdf.append(len(election_df[election_df['first_party']==i]))
    print(partiesdf)

But a more Pythonic method is to use list comprehension like this:

    partywinsdf = [len(election_df[election_df['first_party']==i]) 
            for i in partiesdf]

The result is a list of the number of seats for each party. Here is the list of parties, followed by the list of wins (i.e. seats) we created, above:

    ['Lab' 'Con' 'SNP' 'PC' 'LD' 'DUP' 'SF' 'SDLP' 'Green' 'Spk' 'Alliance']
    [202, 365, 48, 4, 11, 8, 7, 2, 1, 1, 1]

We can see from this that the Labour Party gained 202 seats, the Conservatives have 365 seats, the Scottish National Party (SNP) have 48 seats, and so on.

This method using Pandas works well but how does it compare to SQL. Here is the equivalent SQL version, again using list comprehension. Most of the processing is in a function ```getWins(i)```  which fetches the number of wins for a party represented by ```i```.

    def getWins(i):
        query = f"""
            SELECT * 
            FROM elections
            WHERE first_party = '{i[0]}'
        """
        cur = conn.execute(query)
        rows = cur.fetchall()
        return(len(rows))

    partyWinsdb = [ getWins(i) for i in partiesdb]

The real work is done by the ```SELECT``` statement which uses a ```WHERE``` clause to filter the selection such that the value of ```first_party``` is equal to the party name that is passed to the function. Remember that in the SQL version the party names are in tuples hence the party name is ```i[0]```, the first (and only) element of the tuple.

    SELECT * 
    FROM elections
    WHERE first_party = '{i[0]}'

Comparing this to the Pandas version the syntax of the SQL statement is (at least from my point of view) a bit clearer in its intention than the Pandas equivalent. Having said that it is still rather more verbose.

Let's take a look at the data we have in chart form. To do that we will use Pandas plotting function, as this is one of the most straightforward charting methods.

Our Pandas only version of this is:

    dfdf = pd.DataFrame(partywinsdf,partiesdf)
    dfdf.plot.bar(legend=False,title='Seat allocation per party')

and the SQL version is almost identical:

    import pandas as pd
    dfdb = pd.DataFrame(partyWinsdb,partiesdb)
    dfdb.plot.bar(legend=False, title='Seat allocation per party')

And the chart that we get from either of these is this:

![Seat allocation](https://github.com/alanjones2/Alan-Jones-article-code/raw/master/sqlpandas/images/seatallocparty.png)

And you can see from this that the party with the most parliamentary seats is the the Conservative Party with more than 150 seat advantage over the second place Labour Party. The Scottish National Party (SNP) comes in next followed by the Liberal Democrats, various other regional parties and the Green Party.

We are going to see how this result compares to the percentage of the votes cast for each party, shortly. But first I'm making a decision.

## I choose Pandas

At this point, I think I know what I am more comfortable with. I admit to liking the clarity of the SQL statements but that hasn't persuaded me to ditch Pandas in favor of SQL. 

The next thing I want to do do is to create a new table that contains the results of the analysis in order to plot some charts. I could do this using SQL without too much difficulty but I would then have to convert it into a Pandas dataframe in order to plot it.

So to tie up the rest of the analysis I'll use Pandas - it is mostly about drawing charts, anyway, so I think it is more appropriate (you may disagree, in which case I'd be very glad to hear your views in the comments below).

## Proportional representation

What I'd like to do next is to show what the result of the 2019 elections would have been if the seats in Parliament were allocated in proportion to the number of votes cast.

The first thing to do is to remove the Speaker from the data as he does not represent a real party. The Speaker is element 9 so the easiest thing to do is this:

    partiesdf = list(partiesdf)
    partiesdf.pop(9)
    seats = list(partywinsdf)
    seats.pop(9)

That just removes the 9th element from the ```partiesdf``` list and the ```seats``` list.

Now we work out the total number of votes cast and the number cast for each party.

The total number of votes cast is the sum of the ```valid_votes``` column.

    total_votes=election_df['valid_votes'].sum()

And to get a list of the totl number of votes cast for each party, we sum the values in the column that corresponds to that party.

    total_votes_party = [election_df[i].sum() for i in partiesdf]

Now we can create a new dataframe using this data:

    share_df = pd.DataFrame()
    share_df['partiesdf'] = partiesdf
    share_df['votes'] = total_votes_party
    share_df['percentage_votes']=share_df['votes']/total_votes*100
    share_df['seats'] = seats
    share_df['percentage_seats']=share_df['seats']/650*100
    share_df['deficit']=share_df['percentage_seats']-share_df['percentage_votes']
    share_df['proportional_seats']=share_df['percentage_votes']/100*650

The resulting dataframe looks like this:

![Seat allocation](https://github.com/alanjones2/Alan-Jones-article-code/raw/master/sqlpandas/images/parties_share.png)

So let's see just how representative this is. Below is a bar chart that compares the percentage of seats gained to the percentage of votes cast for a particular party. We create it like this:

    share_df.plot.barh(x='partiesdf',y=['percentage_seats','percentage_votes'],figsize=(15,5))

And here it is:

![Seat allocation](https://github.com/alanjones2/Alan-Jones-article-code/raw/master/sqlpandas/images/seatsvotes.png)

You can see that the Conservative Party, the DUP and the SNP have a higher percentage of seats than votes, whereas the others have a lower percentage of seats than votes. We can see who are the winners and losers in the system by plotting the ```deficit``` column like this:

    share_df.plot.barh(x='partiesdf',y=['percentage_seats','percentage_votes'],figsize=(15,5))

![Seat allocation](https://github.com/alanjones2/Alan-Jones-article-code/raw/master/sqlpandas/images/deficit.png)    

From this chart you can see why the Liberal Democrat Party (```ld``` in the chart) would prefer a proportional representation system of elections in the UK. They have far fewer parliamentary seats than their share of the vote would suggest. The Green Party are also disadvantaged in the same way.

Finally, what would the UK Parliament look like if the number of votes cast for each party were proportionally represented. If we plot the actual number of seats gained against the number of seats that would be allocated under a truly proportional system like this:

    share_df.plot.bar(x='partiesdf',y=['seats','proportional_seats'],figsize=(15,5))

We see that the makeup of the House of Commons would be quite different.

![Seat allocation](https://github.com/alanjones2/Alan-Jones-article-code/raw/master/sqlpandas/images/proportionalseats.png)

Proportionally allocated seats are shown in orange and you can see that the Conservative Party's representation would be substantially reduced and they would lose their majority. The Labour Party would ave a few more seats but the real winners would be the Liberal Democrats whose seat count would shoot up from 11 to 75, and the Greens who would go from 1 to 17.

Whether a move to a proportional system would be an imporovement is not for me to say. However, given the state of British politics at the moment, one could easily see that under the circumstance shown above, a coalition of Labour, Liberal Democrats and Greens would have more seats (about 300) than the Conservatives and could very feasibly form a government with the support of other local parties. Such a result would mean that Britain would have an entirely different type of government.

## So, what now?

My main aim here was to get straight, in my own mind, whether or not to rely more on SQL and less on Pandas for data analysis, rather than to advocate for a new electoral system in the UK. And to that end I have made up my mind that while I won't actively avoid SQL as I have in the past, I'll stick with Pandas for now (I still won't back down on Visual Basic, though).

Thanks for reading, I hope my meander through the vagaries of the British electoral system have been useful in the context of Pandas and SQL (and in the context of electoral reform, if you happen to be British).

If you would like, you can subscribe to my occasional free newsletter, 
[Technofile](https://technofile.substack.com/) which is on 
Substack - I'll post about new articles there.

