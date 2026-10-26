# Figuring out tags in Studio 5000

I'm coming from RSLogix 500, so I'm trying to connect what I'm seeing in Studio 5000 to something I already understand. I kept getting stuck because we were talking about names, scope, data types, physical inputs, and internal bits all at once.

This is the sequence of questions I worked through. It's a record of how I'm trying to piece this together, not me saying I've mastered all of it.

## The questions I was asking

1. Where is my tag? Why did it show up in Controller Tags for my teacher but not for me?
2. Can I change its scope after creating it?
3. Why did cutting and pasting it do nothing?
4. What does an alias actually mean? How is it different from the other tags I created?
5. Does creating a tag on the rung work differently from creating it inside Controller Tags?
6. Can I think of a base BOOL like an internal bit in 500, and an alias like a symbol for an input?
7. Why would I use one over the other?
8. But can't base tags have data types other than BOOL?
9. Are a base and an alias basically the same thing with different names?
10. If I create a base tag, what input is associated with it?
11. If I haven't assigned it a numbered bit address, where does its value go?
12. How does that bit know whether something is energized?

The last few questions were what I needed to get to. Just saying “an alias is another name” didn't answer where the value comes from or what updates it.

## First, where did my tag go?

I created `engine_NOT_at_rated_speed` and couldn't find it in Controller Tags. My teacher had selected the controller name under Scope. Mine was set to `Program_1`, so it appeared under that program's Parameters and Local Tags.

The tag existed. I was looking in the wrong list for the scope I had selected.

- **Controller scope:** the tag can be used across the controller's programs.
- **Program scope:** the tag belongs to that program and can be used by its routines.

Scope tells me where a tag belongs and where it can be used. It doesn't tell me whether it's connected to a physical input.

I also tried cutting and pasting the tag to change its scope, and nothing happened. The screenshot showed I was in **Monitor Tags**. The move instructions use **Edit Tags** in both windows. That's a separate issue from what the tag actually means.

## What I'm used to in 500

In RSLogix 500, I might use:

| Example | What I'm using it for |
| --- | --- |
| `B3:0/0` | An internal bit holding a yes/no condition |
| `N7:0` | A stored whole number |
| `I:0/2` | A physical input bit on my MicroLogix setup |

Those addresses helped me see what I was working with. Moving to names like `Engine_Speed` made me wonder where the address went and what the name was actually connected to.

## A base tag still has somewhere to store its value

If I create `Engine_at_rated_speed` as a **Base BOOL**, Studio 5000 gives it storage for a true/false value. I don't choose a B3 address myself.

For this example, I can relate it to using `B3:0/0` in 500. Both can hold the result of my logic.

If I create `Engine_Speed` as a **Base DINT**, it holds a whole number. That's similar in purpose to using `N7:0`, although a DINT is 32 bits and an N7 integer is 16 bits.

So **“base” doesn't mean “bit.”** It means the tag has its own data. The data type tells me what kind of data it holds.

| Setting | The question it answers |
| --- | --- |
| Base or Alias | Does this have its own data, or refer to existing data? |
| BOOL or DINT, for example | Is the value true/false or a whole number? |
| Scope | Where does the tag belong, and where can I use it? |

I need to keep those questions separate instead of trying to understand all three as the same thing.

## So what input is associated with a base tag?

Creating my own base tag doesn't automatically connect it to an input.

If I create `Engine_Speed`, the name alone doesn't make it read a speed sensor. Something has to put a number into it. In a practice exercise, I can enter that number myself. In an actual application, the program would need a defined way to get the speed value into that tag.

That's like `N7:0` in 500. Having a place to store a number doesn't mean that number automatically comes from a sensor.

## How does an internal bit know whether something is energized?

It doesn't figure that out by itself. **Something updates its value.**

For my internal `Engine_at_rated_speed` BOOL, my rung checks the conditions and the OTE writes the result:

- When the rung is true, the OTE writes **1**.
- When the rung is false, the OTE writes **0**.

That happens when the rung executes. It's the same idea as an OTE writing to `B3:0/0` in 500. The main change I'm getting used to is using a named BOOL instead of choosing that numbered internal address.

A physical input gets updated differently. The input module detects the electrical signal and updates the input bit. That bit is then available for my logic to check.

## Where the alias fits

In my screenshot, `Engine_Running` is an alias for:

`Local:1:I.Data.2`

That input bit already exists. I'm giving it a name that tells me what the signal means in my program.

For my current card, the address means:

| Part | What it points to |
| --- | --- |
| `Local:1` | The locally connected card in slot 1 |
| `:I` | Information coming from the module into the controller |
| `.Data` | The member containing its input states |
| `.2` | Bit 2 |

The `Data` part was unfamiliar. The module tag also contains other information, such as fault information, so I select Data to get to the input states.

I can put either `Engine_Running` or `Local:1:I.Data.2` on the contact, and it checks the same input bit. **The alias doesn't create a second value that needs to be updated separately.**

That's why thinking of it like a symbol for an input address in 500 helps me.

## Why “it's just another name” wasn't enough for me

I was comparing two different things in my exercise:

| Tag | What gives it a value? |
| --- | --- |
| `Engine_Running`, aliased to the input | The input module updates the input bit it refers to. |
| `Engine_at_rated_speed`, a Base BOOL | My logic writes its result through the OTE. |

So there was more going on than just choosing a different name. One was my own stored result. The other referred to existing input data.

If I created `Engine_Running` as a separate Base BOOL, it wouldn't follow that input automatically. I'd need logic or another source to update it.

The part I need to be careful with is assuming every alias means a physical device. **An alias can refer to internal data too.** In my example, it happens to refer to a physical input.

## Does it matter where I click Create New?

Creating a tag from the rung or creating it in the tag table are different ways to create a tag. That alone doesn't determine whether it has its own value or refers to an input.

I still need to check the settings:

- Is it Base or Alias?
- If it's an alias, what is it Alias For?
- What data type does it use?
- Which scope is selected?

Also, the tag and instruction aren't the same thing. `Engine_Running` is the data being checked. The **XIC contact** is the instruction checking it.

## The automatically populated tags are still part of the picture

The explanation we went through was that adding the I/O cards creates their module tags, such as `Local:1:I`, in Controller Tags. Expanding Data shows the input bits. Then I can refer to one of those existing bits with an alias.

I'm still connecting that to what I'm used to seeing in 500. I don't want to describe it as something I've fully learned just because it has been explained once.

What helps me is tracing one value at a time: **where it exists, what updates it, and how my rung refers to it.**

## The questions I can come back to

When I'm unsure whether I need a base tag or an alias, I can ask:

1. Am I trying to store a separate value, such as the result of my logic?
2. Or does the data already exist, and I want to refer to it with a useful name?
3. What will actually update that value: an input module, my logic, or something else?

For my exercise, a Base BOOL stores the internal result, and the alias gives the existing running-feedback input a readable name. That's the comparison I'm using while I work through the broader idea.
