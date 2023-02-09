# Basic Information


## Creating File

Create an Markdown file (.md) in the right location (check the file structure below), then create a new branch for your commit and start a pull request. Once it is approved your page will be added to the website

## Editing File

To edit a document, from the website page directly click on the "Edit this page" button on the top right of your document/page you will be automatically redirected into the github repository in order to change the md file.
Or just search for the file in the repository manually.
Once you finish editing make a pull request for your changes and wait for the merge approve to see the changes.

## File Structure

Follow this structure and the top and side bars will be automatically updated. Here is a example of the current file structure, keep the same pattern.

```
+-- ..
|-- (files)
|
|-- docs
|   |-- home.md 
|   |-- documentation
|   |   |-- basic.md 
|   |   |-- features.md
|   |
|   |-- pyspark
|   |   |-- python.md
|   |   |-- spark.md
|   |   |-- examples
|   |   |   |-- first.md
|   |
|   |-- (other md files)
|   +-- ..
|
|-- (files)
+-- ..
```
So consider it as a normal folder and files. The inital folder will be added as a top navigation item and everything inside this folder will be in the side navigation bar, where you can also create another folder and you will have dropdown in the sidebar.



