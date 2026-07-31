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

RuleBrain receives a stack of egocentric image channels — terrain, water, food, threat,
mate, plus a radial distance channel — a little `K×K` picture centred on each animal. A neural
network eats that natively. Hand-written `if` statements cannot. So the rule brain's first job
is to *undo* the picture: **decode each channel back into a single target.**

That decoding is two operations, and the difference between them is quietly the difference
between a grazer and a hunter:

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
3. **Needs.** Food drive is `max(hunger, 1 − energy)` — deliberately *not* hunger alone, because
   hunger rises too slowly to save an animal whose reserve is draining. Whichever need is more
   pressing, food or water, sets the heading, but only if it's genuinely *urgent*; a mild need
   doesn't get to cancel courtship. Eating and drinking, though, are **opportunistic**: if
   there's grass or water right here and you're not full, top up regardless of what you were
   doing. That one detail is why the animals spend most of their lives free to breed and explore
   instead of permanently commuting to lunch.
4. **Flee** (overrides everything). If a predator is close — within 45% of sensory range, not
   merely *somewhere* in view — turn and run directly away, and drop every gate: no eating, no
   drinking, no mating while being hunted.

Then one last column: the **speed throttle.** An animal that has *arrived* — gates up, target
adjacent, no urgent unmet need — throttles to zero and stands still to feed or court, saving the
locomotion energy it would burn overshooting the thing it wanted. Everyone else sprints. Which
matters more than it sounds: a fleeing sheep and a chasing fox both run flat out, so the
knife-edge chase balance the whole ecosystem depends on is untouched.

And that's the whole mind. No memory, no plan, no state between ticks — a pure function from
"what I can see right now" to six numbers. Crude, and yet enough to keep a world alive, which
is exactly what makes it a usable teacher.

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

![The analysis report](highlights/analysisOut.png)

*Every run produces this: populations over time (the predator–prey pulse), the vegetation
biomass rising and falling underneath it, the slow drift of a sheep trait across generations —
the actual signal of evolution happening — and the phase plot, sheep against foxes, tracing
the orbit that Lotka and Volterra predicted with pen and paper a century ago. Except this one
is made of thousands of individuals, each seeing the world through its own eyes.*

That last panel still gets me. A loop I first met as an abstract pair of differential
equations, redrawn from the bottom up by a crowd of little agents who have no idea they're
collectively obeying it.

## Imitation learning — and it worked

So why start by *copying* the rules instead of letting the animals figure life out themselves?
Because of how staggeringly improbable "figuring it out" is from scratch.

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

And it could. **The cloned brains self-sustain.** Drop a learned sheep policy and a learned
fox policy into a fresh world, let go, and the populations don't collapse — they graze, they
hunt, they breed, they cycle. A neural network, driving animals through the exact same
contract as the rules, keeping an ecosystem alive. That was the first moment the whole "build
it AI-first" bet paid off: the drop-in was genuinely a drop-in.

This is the frontier I've actually reached. The animals can be run by learned brains that
imitate a competent teacher and keep their world running.

## A detour that convinced me the eyes work

Somewhere in the middle of this, I ran a small experiment that I still think is the most
*charming* result in the whole project. I wanted to know: can a network actually *read* those
egocentric perception grids spatially? Not "does the pipeline run" — does the model genuinely
understand *where* something is from the raw channels?

So I built a tiny, focused dataset. Each sample was a fox's-eye view — its terrain channel and
its "where's the prey" channel — paired with a heading, labelled by whether that heading
pointed **toward** the sheep or away from it.

![The fox's vision dataset](highlights/foxVisionLearningDataset.gif)

*A flip-book through the training data: the fox sits at the centre, a sheep blip appears
somewhere in its field of view, and each frame is labelled by whether a candidate heading
points toward the prey or off in the wrong direction. This is the raw material — pure
perception grids — that the model learns to read.*

Then I trained a small vision model on it and asked it the only question that matters: *given
a fresh view, which way is the sheep?*

![The fox learned to point](highlights/foxVisionTrainedResult.png)

*Four unseen scenarios, the sheep placed in a different direction each time. In every one, the
trained model points its arrow straight at the prey. The network isn't pattern-matching a
memorized layout — it's genuinely recovering direction from the spatial arrangement of the
input channels.*

That figure is small but it settled a big worry. The perception design isn't just
*convenient* for a CNN — it's *legible* to one. The spatial information a hunter needs really
is in there, and a network really can extract it. Everything downstream depends on that being
true, and here it was, being true.

*(The sheep got the same treatment — its own vision dataset of terrain, food, and threat
channels — to confirm the prey side of the world is just as readable.)*

## Reinforcement learning — the way and the wall

Cloning a teacher is wonderful, but it has a ceiling: **the student can never be better than
the teacher.** A cloned fox hunts exactly as well as my hand-written rules, no better. If I
want animals that discover strategies I never programmed — *smarter* predators, preys that understands terrain and elevation to run faster — I need them to learn from **consequences**, not from imitation. I need
reinforcement learning.

So I built it. The setup is, I think, rather elegant on paper. Animals **learn while they're
awake and update while they sleep**: each individual quietly records its own experience as it
lives, and when enough of the population beds down for the night, the simulation pauses and
the policy takes a learning step — a little smarter by dawn. The imitation clones from Step 2
are the warm start, so nobody begins from pure noise. Rewards are inferred by watching what
changes tick to tick: energy gained from a kill or a good graze, thirst genuinely quenched,
offspring produced, health lost — and a death penalty that fires for *avoidable* deaths but
forgives dying peacefully of old age. I even built an offline version, so I could log a run
once and then train against that frozen dataset over and over without touching the sim.

It's a clean design. And it **didn't work — not yet.** The returns are a mess of noise. The
policies wobble and slide backward as often as forward. Let me show you what failure looks
like, because the shape of the failure is genuinely interesting:

![Reinforcement learning, refusing to converge](highlights/foxRlout.png)

*Training a fox with reinforcement learning. Mean return per cycle (top-left) should climb;
instead it thrashes. Policy entropy, population, and the KL "trust region" all lurch around.
This is not a bug in the plumbing — it's the world fighting back.*

Three things about *this particular world* conspire against a simple RL learner, and together
they're brutal:

**1. The ground keeps moving.** Sheep and foxes learn *at the same time.* As the sheep get
better at fleeing, the fox's world quietly changes — the same hunting move that worked last
night works worse tonight, through no fault of the fox's own learning. Every animal is aiming
at a target that's aiming back. The problem isn't stationary, and most of the simple RL theory
quietly assumes it is.

**2. The luck is enormous.** As I'll get to in a moment, predator–prey coexistence here is
*chaotic and fragile by nature.* One lucky hunt or one unlucky prey crash can dominate an
animal's entire lifetime reward. Without a "critic" network to smooth out that variance — and
I deliberately kept the design critic-free to keep the deployable brain simple — the learning
signal is so noisy that the updates are as likely to hurt as help, and the safety brake that's
supposed to stop bad updates just keeps slamming on.

**3. The reward arrives late, and to the wrong address.** The things that actually matter —
successfully raising offspring, surviving a lean stretch — happen *many* moments after the
decisions that caused them. Which earlier step earned that kill? Which earlier turn avoided
that death? The reward can't cleanly say. For a memoryless learner with no value function to
bridge the gap, that's the hardest possible regime: sparse, delayed, and mis-attributed
credit.

None of these are plumbing bugs. They're the *nature of the problem* — the exact reasons that
open-ended, multi-agent, co-evolutionary RL is a genuine research frontier and not a solved
recipe. Naming them precisely is, honestly, part of the result. I know exactly which wall I'm
standing at, and why it's hard, which is a much better place to be than "it just doesn't work."

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
into something far more alive. Let me tell you about both horizons: the next steps I can
already see clearly, and the distant one I'm actually building toward.

## The near horizon: a richer mind, a richer world

The immediate roadmap is about giving the animals *more to think about* and *more to do*:

- **Minds that choose their mates.** Right now, breeding is mostly opportunity. The next step
  is real mate *selection* — animals evaluating partners, so sexual selection becomes a force
  that shapes the genome, not just a coin flip.
- **Brains that belong to groups.** A herd or a pack could share a brain, with new lineages
  budding off their own as groups split and grow — the beginnings of social structure encoded
  directly in how minds are inherited.
- **Animals that can see their own kind — and cooperate, or compete.** Add a "same species"
  perception channel and suddenly the door opens to territory, group hunting, and the whole
  rich game-theory of living alongside your rivals.
- **Communication.** A learnable signal one animal emits and others can perceive — the raw
  ingredient of alarm calls, coordination, maybe eventually something like language. (There's
  a whole field of multi-agent communication research to draw on here.)
- **A world you can manipulate.** Rocks that block line of sight. Trees tall enough to feed
  from — and that fall and open a clearing when they die. Objects that are *part of the
  problem*, not just scenery.
- **New worlds within the world.** Fish and amphibians in the oceans. Burrowing creatures in
  the depths. Whole ecosystems stacked in the same planet.
- **A truly interactable body.** New actions beyond move-and-eat: **swim, dig, pick things up,
  put them down.** The moment an animal can *change* its environment instead of just crossing
  it, the space of possible strategies explodes.

Each of these fits *through the same doorway* from Part 2. A new action is a new column in the
action matrix. A new sense is a new perception channel. A new species is a declaration, not a
rewrite — I can already add one as pure configuration, give it a diet and a set of genes, and
the perception system, the genome, and the physics all adapt around it automatically. The
foundations were laid for exactly this kind of growth.

## The far horizon: bodies that evolve

Here's the vision that actually keeps me up at night.

Today, an animal's *body* is fixed. A sheep is a sheep; a fox is a fox. Genes tune numbers —
how fast, how far it sees, how big — but the fundamental body plan is hardcoded. That's the
last big thing I want to tear down.

Imagine instead that the **body itself is part of the genome** — not just parameters, but
*structure*. Animals that can grow new parts and organs over generations, in response to how
they actually live. And then let the world do the selecting:

- Creatures that spend generations in the ocean gradually develop **gills**, and become
  something like **fish**.
- Creatures that jump constantly to escape or to reach food slowly develop **wings**, and
  become something like **birds**.
- Creatures that climb develop the limbs for it and become something like **monkeys**.
- Creatures that dig and live in colonies become something like **ants** — with all the
  emergent complexity that implies.
- And somewhere out at the edge: creatures that learn to **use tools**, or to **build nests** —
  not because I programmed a "build nest" behaviour, but because the animals that happened to
  do it survived better, and their descendants inherited the habit and the body to support it.

The dream is a world where you don't *define* the species. You define the **physics, the
possible parts, and the pressures** — and then you press play and watch the tree of life fill
itself in. Where "bird" and "fish" and "digger" aren't categories I typed into a config file,
but *answers the world discovered* to the problem of staying alive in the ocean, or the air,
or underground.

That's why every design choice fought so hard to keep the brain and the body as **numbers a
model can evolve and a network can learn**, rather than logic a programmer has to hand-write.
Hardcoded rules can't grow wings. Distributions, genomes, and differentiable policies can.

## Why bother? What it's actually *for*

This isn't just a very elaborate screensaver. A world like this is a **laboratory for
questions you cannot ethically or practically ask of the real one**:

- **Evolution, sped up and rewound.** Run the same world a thousand times with different luck
  and ask which outcomes were destiny and which were chance. Replay a lineage. Freeze a trait
  and see what breaks. Deep time, on a laptop, perfectly reproducible.
- **Ecology, under the microscope.** What *exactly* makes a predator and prey coexist versus
  collapse? I found out the hard way that it's a knife's edge of refuge size, hunting
  efficiency, and metabolism — and now I can *vary each one and measure*. That's a real
  ecological question with a real, manipulable answer.
- **The emergence of complexity itself.** At what point does communication appear? When does
  cooperation beat going it alone? What conditions summon social structure, or tool use, or
  something that looks, if you squint, like culture? These are among the deepest questions in
  biology, and a synthetic world lets you *run the experiment* instead of just arguing about it.
- **A sandbox for open-ended AI.** And selfishly: this is one of the hardest environments I
  can imagine for machine learning — multi-agent, co-evolving, chaotic, sparse-reward. If a
  learning algorithm can produce genuinely adaptive intelligence *here*, that's worth knowing.
  The wall I hit earlier is a wall the whole field is still climbing.

## Come build in it

darwinism is open source, under a non-commercial research license, and it's a **framework, not
just an app.** You can `pip install` it, spin up a world in a few lines of Python, and extend
it around four clean seams — **species, brains, tick-systems, and heritable traits** — without
touching the core. Add a rabbit as a block of config. Bolt on a disease that sweeps through
the herds. Write your own brain — rules, a neural net, whatever — and watch it try to survive.

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
