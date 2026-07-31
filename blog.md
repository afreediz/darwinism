# darwinism: I built a world so I could watch things evolve

*A two-part series on building an AI friendly ecosystem-and-evolution
simulation — a whole planet of terrain, weather and seasons where agents as animals hunt, flee,
breed, and evolve across generations, where everything is engineered from the ground up so that a neural
network can one day take over their minds.*

*Part 1 of 2 — the idea, the world and the agents*

---

![The world, alive](highlights/demo.gif)

*(White dots are sheep. Red dots are foxes. The green haze is grass. A tiny black dot on an
animal's shoulder marks a male; the ones without are female.)*

## The Idea
I've always been a little obsessed with one question: **where does life come from?** How does
a pile of lifeless compounds ever become something that can move, breed, *think* — and then,
astonishingly, turn around and start unravelling the mystery of what it came from?  
We have
answers, but almost all of them are *hypothesis*: we read the clues left in today's world,
run the story backward to guess how things must once have been, and then hunt for whatever
else that guess would predict. It's clever, but it introduces many theories where only one of them can be actually true. Wouldn't it be much cooler to just *watch*
the animal evolve? But nothing worth evolves on a human lifescale and we will never live live long enough to see one. So I built a way to cheat.  
**In here, time is a dial I get to turn.** Ten
seconds of my afternoon can be a full day inside the world; whole generations can rise and
fall while my coffee goes cold. Evolution stops being a story I read about and becomes
something I can lean in and *watch happen*. The animals in it are built to be AI-friendly, in the sense that its brain never sees a
picture or an object — it only ever sees **numbers**. Perception hands it a small stack of
numeric grids centred on itself (what terrain is around me, where the water is, where the food
is, where the threats are), plus a short vector of how it's doing inside (health, hunger,
thirst, energy, age). The brain's answer is numbers too: which way to head, how hard to run,
and whether to eat, drink or breed. That's the entire conversation between a mind and this
world — numbers in, numbers out — which means the thing doing the deciding can be a page of
hardcoded rules today and a neural network tomorrow, and the world never notices the swap.
To push it further more i converted it to a framework where anyone can build around it, add rabit and make the fox chase, add a desease system which will wipe most species every 500 years, checkout the project to learn more https://github.com/afreediz/darwinism/.

# The World
Before anything can live, there has to be somewhere to live. The world is generated from
**layered simplex noise** — the same family of math that gives video games their rolling,
organic terrain. A few octaves of it become an elevation map; another becomes moisture. Those
two fields are the *only* real inputs. Everything else is derived from them:

- **Water** flows downhill. Rivers start from a handful of high-elevation sources and trace
  the steepest descent to the sea, pooling into lakes along the way. The ocean is simply
  everything below sea level.
- **Temperature** is latitude minus altitude — warm near the equator, cold up a mountain.
- **Biomes** fall out of the combination: wet and warm becomes forest; dry and hot becomes
  desert; cold becomes tundra; high becomes bare mountain; the rest is open plains.
- **Soil nutrients** and **plant suitability** follow the moisture and the lowlands, so grass
  is lush in the river valleys and sparse on the rocks.

I like that ordering because it's the one the real world uses. I never *place* a forest. I
place moisture and elevation, and the forest is what they imply. The map is a consequence, not
a decision.

![The world under the hood](highlights/worldXray.png)

*The same world, X-rayed into the raw fields. ocean mask, elevation, moisture, temperature, the biome labels, and the soil-nutrient pool. To the
animals none of this is a picture — it's just numbers on a grid.*

### The Beauty of Dynamicity -

A pretty map is scenery. What makes it a *world* is that it changes, and that the changes have
teeth. On top of the static terrain runs a small stack of moving parts:

- **Hydrology** — rivers carved from the mountains down to the sea, so fresh water is a
  *place* animals have to travel to, not a stat they top up anywhere.
- **Weather** that rolls between clear, rain and heat.
- **Seasons** that swing the temperature up and down as virtual years pass.
- **A day/night cycle**, because a world where nothing ever rests is missing half of biology.
- **Grass** growing in every cell up to what that soil can carry — thick on wet grassland, none
  on rock — so grazing strips a patch and the herd has to move on while it grows back.

these aren't decoration — each one reaches into the animals' arithmetic. It gets cold in
winter and the grass grows slower leading to food shortage. It gets hot and
animals get thirstier which keeps them closer to rivers. Night falls and they sleep. Each to swing species through different conditions forcing them to adapt.

## Seed system, determinism for a chaotic planet

**the world is completely reproducible.** Every stochastic thing that happens in a run — where
animals spawn, which way the weather turns, who catches whom, who breeds, how a genome mutates
— is drawn from a *single* seeded random generator that gets threaded through every system.
There is no hidden randomness anywhere, give it
the same seed and you get the same run back: tick for tick, down to the last byte.

Sitting next to that is the **world seed** — It is the map generator, feeds the terrain noise and the river carving.

That's what turns the toy into an instrument — hold one variable still, wiggle the other, and
whatever changed is a result rather than luck. Change one mechanism — how much cover the forest
gives, how fast grass regrows — rerun the *same* seed, and the difference in the population
curves belongs to that mechanism and nothing else. Sweep the same setup across fifty run seeds
and a single story turns into a distribution you can actually put a number on. Vary only the
world seed and you're asking whether a wetter planet carries more life, with the rules held
fixed. And swap a neural network in where the rule brain was — same map, same luck — and what
you're measuring is the mind, not the dice.

## We own's the TIME

The world hands you **control over time itself.** The simulation advances in fixed steps —
240 of them make a day in there — and how many of those steps you spend per second of *your*
time is entirely your call. Slow it down and one minute of sitting at your desk buys you ten
days inside the world: you can follow a single sheep around, watch it get thirsty, walk to a
river, drink, and wander off to find grass. Turn the dial the other way and that same minute
buys you a thousand days — three years of seasons, birth, starvation and mutation, gone past
while you're still holding the coffee. Nothing about the world changes when you do this. The
sheep doesn't know it's living faster. You've just decided how much of it you want to see.

![Time slowed to a quarter speed](highlights/demo0.25x.gif)

![The same world at four times speed](highlights/demo4x.gif)

The engine which runs whole logic and the viewer which present it as GUI are seperate. There's a
**live viewer** — a window, an observer, draws the pixels.
**headless runner** — which has no window at all and is where the real work happens. It is made this way because of frame rate, which is a ceiling that could limit us the amount of steps we can do per second, most game/GUI engine sets FPS limit as nobody can read 10,000 frames a second. Take the drawing
away and the ceiling disappears; the loop runs as fast as the cpu allows, no limit on how faster the world can run.


# The Agents

The world is the stage; the agents are the things that actually live on it. "Agent" here is
just the engineering word for an animal. Each one is a body with state (health, hunger, thirst,
energy, age, sex), a **genome** it inherits and passes on, and a **brain** that decides what to
do next — fed only an **egocentric local view**: a small window of the world centred on the
animal itself, out to the limit of its sensory range.

```
brain.decide(observation) -> action
```

The world hands a brain an **observation** — everything the animal can currently
perceive. The brain hands back an **action** — what it wants to do.

![The brain↔world contract](highlights/brainContract.png)

*Perception goes in as pure tensors; a decision comes out as a
plain matrix of numbers; the world's systems enforce what's actually allowed. Swap the middle
box — a rule brain today, a neural network tomorrow — and nothing else has to change. The rule-based brain and a future PyTorch
brain implement the **same interface**, take the **same observation**, and return the **same
action format**. You can choose which one drives each species at launch:*

```bash
# rule-driven sheep vs. a trained neural-network fox
darwinism-run --fox-brain models/fox.pt --ticks 20000
```

A neural fox hunting rule-based sheep. A learned sheep fleeing a scripted fox. Any mix. The
world doesn't know or care which is which — because they all speak through the same doorway.

> Natively there are only two animals — **sheep**, which eat grass, and
> **foxes**, which eat sheep. But darwinism is a framework, so you can add your own species, example :

```python
RABBIT = 2
cfg.species[RABBIT] = dw.SpeciesConfig(
    name="rabbit", species_id=RABBIT, init_count=90,
    diet=[dw.FieldFood("vegetation", eat_value=0.7)],       # herbivore
    gene_ranges={"max_speed": dw.GeneRange(0.8, 2.2), "burrow_depth": dw.GeneRange(0, 1), ...},
)
```

> Declare its diet, its starting count and its genes, and perception, metabolism, reproduction,
> mutation and logging all pick it up for free — including `burrow_depth` as a brand-new
> heritable trait for selection to push on.

## Perception: the world as numbers, seen through one animal's eyes
![What an agent actually sees](highlights/whatAgentSee_15x15.png)

(*A 15×15 slice of what an animal perceives*)

Here's where the "AI-friendly" idea stops being a slogan and becomes concrete geometry.

When an animal perceives the world, it doesn't get a description. It gets a stack of small
**grids** — a square window centred on itself, a handful of cells in every direction, out to
the limit of its own heritable vision. Each grid is one *channel* of information:

- **terrain** — what kind of ground is around me
- **water** — where can I drink
- **food** — for a sheep, where the grass is richest; for a fox, where the exposed prey is
- **threat** — where the predators are (sheep have this; foxes, as apex predators, don't)
- **mate** — where the eligible partners are
- **distance** — how far each cell is from me

Every one of those grids is read the same way: the animal is always at the exact centre, and
distance grows outward from it. That's what "egocentric" means — the window doesn't move
with the map, it moves with the animal, and cell `(0, 0)` is always *the agent itself*:

```
              distance channel, vision range R = 3

              .    .    .   3.0   .    .    .
              .   2.8  2.2  2.0  2.2  2.8   .
              .   2.2  1.4  1.0  1.4  2.2   .
             3.0  2.0  1.0 [ me ] 1.0  2.0  3.0
              .   2.2  1.4  1.0  1.4  2.2   .
              .   2.8  2.2  2.0  2.2  2.8   .
              .    .    .   3.0   .    .    .
```

(*The `distance` channel: zero under the animal's feet, rising with radius, and cut back to
zero outside its heritable vision range and off the edge of the world.*)

Everything outside the animal's personal vision range is zeroed out. A sheep with weak eyes
sees a small disc; a sharp-eyed fox sees a big one. Nothing is global. Blindness is real —
and the time an animal spends *not* seeing anything interesting becomes exploration.

Why grids? Because a stack of egocentric grids is *exactly* the input a convolutional neural
network eats for breakfast. This is the same data structure as an image with channels. When I
finally drop a CNN into the brain slot, there is **no translation layer** — perception is
already a tensor, already shaped `(channels × height × width)`, already normalized. I didn't
bolt on machine-learning compatibility at the end. I made perception *be* the thing an ML
model wants, from the first line of code.

Alongside the grids, each animal also gets a short **scalar vector** — its own internal state
and a few global facts: hunger, thirst, energy, health, age, sex, the temperature where it's
standing, the time of day, the season, and how far its eyes reach. Body sense plus world
sense. Exactly the mix of "what a CNN sees" (the grids) and "what an MLP reasons over" (the
scalars) that a modern policy network is built to fuse.

And crucially, **perception is per-species**. A fox never carries a "threat" channel it would
never use; a sheep never carries channels for prey it doesn't hunt. Each species gets only the
inputs that mean something to it — so a future per-species network has no dead neurons wired
to nothing.

## Actions: let brain cook

If perception is the input a network wants, actions are the output a network can actually
*produce*. decisions is a row of six numbers:

- **heading** — a direction to move (two numbers, x and y)
- **speed** — a throttle from 0 (hold still) to 1 (full sprint)
- **eat / drink / reproduce** — three "gates," each a value between 0 and 1

actions are **continuous and differentiable.** so a
neural network can learn by gradient descent.

## The world pushes back: propose vs dispose

There is no free will, If the brain says "eat," should the
animal just… eat? What if there's no food next to it? What if it's lying about being next to a
mate?

The answer is that **the brain only ever *proposes*.** It expresses intent. Whether that
intent becomes reality is decided by the world's **systems** — separate pieces of logic that
enforce the actual rules of physics and biology. The brain says "I want to eat"; the
consumption system checks whether there's really grass in the cell the animal is standing on.
The brain says "I want to breed"; the reproduction system checks whether there's genuinely an
eligible partner within arm's reach, and whether both animals have the energy to spare.

This split is doing real work. It means a neural network can be **wrong, greedy, or confused**
— it can want to eat empty ground, or flee toward a wall — and the world simply won't let the
impossible happen. The brain is free to be a messy approximator, because the systems are the one enforcing what happens. Learning is safe precisely because intent and consequence are
separated.

> The systems *are* the rules — and they're just a list you hand to the simulation. You can
> add a new law of nature yourself, example :

```python
class MyDiseaseSystem(dw.System):
    """A contagious illness: sick animals drain health and infect close neighbours."""

    def __init__(self, infect_radius=3.0, infect_chance=0.02, drain=0.004):
        self.infect_radius, self.infect_chance, self.drain = infect_radius, infect_chance, drain
        self.sick = None

    def apply(self, ctx):
        ent = ctx.ent
        if self.sick is None: # lazily size the state to the SoA
            self.sick = np.zeros(ent.capacity, dtype=bool)

        idx = ctx.idx # slots alive this tick
        ill = idx[self.sick[idx]]
        ent.health[ill] -= self.drain # illness costs condition
        self.sick[ent.health <= 0.0] = False # the dead stop being contagious

        if len(ill): # spread to nearby healthy animals
            dx = ent.x[idx][:, None] - ent.x[ill][None, :]
            dy = ent.y[idx][:, None] - ent.y[ill][None, :]
            near = (dx * dx + dy * dy <= self.infect_radius ** 2).any(axis=1)
            roll = ctx.rng.random(len(idx)) < self.infect_chance
            self.sick[idx[near & roll]] = True

sim = dw.Simulation(cfg, systems=[*dw.default_pipeline(cfg), MyDiseaseSystem()])
```

> One method, no core edits. Epidemics now run every tick on the same shared state and the same
> seeded RNG — and the same hook takes droughts, wildfires, migration, territoriality. You can
> reorder or replace the built-in systems too.

## Every animal carries a genome

Each animal is born with a genome: speed, vision range, metabolism, body size, lifespan,
aggression, boldness, even a personal "chronotype" that decides whether it's an early riser or a night owl. A high vision gene widens the window of the world that animal can see; a high metabolism gene burns its energy faster.

When two animals breed, their child's genes are a shuffle of both parents' plus a little
random mutation. Nothing selects for anything on purpose. But the animals whose numbers happen
to suit the world leave more children, and their numbers become more common. Do that for a few
hundred generations and the population *isn't the same population anymore*. It has adapted.

### Tick system

All of this happens in a fixed heartbeat. Every tick, the world runs the sequence:

![The tick pipeline](highlights/tickPipeline.png)

*The world advances the clock and weather, rebuilds its spatial index, shows every animal its
local view, asks each brain to decide, lets the sleepers rest, then moves, feeds, ages, and
breeds everyone — and finally logger writes down what happened.*

### Optimization: amonium for the potato

One design pressure shapes a surprising amount of the code: I wanted **thousands** of animals
running for **tens of thousands** of ticks, fast enough to actually iterate on. Two things stand
in the way, and either one alone is enough to bring the whole thing to a crawl and cook your
machine. The first is the classic design **object per animal**, a big list to loop over —
where every single tick means walking thousands of scattered objects in python which will easily cook our RAM out of PC. The second is the question *"who is near me?"*,
which every animal asks every tick about food, threats and mates. Compare every animal to every
other is **quadratic** — double the population and you quadruple the cost — so the
moment the world gets crowded, the exact moment it gets interesting, the machine dies. So overcoming both challenge is the only option to scale, and we did it :

**[Structure Of Array(SoA)](darwinism/sim/entities.py) :** *no animal objects at all.* Every animal is a row index into parallel NumPy
arrays — one array of x-positions, one of energies, one of hunger values — which means "make
every hungry animal a little hungrier" is a single vectorized operation over the whole
population, this approach with the lightning speed of numpy makes the whole operations alot faster and smooth.

**[Spatial hash grid](darwinism/sim/grid.py) :** the world is diced into buckets, each animal drops
into its bucket once per tick, and neighbour queries only look at the handful of nearby buckets —
so "who is near me?" costs a constant peek at a few cells instead of a sweep over the whole
population. Perception gets the same
treatment — the egocentric grids are built as a sliding window over the padded world fields,
masked by each animal's vision disc, all at once instead of per animal.

---

So that's the machine: perception that's already a tensor, actions that are already
differentiable, a brain that proposes which the world disposes, a deterministic
heartbeat, and an engine optimized and faster enough to evolve populations across lengthier time. A world built,
from the first commit, to be *inhabited by something that learns.*

Now the whole stage is set, its time to put some agents with brain in it and watch things evolve, building strategy to make it learn and survive.
That's Part 2 of this series.

---