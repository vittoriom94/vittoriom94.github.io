---
author: "Vittorio Mattei"
date: 2022-07-22
linktitle: Python if-else alternatives
title: Python if-else alternatives for better design
type:
- post 
- posts
weight: 1
categories:
- Programming
tags:
- programming
- python
- beginner
- design
series:
- Python programming
---

There is no doubt that one of the first conditional construct a programmer will encounter is the if-elif-else construct.  
While the syntax may change across languages, the logical meaning is always the same:  
```
if condition_1:
    do something
else if condition_2:
    do something different
else:
    do something generic
```
While this construct is incredibly easy to use, there are some drawbacks to it, when the logic starts expanding.  
Note that obviously there isn't a best rule, the only way to decide which solution to use is on a case by case scenario. Refactoring is also a fundamental tool. Nothing stops you from using one method and then changing to another later, when you realize your code can be simplified by a refactoring.
# Case 1: Multiple well defined conditions
In the first case we'll set up the following scenario.  
We are creating a texting game. The player can cast spells to attack enemies, and we just decided to implement two simple spells: "fireball" and "thunderbolt".  
The setup for this part of the code is simple: the player inputs a command **"spell *<spell_name>*"** and the corresponding spell is cast to the enemy. We assume that the enemy is already selected by a previous command and we just want to cast the spell against him. We also assume that the spell command is already interpreted, so the only part to handle is the spell selection and cast.

To cast a spell, we need to call the method *cast* of the object *player*. The method takes two parameters: the spell object and the enemy.
```
# player.py
class Player(Entity):
    ...
    def cast(spell: spells.Spell, enemy: enemy.Enemy):
        # do something
    ...
```
The **issue** that we want to resolve is how to map a string to a spell class.  
The simplest solution is to use a typical if-elif-else chain. If we find the spell, we can call the *cast* method, otherwise we raise an exception, that can be handled from outside when there isn't a match for a command.

```
# spells.py
class Spell:
    ...
class Fireball(Spell):
    ...
...
# Assume that we defind all possible spells as classes that extend Spell

def elaborate_input_spell(spell_name: str, player: player.Player, enemy: enemy.Enemy):
    if spell_name == "fireball":
        return player.cast(Fireball(), enemy)
    elif spell_name == "thunderbolt":
        return player.cast(Thunderbolt(), enemy)
    else:
        raise Exception("Spell not found.", spell_name)
```

This solution is simple and easy to understand, but what if we need to add more spells? That code might easily devolve into something like this:

```
# spells.py

def elaborate_input_spell(spell_name: str, player: player.Player, enemy: enemy.Enemy):
    if spell_name == "fireball":
        return player.cast(Fireball(), enemy)
    elif spell_name == "thunderbolt":
        return player.cast(Thunderbolt(), enemy)
    elif spell_name == "blizzard":
        return player.cast(Blizzard(), enemy)
    elif spell_name == "meteor":
        return player.cast(Meteor(), enemy)
    elif spell_name == "sap":
        return player.cast(Sap(), enemy)
    elif spell_name == "whirlwind":
        return player.cast(Whirlwind(), enemy)
    elif spell_name == "storm":
        return player.cast(Storm(), enemy)
    elif spell_name == "bury":
        return player.cast(Bury(), enemy)
    elif spell_name == "blind":
        return player.cast(Blind(), enemy)
    else:
        raise Exception("Spell not found.", spell_name)

```

... and so on and so on.  
Note how all the conditions are a simple equality of strings, and each string is then **mapped** to a fixed class. Why not use a [dictionary](https://docs.python.org/3/tutorial/datastructures.html#dictionaries) for this?  
A dictionary allows to map a *key* to a *value*. Keys and values can be of any type so this is not only restricted between strings and classes but you can do any kind of mapping.

Here's how we can refactor the above code by using a dictionary:
```
# spells.py
# Spell classes definitions go here

spells = {
    "fireball": Fireball,
    "thunderbolt": Thunderbolt,
    "blizzard": Blizzard,
    "meteor": Meteor,
    "sap": Sap,
    "whirlwind": Whirlwind,
    "storm": Storm,
    "bury": Bury,
    "blind": Blind
}

def elaborate_input_spell(spell_name: str, player: player.Player, enemy: enemy.Enemy):
    spell = spells.get(spell_name)
    if spell:
        return player.cast(spell(), enemy)
    raise Exception("Spell not found.", spell_name)
```

When we have new spells we just need to add them to *spells*.  
Note that **dict().get(value)** will return **None** if the key is not found, so we call cast only if spell is not **None**. The return statement will ensure that we don't raise an Exception in the normal flow.  
Not also that spells is defined outside the method, as to ensure that the variable is initialized only at startup and not in every method call! Also note how the values are classes and not instances, otherwise we would return the same instance everytime we cast a spell, instead of creating a new one.  

It's also import to note that now you can import the spells variable in other modules, so when you need to use spells for additional operation, you don't need to copy the whole if-else tree, but you only import spells.spells.

# Case 2: composed conditions
Let's now have a different case. This time, we want to sell an item. The sell function will be called when the player inputs the command **"sell *\<item\>*"**.  
We assume that the player is already talking to a merchant and the item to sell has already been selected

There are different elements to consider:
- The item rarity
- The item type
- The item durability (new, used, destroyed)

```
# items.py
item_rarities = ["common", "rare", "epic"]
item_types = ["weapon", "armor"]
item_durabilities = ["new", "used", "destroyed"]

class Item:
    rarity: str
    item_type: str
    durability: str
```

Let's assume we start by implementing a sell function that only takes into consideration of the *durability* property. We can just check it against *item_types* and return a different sell value:
```
# merchant.py

def sell(item: items.Item):
    if item.item_type == "weapon":
        return 1000
    elif item.item_type == "armor":
        return 500
    else:
        return 0
```

This is pretty straightforward and we easily implemented a sell function for our game.  
But what if now we want to take into consideration the durability?
```
# merchant.py

def sell(item: items.Item):
    if item.item_type == "weapon" and item.durability == "new":
        return 1000
    elif item.item_type == "armor" and item.durability == "new":
        return 500
    elif (item.item_type == "armor" and item.durability == "used") or (item.item_type == "weapon" and item.durability == "used"):
        # We could only check if durability is used, but this may not be true for additional item types
        return 50
    elif item.durability == "destroyed":
        return 10
    else:
        return 0
```

This starts getting a little bit more confusing and it easily becomes unmaintainable when we add new item types, or we take the rarity into consideration.  
There are many solutions in this case, it mostly depends on the specific scenario you're analyzing and the action you're doing.  
In this case we can leverage the fact that we are using a mathematical formula, so we could create maps that associate a base value to the item type and multipliers depending on durability and rarity.  
### Match-case
What I want to show next is a different approach that has been available since the release of **Python 3.10**. If you come from Java, C# or C++ you're probably already familiar with this approach as they already had switch-case statement.  
Python implemented this recently and called them [match-case](https://peps.python.org/pep-0636/), as they allow developers to create pattern matching structures. When if-else conditions start getting long, complicated and depending on each other, you can use match-case to bring some order to the code.  
Let's rewrite the code seen before
```
# merchant.py

def sell(item: items.Item):   
    value_elements = (item.item_type, item.item_durability)
    match value_elements:
        case ("weapon", "new"):
            return 1000
        case ("armor", "new"):
            return 500
        case (("weapon" | "armor"), "used"):
            return 50
        case (_, "destroyed"):
            return 0
```

Pattern matching allows really powerful conditions and you should check out python documentations for powerful examples. As you can see, it's really easy to add conditions this way, without copying *if* conditions multiple times in the code. Just remember that this is only possible from **Python 3.10**!

## Conclusion
As we've seen, while an if-else approach works, over time it becomes hard to maintain and it easily creates bugs around in your code when you lose tracks of conditions or you start copying your code around. The two easiest solutions depend on the particular case.

When you have an if-else tree that associates a certain condition with a fixed action, consider using a **dict**. They're easy to implement, they can be imported in other modules and it's easy to modify or improve thm.

When you're dealing with complex conditions, common actions for multiple conditions, conditions depending only on some parameters, consider using **match-case**. You can probably express the same logical meaning but in a more readable and improvable way.