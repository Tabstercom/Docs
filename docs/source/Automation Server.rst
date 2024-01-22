# Automation Server
## Overview

### Controls

To **open** please run it in vscode, I will complie it at somepoint, but its not worth it yet.

To **close** please Prese '**e**' then enter (if you miss-type, you will prompted to try again)

**notes!**
This will begin the shut down Process. This can take sometime (mins) as it has to wait for 2 seperate threads to finish their current processing. Then it will begin a 10 second countdown before closing. This is so that It can be sure its closed all connections to **MongoDb**

### Main Pull Loop

**Overview**
In a continous loop , The server looks in the **MongoDB**  For any **Monday** boards to pull data from. It downloads all the Item {object}'s, &  sanitizes the data. It then determins if any can be Automated (or why they cant e.g columns not correctly marked 'Done'). It then loops through the stantized data that can be automated and tries to write **CLI** for them. If it hits an error at any stage (i.e cant find the RC file) it notifys monday and sets the item to Error. It Caches this Data in **MongoDB**, replacing the data from the previous pull.

**Data Sanitation**
This process, we create a new Item{} with only the data we need, an items columns data & the id of each column (for updating monday). We also add its modified priority and run initial checks to look for formatting errors (i.e no date)

**Time Locks**  
 Every Time an Cached Item is updated on **MongoDb** we save the Time edited. Since the pull and CLI building of a board can take time (Monday is slow). When its time to replace the old Cached items, the 'new' pulled data might no longer be accurate, as Monday was update during its pull/processing. So we comare the time we started to pull the data to the last edit. If the 'new' data is out of data we do not cache it.

**notes!**
 - all the **CLI** building uses Settings created in the **GUI** e.g project settings, naming convertions, scanning board settings.... 
 - All cached Items are deleated when the server is Started, 
 
 ### Tasks/Queue System
Due to the asynchronicity of multiple agents. There is a Qeue system built into the server to ensure Items are not sent to multiple agents.

This is set up like a ***deli-counter ticket system***

Firstly Agents have a Task_ID Global variable, if its None then they will ping the server for a new Task_ID 

The Server creates a new task{Task_id:AgentName} and adds it to a queue

A Seperate Thread continuously copys the current queue and trys processes each Task.  The Task is to find the item with the heighest prioirty that this Agent can process (taking into account the agents installed programs, and how many it has open already). If it finds an item it packages up the CLI and Time locks the item. Otherwise it packages up nothing. And adds its to a completed pile. 

meanwhile the Agents is constantly asking the server if its Task_ID task is in the completed pile, when it finally is, the Server sends the it over, and updates Monday & settings (and releases the Item from its Time Lock)

The agent then runs any CLI (if it has any) and deleates its Task_ID and beings to ping for a new one

**notes!**
if the Task is never collected (30 seconds timeout) the task is delated and the item is released from its lock.


### Design Priciples
1) Monday holds the Ground truths, when making changes, it always updates Monday and waits to download that data. This is slower (mainly for UI responsivness)  but much more reliable.
1.1 Errors break this rule, but this since this doesn't 
2) The Agents and the GUI do no processing data of themselves, the server handles all the 'buisness logic' (Apart from sorting the 'lightpass', the agent does this before Tiffing as it would slow the server down to handel this)
3) The Agent has no idea about whats its settings are, apart from what Programs it has found installed [Tiffer/RC/Marmoset] or any idea about what its doing, it just runs CLI in subprocess and reports back when they have closed.
4) The Gui can be run in multiple instaces from any machne simulatinously. 

```mermaid
graph LR
m[Monday] --Get Items --> A
A --Update changes  --> m
A[Automation Server] --Send CLI --> B[Agent]

B -- Program Finished --> A
B -- Start  --> f

f --Update progress --> e
A --Cache Data --> e
e -- Check Item Progress--> A
e[DataBase]
f[Program]

g[GUI] -- Change Settings --> A
A --live data --> g

