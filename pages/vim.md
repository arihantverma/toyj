- `cw` (normal mode): deletes the end of current word + switch to insert mode
- `dw` (write mode): delete word under cursor
- ## How to open vim without customisations
- `vim -u NONE -N`: open vim without customisations
	- > The -u NONE flag tells Vim not to source your vimrc on startup
	- > When Vim starts up without loading a vimrc file, it reverts to vi compatible mode, which causes many useful features to be disabled. The -N flag prevents this by setting the ‘nocompatible’ option
	- > Some of Vim’s built-in features are implemented with Vim script,
	  which means that they will only work when plugins are enabled. This file
	  contains the absolute minimum configuration that is required to activate
	  Vim’s built-in plugins:
		- ```vim
		  "essential.vim
		  set nocompatible
		  filetype plugin on
		  ```
			- `vim -u code/essential.vim`
- `!q`: quit editor discarding any changes made
- `x` (n): deletes character under cursor
- Many commands that change text made from: **Operator** and **Motion**
	- `d <motion>`: d for delete and motion for particular motion
	- Motions:
		- `w`: until the start of next word, EXCLUDING its first character
		- `e`: to the end of the current word INCLUDING its last character
		- `$`: to the end of the line, INCLUDING last character.
		- `0`: to the start of the line
		- ***Pressing just the motion while in Normal mode without an operator will move the cursor as specified***
			- *this can be added with numbers for that jump, example `2w` for 2 word jump*
- `dd`: delete the whole line
	- similarly `2dd` delete two lines
- `u`: undo
- `U`: undo all changes one a line
- `<C-r>`: redo
- `p`: put previously deleted text **after** the cursor
- `P`: before the cursor
- `rx`: replace <r> cursor selection with <x>
- `ce`: to change from the cursor until the end of the word. This initiates write mode.
	- `c` is for change
	- all other motions work like with `d`
- start with lesson 4.1