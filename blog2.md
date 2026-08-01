# darwinism, Part 2: Teaching the agents to think — and where this is all going

*Part 2 of 2. [Part 1](#) built the world and explained why every piece of it is shaped for a
neural network that doesn't exist yet. This part is what happened when I finally put a learner
in it — and where the whole thing is headed.*

---

By the end of Part 1, the world was ready for a brain transplant. Perception came out as
tensors, actions went in as soft numbers, and the rule-based mind driving the animals could be
unplugged and replaced through a single clean interface. The scaffolding was all there.

Now for the honest part: teaching a neural network to actually *live* in this world is a
staircase, and I've climbed some of it and slipped on the rest. Here's the climb, step by
step — the wins and the wall.

## The first Brain: hardcoded rules

Before reaching for a *learnable* brain, We had to prove survival is possible — that these
observations can be mapped to actions that keep an animal alive, by *any* algorithm at all. If
hand-written rules can't do it, no gradient is going to find its way there either. So the first
mind is deliberately hardcoded: a few hundred lines of `if` statements called
[`RuleBrain`](darwinism/sim/brain.py). It's the
existence proof, and once it works it doubles as the teacher every learned brain gets measured
against.

RuleBrain receives a stack of egocentric image channels — terrain, water, food, threat,
mate, plus a radial distance channel — a little `K×K` picture centred on each animal. A neural
network eats that natively. Hand-written `if` statements cannot. So the rule brain's first job
is to *undo* the picture: **decode each channel back into a single target.** which are two operations

- **Nearest.** For threats, water, and mates, all that matters is *which one is closest.* Scan
  the channel, take the non-zero cell with the smallest radial distance, and report back its
  offset and how far away it is. A sheep doesn't care which fox — it cares about the near one.
- **Best.** For grass, closest isn't right. A sparse cell underfoot is worse than a lush patch
  a short walk away, so the score is `value − 0.02 × distance`: pick the *richest* patch in
  sight, with a mild pull toward the near ones. Each species declares which reduction its food
  channel wants, so foxes read "nearest exposed prey" and sheep read "best pasture" through the
  exact same code path.

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

And that's the whole hardcoded mind. No memory, no plan, no state between ticks — a pure function from
"what I can see right now" to six numbers. Crude, and yet enough to keep a world alive.
## Coexistence

This world is "fragile," because it's the most
counterintuitive thing I learned building darwinism.

Getting sheep and foxes to *coexist* — to settle into sustained oscillations instead of one
side wiping out and dragging the other down with it — took a stack of small, realistic
mechanisms, and pulling out almost any single one collapses the predator. A few of them:

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
```python
python -m darwinism.analysis.plots runs/run.csv --out analysis/out # to plot the analysis graph from a csv

darwinism-run --ticks 20000 --monitor # for live plotting
```
![The analysis report](highlights/analysisOut.png)

*The analysis outputs 4 graphs: populations over time, the vegetation
biomass rising and falling, the slow drift of a sheep trait across generations —
the actual signal of evolution happening — and the phase plot, sheep against foxes*

That last graph is the beautiful part. Nowhere in this codebase is there a Lotka–Volterra
equation — no predator–prey model. Just thousands of little agents
eating, fleeing and breeding on their own. The orbit Lotka and Volterra derived with pen and
paper a century ago simply *showed up*, uninvited.

## First question: can a CNN even read this world?

Before handing a network the whole job of staying alive, there's a much smaller question worth
answering: can it *see* at all? Every plan I had downstream assumed a CNN could look at those
egocentric perception grids and recover **where** things are. If that assumption is wrong,
nothing built on top of it can work.

So I built a tiny, focused dataset. Each sample was a fox's-eye view — its terrain channel and
its "where's the prey" channel — paired with a heading, labelled by whether that heading
pointed **toward** the sheep or away from it.

![The fox's vision dataset](highlights/foxVisionLearningDataset.gif)

*A flip-book through the training data: the fox sits at the centre, a sheep blip appears
somewhere in its field of view, and each frame is labelled by whether a candidate heading
points toward the prey or off in the wrong direction. This is the raw material — pure
perception grids — that the model learns to read.*

Then I trained a small vision model on it and asked, *which way is the sheep?*

![The fox learned to point](highlights/foxVisionTrainedResult.png)

*Four unseen scenarios, the sheep placed in a different direction each time. In every one, the
trained model points its arrow straight at the prey. The network isn't pattern-matching a
memorized layout — it's genuinely recovering direction from the spatial arrangement of the
input channels.* [`notebook`](notebooks\vision_learning\fox\train_model.ipynb)

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

And sitting with that is genuinely one of the more haunting things this project has made me
think about. Maybe *this* is part of why life on Earth took so unfathomably long to get
interesting — billions of years of single cells, barely changing, because the first step is
always the improbable one. Not because complexity is hard to *build*, but because the very first
lucky coincidence is so vanishingly rare that only a whole planet running the experiment in
parallel for an eternity is enough to find it at all. And then look what happens once a
breakthrough *is* found: eyes, predation, hard body parts — and the Cambrian goes off like a
bomb, half the body plans we'd ever see appearing in a geological blink. The waiting is the slow
part. Once something basic is finally learned, the explosion is almost immediate.

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
cloning.**.

1. **Collect.** Run the rule brain across many different worlds and record, for every animal
   at every moment, exactly what it saw (the observation) and exactly what it did (the action).
   A giant dataset of "in *this* situation, a competent animal does *that*."
2. **Clone.** Train a neural network — a CNN reading the perception grids, fused with a small
   network reading the scalar state — to predict the teacher's action from the observation.
   Pure supervised learning. No survival, no reward, no simulation in the loop. Just: given
   what you see, output what the teacher would have done. The question here would be whether the neural network be able to approximate the algorithm we defined in RuleBrain.
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
the teacher.** A cloned fox at most performs as best as the hand-written rules are, no better. If I
want animals that discover strategies I never programmed — *smarter* predators which builds different strategies to catch preys, preys that understands terrain and elevation to run faster — I need them to learn from **consequences**, not from imitation. So we need
reinforcement learning.

So I built it. Animals **learn while they're
awake and update while they sleep**: each individual quietly records its own experience as it
lives, and when enough of the population beds down for the night, the simulation pauses and
the policy takes a learning step — a little smarter by dawn. Rewards are inferred by watching what
changes tick to tick: energy gained from a kill or a good graze, thirst quenched,
offspring produced, health lost — and a death penalty that fires for *avoidable* deaths but
forgives dying peacefully of old age. I even built an offline version, so I could log a run
once and then train against that frozen dataset over and over without touching the sim.

It's a clean design. And it **didn't work — not yet.** The returns are a mess of noise. The
policies wobble and slide backward as often as forward. Let me show you what failure looks
like, because the shape of the failure is interesting :

![Reinforcement learning, refusing to converge](highlights/foxRlout.png)

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
almost never happens: because every other agent is also moving, the terrain differs between
worlds, and shifting a couple of cells rewrites the whole observation, an agent essentially
never sees the *same* situation twice. Policy gradient learning is fundamentally a statistical
argument over repeated visits to similar states. Strip out the repetition and each gradient
step is estimated from different samples, so the updates point in inconsistent directions
and mostly cancel.

**3. Chaotic dynamics → catastrophic return variance.** Predator–prey coexistence in this
world is *chaotic and fragile by construction* — small perturbations diverge into wildly
different population trajectories. That means the return-to-go for two nearly identical states
can differ by an order of magnitude: one lucky hunt, or one deep prey trough the agent had no
part in causing, dominates an entire lifetime's reward. A critic exists precisely to absorb
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

So that's the state of the climb: a world purpose-built for learning, learned brains that
successfully imitate a competent teacher and keep their ecosystem alive, proof that the
perception really is legible to a network — and an honest, well-understood wall where
open-ended reinforcement learning meets a chaotic multi-agent world.

But this whole project was never really about sheep and foxes. They're the *simplest possible*
test of an idea that's much bigger — so before I close, let me tell you where this is actually
going.

## From two species to a living planet

Everything I've shown you so far — the noise-carved world, the egocentric eyes, the drop-in
brains, the fragile predator–prey dance — is, in the end, the *smallest interesting version*
of an idea. Two species. A handful of actions. A flat world you walk across.

I built it small on purpose, because I needed the foundations to be right before making them
big. But the whole reason the architecture is shaped the way it is — a world of pure numbers,
a single brain contract, bodies described by an evolvable genome — is that it's meant to grow
into something far more alive. Let me tell you about both horizons: the next steps we are going to take and the distant one we are actually building toward.

## The near horizon: a richer mind, a richer world

As we move forward we are going to imitate the real world as much as possible, for it we need more:

- **Smarter entities — and court, cooperate, or compete.** A "same
  species" perception channel is the small change that unlocks a lot at once. Breeding stops
  being mostly opportunity and becomes real mate *selection*, with animals evaluating partners
  so sexual selection shapes the genome instead of a coin flip. And the same channel opens the
  door to territory, group hunting, and the whole rich game-theory of living alongside your
  rivals. Give them a **learnable signal** on top of it — something one animal emits and others
  can perceive — and you have the raw ingredient of alarm calls, coordination, maybe eventually
  something like language. (There's a whole field of multi-agent communication research to draw
  on here.) And a **truly interactable body** to act on all of it: new actions beyond
  move-and-eat — **swim, dig, pick things up, put them down.** The moment an animal can *change*
  its environment instead of just crossing it, the space of possible strategies explodes.
- **Wider worlds.** More *stuff* in the world, and more world to put it in. Rocks that block
  line of sight. Trees tall enough to feed from — and that fall and open a clearing when they
  die. Objects that are *part of the problem*, not just scenery. And beyond the grassland:
  fish and amphibians in the oceans, burrowing creatures in the depths — whole ecosystems
  stacked in the same planet.
- **Neuroevolution — minds shaped by survival, not by a gradient.** Instead of training a brain
  against a reward function I invented, let the weights (and eventually the architecture) be
  mutated and selected by who actually lives long enough to breed. No hand-written objective,
  just lineages of minds competing in the world. One appealing way to get there is **brains that
  belong to groups**: a herd or a pack shares a brain, and new lineages bud off their own as
  groups split and grow — so social structure ends up encoded directly in how minds are
  inherited.

## The far horizon: a real world, and entities that evolve into it

Here's the vision that actually keeps me up at night. It has two halves, and they only work
together: a world made of **matter** instead of adjectives, and creatures whose **bodies** are
free to change in response to it. A richer world with fixed bodies is just prettier scenery;
evolvable bodies in a world of labels have nothing to evolve *toward*. Put both in and the
world starts posing problems nobody typed in.

### A Real World

Right now a tile is a label — `forest`, `water`, `rock` — a word the systems agree to
interpret. The far-horizon version deletes the labels and builds the world out of **matter**:
fundamental particles and the compounds they form, each with real properties — mass, hardness,
density, melting point, flammability, conductivity, solubility — and rules for how they
interact. Terrain stops being a texture and becomes a *material*. Once that's true, an enormous
amount stops needing to be programmed:

- **Ground you can dig, and stuff you can carry.** Terrain isn't a passable/impassable flag;
  it's a volume of particles you can displace. Dig and you get a hole, and a pile of what used
  to be in it — and that pile, like everything else in the world, is an object with mass. Any
  piece of it can be picked up, dragged or carried if the creature is strong enough, a straight
  comparison of the object's mass against the animal's strength: sticks, stones, leaves, bones,
  corpses, other animals. So burrows, tunnels, dens and warrens become things animals *make*,
  not features I implement — and nest-building isn't a "build nest" action, it's a body strong
  enough to move twigs plus a lineage for whom moving twigs paid off. Dig too deep in the wrong
  material and it collapses on you.
- **Chemistry all the way down.** Things rot, rust, ferment, dissolve, and
  poison. Nutrients cycle because a corpse actually decomposes into the soil that grows the
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
  inherited the habit *and* the body to support it, the elder ones in the colonies communicate informations to the younger ones. This is where the two halves meet — a
  material world hands the creature something to pick up, and an evolvable body eventually
  grows the thing that can pick it up.

The dream is a world where you don't *define* the species. You define the **physics, the
possible parts, and the pressures** — and then you press play and watch the tree of life fill
itself in. Where "Terrestrial", "Aquatic" and "Aerial" aren't categories I typed into a config file,
but *answers the world discovered* to the problem of staying alive in the ground, ocean or air.

That's why every design choice fought so hard to keep the brain and the body as **numbers so a
model can evolve and a network can learn**, rather than logic a programmer has to hand-write.
Hardcoded rules can't grow wings. Distributions, genomes, and differentiable policies can.

## Why bother? What it's actually *for*

This isn't just a very elaborate screensaver. A world like this is a **laboratory for
questions you cannot ethically or practically ask of the real one**:

- **Evolution, sped up and rewound** Simulate the same world thousands of times under different random conditions to separate determinism from chance. Replay entire lineages. Lock a single trait in place and observe the consequences. Millions of years of evolution, compressed into a reproducible controllable user-paced experiment on your laptop.
- **Ecology, under the microscope.** What *exactly* makes a predator and prey coexist versus
  collapse? I found out the hard way that it's a knife's edge of refuge size, hunting
  efficiency, and metabolism — and now I can *vary each one and measure*. That's a real
  ecological question with a real, manipulable answer.
- **The emergence of complexity itself.** At what point does communication appear? When does
  cooperation beat going it alone? What conditions summon social structure, or tool use, or
  something that looks, if you squint, like culture? These are among the deepest questions in
  biology, and a synthetic world lets you *run the experiment* instead of just arguing about it.
- **A sandbox for open-ended AI.** : this is one of the hardest environments I
  can imagine for machine learning — multi-agent, co-evolving, chaotic, sparse-reward. If a
  learning algorithm can produce genuinely adaptive intelligence *here*, its a break through for various fields in science.

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
simple things bumping into each other?* — and ended up building a machine whose entire purpose
is to *keep asking it*, faster and deeper than the real world ever could. Two species and a
grassy plain today. A planet filling itself with invented life tomorrow.

If any of that resonates, come poke at it. Break it. Grow something strange in it. And if it
made you think — a star on the repo genuinely helps, and is much appreciated.

**Thanks for reading both parts.** 🌍

*— darwinism, by Afreedi Z.*
