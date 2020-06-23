---
title: "Part 3 - Generating a dungeon"
date: 2020-06-15T10:20:18-07:00
draft: true
---

Remember how we created a wall in the last part? We won't need that anymore. Additionally, our dungeon generator will start by filling the entire map with "wall" tiles and "carving" out rooms, so we can modify our `GameMap` class to fill in walls instead of floors.

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
class GameMap:
    def __init__(self, width: int, height: int):
        self.width, self.height = width, height
-       self.tiles = np.full((width, height), fill_value=tile_types.floor, order="F")
+       self.tiles = np.full((width, height), fill_value=tile_types.wall, order="F")

-       self.tiles[30:33, 22] = tile_types.wall
        ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>class GameMap:
    def __init__(self, width: int, height: int):
        self.width, self.height = width, height
        <span class="crossed-out-text">self.tiles = np.full((width, height), fill_value=tile_types.floor, order="F")</span>
        <span class="new-text">self.tiles = np.full((width, height), fill_value=tile_types.wall, order="F")</span>

        <span class="crossed-out-text">self.tiles[30:33, 22] = tile_types.wall</span>
        ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

Now, on to our dungeon generator.

The original version of this tutorial put all of the dungeon generation in the `GameMap` class. In fact, this was my plan for this tutorial as well. But, as HexDecimal (author of the TCOD library) pointed out in a pull request, that's not very extensible. It puts a lot of code in `GameMap` where it doesn't necessarily belong, and the class will grow to huge proportions if you ever decide to add an alternate dungeon generator.

The better approach is to put our new code in a separate file, and utilize `GameMap` there. Let's create a new file, called `procgen.py`, which will house our procedural generator.

Let's start by creating a class which we'll use to create our rooms. We can call it `RectangularRoom`:

```py3
from typing import Tuple


class RectangularRoom:
    def __init__(self, x: int, y: int, width: int, height: int):
        self.x1 = x
        self.y1 = y
        self.x2 = x + width
        self.y2 = y + height
    
    @property
    def inner(self) -> Tuple[slice, slice]:
        """Return the inner area of this room as a 2D array index."""
        return slice(self.x1 + 1, self.x2), slice(self.y1 + 1, self.y2)
```


The `__init__` function takes the x and y coordinates of the top left corner, and computes the bottom right corner based on the w and h parameters (width and height).

The `inner` property returns two "slices", which represent the inner portion of our room. This is the part that we'll be "digging out" for our room in our dungeon generator. It gives us an easy way to get the area to carve out, which we'll demonstrate soon.

We'll be adding more to this class shortly, but to get us started, that's all we need.

What's with the + 1 on `self.x1` and `self.y1`? Think about what we're saying when we tell our program that we want a room at coordinates (1, 1) that goes to (6, 6). You might assume that would carve out a room like this one (remember that lists are 0-indexed, so (0, 0) is a wall in this case):

``` 
  0 1 2 3 4 5 6 7
0 # # # # # # # #
1 # . . . . . . #
2 # . . . . . . #
3 # . . . . . . #
4 # . . . . . . #
5 # . . . . . . #
6 # . . . . . . #
7 # # # # # # # #
```

That's all fine and good, but what happens if we put a room right next to it? Let's say this room starts at (7, 1) and goes to (9, 6)

``` 
  0 1 2 3 4 5 6 7 8 9 10
0 # # # # # # # # # # #
1 # . . . . . . . . . #
2 # . . . . . . . . . #
3 # . . . . . . . . . #
4 # . . . . . . . . . #
5 # . . . . . . . . . #
6 # . . . . . . . . . #
7 # # # # # # # # # # #
```

There's no wall separating the two\! That means that if two rooms are one right next to the other, then there won't be a wall between them\! So long story short, our function needs to take the walls into account when digging out a room. So if we have a rectangle with coordinates x1 = 1, x2 = 6, y1 = 1, and y2 = 6, then the room should actually look like this:

``` 
  0 1 2 3 4 5 6 7
0 # # # # # # # #
1 # # # # # # # #
2 # # . . . . # #
3 # # . . . . # #
4 # # . . . . # #
5 # # . . . . # #
6 # # # # # # # #
7 # # # # # # # #
```

This ensures that we'll always have at least a one tile wide wall between our rooms, unless we choose to create overlapping rooms. In order to accomplish this, we add + 1 to x1 and y1.

Before we dive into a truly procedurally generated dungeon, let's begin with a simple map that consists of two rooms, connected by a tunnel. We can create a new function to create our dungeon, intuitively named `generate_dungeon`, which will return a `GameMap`. As arguments, it will take the needed width and the height to create the `GameMap`, and it will utilize the `RectangularRoom` class to create the needed rooms. Here's what that looks like:

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
from typing import Tuple

+from game_map import GameMap
+import tile_types


class RectangularRoom:
    def __init__(self, x: int, y: int, width: int, height: int):
        self.x1 = x
        self.y1 = y
        self.x2 = x + width
        self.y2 = y + height
    
    @property
    def inner(self) -> Tuple[slice, slice]:
        """Return the inner area of this room as a 2D array index."""
        return slice(self.x1 + 1, self.x2), slice(self.y1 + 1, self.y2)


+def generate_dungeon(map_width, map_height) -> GameMap:
+   dungeon = GameMap(map_width, map_height)

+   room_1 = RectangularRoom(x=20, y=15, width=10, height=15)
+   room_2 = RectangularRoom(x=35, y=15, width=10, height=15)

+   dungeon.tiles[room_1.inner] = tile_types.floor
+   dungeon.tiles[room_2.inner] = tile_types.floor

+   return dungeon
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>from typing import Tuple

<span class="new-text">from game_map import GameMap
import tile_types</span>


class RectangularRoom:
    def __init__(self, x: int, y: int, width: int, height: int):
        self.x1 = x
        self.y1 = y
        self.x2 = x + width
        self.y2 = y + height
    
    @property
    def inner(self) -> Tuple[slice, slice]:
        """Return the inner area of this room as a 2D array index."""
        return slice(self.x1 + 1, self.x2), slice(self.y1 + 1, self.y2)


<span class="new-text">def generate_dungeon(map_width, map_height) -> GameMap:
    dungeon = GameMap(map_width, map_height)

    room_1 = RectangularRoom(x=20, y=15, width=10, height=15)
    room_2 = RectangularRoom(x=35, y=15, width=10, height=15)

    dungeon.tiles[room_1.inner] = tile_types.floor
    dungeon.tiles[room_2.inner] = tile_types.floor

    return dungeon</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

Now we can modify `main.py` to utilize our now `generate_dungeon` function.

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
#!/usr/bin/env python3
import tcod

from engine import Engine
from entity import Entity
-from game_map import GameMap
from input_handlers import EventHandler
+from procgen import generate_dungeon


def main() -> None:
    ...

    entities = {npc, player}

-   game_map = GameMap(map_width, map_height)
+   game_map = generate_dungeon(map_width, map_height)

    engine = Engine(entities=entities, event_handler=event_handler, game_map=game_map, player=player)
    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>#!/usr/bin/env python3
import tcod

from engine import Engine
from entity import Entity
<span class="crossed-out-text">from game_map import GameMap</span>
from input_handlers import EventHandler
<span class="new-text">from procgen import generate_dungeon</span>


def main() -> None:
    ...

    entities = {npc, player}

    <span class="crossed-out-text">game_map = GameMap(map_width, map_height)</span>
    <span class="new-text">game_map = generate_dungeon(map_width, map_height)</span>

    engine = Engine(entities=entities, event_handler=event_handler, game_map=game_map, player=player)
    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

Now is a good time to run your code and make sure everything works as expected. The changes we've made puts two sample rooms on the map, with our player in one of them (our poor NPC is stuck in a wall though).

![Part 3 - Two Rooms](/images/part-3-two-rooms.png)

I'm sure you've noticed already, but the rooms are not connected. What's the use of creating a dungeon if we're stuck in one room? Not to worry, let's write some code to generate tunnels from one room to another. Add the following methods to `procgen.py`:

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
        ...
        return slice(self.x1 + 1, self.x2), slice(self.y1 + 1, self.y2)


+def create_horizontal_tunnel(gamemap: GameMap, x1: int, x2: int, y: int) -> None:
+   min_x = min(x1, x2)
+   max_x = max(x1, x2) + 1

+   gamemap.tiles[min_x:max_x, y] = tile_types.floor


+def create_vertical_tunnel(gamemap: GameMap, y1: int, y2: int, x: int) -> None:
+   min_y = min(y1, y2)
+   max_y = max(y1, y2) + 1

+   gamemap.tiles[x, min_y:max_y] = tile_types.floor


def generate_dungeon(map_width, map_height) -> GameMap:
    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>        ...
        return slice(self.x1 + 1, self.x2), slice(self.y1 + 1, self.y2)


<span class="new-text">def create_horizontal_tunnel(gamemap: GameMap, x1: int, x2: int, y: int) -> None:
    min_x = min(x1, x2)
    max_x = max(x1, x2) + 1

    gamemap.tiles[min_x:max_x, y] = tile_types.floor</span>


<span class="new-text">def create_vertical_tunnel(gamemap: GameMap, y1: int, y2: int, x: int) -> None:
    min_y = min(y1, y2)
    max_y = max(y1, y2) + 1

    gamemap.tiles[x, min_y:max_y] = tile_types.floor</span>


def generate_dungeon(map_width, map_height) -> GameMap:
    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

Let's put this code to use by drawing a tunnel between our two rooms.

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
    ...
    dungeon.tiles[room_2.inner] = tile_types.floor

+   create_horizontal_tunnel(dungeon, 25, 40, 23)

    return dungeon
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>        ...
    dungeon.tiles[room_2.inner] = tile_types.floor

    <span class="new-text">create_horizontal_tunnel(dungeon, 25, 40, 23)</span>

    return dungeon</pre>
{{</ original-tab >}}
{{</ codetab >}}

Run the project, and you'll see a horizontal tunnel that connects the two rooms. It's starting to come together!

![Part 3 - Two Rooms](/images/part-3-two-rooms-connected.png)

Now that we've demonstrated to ourselves that our room and tunnel functions work as intended, it's time to move on to an actual dungeon generation algorithm. Our will be fairly simple; we'll place rooms one at a time, making sure they don't overlap, and connect them with tunnels.

We'll need two additional things in the `RectangularRoom` class to ensure that two rooms don't overlap. First, we'll need to know where the "center" of each room is. Secondly, we'll want a method that tells us if our room is intersecting with another room.

Enter the following methods into the `RectangularRoom` class:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
+from __future__ import annotations

from typing import Tuple

from game_map import GameMap
import tile_types


class RectangularRoom:
    def __init__(self, x: int, y: int, width: int, height: int):
        self.x1 = x
        self.y1 = y
        self.x2 = x + width
        self.y2 = y + height

+   @property
+   def center(self) -> Tuple[int, int]:
+       center_x = int((self.x1 + self.x2) / 2)
+       center_y = int((self.y1 + self.y2) / 2)

+       return center_x, center_y

    @property
    def inner(self) -> Tuple[slice, slice]:
        """Return the inner area of this room as a 2D array index."""
        return slice(self.x1 + 1, self.x2), slice(self.y1 + 1, self.y2)

+   def intersects(self, other: RectangularRoom) -> bool:
+       """Return True if this room overlaps with another RectangularRoom."""
+       return (
+               self.x1 <= other.x2
+               and self.x2 >= other.x1
+               and self.y1 <= other.y2
+               and self.y2 >= other.y1
+       )


class GameMap:
    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre><span class="new-text">from __future__ import annotations</span>

from typing import Tuple

from game_map import GameMap
import tile_types


class RectangularRoom:
    def __init__(self, x: int, y: int, width: int, height: int):
        self.x1 = x
        self.y1 = y
        self.x2 = x + width
        self.y2 = y + height

    <span class="new-text">@property
    def center(self) -> Tuple[int, int]:
        center_x = int((self.x1 + self.x2) / 2)
        center_y = int((self.y1 + self.y2) / 2)

        return center_x, center_y</span>

    @property
    def inner(self) -> Tuple[slice, slice]:
        """Return the inner area of this room as a 2D array index."""
        return slice(self.x1 + 1, self.x2), slice(self.y1 + 1, self.y2)

    <span class="new-text">def intersects(self, other: RectangularRoom) -> bool:
        """Return True if this room overlaps with another RectangularRoom."""
        return (
                self.x1 <= other.x2
                and self.x2 >= other.x1
                and self.y1 <= other.y2
                and self.y2 >= other.y1
        )</span>


class GameMap:
    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

`center` is a "property", which essentially acts like a read-only variable for our `RectangularRoom` class. It describes the "x" and "y" coordinates of the center of a room. It will be useful later on.

`interects` checks if the room and another room (`other` in the arguments) intersect or not. It returns `True` if the do, `False` if they don't. We'll use this to determine if two rooms are overlapping or not.

We're going to need a few variables to set the maximum and minimum size of the rooms, along with the maximum number of rooms one floor can have. Add the following to `main.py`:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
    ...
    map_height = 45

+   room_max_size = 10
+   room_min_size = 6
+   max_rooms = 30

    tileset = tcod.tileset.load_tilesheet(
    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>    ...
    map_height = 45

    <span class="new-text">room_max_size = 10
    room_min_size = 6
    max_rooms = 30</span>

    tileset = tcod.tileset.load_tilesheet(
    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

At long last, it's time to modify `generate_dungeon` to, well, generate our dungeon\! You can completely remove our old implementation and replace it with the following:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
from __future__ import annotations

+from random import randint
-from typing import Tuple
+from typing import List, Tuple, TYPE_CHECKING

from game_map import GameMap
import tile_types


+if TYPE_CHECKING:
+   from entity import Entity

...

-def generate_dungeon(map_width, map_height) -> GameMap:
-   dungeon = GameMap(map_width, map_height)

-   room_1 = RectangularRoom(x=20, y=15, width=10, height=15)
-   room_2 = RectangularRoom(x=35, y=15, width=10, height=15)

-   dungeon.tiles[room_1.inner] = tile_types.floor
-   dungeon.tiles[room_2.inner] = tile_types.floor

-   create_horizontal_tunnel(dungeon, 25, 40, 23)

-   return dungeon


+def generate_dungeon(
+   max_rooms: int,
+   room_min_size: int,
+   room_max_size: int,
+   map_width: int,
+   map_height: int,
+   player: Entity,
+) -> GameMap:
+   """Generate a new dungeon map."""
+   dungeon = GameMap(map_width, map_height)

+   rooms: List[RectangularRoom] = []

+   for r in range(max_rooms):
+       room_width = randint(room_min_size, room_max_size)
+       room_height = randint(room_min_size, room_max_size)

+       x = randint(0, dungeon.width - room_width - 1)
+       y = randint(0, dungeon.height - room_height - 1)

+       # "RectangularRoom" class makes rectangles easier to work with
+       new_room = RectangularRoom(x, y, room_width, room_height)

+       # run through the other rooms and see if they intersect with this one
+       if any(new_room.intersects(other_room) for other_room in rooms):
+           continue  # this room intersects, so go to the next attempt
+       # if there are no intersections then the room is valid

+       # dig out the rooms inner area
+       dungeon.tiles[new_room.inner] = tile_types.floor

+       # center coordinates of new room, will be useful later
+       (new_x, new_y) = new_room.center

+       if len(rooms) == 0:
+           # this is the first room, where the player starts at
+           player.x = new_x
+           player.y = new_y
+       else:
+           # all rooms after the first:
+           # connect it to the previous room with a tunnel

+           # center coordinates of previous room
+           (prev_x, prev_y) = rooms[-1].center

+           # flip a coin (random number that is either 0 or 1)
+           if randint(0, 1) == 1:
+               # first move horizontally, then vertically
+               create_horizontal_tunnel(dungeon, prev_x, new_x, prev_y)
+               create_vertical_tunnel(dungeon, prev_y, new_y, new_x)
+           else:
+               # first move vertically, then horizontally
+               create_vertical_tunnel(dungeon, prev_y, new_y, prev_x)
+               create_horizontal_tunnel(dungeon, prev_x, new_x, new_y)

+       # finally, append the new room to the list
+       rooms.append(new_room)

+   return dungeon
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>from __future__ import annotations

<span class="new-text">from random import randint</span>
<span class="crossed-out-text">from typing import Tuple</span>
<span class="new-text">from typing import List, Tuple, TYPE_CHECKING</span>

from game_map import GameMap
import tile_types


<span class="new-text">if TYPE_CHECKING:
    from entity import Entity</span>

...

<span class="crossed-out-text">def generate_dungeon(map_width, map_height) -> GameMap:</span>
    <span class="crossed-out-text">dungeon = GameMap(map_width, map_height)</span>

    <span class="crossed-out-text">room_1 = RectangularRoom(x=20, y=15, width=10, height=15)</span>
    <span class="crossed-out-text">room_2 = RectangularRoom(x=35, y=15, width=10, height=15)</span>

    <span class="crossed-out-text">dungeon.tiles[room_1.inner] = tile_types.floor</span>
    <span class="crossed-out-text">dungeon.tiles[room_2.inner] = tile_types.floor</span>

    <span class="crossed-out-text">create_horizontal_tunnel(dungeon, 25, 40, 23)</span>

    <span class="crossed-out-text">return dungeon</span>


<span class="new-text">def generate_dungeon(
    max_rooms: int,
    room_min_size: int,
    room_max_size: int,
    map_width: int,
    map_height: int,
    player: Entity,
) -> GameMap:
    """Generate a new dungeon map."""
    dungeon = GameMap(map_width, map_height)

    rooms: List[RectangularRoom] = []

    for r in range(max_rooms):
        room_width = randint(room_min_size, room_max_size)
        room_height = randint(room_min_size, room_max_size)

        x = randint(0, dungeon.width - room_width - 1)
        y = randint(0, dungeon.height - room_height - 1)

        # "RectangularRoom" class makes rectangles easier to work with
        new_room = RectangularRoom(x, y, room_width, room_height)

        # run through the other rooms and see if they intersect with this one
        if any(new_room.intersects(other_room) for other_room in rooms):
            continue  # this room intersects, so go to the next attempt
        # if there are no intersections then the room is valid

        # dig out the rooms inner area
        dungeon.tiles[new_room.inner] = tile_types.floor

        # center coordinates of new room, will be useful later
        (new_x, new_y) = new_room.center

        if len(rooms) == 0:
            # this is the first room, where the player starts at
            player.x = new_x
            player.y = new_y
        else:
            # all rooms after the first:
            # connect it to the previous room with a tunnel

            # center coordinates of previous room
            (prev_x, prev_y) = rooms[-1].center

            # flip a coin (random number that is either 0 or 1)
            if randint(0, 1) == 1:
                # first move horizontally, then vertically
                create_horizontal_tunnel(dungeon, prev_x, new_x, prev_y)
                create_vertical_tunnel(dungeon, prev_y, new_y, new_x)
            else:
                # first move vertically, then horizontally
                create_vertical_tunnel(dungeon, prev_y, new_y, prev_x)
                create_horizontal_tunnel(dungeon, prev_x, new_x, new_y)

        # finally, append the new room to the list
        rooms.append(new_room)

    return dungeon</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

That's quite a lengthy function! Let's break it down and figure out what's doing what.

```py3
def generate_dungeon(
    max_rooms: int,
    room_min_size: int,
    room_max_size: int,
    map_width: int,
    map_height: int,
    player: Entity,
) -> GameMap:
```

This is the function defition itself. We pass several arguments to it.

* `max_rooms`: The maximum number of rooms allowed in the dungeon. We'll use this to control our iteration.
* `room_min_size`: The minimum size of one room.
* `room_max_size`: The maximum size of one room. We'll pick a random size between this and `room_min_size` for both the width and the height of one room to carve out.
* `map_width` and `map_height`: The width and height of the `GameMap` to create. This is no different than it was before.
* `player`: The player Entity. We need this to know where to place the player.

```py3
    """Generate a new dungeon map."""
    dungeon = GameMap(map_width, map_height)
```

This isn't anything new, we're just creating the initial `GameMap`.

```py3
    rooms: List[RectangularRoom] = []
```

We'll keep a running list of all the rooms.

```py3
    for r in range(max_rooms):
```

We iterate from 0 to `max_rooms` - 1. Our algorithm may or may not place a room depending on if it intersects with another, so we won't know how many rooms we're going to end up with. But at least we'll know that number can't exceed a certain amount.

```py3
        room_width = randint(room_min_size, room_max_size)
        room_height = randint(room_min_size, room_max_size)

        x = randint(0, dungeon.width - room_width - 1)
        y = randint(0, dungeon.height - room_height - 1)

        # "RectangularRoom" class makes rectangles easier to work with
        new_room = RectangularRoom(x, y, room_width, room_height)
```

Here, we use the given minimum and maximum room sizes to set the room's width and height. We then get a random pair of `x` and `y` coordinates to try and place the room down. The coordinates must be between 0 and the map's width and heights.

We use these variables to then create an instance of our `RectangularRoom`.

```py3
        # run through the other rooms and see if they intersect with this one
        if any(new_room.intersects(other_room) for other_room in rooms):
            continue  # this room intersects, so go to the next attempt
```

So what happens if a room *does* intersect with another? In that case, we can just toss it out, by using `continue` to skip the rest of the loop. Obviously there are more elegant ways of dealing with a collision, but for our simplistic algorithm, we'll just pretend like it didn't happen and try the next one.

```py3
        # if there are no intersections then the room is valid

        # dig out the rooms inner area
        dungeon.tiles[new_room.inner] = tile_types.floor

        # center coordinates of new room, will be useful later
        (new_x, new_y) = new_room.center
```

Here, we "dig" the room out. This is similar to what we were doing before to dig out the two connected rooms. We're also grabbing the center `x` and `y` from the room, which we can use to draw our tunnels in a moment.

```py3
        if len(rooms) == 0:
            # this is the first room, where the player starts at
            player.x = new_x
            player.y = new_y
```

We put our player down in the center of the first room we created. If this room isn't the first, we move on to the `else` statement:

```py3
        else:
            # all rooms after the first:
            # connect it to the previous room with a tunnel

            # center coordinates of previous room
            (prev_x, prev_y) = rooms[-1].center

            # flip a coin (random number that is either 0 or 1)
            if randint(0, 1) == 1:
                # first move horizontally, then vertically
                create_horizontal_tunnel(dungeon, prev_x, new_x, prev_y)
                create_vertical_tunnel(dungeon, prev_y, new_y, new_x)
            else:
                # first move vertically, then horizontally
                create_vertical_tunnel(dungeon, prev_y, new_y, prev_x)
                create_horizontal_tunnel(dungeon, prev_x, new_x, new_y)
```

If it's not the first room, we need to connect the room to the previous one. We grab the center `x` and `y` from the room by grabbing the room at the -1 index in the list, which points to the last item.

From there, we make a random, binary choice (like flipping a coin) by getting either integer 0 or 1. The only difference is whether we drap the vertical or horizontal path first. Either way, we tunnel one way, then the other, to create a tunnel that connects both rooms. So long as we always connect to the previous room we created, our dungeon should always stay connected between rooms.

```py3
        # finally, append the new room to the list
        rooms.append(new_room)
```

Regardless if it's the first room or not, we want to append it to the list, so the next iteration can use it.

So that's our `generate_dungeon` function, but we're not quite finished yet. We need to modify the call we make to this function in `main.py`:

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
    ...
    entities = {npc, player}

-   game_map = generate_dungeon(map_width, map_height)
+   game_map = generate_dungeon(
+       max_rooms=max_rooms,
+       room_min_size=room_min_size,
+       room_max_size=room_max_size,
+       map_width=map_width,
+       map_height=map_height,
+       player=player
+   )

    engine = Engine(entities=entities, event_handler=event_handler, game_map=game_map, player=player)
    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>    ...
    entities = {npc, player}

    <span class="crossed-out-text">game_map = generate_dungeon(map_width, map_height)</span>
    <span class="new-text">game_map = generate_dungeon(
        max_rooms=max_rooms,
        room_min_size=room_min_size,
        room_max_size=room_max_size,
        map_width=map_width,
        map_height=map_height,
        player=player
    )</span>

    engine = Engine(entities=entities, event_handler=event_handler, game_map=game_map, player=player)
    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

And that's it\! There's our functioning, albeit basic, dungeon generation algorithm. Run the project now and you should be placed in a procedurally generated dungeon\! Note that our NPC isn't being placed intelligently here, so it may or may not be stuck in a wall.

![Part 3 - Generated Dungeon](/images/part-3-dungeon.png)

*Note: Your dungeon will look different from this one, so don't worry if it doesn't match the screenshot.*

If you want to see the code so far in its entirety, [click
here](https://github.com/TStand90/tcod_tutorial_v2/tree/part-3).

The next part of this tutorial will be available on June 30th. Check back soon!