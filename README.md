# Obsidian Board

![video](https://github.com/user-attachments/assets/c40da606-d3a4-4a4e-b96c-54f9b8f17819)


## Requirements

1. [Dataview](https://blacksmithgu.github.io/obsidian-dataview/)

## Usage

1. Copy the template file (link below) to your templates folder
2. Add the kanban.css (link below) file to snippets and enable it
3. In edit mode, update the folder of the tasks inside the newly created board (I've automated this with [Templater](https://silentvoid13.github.io/Templater/introduction.html))

## Task File Requirements

The items of the board will be files that has the following frontmatter fields:
- `status`: The value of it should be one id of the COLUMNS variable in the template.
- `order`: You can set this initially to 0. It represents the cards order in the related column.
- `priority`: A numeric value that is controlled via the P_LABEL variable in the template. In my template 0 has no meaning, 1 is lowest, 2 is low and so on...
- `title`: of course you need a title field for the card's text


You can always enhance the template and add your own fields to task's frontmatter.
