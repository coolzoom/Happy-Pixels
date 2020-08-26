# Happy-Pixels
This repository was handed to be the past owner. See old thread see here: https://www.ownedcore.com/forums/showthread.php?p=4127807

Welcome to my long-running guide on how to make a WoW bot. This guide will be focused on teaching the community how to make a bot which runs on WoW Classic. This guide is completely open source and free. This guide will be compartmentalized and have new components released as often as I can update. The skills and code will be transferable to retail with some tweaks.

NOTE: If you have questions about the code itself or have an issue with something not working, PLEASE OPEN AN ISSUE ON THE GITHUB PROJECT. If you would like a response, please star the Github project. I will not be solving technical problems on this thread. If you have a non-technical question, such as asking about what the plans for this project are, feel free to post here.

What we are building:
-An AddOn to emit game data (Lua)
-A program to interpret the lua data (NodeJS)
-An engine to complete actions in the game based on the interpreted data (NodeJS)
-A program to emit cursor data (C#)
For more details on the features of this bot, see my other thread: https://www.ownedcore.com/forums/wow...g-partner.html (I have developed a fully functioning pixel bot and I am looking for a partner.)

A few suggested prerequisites (none are required):
-Some knowledge of Lua in order to build and tweak AddOns
-Good knowledge of Javascript to build the engine
-A little knowledge of NodeJS and npm to work with some of the amazing software that is already built for us
-A very small understanding of how image recognition works

1. Making your pixel data
This bot will be a pixel-based bot and display data using the WoW API. This is one of the less detectable ways of displaying data, as we don't have to user a lua unlocker in order to get information. Everything you need can be found in this DataToColor Folder

Contents:
-/libs contains your dependencies
-AddonCore.lua will make registering events easier for us
-DataToColor.lua is the main addon in which we will be writing our code
-DataToColor.toc is our table of contents for our AddOn and contains a list of dependencies
-embeds.xml is going to help with initializing our AddOn
-white.tga is a background image we are going to display as a frame in order to prevent our displayed data from changing hex codes due to any present opacity.

Download this file (/DataToColor) and place it in your AddOn folder for classic.

When you open up your game, you should see something similar to the following colors in your top left corner:

1FirstD2C.png

What is exactly is going on here?
Each square of color here represents unique data in the game. For example, the second and third squares respectively represent the players x/y-coordinate in its current zone. The 15th square represents the character's current level.

How does this work?
By calling the WoW API with commands such as:

```
UnitLevel("player")
```

We get a value returned:

Code:
```
60
```

Try it yourself. In your game client, hit enter to open up the chatbox and type the following:

```
/run print(UnitLevel("player"))
```

In the chatbox, you'll see an output with your level.

We then take the returned value and change it into a color. Every color on your machine is assigned a number from 0 all the way up to 16,777,216 (256 ^ 3). If we found out that our character level is **60** with **UnitLevel("player")** we now have to change that into a hexidecimal value and display that color.

Let's write that function now. I'll be commenting line by line to try and explain what each component is doing. Comments in lua are denoted with a double dash "--" like so:

```
-- This is a comment.
```

Here is our function that will be used to convert decimal values to hexidecimal values:

```
-- i is converted to a 6 digit hexidecimal number, even if the decimal number passed in has fewer or greater digits.
function integerToColor(i)
    -- Checks that the variable passed to this function is a whole number
    if i ~= math.floor(i) then
        -- Prints an error if the variable passed is not an integer 
        error("The number passed to 'integerToColor' must be an integer")
    end
    -- Checks that the integer passed to this function is less than (256 ^ 3) - 1
    if i > (256 * 256 * 256 - 1) then -- the biggest value which can be represented by 3 bytes of color
        error("Integer too big to encode as color")
    end
    
    -- r,g,b are integers of range 0-255
    -- blue (b) represents the 1st & 2nd numbers in a blue, green, red [bgr] hexidecimal color scheme. 
    -- eg: [ff]0000 = 16711680
    local b = Modulo(i, 256)
    -- Changes i to a 4 digit hexidecimal number
    i = math.floor(i / 256)
    -- green (g) represents the 3rd & 4th numbers in a bgr hexidecimal color scheme. 
    -- eg: 00[ff]00 = 65280
    local g = Modulo(i, 256)
    -- Changes i to a 2 digit hexidecimal number
    i = math.floor(i / 256)
    -- red (r) represents the 5th & 6th numbers in a bgr hexidecimal color scheme. 
    -- eg: 0000[ff] = 255
    local r = Modulo(i, 256)
    -- The WoW Color Scheme requires hexidecimal values to be represented on a scale of (0 - 1).
    -- Here we return the table of colors with values ranging from 0 - 1.
    return {r / 255, g / 255, b / 255}
end
```

Now when we pass in 60 to this function, the output should be {0,0,.2352941} which when displayed as a WoW frame, gives us a nice blue.

1.1 Special Data Types
Before I begin talking about how to read the data, I wanted to discuss how to encode data as color when dealing with strings and decimals.

Decimals are somewhat straightforward. Simply multiply your decimal by its number of decimal places. In WoW, I didn't find any decimals I wanted to represent that are larger than 9.99999 (x,y coords, corpse coordinates, etc. are all represented by numbers from [0 - 10].

```
-- This function is able to pass numbers in range 0 to 9.99999 (6 digits)
-- converting them to a 6-digit integer.
function fixedDecimalToColor(f)
    if f > 9.99999 then
        -- error("Number too big to be passed as a fixed-point decimal")
        return {0}
    elseif f < 0 then
        return {0}
    end
    -- "%f" denotes formatting a string as floating point decimal
    -- The number (.5 in this case) is used to denote the number of decimal places
    local f6 = tonumber(string.format("%.5f", 1))
    -- Makes number an integer so it can be encoded
    local i = math.floor(f * 100000)
    return integerToColor(i)
end
```
Strings are a little more complicated, but simple once you understand, and an absolutely invaluable asset. Being able to send your target's name or a zone name over to your engine is paramount to its awesomeness. Here's how it works. All characters on a computer are assigned an ASCII value, for example:
```
The value of "A" is 65.
The value of "B" is 66.
The value of "C" is 67.
```
and so on.

For a complete list of ASCII character codes, see here.

So, if we wanted to represent the string "THRALL", we just need to grab the ASCII values of the letters and concatenate them.
```
T = 84
H = 72
R = 82
A = 65
L = 76
L = 76
```
So, our final integer would be 847282657676, representing "THRALL". As mentioned above, the maximum integer we can store in one color is only ~16,500,000, though. So to deal with this string, we have to split it up into two colors:

847282 == "THR"

657676 == "ALL"

We run these two integers through our integer to color function above, and we are ready to rumble. Below is the function to convert any given 3 string up to 3 characters at a time:

```
-- Pass in a string to get the upper case ASCII values. Converts any special character with ASCII values below 100
function DataToColor:StringToASCIIHex(str)
    -- Converts string to upper case so only 2 digit ASCII values
    -- All lowercase letters have a decimal ASCII value >100, so we only uppercase numbers which are a mere 2 digits long.
    str = string.sub(string.upper(str), 0, 6)
    -- Sets string to an empty string
    local ASCII = ''
    -- Loops through all of string passed to it and converts to upper case ASCII values
    for i = 1, string.len(str) do
        -- Assigns the specific value to a character to then assign to the ASCII string/number
        local c = string.sub(str, i, i)
        -- Concatenation of old string and new character
        ASCII = ASCII .. string.byte(c)
    end
    return tonumber(ASCII)
end
```

2. Extracting pixel data

How our data structure works: 2SecondD2C.png

2.1 Installing dependencies
Now that we have encoded our data with color, we need to decode it into an object our engine can interact with. The first step of this whole process is taking an image of our block of pixel frames. You will need to install the latest version of NodeJS for this part: Download | Node.js

You should also go to the Github project and download the NodeGameBot folder

I decided to use a package called robotjs to read my colors. RobotJS also allows us to code keyboard and mouse inputs/movements in addition to the image reading software it offers. To get robotjs, open up cmd and run
Code:
npm install robotjs
A file called "node_modules" should have appeared in whichever directory you just ran your npm install.

2.2 Locating the pixel frames

We will now have to tell our program where each pixel frame is on the screen so that it knows where to look when interpreting data. I used to do this manually, going pixel by pixel and inputting the frame coordinate. This wasn't a problem, until I started adding and changing data points all of the time for different character profiles. There is a handy function built into DataToColor.lua which will allow you to automatically assign your frame coordinates. In order to do this, which I highly suggest, do the following:

1) Make certain that your left-hand monitor is your primary monitor

2) Put WoW into full-screen mode

3) Open the WoW chat (console) and type:

```
/dc
```

All of the colors should be black, except for the very first square, the metadata square.

4) Open cmd with your WoW window and navigate to \NodeGameBot\Engine and run:
node data.js

Wait a few seconds for a message to appear which should say:
New frame coordinates saved.

If you get an error message, such as alerting you that your game client isn't open, try rectifying whatever issue it gives you and run the same command again.

2.2 Decoding our data

Getting our data is relatively simple now. This will serve as a walk through to the beginDataProcessing function. Follow along in data.js.

We must first find which section of the screen we are going to be reading our pixel data from. We will use the frame coordinates we saved using the /dc command and data.js

```
    let beginDataProcessing = () => {
    // Grabs the coordinate array of objects generated by ConfigureBitmapCoords.js and /dc
    let f = JSON.parse(fs.readFileSync(path.resolve(__basedir, './Database/lib/frameCoordinates.json')))
    // Our goal is to take as small of an image capture as possible to save processing time
    // Because we do not know exactly where our minimum and maximum will be (yet)
    // we assigned a temporarily infinite plane.
    let readerMapXMin = Infinity
    let readerMapYMin = Infinity
    let readerMapXMax = 0
    let readerMapYMax = 0

    // Assigns the dimensions of the smallest bitmap possible that we will pull our data from
    for (let i = 0; i < f.length; i++) {
        // Finds the maximum value of x in our plane
        if (f[i].x > readerMapXMax) {
            readerMapXMax = f[i].x + 1
        }
        // Finds the maximum value of y in our plane
        if (f[i].y > readerMapYMax) {
            readerMapYMax = f[i].y + 1
        }
        // Finds the minimum value of x in our plane
        if (readerMapXMin > f[i].x) {
            readerMapXMin = f[i].x - 1
        }
        // Finds the minimum value of y in our plane
        if (readerMapYMin > f[i].y) {
            readerMapYMin = f[i].y - 1
        }
    }
```
Next, we need to declare which data we would like to grab. I decided to build this profile for a frost mage, so you'll see spells corresponding to the frost mage.

```
 // Variable names that will be stored in the info object
    let xcoord, ycoord, direction, needWater, needFood, needManaGem, targetIsDead, target, targetInCombat, playerInCombat, health, healtlhMax, healthCurrent, mana, manaMax, manaCurrent, level, range, gold, targetFrozen, targetHealth
    let deadStatus, talentPoints, skinning, gossipWindowOpen, itemsAreBroken, bagIsFull, bindingWindowOpen, metaData, zone, flying, frameCols, dataWidth, gossipOptions, corpseX, corpseY, fishing, gameTime, playerClass, unskinnable,
        hearthZone, targetOfTargetIsPlayer, processExitStatus, bitmask

    let spell = {
        melee: {},
        fireball: {}, // Slot 2
        frostbolt: {}, // Slot 3
        fireBlast: {}, // Slot 4
        frostNova: {}, // Slot 5
       // etc..
    }
    let item = []
    let equip = []
    let bags = []
```

We're also going to set an unending interval to frequently update our data in real time, that way, as soon as your character moves or casts a spell, our data in our engine will update in mere milliseconds.
```
    // Globally exported object to be used in other files
    setInterval(() => {
        this.info = {
            metaData: metaData,
            xcoord: xcoord,
            ycoord: ycoord,
            direction: direction,
            itemsAreBroken: itemsAreBroken,
            bagIsFull: bagIsFull,
            zone: zone,
            playerClass: playerClass,
            targetHealth: targetHealth,
            flying: flying,
            targetOfTargetIsPlayer: targetOfTargetIsPlayer
            // etc.. 
        }
    // If you don't set a time on your interval, it will run as fast as possible
    })
    
```

We next create a class which can be used to convert our color cells to actually usable decimal data:
```
    // Class system used to convert colors to usable data
    class SquareReader {
        constructor(pixels) {
            this.pixels = pixels;
        }

        // Robotjs function to find the hexidecimal color of a pixel based on a given x,y coordinate
        getColorAtCell(cell) {
            return this.pixels.colorAt(cell.x, cell.y)
        }
        // Converts a cell's hexideciml color code to decimal data
        getIntAtCell(cell) {
            // Finding the hexidecimal color
            let color = this.getColorAtCell(cell)
            // Converting from base 16 (hexidecimal) to base 10 (decimal)
            return parseInt(color, 16)
        }

        // Converts a cell's hexidecimal color to a 6 point decimal
        getFixedPointAtCell(cell) {
            return this.getIntAtCell(cell) / 100000
        }
        // Converts a cell's hexidecimal color to a 3 character string
        getStringAtCell(cell) {
            // Converting cell coordinates to a hexdecimal color
            let color = this.getIntAtCell(cell)
            // Checking that color exists
            if (color && color !== 0) {
                color = color.toString()
                let word = ''
                // Iterates through each ASCII code and sets it equal to relevant character
                for (let i = 0; i < 3; i++) {
                    let char = color.slice(i * 2, (i + 1) * 2)
                    word = word + String.fromCharCode(char)
                }
                // Removes null bytes if any, but leaves spaces
                word = word.replace('\0', '')
                return word
                // If data input is 0, outputs empty string. e.g. no target
            } else {
                return ''
            }
        }
    }
 
```
Finally, we will update the variables using another interval. There are more complicated structures in the interval than the one I have posted below, so for now, let's just record what we need to walk around in the game: the xcoord, ycoord, direction the player is facing, and their zone.
```
    // Locates the pixels, records their hex, translates values into decimal.
    // xcoord, ycoord, and radians are returned for each respective variable.
    setInterval(() => {
        // Grabs the bitmap of a new frame
        let dataBitmap = robot.screen.capture(readerMapXMin, readerMapYMin, readerMapXMax, readerMapYMax) // Takes a bitmap of the hex encoded variables
        let reader = new SquareReader(dataBitmap)
        xcoord = reader.getFixedPointAtCell(f[1]) * 10
        ycoord = reader.getFixedPointAtCell(f[2]) * 10
        direction = reader.getFixedPointAtCell(f[3])
        zone = reader.getStringAtCell(f[4]).concat(reader.getStringAtCell(f[5])) // Checks current geographic zone
    })
```
Congratulations. You have successfully exported data from the game using only pixel frames. Copy and paste the following to the very bottom of data.js. (You can also change this.info.xcoord/ycoord/direction/whatever to any other given variable in the program, for example, you could use this.info.level to get your character level, or this.info.targetHealth to get the current health percentage of your target)

```
setInterval(() => {
        console.log("xcoord:", this.info.xcoord)
        console.log("ycoord:", this.info.ycoord)
        console.log("direction:", this.info.direction)
}, 100)
```
After you save, once again run:

```
node data.js
```
You should see your desired values being printed out. Enjoy walking around the game and having your data available on your local machine.

3. Recording a path and walking around the game

In this section, I will be referring to this map of coordinates I have recorded: https://i.imgur.com/P6UlPc9.png

This is plainly a map of Durotar, the region south of Orgrimmar. The first image is a scaled down version of the paths that have been recorded. The second image is the ingame map of Durotar. The third image is a blown up version of the map so that you can see a bit better.

The legend for this map only has four components:

-Grey dots represents a recorded coordinate in a path
-The blue dot represents the current position of the player
-The orange dot represents the closest point on the path to the player (seen directly under the blue dot at the very top of the map)
-The red dots represent path nodes, which I will not explain.

Whenever a path intersects with another path, or shares roughly the same start/endpoint as another path, a node is recorded. Nodes (red dots) signal a place in a path which the character can switch off to another path, should it get them to their destination faster. To compute what the fastest route possible is, we use Dijkstra's algorithm, seen here. The technicalities behind this algorithm are fascinating if you'd like to read, but all you need to know is that this is one of the most efficient pathing algorithms in computer science. By using this node system, we can automatically link up all of our paths and make walking across the continent an absolute breeze. I will be uploading all path files at a later date, should you wish to use them or expand the pathing network.

In order to record your first path, you'll need to download two new folders from the project if you haven't already:

1) PathCoordinates This folder contains all of the code we need to process, create, and modify paths.

2) Assorted This folder contains some useful shortcut functions used throughout the project.

Once you have the files downloaded, open up the file containing recordCoordinates.js.

A few lines from the top, you should see a variable called "PATH_FILE_NAME"

Code:
// Input the name of the path you are about to record here:
const PATH_FILE_NAME = "Demo"
Change this to whatever some name of the path you'd like to record. This will save as a .JSON file be default, and there's no need to add a file extension yourself, unless you'd like to change it to your own format, such as .env or .txt

I recommend keeping the settings the same, but if you would like more precise or more randomized walking, you can play around with these settings.
```
// The minimum amount of time to be randomly added in addition to the base increment of recorded points. Default: 0
const PATH_RANDOMIZER_MIN_ADDITION = 0
// The maximum amount of time to be randomly added in addition to the base increment of recorded points. Default: 150.
const PATH_RANDOMIZER_MAX_ADDITION = 150
// // The base amount of time to be incrementally used between each recorded point
const PATH_BASE_MS_INCREMENT = 300
```
After you have decided where you are going to walk, you can now open up your terminal and navigate to the folder which recordCoordinates.js is in, and run the following:
```
node recordCoordinates.js
```
You will have about 0.5-1s before node initializes, and then it's off to the races. Begin moving your character to the desired destination. Once you have arrived there, used Ctrl+C in your terminal to stop recording your path.

Your path will be saved in Database/lib/PathCoordinates/Path text files/ along with the rest of the path files which I have already uploaded. View your path in there, and feel free to remove duplicate points that most likely occur at the beginning end of your path.

3.1 Walking Around

Before we begin this subsection, it is important to note that this is note the path system we will be using later, but a rudimentary implementation to help us understand how navigation works. The ideal path system will use an automatically generated network of paths to determine the shortest path to a given point of interest, determined by any given start or end point. Dynamic pathing is the end implementation, but for now we will use a static pathing system for demonstrative purposes. This means the start and end point are defined before initialization in our .json file, and that we must be located at or near the predefined start point in order to successfully navigate to our destination.

Make sure that you have the following files for this section of the guide:

-NodeGameBot/Engine/roamer.js
-play.js
-A path which you have recorded, should be a .json file.

Make sure that you have WASD movement enabled. If you don't, that's OK. We can talk about how to use other keys for movement soon.

Open up your play.js file. And add the following if not already there:

```
const robot = require("robotjs")
const roam = require("./NodeGameBot/Engine/roamer")

robot.moveMouse(500, 500)
robot.mouseClick()
```
These are the dependencies required to run your walking bot. robotjs taps your keys keys (WASD is the default). The robot.mouseMove and robot.mouseClick functions simply focus your WoW client if not already focused.

Below the above dependencies and function, use the following:

```
let walk = () => {
    roam.givePath("RazorHillToOrgrimmar")
    roam.walkPath(() => {
        console.log("Finished Walking.")
    })
}

setTimeout(() => {
    walk()
}, 500)
```
I have named my path RazorHillToOrgrimmar, but you can use any name of any path you have already generated. It will walk me from the Razor Hill inn entrance to the center of Orgrimmar.

I generated a second folder for my paths in Database/libs/ that I called /Paths/. I highly recommend keeping two separate paths folders for two reasons:

1) If you accidentally overwrite your path when using recordCoordinates, your path will be permanently lost if only using one folder.
2) I like to edit the path manually sometimes. It's a good practice to keep your original path and an edited path. You can always go back to your original if you want to edit out a sharp corner or manually add every fine detailed points such as when walking on a narrow cliff, bridge, ramp or when navigating through crowds of MOBs.

Now, if you run this program with:
```
play.js
```
Your character will walk from their start and end point somewhat seamlessly, granted that you are already standing at your start point when the path begins. So how does this all work?

Navigate to roamer.js and let's have a brief walk through. Most of us took some form of trigonometry as teenagers. This is a combination of very simple trigonometry and arithmetic.

First we add our dependencies:
```
let robot = require("robotjs")
let data = require("./data.js")
let pathModule = require("path")
let fs = require("fs")
```
Once again, robotjs to the rescue. It allows us to tap our keys, get unstuck, and complete every input we need.
We need to import data.js because that tells us which direction our character is facing, what x,y coordinate and zone we're in, and gives us a host of other information that we added earlier.
pathModule and fs are simply used to import other files we'll need later.

Next, we have navVariables:
```
let navVariables = fs.readFileSync(pathModule.resolve(__dirname, '../../Database/lib/roamerConstants.json'))
navVariables = JSON.parse(navVariables)
```
Perhaps the most important part of the walking program, our settings. Every computer is different. I'm fortunate to have a nice computer which reacts very quickly to overadjusting on turns, reading data, etc. You may feel that walking is sometimes a little bit jerky, sudden, or even delayed. Playing around with the settings in the roamerConstants.json file will help fix these problems with a bit of experimentation.

The file also contains settings for using blink if you're a mage as well as different settings for unstucking yourself if your character gets trapped in a corner.


```
const file_path = "../../Database/lib/PathCoordinates/Paths/"
```
This is the predefined path I'm using to call paths by name. This should point to wherever you are storing the paths you would like to use for walking around.

After a slue of variable declarations, we arrive at our givePath function:
```
exports.givePath = (name, reverse) => {
    let ourPath = JSON.parse(fs.readFileSync(pathModule.resolve(__dirname, file_path + name + ".json")))
    if (reverse) {
        exports.assignPath(ourPath[name].reverse())
    } else {
        exports.assignPath(ourPath[name])
    }
}
```
We accept the name of path as a string, for example "ValleyOfTrialsToOrgrimmar" and a second parameter discerning whether we would like to reverse our path, that way we don't have to record two paths.

This simply sorts our path into data that can be easily interpreted by our roamer. Some of that data won't be used until later.
```
// Assigns an array of coordinates to their respective [x,y] positions to be used in our next roaming path
exports.assignPath = (coordinates, indexStart = 1) => {
    console.log(coordinates)
    robot.setKeyboardDelay(0) // default delay time
    resetRoamer = false
    // store last path travel so it can be recalled
    previousPath = path.slice()
    path.length = 0
    path = coordinates.slice()
    // Resets the currentPoint counter when we start a new path
    currentPoint = indexStart
    // stores variables from path array into x,y coords
    for (i = 0; i < path.length; i++) {
        pathx[i] = path[i][0] //stores x coord
        pathy[i] = path[i][1] //stores y coord
        direction[i] = path[i][2] //stores direction
        zone[i] = path[i][3] //stores zone
    }
}
```
Feel free to look at some of the Math functions, but they are all pretty self-explanatory shortcut functions, such as:
```
// Converts from degrees to radians.
Math.degToRad = function(degrees) {
    return degrees * Math.PI / 180
}
```
Now onto the good stuff:

```
class TurningControl
```
This class is filled with methods which allow us to turn in any given direction and to stop turning when we've turned for x amount of time.

```
class WalkingControl
```
This allows us to walk forwards, backwards, stop walking, etc. For example, beginning to walk backwards is as simple as writing:
```
walkingControl.startWalkingBackwards()
```
We only use the following class to detect lingering in a small range of x-y coordinate for too long, as well as attempting to get out of that area and get back on track:
```
class StuckPrevention
```
There is also an inbuilt method for changing zones, as the coordinate system changes for every zone that you are in
```
let keepWalkingZoneChange = (zone) => {
    return new Promise((res, na) => {
        //if current zone is one of the 5 major cities, change the roamer navigation variables to accomodate the new coordinate system
        ROAMVAR = (zoneList.includes(zone)) ? navVariables.special : navVariables.default
        setTimeout(() => {
            res()
        }, 1500)
    })
}
```
I hope this was helpful. If you have a specific question about the functionality or math, please write to me so that I can help explain what is confusing. I plan to go over each component, but I'm a bit low on time at the moment. I hope this is sufficient for those of you asking me about walking around.

The next part of this guide will either focus on either how to begin programming your rotations for combat, or how to make a more robust path system. I haven't decided which to release and refactor yet, so feel free to weigh in.

Please leave your questions as an issue on the github project (and star the project): GitHub - FreeHongKongMMO/Happy-Pixels: Learn how to develop a WoW Bot

As I write this update, it is February 16th, 2020. My writing inactivity for this guide has apparently been noticed by the couple of dozen of you that have messaged me. I regret to say I have become suddenly very busy IRL right as started on the process for open sourcing this project. I plan to continue to update this as frequently as possible. Your messages both here and the stars on Github really pushed me to do this next part over the weekend. I wish I could add more detail for all of you. I will try to update more often.

On another note, some of you mentioned compile issues with the lua program. PLEASE post screenshots of your errors so that I can help you solve them. Please also ensure you are using all of the libraries included, they are absolutely essential!
