#!/usr/bin/env python
#
#       Adventure
#
# Copyright (C) 2013 Chris Miller <cmiller@learningtech.org>
#
# Special thanks to Tim Kortenkamp and Greg Miller for help
# generating items and rooms.
#
#
# A tribute to classic text-based adventure games, written in Python.
# Developed for educational purposes and just for fun; designed as a lesson
# on using Python lists in a playful way. The game world can be easily expanded
# by simply adding more items to the appropriate lists.
#
# Should run on any version of Python v2.x - currently untested on Python v3.
# In particular, the raw_input() function may break on Python v3.
#
#
# This program is a just a subset of the full game - the core movement engine
# without all the fancy stuff like combat, inventory, etc. There are 5 starter rooms
# supplied here - the original 5 rooms I created to first test my movement engine.
#
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#


# I use these globals to quickly fine-tune the combat system.
# Note: even small changes to the multipliers will have huge effect as enemy & player levels scale up
# Most of this isn't necessary for the movement-only engine, however some of it is required by the
# game framework so I just included it all.
HP = 13 # player's HP multiplier (playerHealth = playerLevel * HP)
ENEMYHP = 4 # enemy's HP multiplier (enemy hitpoints = enemy lvl * ENEMYHP)
ENEMYXP = 15 # enemy's XP multiplier (XP earned = enemy lvl * ENEMYXP)
LEVELUPXP = 160 # next lvl XP multiplier (XP needed to lvl up = (playerLevel * playerLevel) * LEVELUPXP)
CAMPCOST = 3 # resting price multiplier (max price = playerLevel * CAMPCOST)
ENEMYGOLD = 3 # gold drop multipler (max gold drop = enemy lvl * ENEMYGOLD)
GOLDCHANCE = 60 # percent chance for a gold drop
POTIONCHANCE = 5 # percent chance for a potion drop
WEAPONCHANCE = 10 # percent chance for a weapon drop
ARMORCHANCE = 10 # percent chance for an armor drop
CAMPCHANCE = 10 # percent chance camp will be interrupted by a random spawn
SPAWNRATE = 5 # respawn multiplier (playerLevel * SPAWNRATE out of 1000 chance each turn)
HPREGEN = 2 # health regen multiplier (regen chance = playerLevel * HPREGEN)
HEALTHPOTIONS = 10 # append health potions to the list to increase their drop rate over other potion types
WEAPONVALUE = 2 # sale value modifier (min sale price = weapon lvl * WEAPONVALUE)
ARMORVALUE = 3 # sale value modifier (min sale price = armor lvl * ARMORVALUE)
POTIONVALUE = 5 # direct sale value (min sale price = POTIONVALUE)
TESTING = False # used only for testing purposes, spoils combat and thus general game play

import random

class Adventure():
    def __init__(self):
        # These are the starting values for the player. Most of this isn't needed for simple movement
        self.playerInventory = []
        self.currentRoom = 1
        self.playerLevel = 1
        self.playerExp = 0
        self.playerHealth = HP
        self.playerAtkVal = 1
        self.playerDefVal = 1
        self.playerWeapon = None
        self.playerArmor = None
        self.playerGold = 0

        # This list of recognized commands is echoed back by the help() function, and used by the
        # command interpreter to verify user input.
        self.commandList = ['help','look','inventory','stats','north','south','east','west','wait','quit','exit']


        # This is the list of room names. Adding new rooms is easy, however you MUST update all
        # 3 lists (roomName, roomDescription, roomExits) to avoid list indexing errors.
        # The roomName list order must exactly match the corresponding list
        # order for the roomDescription & roomExits lists to work correctly.
        self.roomName = [None,     #0, Null room
            'Crossroads',   #1
            'North road',   #2
            'South road',   #3
            'East road',    #4
            'West road',    #5
        ]

        # This is the list of room descriptions, in matching order of the room names.
        self.roomDescription = [None,    #Null room
            'You are at a four way intersection on a deserted road. The road continues in all directions as far as the eye can see.\nA crudely-made signpost reads: ALL ROADS LEAD TO EVER-INCREASING PERIL\n',    #Crossroads
            'You are traveling on a long, narrow road that leads to the north. A small structure is visible in the distance.',    #North road
            'You are traveling on a sprawling, tree-lined path snaking its way to the south. In the distance you see a vast forest.',    #South road
            'You are traveling on a long road that leads to the east. A large body of water stretches off into the horizon.',    #East road
            'You are traveling on a winding road that heads generally westward. Further on, a long range of jagged mountain peaks zags along the coast.',    #West road
        ]
        
        # roomExits is a list of lists, storing room connections by name in order of [north,south,east,west] exits
        # from the current room. None means no room connection for that exit
        self.roomExits = [[None,None,None,None],    #Null room
            ['North road','South road','East road','West road'],    #Crossroads
            [None,'Crossroads',None,None],    #North rd
            ['Crossroads',None,None,None],    #South rd
            [None,None,None,'Crossroads'],    #East rd
            [None,None,'Crossroads',None],    #West rd
        ]
        
    def help(self): # Print the list of recognized commands and their shortcuts
        print ('Available commands: ' + str(self.commandList))
        print ('\nShortcuts: n,s,e,w = move, l = look')


    def listExits(self): # Return a list of available exit directions from current room
        exits = []
        if self.roomExits[self.currentRoom][0] != None:
            exits.append('north')
        if self.roomExits[self.currentRoom][1] != None:
            exits.append('south')
        if self.roomExits[self.currentRoom][2] != None:
            exits.append('east')
        if self.roomExits[self.currentRoom][3] != None:
            exits.append('west')
        return exits


    def look(self): # Print the room name, description, and list of exits.
        print(' ')
        print (self.roomName[self.currentRoom])
        print self.roomDescription[self.currentRoom]
        print ('Available exits: ' + str(self.listExits()))


    def inventory(self):
        if len(self.playerInventory) > 0:
            print('\nPlayer inventory: ' + str(self.playerInventory))
        else:
            print ('You are not carrying any items right now.')
        print('Weapon: ' + str(self.playerWeapon))
        print('Armor: ' + str(self.playerArmor))
        print('Gold: ' + str(self.playerGold))


    def getInput(self): # This part of the command interpreter asks for user input and verifies their cmd is known.
        cmd = ' '
        while (cmd not in self.commandList) and (cmd not in ['n','s','e','w','l']):
            cmd = raw_input('\nWhat would you like to do? ')
            if len(cmd) != 0:
                try:
                    cmd = cmd.split()[0].lower()    #just take the first word, ignore the rest for now
                    if cmd == 'n':
                       cmd = 'north'
                    if cmd == 's':
                       cmd = 'south'
                    if cmd == 'e':
                       cmd = 'east'
                    if cmd == 'w':
                       cmd = 'west'
                    if cmd == 'l':
                       cmd = 'look'
                    return cmd # We've confirmed the cmd is known, so return it
                except:
                    pass #ignore user input issues & move on


    def stats(self):
        print('\nPlayer Statistics')
        print('Level: ' + str(self.playerLevel))
        print('Health: ' + str(self.playerHealth) + '/' + str(self.playerLevel * HP))
        print('Attack: ' + str(self.playerAtkVal))
        print('Defense: ' + str(self.playerDefVal))
        print('XP: ' + str(self.playerExp) + '/' + str((self.playerLevel * self.playerLevel) * LEVELUPXP))
        print('Gold: ' + str(self.playerGold))


    def movePlayer(self, direction):
        # This is the core of the movement engine. First it checks if the direction
        # the player wants to go in has a valid exit, by making sure the room name stored
        # in the current room's exits list for that direction is not None. This is accomplished
        # using the first list index to match room numbers.
        # self.currentRoom is the index number used for reference, and matches the corresponding
        # indices for the roomName and roomDescription lists, and as the first index of the
        # roomExits lists.
        
        # The second index is a list of room names in order of north, south, east, west
        # self.roomExits[self.currentRoom][0] returns the name of the room to the north of the
        # current room.
        # [0]=north room, [1]=south room, [2]=east room, [3]=west room
        
        # After confirming the room name in that direction isn't None, we just change the
        # currentRoom to match the index number of the new room's name - effectively moving
        # the player to that room, then calling look()
        
        if (direction == 'north') and (self.roomExits[self.currentRoom][0] != None):
            self.currentRoom = self.roomName.index(self.roomExits[self.currentRoom][0])
            self.look()
        elif (direction == 'south') and (self.roomExits[self.currentRoom][1] != None):
            self.currentRoom = self.roomName.index(self.roomExits[self.currentRoom][1])
            self.look()
        elif (direction == 'east') and (self.roomExits[self.currentRoom][2] != None):
            self.currentRoom = self.roomName.index(self.roomExits[self.currentRoom][2])
            self.look()
        elif (direction == 'west') and (self.roomExits[self.currentRoom][3] != None):
            self.currentRoom = self.roomName.index(self.roomExits[self.currentRoom][3])
            self.look()
        else:
            # Provide an error message if they tried an invalid movement, then repeat the list of exits
            print('You cannot proceed to the ' + direction + ' from here.')
            print('Available exits: ' + str(self.listExits()))
        

    def processInput(self, cmd):
        # This is the core of the command interpreter. We already verified the user input
        # was recognized, so now we can just call the function they asked for.
        if cmd == 'help':
            self.help()
        if cmd == 'look':
            self.look()
        if (cmd == 'inventory') or (cmd == 'inv'):
            self.inventory()
        if cmd == 'stats':
            self.stats()
        if (cmd == 'wait'):
            print('Time passes...')
        if (cmd in ['north','south','east','west']):
            # If player specified a direction, move that direction
            self.movePlayer(cmd)


def adventure():
    a = Adventure()
    print("\n\n    Welcome to Chris Miller's unique text adventure!\n\n")
    print("Type 'help' for a list of available commands.")
    print('\nYou wake up by the side of the road, dizzy and disoriented. ' +
        'You instinctively grab the back of your head, and let out a soft moan of pain as you discover the fresh blood still clotting on your cracked skull. ' +
        'You appear to have been mugged and left for dead, out in the middle of nowhere.\n')
    
    a.look()

    while a.playerHealth > 0:

        #get player's next move
        cmd = a.getInput()
        if (cmd == 'quit') or (cmd == 'exit'):
            break
        else:
            a.processInput(cmd)

    # basic Game Over statement, will print regardless of death, 'quit', or player win
    print('\nYour game ended at level ' + str(a.playerLevel) + ' with ' + str(a.playerExp) + ' experience and ' + str(a.playerGold) + ' gold.\n')

adventure()
