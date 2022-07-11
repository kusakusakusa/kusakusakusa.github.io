---
title:  "How To Restrict File Search In Sublime Based On Project"
date:   2019-12-17 09:00:00
permalink: how-to-restrict-file-search-in-sublime-based-on-project
---

This is a documentation on how to restrict text search to within specific directories per project you are working on in [Sublime](https://www.sublimetext.com/).

You may often find it annoying that a simple text search is searching in folders that is not part of your source code. While you can easily flip the switch in the user settings of the sublime text editor, it might not be ideal if you wear multiple hats like me and work on different frameworks.

One framework’s trash may be another’s treasure. Folders that are considered junk in one framework might be important in another. And if we happen to work on these framework together simultaneously, we would have to constantly flip the switch on and off as we jump between working on these projects.

INSERT MEME sublime meme

If you are using sublime text because the framework you are working on is simple enough to manage and you do not want the computation-heavy indexing and compilation process to be running in the background constantly, this is an article for you to boost your productivity.

## The User Setting Way

The commonly documented way of configuring your sublime editor is to tweak the configuration option in the user setting. Using restricted file search as an example, we can simple press CMD + , (assuming you are on a mac) to call up the user setting files in the sublime editor.

Next add this setting under the Preferences.sublime-settings file as shown:

```
{
  ...
  "binary_file_patterns": [
    "node_modules/*",
    "public/packs/",
    "public/assets/",
    "public/packs-test/",
    "tmp/*",
    "*.jpg",
    "*.jpeg",
    "*.png",
    "*.gif",
    "*.ttf",
    "*.tga",
    "*.dds",
    "*.ico",
    "*.eot",
    "*.pdf",
    "*.swf",
    "*.jar",
    "*.zip"
  ]
  ...
}
```

The `binary_file_patterns` option will instruct Sublime to treat these files as binary files. Binary files are not readable by human, hence it is not considered in Goto Anything or Find in Files functions by default.

Line 4 to 8 are folders that we want excluded from our search process.

The rest are referring to specific files base on their extensions.

This setup is what I typically use for my Rails projects.

## The Project-specific Way

The better alternative is to base the setting on the project level so that we do no overlap the settings between projects.

Save the project as a sublime project via File > Save As…

If moving the mouse is not your thing, can you simply create a file in the root directory with the .sublime-project extension.

Next, add the same binary_file_patterns setting as shown previously under the folders key as shown:

```
{
  "folders":
  [
    {
      "path": ".",
      "binary_file_patterns": [
        ...
      ]
    }
  ],
}
```

Line 5, the path key is required. This is added by default when we save our project as a sublime project. More information on its function and purpose can be found [here](https://www.sublimetext.com/docs/projects.html).

Next, close sublime. This time, open our project via the sublime project file that you have created.

Make a search now and you will realise that you are no longer searching in the folders that you have no interest in. The search process is also much faster than before because the program is going through less files, and ignoring the bulkier ones.

Hallelujah!

## Housekeeping

Once we created the `.sublime-project` file, another file with the extension `.sublime-workspace` will also be created. The latter contains user specific data and you will not want to share it with other developers who may be working on the same source code as you. Add this file to our <code>.gitignore</code> file to achieve this.
