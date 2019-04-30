# vi notes

Throughout the lab you will need to create and edit YAML files using `vi`

There are only three things that you need to be able to do:

1. Add and edit text in the YAML file
2. Save changes and exit
3. Discard changes and exit

## Add and edit text

**vi** needs to be in `INSERT` mode for you to be able to add and edit text.

Press the `Ins` key on your laptop to put **vi** into `INSERT` mode. The editor will show `-- INSERT --` at the bottom left of the screen to indicate it is in `INSERT` mode.

Use the normal arrow keys to move around the text file.

Press `Esc` if you need to come out of `INSERT` mode.

## Save changes and exit

When you are finished adding or editing text, press `Esc`, type `:wq` and press `Enter`. This will save your work and exit the `vi` editor.

`w` tells vi to write your changes, `q` tells vi to quit back to the terminal.

## Discard changes and exit

If you've made a mistake and want to quit without saving, press `Esc` and type `:q!`. You will be returned to the terminal without saving changes.

## Quick reference

- Add text: `Ins`
- Come out of `INSERT` mode: `Esc`
- Save changes and: Press `Esc` then type `:wq`
- Discard changes and exit: Press `Esc` then type `:q!`
