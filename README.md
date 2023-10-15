# Gitlet

## The principle of Gitlet 
In principle, Gitlet is no different from Git. Both save the contents of all files in a version and allow switching between different versions, ensuring that all existing versions are not lost. So, how do you distinguish a version? The answer is to create a version snapshot with the "commit" command. It's like taking a photo of that moment with a camera and stringing the photos together in order. When you want to go back to a certain version, you just have to flip back in sequence. 

When Gitlet is initialized, the system automatically creates an initial commit, called the "init Commit". This commit doesn't record any files. Subsequently, with every new version, a new commit is generated, which records the information of the files in that version.

The most recent version's commit is marked by a "Head". In this instance, it's "Commit2", meaning the current directory contains only the files 3.txt and 4.txt. If there's a need to switch to Commit1, we can see that Commit1 has recorded the files 1.txt and 2.txt. So, how does one transition between versions? All that's needed is to shift the Head to Commit1, read the files 1.txt and 2.txt that Commit1 has recorded, write these files back into the current directory, and then delete the files 3.txt and 4.txt. After this operation, the directory will only have the files 1.txt and 2.txt.

Commit1 and Commit2 have distinct contents. However, if both commits contain the same files, there's no need to store duplicates. This approach can save a considerable amount of space.

This way, when switching versions, all that's needed is to delete 4.txt and add the new 1.txt.

Gitlet also supports branching functionality. When multiple people are working on development concurrently and to ensure there's no interference between them, multiple paths can be taken simultaneously. In Gitlet, this is called a branch. As illustrated below (though I cannot see the actual image), BranchA and BranchB represent two distinct branches. HeadA and HeadB are used to track the heads of all branches in Gitlet, while the Head points to the current commit. They are independent of each other. This means that the current directory's files are still just 2.txt and 4.txt. HeadB and HeadA merely serve a recording purpose and don't represent the directory's current commit. For instance, both the Head and HeadA can point to Commit3A simultaneously.

## Gitlet Internal Structure Implementation:
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


## Objects
"Objects" is a crucial concept in Gitlet. Each Commit corresponds to a Commit object, and each file corresponds to a Blob object. But how do you define an object? The answer lies in its unique hashcode. This hashcode is determined based on the unique information of each object, generated using the SHA-1 algorithm. I refer to this hashcode as an "ID". Only entities with the same ID can be considered the same object. All these objects are serialized and written into the "objects" directory. Each object corresponds to a file, with the filename being the 40-character object ID.

For every file in Gitlet, it's stored in the form of a Blob. Different files correspond to different Blobs, while identical files correspond to the same Blob. How do we differentiate between files? Only files with both the same filename and content can be regarded as identical.

In terms of **Commit**, the identical object should encompass consistent attributes like `message`, `parents`, `date`, and `blobID`.

- **message**: This is the description provided every time the `gitlet commit` command is executed. Examples include annotations like "init commit", "add file1", and so on.

- **parents**: This holds the ID information of the prior Commit. After executing a merge operation that consolidates two branches, the resultant Commit will reference two preceding commits. This means that this Commit will have two IDs listed within its `parents`.

- **date**: This is the timestamp automatically generated during the commit action. It's crucial to note that gradescope insists on the date format "Wed Dec 31 16:00:00 1969 -0800". Any divergence from this format will yield an error when the `gitlet log` command is launched.

- **blobID**: Each Commit is linked to several retained files. By keeping a file's `blobID` within the Commit, it's straightforward to identify the corresponding Blob object in the `objects` directory.

To summarize, the `objects` directory preserves all relevant Commit and file data. Irrespective of being a Commit or a file, each is referred to as an object. Every object is distinguished by a unique ID. Consequently, to identify a specific object, knowing its ID suffices to locate it within the `objects` directory.

## refs
The `refs` directory contains the endpoint information for all branches. Within the `heads` folder, there are multiple files, each named after its respective branch. Taking the illustration below as an example:

```
     |--refs
     |    |--heads
     |         |--master
     |         |--61abc
```

There are two branches: the default branch called `master` and a newly created one named `61abc`. The content of each file represents the 40-character ID of the terminal Commit. The content of the `master` file corresponds to the ID of `Commit3A`, while the content of the `61abc` file corresponds to the ID of `Commit3B`.

## HEAD
The `HEAD` file contains the ID of the current pointing Commit. In the given illustration, the content of the `HEAD` file corresponds to the 40-character ID of `Commit2`.

## stage
The `stage` serves as a staging area. When executing the command `gitlet add "a.txt"`, the file a.txt is merely staged temporarily. At this point, the Blob object for a.txt is created, but the Commit hasn't yet pointed to this Blob. Only after executing the `commit` command is the connection between the Commit and Blob truly established. For a more detailed understanding, you can refer to animations illustrating the `add` and `commit` processes in certain articles. In my implementation, I've separated `stage` into `addstage` and `removestage`. These respectively correspond to the operations needed for the `add` and `rm` commands, making the code easier to write.


