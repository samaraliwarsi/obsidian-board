# Obsidian Board

## Forked update - Kanban Board

## Images
![Board](/kanbanMD-Updated/board_img.png)
![Setting](/kanbanMD-Updated/boardSettings_img.png)

### Usage for full vaul - New method
> The files related to the new method are inside the folder [KanbanMD_Updated](/kanbanMD-Updated/)

1. Copy the [kanbanMD_wConfig.md](/kanbanMD-Updated/kanbanMD_wConfig.md) to your vault. 
2. Add the [kanban.css](/kanbanMD-Updated/kanban-board.css) file to `.obsidian/Snippets` and enable it from `Settings/Apperance` in Obsidian.
3. The board should create a file `kanban-config.json` in a folder called `.kanban/` at the root of your Obsidian vault. If it doesn't, create it. This file will contain the settings related to the board. 
4. Use the file - Open Board settings and set your target folder, exclude folder, exclude tags and other settings. 
5. Frontmatter requirements for notes same as mentiond in [Task File Requirements](#task-file-requirements)

### Note about Settings
1. You can select any target folder. 
2. Exclude list is folders to exclude inside the target folder. Exclude list for tags filters out any tags mentioned. 
3. Default cards per collum is the maximum cards loaded without using **Load More** button. Each iteration of load more button, loads the same number of files as the number in default cards per column setting. 
4. Column names are based on the `status` frontmatter in properites. This can be edited in settings. 
5. Folder colors are specific colors that can be assigned to left-border of the cards populated for folders that are already in the target folder. Any folder not mentioned will get an accent color border based on general obsidian settings. This setting doesn't affect what folders are populated, only if they get a specific color. 
6. Progress Colors are based on the progress frontmatter in the properties. It's coded to produce a progress bar from 0-5 range. Each level of progress produces a different color which can be changed using this setting. For deeper editing, adding more levels, code editing is required. 
7. Config File Location - this is just a visible cue for where file is located. Can't be edited as of now. 

### Usage and Ownership

All credits to the original owner of the repository [Louai99k](https://github.com/Louai99k). The changes here are based on my own personal usage. 

---

## Orignal board

![video](https://github.com/user-attachments/assets/c40da606-d3a4-4a4e-b96c-54f9b8f17819)


### Requirements

1. [Dataview](https://blacksmithgu.github.io/obsidian-dataview/)

### Usage

1. Copy the template file (link below) to your templates folder
2. Add the kanban.css (link below) file to snippets and enable it
3. In edit mode, update the folder of the tasks inside the newly created board (I've automated this with [Templater](https://silentvoid13.github.io/Templater/introduction.html))

### Task File Requirements

The items of the board will be files that has the following frontmatter fields:
- `status`: The value of it should be one id of the COLUMNS variable in the template.
- `order`: You can set this initially to 0. It represents the cards order in the related column.
- `priority`: A numeric value that is controlled via the P_LABEL variable in the template. In my template 0 has no meaning, 1 is lowest, 2 is low and so on...
- `title`: of course you need a title field for the card's text


You can always enhance the template and add your own fields to task's frontmatter.
