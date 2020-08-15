---
layout: post
title: Splitting minetest-game’s default mod
category: devlog
tags:
  - minetest-game
  - minetest
---

[minetest-game](https://github.com/minetest/minetest_game/) (`mtg`) is the default game bundled in Minetest engine. Most of mods use this game as a base for modding.

This game is currently divised into 32 mods:
`beds`, `binoculars`, `boats`, `bones`, `bucket`, `butterflies`, `carts`, `creative`, `default`, `doors`, `dungeon_loot`, `dye`, `env_sounds`, `farming`, `fire`, `fireflies`, `flowers`, `game_commands`, `give_initial_stuff`, `map`, `player_api`, `screwdriver`, `sethome`, `sfinv`, `spawn`, `stairs`, `tnt`, `vessels`, `walls`, `weather`, `wool`, `xpanes`.

`default` mod contains various features and other mods often depends on it, even if they only require a small subset of what it provides. It [has been proposed](https://github.com/minetest/minetest_game/issues/726) to split this mod to make the game more modular but not implemented because seen as a complicated change.

In this devlog, I will try to implement this step-by-step. Changes discussed here are only experimental and you should *NOT* consider them if you are maintaining a mod because they have not be implemented upstream (yet).

Because many mods relies on `default` (and even unmaintained mods), it is important to preserve backward compatibility. For nodes names it is possible to define aliases (and this has already been implemented in minetest engine), but for textures, sounds, etc. this is not possible yet because [this didn't seems much needed](https://github.com/minetest/minetest/issues/3370). However this feature is required because many mods use `default` textures, often for overlaying (example: [moreores](https://github.com/minetest-mods/moreores/) ore registering use `default_stone.png`).

When doing the split, I will assume that `minetest.register_alias(alias, original_name)` also works for textures and sounds even if it is not currently true (and will see how hard it is to implement this in the engine).

I already started the splitting with the mod [`keys`](https://github.com/minetest/minetest_game/pull/2715) witch is easy to separate from `default` because it is a very independent part.

First, let’s update my fork of `mtg`, create a new branch [`experimental_default_split`](https://github.com/louisroyer/minetest_game/tree/experimental_default_split) on my fork and merge the keys related part in it.

`default` has already some kind of separation for `chests`, `furnace`, `tools`, `torch` and `trees` since they are separate files, so it will be easier to start with them.

Here is how I procede:

- find a name for the new mod (check on contentdb and on the forum the name is not already taken)
- search if there is any reference to the mod in other files of `default`
- move all related lua code, textures, sounds, etc. in an other repository
- write a `mod.conf`, a `README.txt`, and license files
- replaces all references to `default` in the new mod and separate mod API from registrations
- update the game API (`game_api.txt`)
- add aliases
- add the mod in `default` depends (this will create a circular reference if the mod also depends on `default`, but it is temporary) 
- export mod API with the old name (example: `default.register_tool = tool.register_tool`)
- move translations by copying all and running [update translation tool](https://github.com/minetest-tools/update_translations)
- run luacheck again (I use vim syntastic with luacheck, but it is better to double-check)

When all mods are created, it is time to play with depends to no longer have any mod that depends on `default`. Then it is possible to run `mtg` to ensure test everything.

Result of the split:
- `mtg_gui`:
    - contains gui related functions
- `default_lib`:
    - contains helper functions from `default`
- `chests`:
    - depends on `mtg_gui`, `woods` (for the sound), `default_lib`, and  optionally on `ores` (then locked chests can be crafted)
    - locked chests can be disabled by settings
    - [`basic_materials`](https://gitlab.com/VanessaE/basic_materials) in optional depends and if loaded, then the craft use padlock instead of steel, this can be disabled by settings
- `ores`
- `trees`
- `woods`
- `ladders`
- `fences`
- `signs`


Since the goal of this experiment is to make `mtg` more modular, I added new settings to allow disabling easily parts of some mods and I made some mods depends optional.
