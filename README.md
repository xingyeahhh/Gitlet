# Gitlet
## The principle of Gitlet 
In principle, Gitlet is no different from Git. Both save the contents of all files in a version and allow switching between different versions, ensuring that all existing versions are not lost. So, how do you distinguish a version? The answer is to create a version snapshot with the "commit" command. It's like taking a photo of that moment with a camera and stringing the photos together in order. When you want to go back to a certain version, you just have to flip back in sequence. 

When Gitlet is initialized, the system automatically creates an initial commit, called the "init Commit". This commit doesn't record any files. Subsequently, with every new version, a new commit is generated, which records the information of the files in that version.

The most recent version's commit is marked by a "Head". In this instance, it's "Commit2", meaning the current directory contains only the files 3.txt and 4.txt. If there's a need to switch to Commit1, we can see that Commit1 has recorded the files 1.txt and 2.txt. So, how does one transition between versions? All that's needed is to shift the Head to Commit1, read the files 1.txt and 2.txt that Commit1 has recorded, write these files back into the current directory, and then delete the files 3.txt and 4.txt. After this operation, the directory will only have the files 1.txt and 2.txt.

Commit1 and Commit2 have distinct contents. However, if both commits contain the same files, there's no need to store duplicates. This approach can save a considerable amount of space.


