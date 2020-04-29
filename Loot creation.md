#Loot

Loot is dealt in five different table...

global_loot
lootdrop
lootdrop_entries
loottable
loottable_entries



///Loottable


///Loottable_entries
multiplier = how many times this table entry will be applied

///Lootdrop


///lootdrop_entries
multiplier = how many times this table entry will be applied
mindrop = at least X number of items will be placed on npc
equip_item 0=nope 1=yes
chance = % of a chance this table entry will be applied




Multipliers are a means to call on the entry as per your desire. 2 two times. This can stack so be warned.



Theres 2 drop chances as well, one for group, one for the item


1% is 0.01
.5678 would be 56.78%


But basically you roll for loottable then roll for lootdrop then you roll for item.



So 50% loottable, 10% lootdrop, and 1% item is (.5 * .1 *.01) drop rate.


so any loot tables in blobal_loot will populate globally according to min-max level rules?




//////////////////////////
NPCs have a loottable ID, loottables have lootdrops, lootdrops have items.

So to find what items drop on a certain NPC all you need to do is run this query (# is your NPC ID).
Code:
SELECT
    items.*
FROM
    items
INNER JOIN lootdrop_entries ON items.id = lootdrop_entries.item_id
INNER JOIN loottable_entries ON lootdrop_entries.lootdrop_id = loottable_entries.lootdrop_id
INNER JOIN npc_types ON loottable_entries.loottable_id = npc_types.loottable_id
WHERE
    npc_types.id = #;
///////////////////////


There is now probability in loottable_entries, which should be 100 for most purposes, but would ultimately be used to roll on whether or not that loottable entry even tries to drop.
Each lootdrop entry has chance, so for your 5% you would probably want a 100% loottable entry with a lootdrop that has your item in it with a chance of 5%.

It's not especially complicated, so just look for the two relevant chance/probability fields and do the math.

As an aside, I think it's good to investigate any player concerns, but when it comes to RNG complaints I usually find that people expect some kind of coherent noise generator and lack any real understanding of what 'random' is.


//////////////////////////////////////////////////////




Restore your database from a backup (the old system) and run the following queries and ONLY the following queries on loot:

Code:
alter table loottable_entries add `droplimit` tinyint(2) unsigned NOT NULL default 0;
alter table loottable_entries add `mindrop` tinyint(2) unsigned NOT NULL default 0;
alter table lootdrop_entries change `chance` `chance` float not null default 1;
alter table lootdrop_entries add `multiplier` tinyint(2) unsigned NOT NULL default 1;
update loottable_entries set droplimit = 1 where probability != 100;
update loottable_entries set mindrop = multiplier, droplimit = multiplier, multiplier = 1 where probability = 100;
This will convert your tables to the new system, while keeping the old functionality. The problem was when I removed probability the conversion SQL was needed. Since it's added back, the only change needed is to set droplimit to 1 so that the tables will only drop 1 item at a time like in the old system. Multiplier ignores droplimit so that will still work as intended. Your chance and probability do not need to change at all (which is your issue in the example - your probability changed from 8% to 100%, but you didn't change chance to compensate.)

I'm going to go ahead and change the SQL on SVN to reflect this.


////////////////////////////////////////////////////////////




My next commit on the EQEmu SVN will include an overhauled loot system. It will contain a SQL that will convert your loot tables to the new system preserving both drop rates and drop behavior. I just wanted to post so that everybody understands the new system. 

First, the table changes:

Two new columns have been added to loottable_entries called droplimit and mindrop. Probability has been removed from loottable_entries. Multiplier has been added to lootdrop_entries, and chance in lootdrop_entries has been changed to a float so it can handle decimals.

The biggest change is in the new system, each item is rolled individually instead of against every other item in the same lootdrop table. I have already confirmed on PEQ that there is no performance hit with this change. This means that you could potentially have every item on a single lootdrop table per loottable if you so wish. However, multiple lootdrop tables per loottable is still very much supported. The added columns preserve ALL of the functionality the old system, plus adds some new functionality. 

The new columns are: 

mindrop: This value forces the lootdrop table to always drop the number of items specified. If all of the items are rolled and mindrop is not met, everything is re-rolled until it is. 0 is no mindrop, so the table could potentially drop no items if none are 100%.

droplimit: The opposite of the above column. Once this value is met, no more items will drop from the lootdrop table, even if entries with a 100% chance are left to roll. There is also a chance that the droplimit is not met, and fewer items than the value specified will drop. 0 means there is no limit, and the lootdrop table could potentially drop every item. 

multiplier: Same as before really. The simplest way to explain this is it multiplies the other 2 columns. So, if your mindrop is 2 and your multiplier is 4, the lootdrop will drop AT LEAST 8 items. If your droplimit is 2 and your multiplier is 4, the lootdrop will drop NO MORE than 8 items. multiplier is useful if you want a different number range of items to drop every time, instead of just the number of items in the lootdrop table. 

It is important to note using either mindrop and multiplier can cause the same item to be dropped multiple times. 

It is also important to note that all 3 columns will effect the true percentage every item has of dropping. droplimit may prevent items from even getting a roll at all, mindrop may potentially give items extra rolls, and multiplier will give every item an extra roll per each value (multiplier of 4 gives every item 3 extra rolls.) Setting mindrop to 0, droplimit to 0, and multiplier to 1 will give you 100% clean percentages, meaning the percentage specified in chance will be the exact chance the item has of dropping.

The order the items are rolled is randomly chosen every time, so no items will have an advantage over others due to location in the list.

Last, the new multiplier column in lootdrop_entries works exactly the same way as the one above, except it's per item. So, if you need rats eyes to potentially drop twice, but rat tail to only drop once you can do so and still have them in the same lootdrop table. Any items with multiplier set here receive an extra roll per value, while the ones with it set to 1 do not. Only the first successful roll counts towards mindrop above. So, if mindrop is set to 1 and ration's multiplier is set to 4, you can still drop 4 rations however no other items in that lootdrop will be able to drop. 

Any questions, please ask away 

CAVEDUDE

//////////////////////////////////////////////////////////////////////

I know this is easy to over think and I did it too when I was question Cavedude on changing the system before we bring in the data that I parsed from Magelo.

Think of it as a header.

loottable_entry:
Code:
mysql> select * from loottable_entries WHERE loottable_id
 = 1000 limit 1;
+--------------+-------------+------------+-------------+-----------+---------+
| loottable_id | lootdrop_id | multiplier | probability | droplimit | mindrop |
+--------------+-------------+------------+-------------+-----------+---------+
|         1000 |        2014 |          1 |          25 |         1 |       0 |
+--------------+-------------+------------+-------------+-----------+---------+
You have a 25% probability of rolling into the lootdrop:

Inside that lootdrop (2014):

Code:
mysql> select * from lootdrop_entries where lootdrop_id = 2014;
+-------------+---------+--------------+------------+--------+----------+----------+------------+
| lootdrop_id | item_id | item_charges | equip_item | chance | minlevel | maxlevel | multiplier |
+-------------+---------+--------------+------------+--------+----------+----------+------------+
|        2014 |    3329 |            1 |          1 |   8.25 |        0 |      127 |          1 |
|        2014 |    3342 |            1 |          1 |    8.5 |        0 |      127 |          1 |
|        2014 |    3353 |            1 |          1 |   8.25 |        0 |      127 |          1 |
+-------------+---------+--------------+------------+--------+----------+----------+------------+
Each item in this lootdrop now is rolled individually. In other words, all three of these items could drop at the same time but they are all rolled individually instead of ONE roll a random number 1-100 and it being (3329 - 8.25%) - (3342 - 8.5%) - (3353 - 8.25%)

Now it is a roll for each item:
3353 - 8.25 (Rolled 10, well shucks)
3329 - 8.5 (Rolled 4, nice it dropped!)
3342 - 8.5 (Rolled 94, no cigar)


Quote:
Originally Posted by ChaosSlayerZ  
Also, when you say that items will now "roll individually" - I do know that all %% on items were screwed up and place in list was affecting proper percentages, but "roll individually" sounds somewhat vague - do items total still have to add up to 100%?
No items do not have to add up to 100% as they are rolled individually, the existing % and drops for PEQ will be audited against the script I am building to dump the new data and audit the existing data.

Make sense?

AKKADIUS

////////////////////////////////////////////////////////////////////////////////////////////

It's actually a LOT more simple than the old way once you grasp it. But yes, there is a learning curve involved coming from the old way.

That example is simple, you just use mindrop and droplimit to force 1 item to drop every time (since probability was 100%) So:

lootdrop_entries 85, mindrop 1, droplimit 1, multiplier 1

lootdrop_entries 85 would have inside:
item1 10%
item2 10%
item3 10%
item4 10%
item5 10%
item6 10%
item7 10%
item8 10%
item9 10%
item0 10%

Since probability was 100%, none of the percentages will change of course. No more and no less than 1 of those 10 items will drop. 

But changing it up, let's say the old probability was 80%. Then the same example would look like:

lootdrop_entries 85, mindrop 0, droplimit 1, multiplier 1

lootdrop_entries 85 would have inside:
item1 8%
item2 8%
item3 8%
item4 8%
item5 8%
item6 8%
item7 8%
item8 8%
item9 8%
item0 8%

Notice mindrop is now 0 because we are no longer guaranteed an item, and all the percentages dropped because multiplier is 20% lower. But, they all equal 80% - the old probability 

Let's have some fun and show off what the new system can do. 

Same example above, but:
mindrop 0, droplimit 0: 0 to all 10 of those items will drop. 
mindrop 5, droplimit 0: 5-10 of those items will always drop.
mindrop 1, droplimit 2: 1-2 of those items will drop.
mindrop 0, droplimit 0 and all the items have a 50% chance of dropping: 0-10 items can drop, but most of the time 5 will (since "probability" is now 500%.)

CAVEDUDE

///////////////////////////////////////////////////////////////////////////

PEQ PHP editor was updated way back in September, Rev 342. Make sure to grab it from SVN and not the download. I'm not going to be maintaining the downloads anymore. I just don't have the time.

I'll get back to the loot changes soon. I'm going to add probability back as an optional part of the system due to the fact that you can no longer have the same lootdrop in multiple loottables with different percentages. But, I'm 95% certain that is the only functionality the new system lost over the old, so no further changes will be needed. Of course, if we come up with something else the new system cannot do that the old could, I'll look at that as well.

For those who plain just don't like it, you can always revert locally. It's only 1 method, part of another method, and a few database calls. That code is hardly ever touched, so you could probably continue on with it in a conflicted state and never be bothered by it.

CAVEDUDE

/////////////////////////////////////////////////////////////////////////////


There is nothing to "fix." If you have 30 items with 2% chance that each will drop, then you have a 60% chance that at least 1 will drop (30*2.) That's basic mathematical probability and how our loot system should have worked from the beginning. But, I'm not going to get into that rant.

Now, to change it to behavior like you need it to you have 2 ways. 

First, probability (which I almost considered changing the name to probability_modifier because that is what it really is) is back, so you can set that to control the total drop rate of the group of items. 

Next, you can lower the chance of the items. (Remember, the new system now accepts floats so you can have chances below 1%.) This basically is just the drawn out way of doing the above. But from a mathematical standpoint, it is the correct way of doing so.


CAVEDUDE

///////////////////////////////////////////////////////

Probability works exactly as it did before. It is rolled to see if any item in that table will drop at all. If so, then each of the items are rolled against their individual chance to see if they will drop. 

If probability is 100%, then that means the items will always be rolled. Please know that doesn't mean they will drop - that is dependent on their chance. But at the least they will be given a chance to drop. On the other hand, if probability is 1% then 99 times out of 100, none of the items will even be rolled at all and thus have no chance to drop, even if their chance is 100%.

The one part that may be confusing is in the old system, if probability rolled true 1 item would ALWAYS drop (ignoring multiplier for this example.) In the new system that is no longer true. Any number of items can now drop, from 0 to all. If you'd like at least 1 item to always drop (provided the probability rolled true!), set mindrop to 1. If you want no more than 1 item to drop set maxdrop to 1, etc.

CAVEDUDE
//////////////////////////////////////////////////////////

Ah, Cavedude, thanks so much for replying. I understand now.
Quote:
Originally Posted by cavedude  
Probability works exactly as it did before.
From one point of view probability is the same in that it is checked first before moving on to roll against the chance column. (It is the chance roll that is handled differently than it was in the old system.) If I may respectfully offer a different point of view (not to try and straighten anyone out, but for clarification and potentially helping another who may read):

In the old system the probability represented the chance that something (specifically one thing) would drop from a set of items (like a lootdrop table of just gems). The important point here is the probability was associated with a sets of items not just individual items.

In the new system the probability is checked first, and if it "succeeds" if you will, then chance is checked. This way of doing loot is...not as straightforward as just checking loot against one percentage (like the chance column, and eliminating the probability column--as was tried in the first implementation of the new system).

The difference is in the way probability is viewed (or how loot is viewed depending on how you want to explain it). If you view loot as individuals items then the probability column does not make much sense to be sure. If you view loot as sets of items then it is the probability column that is descriptive and applicable to the set of items.

Example:
The set of gems mentioned earlier could be set up with percentages in the chance column (lootdrop_entries) according to how often you want each gem to drop. One the set of items is done it can be reused as many times as one would like for varying levels of MOBs. You can set the probability column (loottable_entries) low for low level MOBs and increase the probability appropriately for higher and higher leveled MOBs. (No need to keep creating lootdrop tables containing gems, just reuse the one you made--is one way of doing it.)

Mind you, the new system can be used similarly. Probability no longer applies to a set of items but just modifies and individual item's chance--you may get none of the items, or you might get three, or all of them if the rolls are just right. But if you want to think of the items as a "set" the overall probabilities are all the same.

The Difference:
From a statistical standpoint, the difference between the old system and the new system is that the new loot system has more variance. On average everything is the same but the fact that you could get no drops or a bunch of drops from a given MOB indicates that the variance is higher in the new system.

Consider 2 loottables each with 100 probability referencing a lootdrop table with two items that have a 50% chance each. In the old system you are guaranteed two drops from this scenario--100% from one set and 100% from the other set of items. In the new system you aren't guaranteed anything--you might get zero drops, or 1, or 2, or 3, or 4. More variance. Whether you think this is good or bad is an individual opinion.

My Opinion:
I like the new system with more variance because there's more random-ness and like gambling it turns players into Skinner rats clicking the button hoping that the next time will be the big payoff.  

Hope this helps someone who needs it.


reloc01c

//////////////////////////////////////////////////////////////////////////////////

npc_types have an entry 'loottable_id' which points to a loottable
loottable_entries with the matching loottable_id point to lootdrop_id
lootdrop_entries are what you're at right now and contain references to items

So you sort of need to go backwards given the lootdrop_id you have you need to find all the loottables that point to it. Then you need to go backwards from there and find each npc_type that points to that loottable.

KLS

////////////////////////////////////////////////////////////


Disabled chance just is a place holder for chance cause there's no other way to disable a slot without deleting it 
except to set it's chance to drop to 0. If you want to re-enable those drops you need to set the chance equal to what's
in disabled chance for the appropriate drops.

KLS

/////////////////////////////////////////////////////////




