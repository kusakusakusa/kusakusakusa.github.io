---
title:  "How To Setup And Debug Google App Script"
date:   2020-04-24 09:00:00
permalink: how-to-setup-and-debug-google-app-script
---

This is a quick start guide on setting up Google App Script and using it in G Suite applications. Mainly, I will go through Google Sheets and Google Drive since they are the most widely used services.

## Motivation

This is meant to remove the layer of repetitive mundane house keeping tasks via automation. In my side hustle, we have shifted our files from dropbox to Google Drive recently, bestowing us the ability to automate our tasks with Google App Scripts.

## Setup

Head to your [Google App Scripts console](https://script.google.com/home/start). There are many functions you can explore here. I will touch on just 3. Mainly the projects, executions and triggers.

Every project holds the script to execute based on a trigger. It is up to your jurisdiction to decide if a project should uphold a [Single Responsibility Principle](https://en.wikipedia.org/wiki/Single-responsibility_principle) or do multiple task as part of a bigger singular operation.

Clicking on the ‘New Project’ button near the top left of the screen will bring you to the code editor where you can code your script.

In a script, you can define any number of functions. The exact function to run can be chosen during the trigger selection process. The usual setup is to have a main function, and the other functions are helpers.

## Debug

The most important thing in coding is debugging. After all, it makes up most of our time as software developers.

Let me break it for you.


INSERT TWEET https://twitter.com/catalinmpit/status/1239632987039334401?ref_src=twsrc%5Etfw

In App Script, there is no luxury to debug the code interactively. You will need to rely on the usual logger. To print a log, write the code as such:

```js
function main() {
  Logger.log('Hello World!');
}
```

There are 2 ways to view the logs.

First, within the script editor, go to View -> Logs. A dialog box will popup to show you the logs. A shortcut on mac is cmd + enter.

Second, is from the “My Executions” page back in the App Script console. I personally prefer looking at the logs through there because it does not only show its output but also other details like whether it is even running in the first place.

Click on the play arrow button to execute the function for debugging purposes.

## Triggers
In the App Script editor, click on the clock icon to bring yourself to the function’s triggers page.

Add a trigger by clicking the bottom right button. A dialog box will popup and you can select the conditions of the trigger.

You can select the function in the script to run, as mentioned earlier, and the type of trigger. For Google Sheets, there are more options for triggering the script. More on that later.

## Basics

Before you begin, it is good to know that every entity in G Suite have an id. This id is the gibberish string that you see in the URL of an opened Google Sheets, Google Docs or even a folder in Google Drive. It is highlighted in yellow in the image below.


## Google Docs ID

Knowing the id of the particular file or folder allow you to carry out your operations without having to write the code to search for it. Then again, you can search them by name.

The entities that can be called and utilized in the App Script are documented [here](https://developers.google.com/apps-script/reference). Let’s take a look at [DriveApp](https://developers.google.com/apps-script/reference/drive/drive-app) for example.

Example With Google Drive On Time Driven Trigger
Let’s say you want to remove editors you previously gave access to in your files. You want them to be removed when the files are moved into a particular folder.

```js
function removeEditors() {
  var folder = DriveApp.getFolderById(ID_OF_FOLDER);

  iterateFiles(folder);
}

function iterateFiles(folder) {
  var files = folder.getFiles();
  while (files.hasNext()) {
    var file = files.next();
    Logger.log(`Looking at file ${file.getName()}`);

    var editors = file.getEditors();
    editors.forEach(function(editor) {
      var email = editor.getEmail();
      var approvedEmails = [
        'luffy@gmail.com',
        'zoro@gmail.com',
        'sanji@gmail.com',
        'usopp@gmail.com',
        'nami@gmail.com',
        'chopper@gmail.com',
        'robin@gmail.com',
        'franky@gmail.com',
        'brooks@gmail.com',
        'jinbei@gmail.com'
      ];

      if (!approvedEmails.includes(email)) {
        Logger.log(`Removing editor: ${email} from ${file.getName()}`);
        file.removeEditor(email);
      }
    })
  }
}
```

Given the ID_OF_FOLDER, this script will iterate through all the files in that folder, and for each file it will check if its editors’ emails are under the list of approved emails. If their email is not in the list, they are stripped of the editor role in the file.

Note the way the files variable is being looped. The loop is carried out if hasNext() returns true, and the next file in line is retrieved via next(). This is different from how the editors are looped with forEach, which is, I believe, the usual way developers loop through iterators in JavaScript.

The last step is to setup the time driven trigger to your liking.

The code, unfortunately, cannot be optimized by checking the lastUpdatedDate() of the file and sparing the need to loop through editors for files that were moved into this folder eons ago. This is because moving files into different folders does not update the file’s lastUpdatedDate().

Albeit a small inconvenience, I do expect more features and attributes in the future in App Script to be developed for us to utilize and optimize our codes. Until then, I sincerely hope that it stays free, as it already it right now, without any usage limitation or tiering. In fact, I hope it stays free forever!

## Example With Google Sheet OnEdit Trigger

Let’s look at another example with Google Sheet. In a spreadsheet, we want to compile the values entered in a sheet into another sheet in the same spreadsheet in real time. Simple and straightforward. Let’s see how we can work on it.

Before that, take note of an extremely crucial step. Make sure the spreadsheet is a Google Sheet. If you uploaded a Microsoft Excel sheet and opened it using Google Spreadsheet, you will not be able to run any App Script until you convert it to a Google Sheet. If this applies to you, go to File -> Save as Google Sheet as shown below to make this necessary change.


## Save as Google Sheets

The function is as shown below.

```js
function compile(event) {
  var row = event.range.getRow();
  var column = event.range.getColumn();
  var value = event.range.getValue();
  var spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  var targetSheet = spreadsheet.getSheetByName("Target Sheet")
  targetSheet.getRange(row, column).setValue(value * 2)
}
```

We get the row and column that the user was editing on, whichever sheet that was, and multiply its value by 2 before saving it in the same row and column in the desired sheet with the name Target Sheet.

Now to add the trigger. In the script editor, click on the clock icon. It will bring you to the triggers of this project. Click on the button to add trigger on the bottom right of the page. Under Select event source, you will now see a new option From spreadsheet, and when you select it, you can select the event type to kickstart the function. We are looking for the onEdit event type for this case.

As you can see, this project is associated to only 1 spreadsheet – the spreadsheet it was created from. Additionally, each spreadsheet can have only 1 project as well.

Hence, you cannot have the same script running on different spreadsheet. At least if you did it this way. There may be another way to do so and overcome this restriction but I have yet to explore it.

## Potential

The potential of App Scripts does not end here. Other than scripts, you can even code out html views to present visual dashboard based on real time changes.

On top of that, you can publish you own app scripts, as well as use the scripts other developers have made. This forms a community can supercharge automation to increase productivity.

## Conclusion

Make use of App Script and automate away!

