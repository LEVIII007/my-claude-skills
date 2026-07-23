# Shashank's Real X Posts

These are real posts by @tyagi_Shashankk. Use them as direct reference when writing new posts. Do not summarize. Read each one carefully.

---

## Post 1 — outbox table pattern

outbox table pattern is one of those things i'd never seen before joining a real production codebase.

lot of systems are silently loses events and nobody notices until something goes wrong.

wrote about it in this article

---

## Post 2 — new job learnings

my new job is teaching me something new every single day.

new patterns. how things actually work in production. the stuff no tutorial ever showed me.

I am thinking of writing about whatever catches my eye in the codebase and architecture.

let's see how consistent i can stay

---

## Post 3 — blog automation (finalized)

as a co-founder of @VerlyAI with no big team, i can not write blogs myself.

can't even spend 5 minutes per blog. that is not scalable at all.

so i built an automation which runs every week and publishes 10 new blogs on verly ai.

with proper verified content. with images, diagrams. everything.

it checks competitor blogs → finds anything new they published → modifies the topic for verly ai → generates the blog → critiques it for human touch → publishes.

all of it using claude code routines. zero manual effort.

happy to share it if anyone needs it 🙋

---

## Post 4 — page context awareness

if you run a business that's a bit hard to understand at first glance, this one's for you.

most chatbots don't know what page the user is on.

user lands somewhere confusing. doesn't know what to do. asks the bot. gets a generic answer. leaves.

we just shipped page context awareness for @VerlyAI.

the agent reads the current page → understands what's on it → guides the user to the exact button or section they need.

no more "i don't know where to click"

---

## Post 5 — identity verification

we shipped something crazy in @VerlyAI.

most chatbots treat every user the same.

anonymous. no context. no idea who they're talking to.

so when someone asks "why did my payment fail" or "can you cancel my order" — the bot says "please contact support."

because it genuinely doesn't know who's asking.

this is the thing that blocks companies from giving their chatbot any real power.

now the agent knows exactly who it's talking to.

their account. their orders. their billing. their history.

and it's secure. your user's data never leaks to verly.

so the chatbot can actually do things.

check why a payment failed. track a specific order. perform actions inside that user's account.

not for "a user". for this user. right now.

---

## Notes on what makes these posts work

- Hook is always something he personally experienced or felt — never abstract
- Vulnerability is real ("let's see how consistent i can stay")
- The → arrow format is his natural way of showing steps
- "happy to share it if anyone needs it 🙋" is his signature CTA for builds
- Short lines. No padding. Says what needs to be said and stops.
- Lowercase throughout. Reads like a person thinking out loud.
