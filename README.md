# Quick & Dirty: Using Pygame's DirtySprite & LayeredDirty (A tutorial)

Knowing how to use [Pygame]'s various [sprite classes] is essential to getting good and efficient results when working on Pygame games.

Every computer game is made out of a loop that's constantly re-validating the input and the game data and refreshing the display accordingly. In fact, whenever I read the source code for a game I'm surprised by how many tasks get done on each iteration, that is, dozens of times per second.

You can imagine that drawing things on the screen is one of the heavier tasks, and that's where the [`DirtySprite`] and [`LayeredDirty`] classes come in. By using these two instead of the regular `Sprite` and `Group` classes, you can keep track on what parts of the screen need a refresh and what don't, and in turn, your game renders in a smarter and more efficient way.

In this short tutorial, I will show you a very basic use case for [`DirtySprite`] and how I implemented it in an existing game source.

The full source code for the tutorial, as well as a diff for each step, are all available to browse in the [dirty_chimp] project on GitHub.

## Pummel The Chimp

As a simple basic game, we are going to use [chimp.py], an **extremely** basic game based on the infamous "Win $$$" banner ads of the 1990s.

![Pummel The Chimp, And Win $$$](http://www.pygame.org/docs/tut/chimp/chimpshot.gif)

The game comprises of an image of a chimp that's quickly floating from right to left, and a fist image representing the user's mouse cursor trying to punch it.

Writing this game is described in Pete Shinners' [Line By Line Chimp] tutorial, in case you need to wrap your head around basic Pygame principles. Its original source code is available under `examples/chimp.py` in the Pygame distribution.

There are two sprites used in this game: The chimp and the fist.  
The chimp is always undergoing a change, either in its position (running around) or its angle (spinning when punched).  
The fist follows the cursor movement, if there was any, and on a click event, it is lowered by a few pixels for a second, to simulate a hit.

## The dirt

### Using `DirtySprite`

First things first, we're going to use the `DirtySprite` class for both sprites. Of course, `DirtySprite` is a subclass of `Sprite`, so this change would have no effect for now:

```python
class Fist(pygame.sprite.DirtySprite):
    def __init__(self):
        # call DirtySprite initializer
        pygame.sprite.DirtySprite.__init__(self)
```

```python
class Chimp(pygame.sprite.DirtySprite):
    def __init__(self):
        # call DirtySprite initializer
        pygame.sprite.DirtySprite.__init__(self)
```

Voila, `Fist` and `Chimp` are now `DirtySprite`

Now, if we're going to keep the source as it is, the sprites would disappear immediately after first appearing. That is because we're going to make the rendering be dependent on their `dirty` flag (that is, they will only be updated if they have `dirty` = 1).

So to prevent this, for now, let's keep both of them always dirty after an `update`.  
Add this to both classes:
```python
    def update(self):
        ...
        self.dirty = 1
```

Note that as we mentioned before, the chimp is always either moving or spinning. That means we can have it always being `dirty`. So we're basically done with it!

### Grouping the sprites

Now, in order to use the special dirty-rendering logic, we will need to use `LayeredDirty`, which is a group class for holding and handling multiple dirty sprites.

We're going to use it instead of the regular `RenderPlain` (right before the main loop):
```python
    allsprites = pygame.sprite.LayeredDirty((fist, chimp))
```

Now comes the dirty drawing logic:  
Instead of drawing everything to the screen and using `flip` to replace the screen content every time, we will use `LayeredDirty`'s magic at the end of each loop iteration:

```python
        # Draw Everything
        screen.blit(background, (0, 0))
        rects = allsprites.draw(screen)
        pygame.display.update(rects)
```

So we store the group's `draw` results in `rects`, which is basically only the areas of the screen that needs re-rendering. When passing these to `update`, we get exactly what we wanted.

And, because we keep adding to the screen and not necessarily re-drawing it, we're going to need to tell the `LayeredDirty` instance how to clear previous data, that is, it's going to use our predefined `background` when it clears parts of the screen. Let's add this line right before the main loop, after assigning to `allsprites`:
```python
    allsprites.clear(screen, background)
```

We should be able to run the game now. But we want to be smarter about drawing the fist sprite.

### Re-thinking `Fist`'s rendering

Now, this is all nice but we want to be smarter about dirtying up our sprites.

Let's do that with `Fist`: Of course we'll still need to re-draw it whenever the user moved it (with the mouse cursor) or punched (using a click). But in any other case, i.e., when the mouse does not move, there's no need to change the `Fist` object.  
So, let's change `update`, which currently handles every iteration the same (and sets `dirty` to 1).

We're going to move everything that's related to following the user's mouse to a new method, `move`:

```python
    def move(self):
        "move the fist based on the mouse position"
        pos = pygame.mouse.get_pos()
        self.rect.midtop = pos
        self.dirty = 1
```

And `update` would just have to handle the punch animation, when necessary:

```python
    def update(self):
        "handle the punching fist position"
        if self.punching:
            self.rect.move_ip(5, 10)
            self.dirty = 1
```

As you see, we haven't changed `update`'s code at all, apart from splitting it into 2 functions and adding `self.dirty = 1` when something changed. This would tell `LayeredDirty` to return the sprite's rect as something that requires rendering.

So now, whenever the user clicks the mouse, `self.punching` is going to be 1 (as defined in the `punch` method) and later on, `update` will change the sprite accordingly and mark it dirty.

We just need to add a hook to the mouse motion event, so that we'll call `move` whenever the mouse is moved. Let's add these two lines at the bottom of the event handling block:

```python
            elif event.type == MOUSEMOTION:
                fist.move()
```

### Some final tweaks

We're done with most of the work. There are just a few small issues to address at this point:

First, notice that even though we're using the smarter dirty rendering now, we're still re-rendering the background image on every loop iteration:
```python
        # Draw Everything
        screen.blit(background, (0, 0))
```

This is useless, because we already set our `LayeredDirty` to clear with `background`.  
So you can go ahead and remove that line altogether, we don't need it anymore.

Now, we have another small bug after a punch is made (i.e. a click event): The `Fist` sprite is lowered down in the screen, but never goes up. In the older code, it'd fix its position in the next iteration by following the user's cursor again. But in our code, if there was no mouse motion event, we don't bother updating the sprite.

We can fix that by calling `move` after a punch is finished, that is, at the end of `unpunch`:  
```python
    def unpunch(self):
        "called to pull the fist back"
        self.punching = 0
        self.move()  # reset to mouse's position
```

And finally, if you run the code we have so far you might notice that the chimp image can get on top of the fist sprite! That looks strange and unwanted.  
Fortunately, we can fix that by ordering the sprites we pass `LayeredDirty`, so that the chimp sprite is always rendered first:
```python
    allsprites = pygame.sprite.LayeredDirty((chimp, fist))
```

Now it all looks good.

## And we're done.

This wasn't so difficult, was it?  
Running the code now would render exactly the same game as the good old `chimp.py`.

Yup. Exactly the same game.  
But - we were able to learn along the way, and get to know [`DirtySprite`] and [`LayeredDirty`] which are two classes that could provide a great deal of help in our future works.

Also, comparing the new game with the original version, I managed to see a slight improvement in performance: CPU percents were cut down from 5.5% to about 5.0%, and the process' threads number was fixed on 4 instead of around 5-7.  
These are indeed extremely small changes, but keep in mind that (a) our example game is very simple as it is, and (b) it's not the most *classic* example, as we have the chimp sprite that's always dirty and re-rendering.  
Still, I think that's pretty cool.

You can find the full code of this tutorial under the [dirty_chimp] project on GitHub.  
The code for each step was posted at a separate commit, so you can check out the process in the project's [Commit History] page.

[Pygame]: http://www.pygame.org/
[sprite classes]: http://www.pygame.org/docs/ref/sprite.html
[`DirtySprite`]: http://www.pygame.org/docs/ref/sprite.html#pygame.sprite.DirtySprite
[`LayeredDirty`]: http://www.pygame.org/docs/ref/sprite.html#pygame.sprite.LayeredDirty
[chimp.py]: http://www.pygame.org/docs/tut/chimp/chimp.py.html
[Line By Line Chimp]: http://www.pygame.org/docs/tut/chimp/ChimpLineByLine.html
[dirty_chimp]: https://github.com/n0nick/dirty_chimp
[Commit History]: https://github.com/n0nick/dirty_chimp/commits/master