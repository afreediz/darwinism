# darwinism: Simulating a world for AI and teaching them to Live.

![The world, alive](https://raw.githubusercontent.com/afreediz/darwinism/main/highlights/demo.gif)

*(White dots are sheep. Red dots are foxes. The green haze is grass. A tiny black dot on an
animal's shoulder marks a male; the ones without are female.)*

I've always been a little obsessed with one question: **where does life come from?** How does
a pile of lifeless compounds ever become something that can move, breed, *think* — and then,
astonishingly, turn around and start unravelling the mystery of what it came from?  
We have
answers, but almost all of them are *hypothesis*: we read the clues left in today's world,
run the story backward to guess how things must once have been, and then hunt for whatever
else that guess would predict. It's clever, but it causes a lot of uncertainties and often introduces many theories where only one of them could be actually true. Wouldn't it be much cooler if we could just *watch* how the living beings are evolving with our bare eyes? But nothing worth watching evolves on a human life-scale and we will never live long enough to see one, so I built a way to cheat.
Consider a virtual world where the physical laws are very close to the real world, which has terrains like mountains, forests, deserts, oceans and rivers with dynamic weather, temperature and seasons interacting with each other, and all of these are built with pure numbers such that an AI model can actually see and learn to survive in it, by exploring, grazing, hunting, fleeing and breeding, developing different strategies to survive and adapt, to exploit the best rewards and to live to its fullest. Much more than all this, here time is something that you can control: make it slow enough to inspect one of the sheep and see the world through its own eyes, or make it fast enough to see generations of change and thousands of virtual days passing before your coffee goes cold. To push further, what if we converted this to a framework where anyone can build around it, add a rabbit and make the fox chase it, add a disease system which will wipe most species every 500 years. That is darwinism, checkout the project to learn more https://github.com/afreediz/darwinism/ .

*In this blog we will explore the fundamentals of darwinism, why each is built the way it is, some internal algorithms and techniques used to build this, the surprising results, and the challenges solved and still existing. Without wasting much time, let's get into it :*

# The World
Before anything can live, there has to be somewhere to live. The world is generated from
**layered simplex noise** — the same family of math that gives video games their rolling,
organic terrain. A few octaves of it become an elevation map; another becomes moisture. Those
two fields are the *only* real inputs. Everything else is derived from them:

- **Biomes** fall out of the combination: wet and warm becomes forest; dry and hot becomes
  desert; cold becomes tundra; high becomes mountains; the rest is open plains.
- **Soil nutrients** and **plant suitability** follow the moisture and the lowlands, so grass
  is lush in the river valleys and sparse on the rocks, nutrients & sunlight are absorbed by the plants to grow, which are then eaten by herbivores.
- **Water** flows downhill. Rivers start from a handful of high-elevation sources and trace
  the steepest descent to the sea, pooling into lakes along the way. The ocean is simply
  everything below sea level.
- **Temperature** is latitude minus altitude — warm near the equator, cold up a mountain.

The order is the one the real world uses. We never *place* a forest. We place moisture and elevation, and the forest is what they imply. The map is a consequence, not a decision, and everything from sea level to soil nutrients is [`configurable`](https://github.com/afreediz/darwinism/blob/main/darwinism/config.py) . The whole world — elevation, biome classification, temperature or even water — is pure numbers in a width x height grid. A sample X-ray of the world :

![The world under the hood](https://raw.githubusercontent.com/afreediz/darwinism/main/highlights/worldXray.png)
*The same world, X-rayed into the raw fields: ocean mask, elevation, moisture, temperature, the biome labels, and the soil-nutrient pool. To the
animals none of this is a picture — it's just numbers on a grid.*

### The Beauty of Dynamicity -

A pretty map is scenery. What makes it a *world* is that it changes. On top of the static terrain runs a small stack of moving parts:

- **Hydrology** — rivers carved from the mountains down to the sea, so fresh water is a
  *place* animals have to travel to, not a stat they top up anywhere.
- **Weather** that rolls between clear, rain and heat which affects surroundings like rain boosts the growth of plants and heat increases the thirst of animals.
- **Seasons** that swing the temperature up and down as virtual years pass.
- **A day/night cycle**, because a world where nothing ever rests is missing half of biology.
- **Grass** growing in every cell up to what that soil can carry — thick on wet grassland, none
  on rock — so grazing strips a patch and the herd has to move on while it grows back.

Each one of these reaches into the animals' arithmetic. It gets cold in
winter and the grass grows slower leading to food shortage. It gets hot and
animals get thirstier which keeps them closer to rivers. Night falls and they sleep. Each of these swings the species through different conditions, forcing them to adapt.

## Seed system, determinism for a chaotic planet

**the world is completely reproducible.** Another cool feature to have is reproducibility even when the world appears random and chaotic. Consider you're running the evolution and spot a beautiful act from an entity which is valuable or serves as the proof of one of your research, but you forgot to record it. It's fine, because even though the whole system appears to run randomly, it is generated from a random number generator which we can initialise with a seed — a unique number which gives the same random sequence of numbers for the same seed. So, actually there is no hidden randomness anywhere: give it the same seed and you get the same run back, tick for tick, down to the last byte.
The framework also provides an infinite number of map possibilities by using the world-seed parameter which is used by opensimplex to generate noise.

## We own the TIME

Now the most interesting part, the world hands you **control over time itself.** The simulation advances in fixed steps — 240 of them make a day in there — and how many of those steps you spend per second of *your*
time is entirely your call — we can configure the `dt` which determines how many steps are taken per iteration. Slow it down and one minute of sitting at your desk buys you ten days inside the world: you can follow a single sheep around, watch it get thirsty, walk to a
river, drink, and wander off to find grass. Turn the dial the other way and that same minute
buys you a thousand days — three years of seasons, birth, starvation and mutation, gone past
while you're still holding the coffee. Nothing about the world changes when you do this. The
sheep doesn't know it's living faster. You've just decided how much of it you want to see.

![Time slowed to a quarter speed](https://raw.githubusercontent.com/afreediz/darwinism/main/highlights/demo0.25x.gif)

![The same world at four times speed](https://raw.githubusercontent.com/afreediz/darwinism/main/highlights/demo4x.gif)

The engine which runs the whole logic and the viewer which presents it as a GUI are separate. There's a
[**live viewer**](https://github.com/afreediz/darwinism/blob/main/darwinism/render) — a window, an observer, drawing the pixels — and a
[**headless runner**](https://github.com/afreediz/darwinism/blob/main/darwinism/sim), which has no window at all and is where the real work happens. It is made this way because of frame rate, which is a ceiling that limits the amount of steps we can do per second; most GUI/game engines set an FPS limit, as nobody can read 10,000 frames a second. Take the drawing away and the ceiling disappears; the loop runs as fast as the CPU allows, with no limit on how fast the world can run.

# The Agents

The world is the stage; the agents are the things that actually live on it. "Agent" here is
just the engineering word for an animal. Each one is a body with state (health, hunger, thirst,
energy, age, sex), a **genome** it inherits and passes on, and a **brain** that decides what to
do next — fed only an **egocentric local view**: a small window of the world centred on the
animal itself, up to the limit of its sensory range. The brain receives a set of observations and hands back what it decided to do based on them.

```
brain.decide(observation) -> action
```

Like the real world, the brain is essentially a black box inside the skull. It receives information from our sensory organs, processes it, and sends commands to the rest of the body.

![The brain↔world contract](https://raw.githubusercontent.com/afreediz/darwinism/main/highlights/brainContract.png)

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
> **foxes**, which eat sheep. But darwinism is a framework, so you can add your own species, for example :

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

A new prey species is only half the story though — somebody has to eat it. Predation is just
another diet source, so pointing the fox at the rabbit is one line: add the rabbit's id to the
fox's prey list.

```python
cfg.species[dw.FOX].diet = [
    dw.PreyFood(prey=[dw.SHEEP, RABBIT],                 # now hunts both
                predation_gain=0.72, hunt_success=0.7, hunt_halfsat=90.0),
]
```

The fox's `food` perception channel now lights up for rabbits as well as sheep, the rabbit's
`threat` channel lights up for foxes, and `consumption` runs the same Type III kill roll
against whichever prey it's actually standing next to. Nothing in the tick order changed.

And because `diet` is a *list*, a species isn't limited to one food source or one kind of food.
Here's a lion that eats sheep **and** foxes — a second-order predator sitting on top of the
existing food chain:

```python
LION = 3
cfg.species[LION] = dw.SpeciesConfig(
    name="lion", species_id=LION, init_count=8,
    diet=[dw.PreyFood(prey=[dw.SHEEP, dw.FOX],           # apex: eats prey AND predator
                      predation_gain=0.8, hunt_success=0.55, hunt_halfsat=60.0)],
    gene_ranges={
        "max_speed":       dw.GeneRange(1.2, 2.6),
        "sensory_range":   dw.GeneRange(12.0, 30.0),
        "size":            dw.GeneRange(1.6, 2.6),
        "aggression":      dw.GeneRange(0.5, 1.0),
        "repro_threshold": dw.GeneRange(0.7, 0.9),
    },
    cluster=(2, 4.0),                                    # a couple of prides
    litter_size=2, repro_cost=0.4, repro_cooldown=200.0,
    hunger_rate=0.0018, base_burn=0.0009, population_cap=60,
)
```

Nobody wrote "lions eat foxes" anywhere in the simulation. The predation graph is *derived*
from the diet declarations (`prey_of` / `predators_of` in `config.py`), so the fox — until now
purely a hunter — automatically grows a `threat` channel in its own perception and starts
fleeing something. A third trophic level appears out of two dataclasses.

## Perception: perceiving the world of numbers
![What an agent actually sees](https://raw.githubusercontent.com/afreediz/darwinism/main/highlights/whatAgentSee_15x15.png)

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

Everything outside the animal's personal vision range is zeroed out. A sheep with weak eyes
sees a small disc; a sharp-eyed fox sees a big one. Nothing is global. Blindness is real —
when an animal doesn't have any mate/food within its vision range, it explores the world.

Why grids? Because a stack of egocentric grids is *exactly* the input a convolutional neural
network eats for breakfast. This is the same data structure as an image with channels. When I
finally drop a CNN into the brain slot, there is **no translation layer** — perception is
already a tensor, already shaped `(channels × height × width)`, already normalized.

Alongside the grids, each animal also gets a short **scalar vector** — its own internal state
and a few global facts: hunger, thirst, energy, health, age, sex, the temperature where it's
standing. Exactly the mix of "what a CNN sees" (the grids) and "what an MLP reasons over" (the
scalars) that a modern policy network is built to fuse.

And crucially, **perception is per-species**. Natively, a fox never carries a "threat" channel it would never use; a sheep never carries channels for prey it doesn't hunt. Each species gets only the
inputs that mean something to it — so a future per-species network has no dead neurons wired to nothing.

## Actions: let brain cook

If perception is the input a network wants, actions are the output a network can actually
*produce*. A decision is a row of six numbers:
```
[x, y, s, e, d, r]
```
- **heading** — a direction to move (two numbers, x and y)
- **speed** — a throttle from 0 (hold still) to 1 (full sprint)
- **eat / drink / reproduce** — three "gates," each a value between 0 and 1
Actions are **continuous and differentiable**, so a neural network can learn by gradient descent.

## The world pushes back: propose vs dispose

There is no free will. If the brain says "eat," should the
animal just… eat? What if there's no food next to it? What if it's lying about being next to a
mate?

The answer is that **the brain only ever *proposes*.** It expresses intent. Whether that
intent becomes reality is decided by the world's [**systems**](https://github.com/afreediz/darwinism/blob/main/darwinism/sim/systems) — separate pieces of logic that enforce the actual rules of physics and biology. The brain says "I want to eat"; the consumption system checks whether there's really grass in the cell the animal is standing on.
The brain says "I want to breed"; the reproduction system checks whether there's genuinely an
eligible partner within arm's reach, and whether both animals have the energy to spare.

This split is doing real work. It means a neural network can be **wrong, greedy, or confused**
— it can want to eat empty ground, or flee toward a wall — and the world simply won't let the
impossible happen. The brain is free to be a messy approximator, because the systems are the ones enforcing what happens. Learning is safe precisely because intent and consequence are different.

> The systems *are* the rules — and they're just a list you hand to the simulation. You can
> add a new law of nature yourself, for example :

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

## Genome

Each animal is born with a genome: speed, vision range, metabolism, body size, lifespan,
aggression, boldness, even a personal "chronotype" that decides whether it's an early riser or a night owl. A high vision gene widens the window of the world that animal can see; a high metabolism gene burns its energy faster.

When two animals breed, their child's genes are a shuffle of both parents' plus a little
random mutation. Nothing selects for anything on purpose. But the animals whose numbers happen
to suit the world leave more children, and their genes become more common. Do that for a few
hundred generations and the population *isn't the same population anymore*. It has adapted.

## Tick system

All of this happens in a fixed heartbeat. Every tick, the world runs the sequence:

![The tick pipeline](https://raw.githubusercontent.com/afreediz/darwinism/main/highlights/tickPipeline.png)

*The world advances the clock and weather, rebuilds its spatial index, shows every animal its
local view, asks each brain to decide, lets the sleepers rest, then moves, feeds, ages, and
breeds everyone — and finally the logger writes down what happened.*

### Optimization: ammonium for the potato

One design pressure shapes a surprising amount of the code: I wanted **thousands** of animals
running for **tens of thousands** of ticks, fast enough to actually iterate on. Two things stand
in the way, and either one alone is enough to bring the whole thing to a crawl and cook your
machine. The first is the classic design pattern used by games in which each **entity is an object**, but that leads to a big list to loop over — where every single tick means walking thousands of scattered objects in Python, which will easily cook the RAM out of your PC. The second is the frequently used function **Nearest X**, which every animal uses in every tick to find the nearest prey, the nearest best grass or the nearest threat. Comparing every animal to every other is **quadratic** — double the population and you quadruple the cost — so the moment the world gets crowded, the exact moment it gets interesting, the machine dies. So overcoming both challenges is the only option to scale, or it will waste both computation and memory. To solve them we use some efficient approaches :

**[Structure Of Array(SoA)](https://github.com/afreediz/darwinism/blob/main/darwinism/sim/entities.py) :** *no animal objects at all.* Every animal is a row index into parallel NumPy arrays — one array of x-positions, one of energies, one of hunger values — which means "make every hungry animal a little hungrier" is a single vectorized operation over the whole population, this approach, with the lightning speed of numpy, makes the whole set of operations a lot faster and smoother.

**[Spatial hash grid](https://github.com/afreediz/darwinism/blob/main/darwinism/sim/grid.py) :** the world is diced into buckets, each animal drops
into its bucket once per tick, and neighbour queries only look at the handful of nearby buckets —
so "who is near me?" costs a constant peek at a few cells instead of a sweep over the whole
population. Perception gets the same treatment — the egocentric grids are built as a sliding window over the padded world fields, masked by each animal's vision disc, all at once instead of per animal.

---

So that's all about the world: perception that's already a tensor, actions that are already
differentiable, a brain that proposes which the world disposes, a deterministic
heartbeat, and an engine optimized enough to evolve populations across lengthier time.

# Teaching the agents to think

Life is no longer scripted. Every creature must perceive its world, make decisions, compete for resources, escape predators, find mates, and adapt to a changing environment. From the emergence of intelligence to the forces of evolution, we will now explore how to mimic decision making and intelligence through simple rules, how to let intelligence naturally evolve through reinforcement learning, and how the species will coexist.

## The first Brain: hardcoded rules

Before reaching for a *learnable* brain, we have to prove survival is possible — that by using *any algorithm* these
observations can be mapped to actions that can keep an animal alive, because if
hand-written rules can't do it, no gradient is going to find its way there either. So the first
mind is deliberately hardcoded: a few hundred lines of `if` statements called
[`RuleBrain`](https://github.com/afreediz/darwinism/blob/main/darwinism/sim/brain.py). It's the
existence proof, and once it works it can also be used as the teacher every learned brain gets measured against.

RuleBrain receives a stack of egocentric image channels — terrain, water, food, threat,
mate, plus a radial distance channel — a little `K×K` picture centred on each animal. A neural
network eats that natively. Hand-written `if` statements cannot. So the rule brain's first job
is to *undo* the picture: **decode each channel back into a single target** — which takes two operations.

The distance itself comes straight out of Pythagoras. For a `K×K` window with radius
`R = (K−1)/2`, every cell has an integer offset `(ox, oy)` from the centre, so its distance from
the animal is just `√(ox² + oy²)` — the hypotenuse of the offset triangle. Those offsets and
their distances depend only on `K`, never on *which* animal is looking, so the brain builds that
stencil once and caches it per window size. Decoding a channel then costs one comparison per
cell and no trigonometry at all.

With the stencil in hand, reducing a channel to a target is an `argmin`/`argmax` over a masked
score, vectorised across every animal at once:

- For threats, water and mates the score is *distance* — mask out empty cells by setting them to
  `+∞`, then `argmin`. A sheep doesn't care which fox; it cares about the nearest one.
- For grass, closest isn't right: a sparse cell underfoot is worse than a lush patch a short walk
  away. So cells below the grazing threshold go to `−∞` and the rest score
  `value − 0.02 × distance` — pick the *richest* patch in sight, with a mild pull toward the near
  ones. Each species declares which reduction its food channel wants, so foxes read "nearest
  exposed prey" and sheep read "best pasture" through the exact same code path.

Either way the reduction hands back four numbers per animal: a `present` flag, the cell offset
`(dx, dy)`, and the distance. `present` matters — the `±∞` trick means "nothing found" falls out
as a non-finite score rather than a special case, and every downstream rule is guarded by it.
Distances are then divided by that animal's own `sensory_range` and clipped to `[0, 1]`, which is
the trick that makes the rest of the brain gene-agnostic: a keen-eyed sheep and a myopic one both
reason in fractions of *their own* vision, so a single constant like `0.25` means "close" for
everybody, and the same thresholds keep working while sensory range mutates across generations.

Two more small pieces of arithmetic run through everything. Headings are always unit vectors:
take the raw offset, divide by its magnitude `√(dx² + dy²)`, and fall back to zero when the
magnitude is degenerate — so the brain only ever emits a *direction*, and the speed throttle stays
an independent column. And urgency is a scalar squeezed out of internal state — `food_need`,
`thirst` — compared against fixed thresholds to decide whether a need is worth acting on, and
which of two competing needs wins. Gates are the same idea in binary: a gate is raised when its
target reads within `0.25` of sensory range *and* the corresponding internal state is unsatisfied
— a proposal the world's own systems then re-check against true adjacency.

With channels decoded, the decision itself is **strict priority arbitration** — four layers,
each overwriting the one below it:

1. **Explore** (the default). A random heading, redrawn every tick. The brain is *stateless*;
   wander-smoothing lives in the movement system, so a fresh random direction each tick still
   produces a smooth walk rather than a seizure.
2. **Reproduce.** If energy is high and hunger and thirst are low and a mate is visible, steer
   at the mate — and once it reads adjacent, raise the reproduce gate.
3. **Needs.** Food drive is `max(hunger, 1 − energy)` — *not* hunger alone, because
   hunger rises too slowly to save an animal whose reserve is draining. Whichever need is more
   pressing, food or water, sets the heading, but only if it's genuinely *urgent*; These details are why the animals spend most of their lives free to breed and explore
   instead of permanently commuting to lunch.
4. **Flee** (overrides everything). If a predator is close — within 45% of sensory range, not
   merely *somewhere* in view — turn and run directly away, and drop every gate: no eating, no
   drinking, no mating while being hunted.

Then one last column: the **speed throttle.** An animal that has *arrived* — gates up, target
adjacent, no urgent unmet need — throttles to zero and stands still to feed or court, saving the
locomotion energy it would burn overshooting the thing it wanted. Everyone else sprints. Which
matters more than it sounds: a fleeing sheep and a chasing fox both run flat out, so the
knife-edge chase balance the whole ecosystem depends on is untouched.

And that's the whole hardcoded mind. No territory awareness, no memory, no plan, no state between ticks — a pure function from "what I can see right now" to six numbers. Crude, and yet enough to keep a world alive.

## Coexistence

Getting sheep and foxes to *coexist* — this was one of the challenging things at the beginning: to settle into sustained oscillations instead of one
side wiping out and dragging the other down with it — took a stack of small, realistic
mechanisms, a combination of different parameters and carefully adjusted values, and pulling out almost any single one collapses the predator. A few of them:

- **A refuge.** Sheep hidden in forest cover are invisible and uncatchable to foxes. This
  reservoir stops the prey from ever going fully extinct. But its *size* is a razor's edge:
  too much refuge and the foxes can never crop enough prey — they starve; too little and the
  foxes over-hunt, crash the sheep, and *then* starve. Around 30% of the land is the sweet
  spot, and I found it the hard way.
- **A saturating hunt.** Foxes get dramatically worse at hunting when prey is scarce — which,
  counterintuitively, is *stabilizing.* Make foxes *better* at low-density hunting and they
  finish off the last sheep in a trough and doom themselves. The least aggressive setting is
  the one that survives.
- **A lean predator.** Foxes burn energy about a third slower than sheep, so a fox can ride
  out a hungry stretch between kills instead of dropping dead at the first dip. This one
  number is the single most sensitive lever in the entire simulation — nudge it up slightly
  and the foxes go extinct around tick 3000, every time.
- **Herds and adult founders** — animals start clustered (a lone disperser can't find a mate
  and the population quietly dies of loneliness), and they start as adults (a population of
  newborns starves before it can breed).

The reason I'm telling you this in the RL chapter is that it's *the same fragility that makes
learning so hard.* A world this sensitive to a single parameter is a world where a learning
agent's mistakes are punished savagely and its luck is amplified wildly. The thing that makes
the ecosystem beautiful to watch is the exact thing that makes it merciless to learn in.

## Analysis

One of the features darwinism provides is the plotting of data into meaningful charts, so we can actually observe useful patterns from it.
```python
python -m darwinism.analysis.plots runs/run.csv --out analysis/out # to plot the analysis graph from a csv
darwinism-run --ticks 20000 --monitor # for live plotting
```
![The analysis report](https://raw.githubusercontent.com/afreediz/darwinism/main/highlights/analysisOut.png)

*The analysis outputs 4 graphs: populations over time, the vegetation
biomass rising and falling, the slow drift of a sheep trait across generations —
the actual signal of evolution happening — and the phase plot, sheep against foxes*

That last graph is the beautiful part. Nowhere in this codebase is there a Lotka–Volterra
equation — no predator–prey model. Just thousands of little agents
eating, fleeing and breeding on their own. The orbit Lotka and Volterra derived with pen and
paper a century ago simply *showed up*, uninvited.

## Vision verification: can a CNN actually read this world?

Before handing a network the whole job of staying alive, there's a much smaller question worth
answering: can it *see* at all? Every plan I had downstream assumed a CNN could look at those
egocentric perception grids and recover **where** things are. If that assumption is wrong,
nothing built on top of it can work.

So I built a tiny, focused dataset. Each sample was a fox's-eye view — its terrain channel and
its "where's the prey" channel — paired with a heading, labelled by whether that heading
pointed **toward** the sheep or away from it.

![The fox's vision dataset](https://raw.githubusercontent.com/afreediz/darwinism/main/highlights/foxVisionLearningDataset.gif)

*A flip-book through the training data: the fox sits at the centre, a sheep blip appears
somewhere in its field of view, and each frame is labelled by whether a candidate heading
points toward the prey or off in the wrong direction. This is the raw material — pure
perception grids — that the model learns to read.*

Then I trained a small vision model on it and asked, *which way is the sheep?*

![The fox learned to point](https://raw.githubusercontent.com/afreediz/darwinism/main/highlights/foxVisionTrainedResult.png)

*Four unseen scenarios, the sheep placed in a different direction each time. In every one, the
trained model points its arrow straight at the prey. The network isn't pattern-matching a
memorized layout — it's genuinely recovering direction from the spatial arrangement of the
input channels.* [`notebook`](https://github.com/afreediz/darwinism/blob/main/notebooks/vision_learning/fox/train_model.ipynb)

*(The sheep got the same treatment — its own vision dataset of terrain, food, and threat
channels — to confirm the prey side of the world is just as readable.)*

## Imitation learning — the first real brain

So a CNN *can* read this world: it sees the grids, and it knows where things are. That's the
capability proven — but knowing where the sheep is isn't the same as knowing how to live. The
next step was a whole brain: eyes, internal state, and six output numbers, driving a real
animal through a real world.

And the way I chose to build it was by *copying* the rule brain rather than letting the animals
figure life out themselves. Why? Because of how staggeringly improbable "figuring it out" is
from scratch.

Think about what a random-weight network has to stumble into. A sheep earns its first reward only by happening to stand on grass *and* happening to raise the eat gate on that same
tick; a fox has to close on an uncovered, actively fleeing sheep with the right gate up; and
reproduction wants a mate, the right sex, energy, cooldowns, and the lever pulled all at once.
A random policy does that essentially never — and until it does, there's no reward, no gradient,
nothing pulling the weights anywhere. That's the sparse-reward exploration trap, and here it's
*lethally* sparse: an agent that hasn't learned to eat starves in a few hundred ticks and takes
its lineage with it.

And sitting with that is genuinely one of the more haunting things this project has made me think about. Maybe this is part of why life on Earth took so unimaginably long to become visibly complex—not because evolution wasn't happening, but because some transitions may have been extraordinarily difficult to cross. For billions of years, life remained microscopic even as profound innovations accumulated beneath the surface. Perhaps a whole planet, running countless chemical experiments in parallel over immense spans of time, was needed for one of those breakthroughs to emerge. And then look what happens once a new possibility exists: eyes, predation, hard body parts—and the Cambrian Explosion unfolds with astonishing speed, many of the major animal body plans appearing within a geological instant. The waiting may be the slow part. Once a genuinely transformative innovation arrives, evolution can move with remarkable speed.

Which is a beautiful thought and a terrible engineering constraint. Evolution paid for that
first step with four billion years and an entire planet of parallel attempts. I have a laptop
and an afternoon.

So: don't make them invent eating. **Show them.** The rule brain already knows how to graze,
drink, court, chase, and flee — competently, if not brilliantly — and it can generate an
essentially unlimited supply of correct demonstrations for free. Imitation turns an impossible
exploration problem into an ordinary supervised one: instead of *searching* for the behaviour,
the network is simply *shown* it, thousands of times, with a dense gradient on every single
sample. Millions of ticks of blind luck collapse into a few epochs of fitting.

That's exactly the bargain: cloning gets a network to competent in an afternoon, and *then* the
hard question — can it get better than its teacher? — becomes a question you can actually ask,
because you're no longer starting from an animal that doesn't know food is edible.

With that motivation, the first real learning experiment was the humble one: **behavioural
cloning.**

1. **Collect.** Run the rule brain across many different worlds and record, for every animal
   at every moment, exactly what it saw (the observation) and exactly what it did (the action).
   A giant dataset of "in *this* situation, a competent animal does *that*."
2. **Clone.** Train a neural network — a CNN reading the perception grids, fused with a small
   network reading the scalar state — to predict the teacher's action from the observation.
   Pure supervised learning. No survival, no reward, no simulation in the loop. Just: given
   what you see, output what the teacher would have done. The question here is whether the neural network is able to approximate the algorithm we defined in RuleBrain.
3. **Evaluate.** Take the trained clone, drop it back into the *real* simulation as the actual
   brain, and see if it can stand on its own.


```python
darwinism-live --sheep-brain notebooks/imitation_learning/sheep.pt --fox-brain notebooks/imitation_learning/fox.pt
```

And it worked. **The cloned brains self-sustain.** Drop a learned sheep policy and a learned
fox policy into a fresh world, let go, and the populations don't collapse — they graze, they
hunt, they breed, they cycle. A neural network, driving animals through the exact same
contract as the rules, keeping an ecosystem alive. That was the first moment the whole "build
it AI-first" bet paid off: the drop-in was genuinely a drop-in.

This is the frontier I've actually reached. The animals can be run by learned brains that
imitate a competent teacher and keep their world running.

*(the framework has a `PolicyBrain` that satisfies the same `Brain`
contract and just wraps a PyTorch network. Checkpoints are TorchScript (JIT) archives — code and
weights together — so the model never has to be redefined to load it.)*

## Reinforcement learning — the way and the wall

Cloning a teacher is wonderful, but it has a ceiling: **the student can never be better than
the teacher.** A cloned fox at most performs as well as the hand-written rules do, no better. If I
want animals that discover strategies I never programmed — *smarter* predators that build different strategies to catch prey, prey that understands terrain and elevation to run faster — I need them to learn from **consequences**, not from imitation. So we need
reinforcement learning.

So I [built it](https://github.com/afreediz/darwinism/blob/main/notebooks/live_learning/train.ipynb). The shape is deliberately simple: animals **learn while they're awake and update
while they sleep**. There is no `RuleBrain` anywhere in the loop — both species are driven by
their own learning policy, warm-started from the imitation clones, so the learner begins as an
animal that already knows how to graze, drink and hunt rather than as noise. Here is what
actually happens inside.

**The policy becomes stochastic.** The cloned network output a single action; the learning one
outputs a *distribution* to sample from, because you cannot learn from consequences without
first trying things. The heading is drawn from a Gaussian around the network's predicted
direction, with a learnable spread that lets the policy decide for itself how exploratory to be;
eat, drink, breed and the speed throttle are Bernoulli coin-flips on their logits. Every decision
therefore comes stamped with a log-probability — the number the update needs later. At
deployment the same network switches back to acting greedily (the mean heading, gates
thresholded), so a trained brain is deterministic and a run stays reproducible.

**Reward and pain.** The simulation was never modified to hand out rewards — that would have
polluted the sim core with training concerns. Instead the trainer photographs every animal's
internal state before the tick and diffs it after. Energy gained is the big positive (a good
graze, a successful hunt). Thirst *actually* relieved earns a bonus — only if the animal was
genuinely thirsty, otherwise drinking at a full stomach would be free money. Producing offspring
earns the largest single reward. Health change is signed: recovery pays, injury costs. A small
living wage rewards simply still being here, set against a gentle penalty proportional to
standing hunger and thirst, so being permanently needy is never comfortable. Then death. A death
is detected by the animal's `birth_id` vanishing from its slot, and it delivers **pain** — but
only if it was *avoidable*. If the animal was already past ~90% of its own `max_age` gene when it
acted, it died of old age, which no decision could have prevented, and it is charged nothing.
Penalising that would teach animals that living long is a mistake.

**The buffer.** Each species owns a rollout buffer, and experience is filed into it **per
individual, keyed by `birth_id`** — not pooled into one anonymous stream of transitions. That
key is what makes a trajectory a trajectory: it lets the trainer follow one specific animal's
life from decision to consequence, and it distinguishes "this slot's animal died" from "this slot
was recycled to a newborn". Each stored step is the observation the animal actually saw (grids
kept as float16 — the perception grids are the memory hog), the action it took, that action's
log-probability, its reward, whether it died, and whether the action was genuinely its own.
Tracking is capped at a fixed number of individuals per species per day, so memory doesn't scale
with the population boom.

**Cumulative reward.** A single tick's reward is almost meaningless here — running toward a
distant water source pays nothing until you arrive. So at night every trajectory is walked
*backwards*, accumulating a discounted return-to-go: each step's credit is its own reward plus
0.99 × the credit of the step after it, reset to zero at a death, since death is terminal and
nothing may bootstrap through it. Each action is therefore judged not by what it immediately
earned but by everything that followed it in that animal's life, geometrically discounted. That
backward accumulation is the entire difference from cloning: the signal is *consequence*, not
imitation.

**Critic-free.** Textbook PPO trains a value network beside the policy whose job is to predict
the expected return, so the learning signal becomes (what actually happened − what was expected)
— a low-variance *advantage*. I deliberately went **without one**, because the introduction of a state-value function makes our imitation-learned actor useless and the whole learning has to start from scratch, which is hard and improbable as we discussed earlier.

**The update.** Then a standard clipped-PPO step over the day's batch. The current policy
re-scores every stored observation–action pair; the ratio between the new and old
log-probabilities says how much more (or less) likely each action has become; and the objective
is *clipped* so that no single update can drag the policy far away from the one that actually
gathered the data — the core PPO insight, since the data is only valid near the policy that
produced it. A few epochs of minibatches, an entropy bonus so exploration doesn't collapse in the
first few nights, gradient-norm clipping — and a **KL trust region** on top: the approximate KL
divergence between the old and new policy is measured every minibatch, and if it blows past the
target the update *aborts before applying the step that broke it*. With no critic to hold
variance down, that brake is what stops one freak batch from wrecking a policy that took twenty
nights to build. Then the buffers clear, the animals sleep through the rest of the night, and at
daybreak collection resumes on the improved policy.

**Night is read off the animals, not a clock.** When at least half the living population has
bedded down — and the day has run some minimum length — that's night: pause and train. When the
sleeping fraction falls back to a fifth, that's dawn: collect. The gap between those two
thresholds is hysteresis, so dusk stragglers can't flicker the state machine back and forth.
Sleeping ticks aren't collected at all, and the occasional animal whose action the sleep system
overrides is flagged and dropped from the gradient entirely — it didn't choose that action, so it
must be neither credited nor blamed for it.

**Episodes.** The outer loop rebuilds the simulation from the identical world seed and run seed,
so every episode replays the byte-identical starting world and only the policies, optimizers and
sampling RNG carry across. Extinction is therefore not fatal to training: the world resets, the
lessons persist, and the species gets another run at the same scenario.

I even built an [**offline version**](https://github.com/afreediz/darwinism/blob/main/notebooks/live_learning/offline/train_fox.ipynb), so I could log a run once and then train against that frozen
dataset over and over without touching the sim — same reward function, same returns-to-go, same
clipped update, with collection and learning simply split in time. (The cloning dataset couldn't
be reused for it: that's a reservoir sample of independent observation→action rows with no
rewards, no temporal order and no `birth_id`, so there is nothing to accumulate a return along.)

RL was clean design but it **didn't work — not yet.** The returns are a mess of noise. The
policies wobble and slide backward as often as forward. Let me show you what failure looks
like, because the shape of the failure is interesting :

![Reinforcement learning, refusing to converge](https://raw.githubusercontent.com/afreediz/darwinism/main/highlights/foxRlout.png)

*Training a fox with reinforcement learning. Mean return per cycle (top-left) should climb;
instead it thrashes. Policy entropy, population, and the KL "trust region" all lurch around.
This is not a bug in the plumbing — it's the world fighting back.*

Four properties of *this particular world* conspire against a simple policy-gradient learner,
and together they're brutal:

**1. Non-stationarity from co-adaptation.** Policy gradient methods assume a *fixed* MDP:
one environment, one reward landscape, sitting still while you climb it. Here that assumption
is simply false. Sheep and foxes train simultaneously, and each species' policy is part of the
other's transition dynamics. When the fox learns to chase, it is implicitly approximating "how
sheep move" — but the sheep update overnight too, and the function the fox was fitting no
longer exists by morning. From either side the environment is a moving target, so a policy that
genuinely improved against last night's opponent can score *worse* tonight through no fault of
its own gradient. The optimisation problem changes shape at the same rate you solve it.

**2. Partial observability + state aliasing.** Each animal perceives the world through an
**egocentric** window — a local patch centred on itself, not the map. Two consequences fall out
of that. First, the same underlying world state produces completely different observations
depending on where an agent is standing, so the policy has to learn the
invariance itself, from data, over many episodes. Second — and worse for RL — the *reverse*
almost never happens: because every other agent is moving, the terrain differs between
worlds, the vegetation changes even when an agent is standing still (through growth or consumption by other agents), the temperature changes, and shifting a couple of cells rewrites the whole observation available in the sensory range of an agent — so an agent essentially
never sees the *same* exact situation twice. Policy gradient learning is fundamentally a statistical argument over repeated visits to similar states, so that a cumulative reward/pain assignment can successfully differentiate the good and bad actions taken in the past. Strip out the repetition and each gradient step is estimated from different samples, so the updates point in inconsistent directions and mostly cancel.

**3. Chaotic dynamics → catastrophic return variance.** Predator–prey coexistence in this
world is *chaotic and fragile by construction* — A critic exists precisely to absorb
this variance by predicting the expected return and turning it into a low-variance advantage.
I deliberately kept the design critic-free so I could keep the imitation-learned weights that
already understand the basics — which leaves a whitened-return baseline as the only variance
reduction, and it isn't enough. The result is exactly what the plots show: high-variance,
destabilising updates, and a KL trust region that keeps early-stopping to protect the policy
from its own noisy gradient.

**4. Sparse, delayed, mis-attributed reward.** The outcomes that actually matter — breeding
successfully, surviving a lean stretch, not starving — arrive hundreds of ticks after the
decisions that caused them, while the tick-to-tick reward is inferred from a snapshot diff that
cannot say *which* earlier action earned a later kill or avoided a later death. That is the
classic temporal credit assignment problem, and it is normally solved either by a value
function that bootstraps reward backwards through time, or by memory that lets a policy
condition on what led here. This learner has neither: it is memoryless and critic-free. Sparse,
delayed, mis-attributed credit is the hardest regime there is, and I walked into it with the
fewest tools.

And bolting memory on is not the free fix it sounds like. A recurrent policy no longer reasons
about the current observation alone — it reasons about a *history*, which means the thing being
learned is a mapping from an entire trajectory, and the gradient has to travel back through
every step of that trajectory to assign blame. That multiplies the credit assignment problem
rather than dissolving it: the hidden state is itself learned, so early-training memory is
noise that the policy nonetheless conditions on, and combined with points 1–3 — a moving
opponent, near-unique observations, chaotic returns — the variance goes *up*, not down. Memory
is almost certainly part of the answer here, but it is the kind of thing you add once you have
a critic to hold the variance down, not before.

None of these are plumbing bugs. They're the *nature of the problem* — the exact reasons that
open-ended, multi-agent, co-evolutionary RL is a genuine research frontier and not a solved
recipe.

---

So, that's everything we've built and achieved so far. This whole project was never really about getting some species to fight with each other, making them coexist so we can play the GOD. They're the *simplest possible* test of an idea that's much bigger — so before I close, let me tell you where this project is actually
going.

## A world for models

Everything I've shown you so far — the noise-carved world, the egocentric eyes, the drop-in
brains, the fragile predator–prey dance — is, in the end, the *smallest interesting version*
of a big idea.

I built it small on purpose, because I needed the foundations to be right before making them
big. But the whole reason the architecture is shaped the way it is — a world of pure numbers,
a single brain contract, bodies described by an evolvable genome — is that it's meant to grow
into something far more alive. Let me tell you about both horizons: the next steps we are going to take, and the distant one we are actually building toward.

## The near horizon: a richer mind, a richer world

As we move forward we are going to imitate the real world as much as possible, and for that we need more:

- **Smarter entities.** A "same
  species" perception channel is the small change that unlocks a lot at once. Breeding stops
  being mostly opportunity and becomes real mate *selection*, with animals evaluating partners
  so sexual selection shapes the genome instead of a coin flip. And the same channel opens the
  door to territory, group hunting, and the whole rich game-theory of living alongside your
  rivals. Give them a **learnable signal** on top of it — something one animal emits and others
  can perceive — and you have the raw ingredient of alarm calls, coordination, maybe eventually
  something like language. (There's a whole field of multi-agent communication research to draw
  on here.) And a **truly interactable body** to act on all of it: new actions beyond
  move-and-eat — **swim, dig, pick things up, put them down.** The moment an animal can *change*
  its environment instead of just crossing it, the space of possible strategies explodes. And
  none of that goes very far without **memory**: today's brain is memoryless, every decision a
  pure function of the current frame. Add a recurrent core — an LSTM behind the same contract —
  and an animal can carry a little of its past with it: the water it drank from a hundred ticks
  ago, the fox it just fled, the patch it already grazed bare.
- **Wider worlds.** More *stuff* in the world, and more world to put it in. Rocks that block
  line of sight. Trees tall enough to feed from — and that fall and open a clearing when they
  die. And expanding the habitable world beyond the land surface:
  fish and amphibians in the oceans, burrowing creatures in the depths — whole ecosystems
  stacked in the same planet.
- **Neuroevolution — minds shaped by survival, not by a gradient.** Instead of training a brain
  against a reward function I invented, let the weights (and eventually the architecture) be
  mutated and selected by who actually lives long enough to breed. No hand-written objective,
  just lineages of minds competing in the world. One appealing way to get there is **brains that
  belong to groups**: a herd or a pack shares a brain, and new lineages bud off their own as
  groups split and grow — so social structure ends up encoded directly in how minds are
  inherited. Smarter models survive longer by building strategies, or models choose strength and give hard competition, and different behaviours can be observed inside the same species group.

## The far horizon: a real world, and entities that evolve into it

Here's the vision that we aim for the future. It has two halves, and they only work
together: a world made of **matter** instead of adjectives, and creatures whose **bodies** are
free to change in response to it.

### A Real World

Right now a tile is a label — `forest`, `water`, `rock` — a word the systems agree to
interpret. The far-horizon version deletes the labels and builds the world out of **matter**:
fundamental particles and the compounds they form, each with real properties — mass, hardness,
density, melting point, flammability, conductivity, solubility — and rules for how they
interact. Terrain stops being a texture and becomes a *material*. Once that's true, an enormous
amount stops needing to be programmed:

- **Ground you can dig, and stuff agents can carry.** Terrain isn't a passable/impassable flag;
  it's a volume of particles you can displace. Dig and you get a hole, and a pile of what used
  to be in it — and that pile, like everything else in the world, is an object with mass. Any
  piece of it can be picked up, dragged or carried if the creature is strong enough, a straight
  comparison of the object's mass against the animal's strength: sticks, stones, leaves, bones,
  corpses, other animals. So burrows, tunnels, dens and warrens become things animals *make*,
  not features I implement — and nest-building isn't a "build nest" action, it's a body strong
  enough to move twigs plus a lineage for whom moving twigs paid off. Dig too deep in the wrong
  material and it collapses on you.
- **Chemistry all the way down.** Nutrients cycle because a corpse actually decomposes into the soil that grows the
  grass. A mineral vein is a real deposit somewhere, not a spawn table — and something that eats
  it gets whatever it was made of. Combustion is just one more reaction in that set: strike the
  right rock hard enough against the right rock and you throw sparks; land a spark on something
  dry and flammable and you have a fire that spreads by the actual moisture and fuel of what's
  around it. Lightning does it by accident. A creature that learns to do it on purpose has
  crossed a line I never coded.

The point isn't realism for its own sake. It's that **a world made of materials has far more
affordances than a world made of labels** — and affordances are what evolution has to be clever
about. Tool use isn't a feature you add to a simulation like this; it's something that becomes
*possible*, and then, if the pressure is right, inevitable.

### An Evolving Entity

The other half is the creature. Today an animal's *body* is fixed: a sheep is a sheep, a fox is
a fox. Genes tune numbers — how fast, how far it sees, how big — but the fundamental body plan
is hardcoded. That's the last big thing I want to tear down. Imagine instead that the **body
itself is part of the genome** — not just parameters, but *structure*: animals that grow new
parts and organs over generations, in response to how they actually live. Then let the world do
the selecting:

- **Water selects for gills.** Creatures that spend generations in the ocean become something
  like **fish** — not because there's an aquatic species in a config file, but because that's
  what survives out there.
- **Height selects for grip, and for wings.** Creatures that climb develop the limbs for it and
  become something like **monkeys**; creatures that jump constantly to escape or to reach food
  slowly develop **wings**, and become something like **birds**.
- **Ground selects for digging — and digging selects for society.** Creatures that burrow and
  live in colonies become something like **ants**, with all the emergent complexity that
  implies.
- **And out at the edge: tools and nests.** Not because I programmed a "build nest" behaviour,
  but because the animals that happened to do it survived better, and their descendants
  inherited the habit *and* the body to support it, and the elder ones in the colonies communicated their knowledge to the younger ones. This is where the two halves meet — a
  material world hands the creature something to pick up, and an evolvable body eventually
  grows the thing that can pick it up.

The dream is a world where you don't *define* the species. You define the **physics, the
possible parts, and the pressures** — and then you press play and watch the tree of life fill
itself in. Where "Terrestrial", "Aquatic" and "Aerial" aren't categories we typed into a config file,
but *answers the world discovered* to the problem of staying alive in the ground, ocean or air.

That's why every design choice fought so hard to keep the brain and the body as **numbers so a
model can learn and evolve**, rather than logic a programmer has to hand-write.
Hardcoded rules can't grow wings. Distributions, genomes, and differentiable policies can.

## Why bother? What it's actually *for*

A world like this is a **laboratory for
questions you cannot practically ask of the real one, simulating scenarios, studying intelligence itself**:

- **Evolution, sped up and rewound.** Simulate the same world thousands of times under different random conditions to separate determinism from chance. Replay entire lineages. Lock a single trait in place and observe the consequences. Millions of years of evolution, compressed into a reproducible, controllable, user-paced experiment on your laptop.
- **Ecology, under the microscope.** What *exactly* makes a predator and prey coexist versus
  collapse? I found out the hard way that it's a knife's edge of refuge size, hunting
  efficiency, and metabolism — and now I can *vary each one and measure*. That's a real
  ecological question with a real, manipulable answer.
- **The emergence of complexity itself.** At what point does communication appear? When does
  cooperation beat going it alone? What conditions summon social structure, or tool use, or
  something that looks, if you squint, like culture? These are among the deepest questions in
  biology, and a synthetic world lets you *run the experiment* instead of just arguing about it.
- **A sandbox for open-ended AI.** This is one of the hardest environments I
  can imagine for machine learning — multi-agent, co-evolving, chaotic, sparse-reward. If a
  learning algorithm can produce genuinely adaptive intelligence *here*, it's a breakthrough for various fields in science.

## Come build in it

darwinism is open source, under a non-commercial research license, and it's a **framework, not
just an app.** You can `pip install` it, spin up a world in a few lines of Python, and extend
it around four clean seams — **species, brains, tick-systems, and heritable traits** — without
touching the core. Add a rabbit as a block of config. Bolt on a disease that sweeps through
the herds. Write your own brain — rules, a neural net, whatever — and watch it trying to survive.

```python
import darwinism as dw

cfg = dw.make_config(world_seed=12345, seed=7)   # reproducible world + run
sim = dw.Simulation(cfg)
for _ in range(20000):
    sim.step()
print(sim.populations)                            # {'sheep': ..., 'fox': ...}
```

Run it headless to blast through generations of data, or watch it live in a window and click
on any animal to see the world through its eyes. Same core either way. Same numbers all the
way down.

---

I set out to answer a childhood-sized question — *how does all this complexity come from
simple things bumping into each other?* — I got answers to many parts of it along the way, but ended up with more questions than before, and a project worth sharing and putting effort into.

If any of that resonates, come poke at it. Break it. Grow something strange in it. And if it
made you think — a star on the repo genuinely helps, and is much appreciated.

**Thanks for reading.** 🌍

*— darwinism, by Afreedi Z.*