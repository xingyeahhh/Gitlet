# Gitlet

### The principle of Gitlet 
In principle, Gitlet is no different from Git. Both save the contents of all files in a version and allow switching between different versions, ensuring that all existing versions are not lost. So, how do you distinguish a version? The answer is to create a version snapshot with the "commit" command. It's like taking a photo of that moment with a camera and stringing the photos together in order. When you want to go back to a certain version, you just have to flip back in sequence. 

When Gitlet is initialized, the system automatically creates an initial commit, called the "init Commit". This commit doesn't record any files. Subsequently, with every new version, a new commit is generated, which records the information of the files in that version.

The most recent version's commit is marked by a "Head". In this instance, it's "Commit2", meaning the current directory contains only the files 3.txt and 4.txt. If there's a need to switch to Commit1, we can see that Commit1 has recorded the files 1.txt and 2.txt. So, how does one transition between versions? All that's needed is to shift the Head to Commit1, read the files 1.txt and 2.txt that Commit1 has recorded, write these files back into the current directory, and then delete the files 3.txt and 4.txt. After this operation, the directory will only have the files 1.txt and 2.txt.

Commit1 and Commit2 have distinct contents. However, if both commits contain the same files, there's no need to store duplicates. This approach can save a considerable amount of space.

This way, when switching versions, all that's needed is to delete 4.txt and add the new 1.txt.

Gitlet also supports branching functionality. When multiple people are working on development concurrently and to ensure there's no interference between them, multiple paths can be taken simultaneously. In Gitlet, this is called a branch. As illustrated below (though I cannot see the actual image), BranchA and BranchB represent two distinct branches. HeadA and HeadB are used to track the heads of all branches in Gitlet, while the Head points to the current commit. They are independent of each other. This means that the current directory's files are still just 2.txt and 4.txt. HeadB and HeadA merely serve a recording purpose and don't represent the directory's current commit. For instance, both the Head and HeadA can point to Commit3A simultaneously.

### Gitlet Internal Structure Implementation:
Let's delve into the internal implementation of Gitlet. Firstly, the gitlet init command creates a .gitlet directory. This .gitlet directory is a hidden folder. On Windows, you need to turn on the option to view hidden files to see it. On Linux or macOS, you can view it using the ls -la command. Most of Gitlet's operations take place within this .gitlet directory. The structure of this directory is as shown in the code below:

     .gitlet
     ├── objects
     │    ├── commit and blob
     ├── refs
     │    ├── heads
     │         └── master
     ├── HEAD
     ├── addstage
     └── removestage


### Objects
"Objects" is a crucial concept in Gitlet. Each Commit corresponds to a Commit object, and each file corresponds to a Blob object. But how do you define an object? The answer lies in its unique hashcode. This hashcode is determined based on the unique information of each object, generated using the SHA-1 algorithm. I refer to this hashcode as an "ID". Only entities with the same ID can be considered the same object. All these objects are serialized and written into the "objects" directory. Each object corresponds to a file, with the filename being the 40-character object ID.

For every file in Gitlet, it's stored in the form of a Blob. Different files correspond to different Blobs, while identical files correspond to the same Blob. How do we differentiate between files? Only files with both the same filename and content can be regarded as identical.

In terms of **Commit**, the identical object should encompass consistent attributes like `message`, `parents`, `date`, and `blobID`.

- **message**: This is the description provided every time the `gitlet commit` command is executed. Examples include annotations like "init commit", "add file1", and so on.

- **parents**: This holds the ID information of the prior Commit. After executing a merge operation that consolidates two branches, the resultant Commit will reference two preceding commits. This means that this Commit will have two IDs listed within its `parents`.

- **date**: This is the timestamp automatically generated during the commit action. It's crucial to note that gradescope insists on the date format "Wed Dec 31 16:00:00 1969 -0800". Any divergence from this format will yield an error when the `gitlet log` command is launched.

- **blobID**: Each Commit is linked to several retained files. By keeping a file's `blobID` within the Commit, it's straightforward to identify the corresponding Blob object in the `objects` directory.

To summarize, the `objects` directory preserves all relevant Commit and file data. Irrespective of being a Commit or a file, each is referred to as an object. Every object is distinguished by a unique ID. Consequently, to identify a specific object, knowing its ID suffices to locate it within the `objects` directory.

### refs
The `refs` directory contains the endpoint information for all branches. Within the `heads` folder, there are multiple files, each named after its respective branch. Taking the illustration below as an example:

```
     |--refs
     |    |--heads
     |         |--master
     |         |--61abc
```

There are two branches: the default branch called `master` and a newly created one named `61abc`. The content of each file represents the 40-character ID of the terminal Commit. The content of the `master` file corresponds to the ID of `Commit3A`, while the content of the `61abc` file corresponds to the ID of `Commit3B`.

### HEAD
The `HEAD` file contains the ID of the current pointing Commit. In the given illustration, the content of the `HEAD` file corresponds to the 40-character ID of `Commit2`.

### stage
The `stage` serves as a staging area. When executing the command `gitlet add "a.txt"`, the file a.txt is merely staged temporarily. At this point, the Blob object for a.txt is created, but the Commit hasn't yet pointed to this Blob. Only after executing the `commit` command is the connection between the Commit and Blob truly established. For a more detailed understanding, you can refer to animations illustrating the `add` and `commit` processes in certain articles. In my implementation, I've separated `stage` into `addstage` and `removestage`. These respectively correspond to the operations needed for the `add` and `rm` commands, making the code easier to write.

## Thought process behind writing each functionality.

### init

Initialize the gitlet repository by first setting up the overarching `.gitlet` file structure. Following that, create a Commit object named `initCommit`. The `message` should be "initial commit", while both `parents` and `blobID` should be empty (note that it shouldn't be `null`, but an empty list. If it's `null`, it will cause an error during the hashcode generation). The `date` is set to "Thursday, 1 January 1970", which can be directly generated using the following code:

```java
this.currentTime = new Date(0);
```

The generated timestamp here isn't in the standard format. To get a String type timestamp, you can transform it using:

```java
private static String dateToTimeStamp(Date date) {
    DateFormat dateFormat = new SimpleDateFormat("EEE MMM d HH:mm:ss yyyy Z", Locale.US);
    return dateFormat.format(date);
}
```

Lastly, to generate the ID, you can use the code below. The `sha1()` function takes a String as input, so all variables need to be of String type to get the 40-character ID.

```java
private String generateID() {
    return Utils.sha1(dateToTimeStamp(currentTime), message, parents.toString(), pathToBlobID.toString());
}
```

With this, you've created an `initCommit` object. The next step is to write it into the `objects` directory.

**Failure scenario:** If a `.gitlet` repository already exists, output "A Gitlet version-control system already exists in the current directory." and then exit.

### add

`add [filename]`

The `add` operation will store all unsaved files in the form of a Blob. Remember that the objects you're storing must be instantiated, as:

```java
public class Commit implements Serializable { ... }
public class Blob implements Serializable { ... }
```

Note that variables marked with `static` won't be stored, so avoid using the `static` keyword when declaring variables.

Additionally, all stored filenames and their corresponding `blobID` should be written to the `addstage`, so they can be read during the next `commit` operation. Below are the instance variables of my `Commit`, `Blob`, and `Stage`. They are provided for reference:

```java
/*
 * commit object
*/
private String message;
private Map<String, String> pathToBlobID = new HashMap<>();
private List<String> parents;
private Date currentTime;
private String id;
private File commitSaveFileName;
private String timeStamp;
/*
 * blob object
*/
private String id;
private byte[] bytes;
private File fileName;
private String filePath;
private File blobSaveFileName;
/*
 * stage object
*/
private Map<String, String> pathToBlobID = new HashMap<>();
```

**Failure scenario:** If the file doesn't exist, output "File does not exist."

### commit
`commit [message]`

Create a new Commit object and save it. The specific implementation is as follows:

- The `message` is the provided message information.

- `parents` reference the previous Commit. The ID of the prior commit can be sourced from the HEAD, and the relevant object data can be accessed using this commit ID.

- The `date` can be readily generated using `this.currentTime = new Date();`. Remember to adjust the format accordingly.

- `BlobID` is a mapping structure (though lists can be employed, they may be less intuitive) that maps file paths to their respective file IDs. My approach involves copying the mapping from the previous Commit, then contrasting it against `addstage` to discern which files are to be retained and which are to be deleted. This addition and deletion process leverages the map's key-value pair functionalities.

**Failure scenarios:**
1. If there is no content in `addstage`, output the error message: "No changes added to the commit."
2. If the `message` is empty, output the error message: "Please enter a commit message."

### rm
`rm [filename]`

The `rm` function is designed to delete a file named `filename` from the working directory. This can be categorized into three scenarios:

1. If the file has just been added to `addstage` but hasn't been committed, directly remove the Blob from `addstage`.
  
2. If the file is tracked by the current Commit and exists in the working directory, then add it to `removestage` and delete the file from the working directory. This will be recorded in the next commit.
  
3. If the file is tracked by the current Commit but doesn't exist in the working directory, simply add it to `removestage` (this situation arises if you manually delete the file after a commit, then execute `rm`; the second `rm` corresponds to this scenario).

Other than these three scenarios, there shouldn't be any file named `filename` in the working directory, so `rm` only corresponds to these cases.

**Failure scenario:** If the file is neither in the `addstage` nor tracked by the current Commit, it indicates that the file doesn't exist in the working directory. Output the error message: "No reason to remove the file."

### log
`log`

Starting from the current Commit, it prints all Commit information in reverse until it reaches the `initCommit`. A loop can easily handle this task.

There's a special Commit called the `merge commit`. As shown in the example diagram, a merge commit has two parents. When printing a merge commit, you only need to select the first one from the `parentsList`, because during the merge operation mentioned below, the parent that needs to be printed is placed in the first position of the List. 

**Failure scenario:** None.

**global-log**
`global-log`

Print all Commits without concern for order. Simply read and print all Commits from the `objects` directory. The function below retrieves all Commits from the `objects` directory and stores them in a List form:

```java
List<String> commitList = Utils.plainFilenamesIn(OBJECT_DIR);
```

The `OBJECT_DIR` here is custom-defined, and its definition is as follows. You can learn about these from Lab6. Make sure not to blindly copy and paste; you need to understand the logic :joy:.

```java
public static final File GITLET_DIR = join(CWD, ".gitlet");
public static final File OBJECT_DIR = join(GITLET_DIR, "objects");
```

**Failure scenario:** None.

### find**
`find [commitmessage]`

Print all IDs of Commits with the same message as the input. Similar to the previous logic, read all the Commits, find the ones that match, and print their IDs. If multiple results are found, print each on a new line.

**Failure scenario:** None.

### status
`status`

Display the following:

```
=== Branches ===
*master
other-branch

=== Staged Files ===
wug.txt
wug2.txt

=== Removed Files ===
goodbye.txt

=== Modifications Not Staged For Commit ===
junk.txt (deleted)
wug3.txt (modified)

=== Untracked Files ===
random.stuff
```

Start by printing all branches. You can read the names directly from the `Heads` directory. The current branch is identified by the contents of the `HEAD` file; just prefix it with an asterisk.

The second section prints filenames stored in `addstage`.

The third section prints filenames stored in `removestage`.

The last two sections are from extra credit content I didn't implement, so just print the headings.

**Failure scenario:** None.

### checkout
`checkout` has three usage scenarios:

1. 
`checkout -- [filename]`

After committing, if a file `1.txt` exists in the working directory and you wish to modify its content or delete it, and later regret the decision, use this command to recover the file. If the file `filename` is tracked by the current Commit, write it to the working directory. If a file with the same name exists, overwrite it. If it doesn't, write directly.

**Failure scenario:** If the file `filename` doesn't exist in the current Commit, output "File does not exist in that commit."

2. 
`checkout [commitID] -- [filename]`

Similar to the first case, but now retrieve the file `filename` from a specific past Commit using its `commitID`.

3. 
`checkout [branchname]`

Switch to a different branch. The process involves updating the working directory with files from the latest Commit of the target branch. If there's any overlap between the current and target branches' files, the ones from the target branch will overwrite those in the working directory.

