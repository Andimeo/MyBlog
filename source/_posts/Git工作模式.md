title: Git工作模式
date: 2014-08-23 14:14:50
categories: 工具
tags:
- Git
- 译文
---
昨天晚上小伙伴问到我，如何回滚git repository的远程版本。折腾了近一个小时后，发现需要做的是，

* git reset --hard $version_number
* git push -f

找到答案之后，在stackoverflow上又看到这样一段对于git工作模式的描述，
{% blockquote %}
A typical distributed workflow using Git is for a contributor to fork a project, build on it, publish the result to her public repository, and ask the "upstream" person (often the owner of the project where she forked from) to pull from her public repository. Requesting such a "pull" is made easy by the git request-pull command.
{% endblockquote %}
此时我不禁发现，对git的理解还是太浅薄。当初推荐小伙伴使用git时，我只能说出git比svn好，却不知为何更好。显然这样的表达是毫无说服力的。在网上一番搜索之后，发现一篇文章，探讨了四种常见的Git工作模式。我决定将本文翻译于此，算是对用了半年git的一个交代吧。

值得注意的是，如文章作者在一开始交代的，这四种方式只是四个典型的应用方式。我们不应该仅限于这五种方式，我们可以对这些方法进行融汇、修剪，创造出最适合自己团队的工作模式。

<!-- more -->
[Git Workflows](https://www.atlassian.com/git/workflows)

#Overview
由于不知道Git可能的工作模式，刚刚在工作中接触Git的人可能会感到有些困难。本文描述了最常见的几种Git工作模式，希望以此作为新人探索Git世界的一个起点。

当你在阅读本文的时候，这些工作模式只是一些指导意见，而不是不可违背的规则。我们希望通过告诉你什么是可能的，让你能够根据个人的需求混合、订制自己的工作模式。
#Centralized Workflow
转到一个分布式的版本控制系统看起来是一个令人畏惧的任务，但是你并不必改变现有的工作模式就可以享受Git带来的好处。你的团队可以按照与svn一样的模式进行工作。

{% img /img/central1.png %}

然而，相比svn而言，在你的工作流程中使用Git会带来几个好处。第一，每个开发者都是整个项目的本地备份。这种隔离的工作环境可以使每个开发者独立的工作，他们可以提交修改到自己的本地仓库，可以忽视忘记上游的开发过程。直到合适的时候再通知上游。

第二，Git提供了健壮的分支和合并模型。与svn不同，Git的分支为集成代码和分享改动采用了一种fail-safe的机制。

##How it works
就像Subversion一样，Centralized-workflow使用一个中央仓库作为代码提交的唯一入口。Svn中使用trunk，而Git中默认的开发分支叫作mster，所有的改动都会提交到这个分支。这种工作模式master以外的其他任何分支。

{% img /img/central2.png %}

开发者首先clone这个中央仓库。在他们自己的本地备份中，他们修改文件、提交改动，就和在svn中一样。然而，这些提交仅仅储存在本地，它们和远程的那个中央仓库是完全隔离的。这就允许开发者可以在自己满意的一个时间点同步自己的代码到上游仓库中。

开发者通过push，将自己的master分支推送到中央仓库，来完成发布改动的过程。这和svn commit等价，除了git push会把所有本地提交里远程仓库中没有的都提交给中央仓库的master分支。

** Managing Conclicts **

中央仓库代表官方工程，所以它的提交历史应该是正式且不可变的。如果一个开发者的本地提交偏离了中央仓库，Git将会拒绝push他的改动，因为这会覆盖掉官方的提交。

{% img /img/central3.png %}

在开发者发布他们的修改之前，他们需要先取到中央仓库中本地没有的那些提交，将自己的改动建立在那些提交之上。这就好像是说，“我想把自己的改动添加到其他人已经做过的改动之上。”结果是一个绝对线性的提交历史，就跟传统的svn工作模式一样。

如果本地改动和上游的提交直接产生了冲突，Git将会暂停并让你手动的解决这些冲突。Git的好处之一是它使用了git status和git add同时用于生成提交和解决合并冲突。这使得新开发者更容易管理自己的合并。而且，如果他们陷入了麻烦，git就会停止合并过程，让开发者重试或者寻求帮助。

##Example
让我们来一步一步的看看一个典型的小团队如何在这种模式下合作。我们将会看到两个开发者，John和Mary，通过一个中央仓库，分别进行两个feature的开发并且分享他们的贡献。

** Someone initializes the central repository **

{% img /img/central4.png %}

首先，有个人需要在一台服务器上创建中央仓库。如果这是一个新工程，你可以初始化一个空的仓库。否则，你需要导入一个已有的项目。

中央仓库应该是空的仓库，它不应该是一个已有的工作目录。我们可以这样创建，
```
ssh user@host
git init --bare /path/to/repo.git
```
确保user是正确的SSH Username，host是你的服务器的域名或者IP，还有你希望存放仓库的位置。

** Everybody clones the central repository **

{% img /img/central4.png %}

接下来，每个开发者创建一个整个工程的本地备份。我们可以通过命令git clone完成，
```
git clone ssh://user@host/path/to/repo.git
```
当你克隆了一个仓库，Git将会自动为你添加一个叫作origin的标签，这个标签指回到父仓库，以便你今后和其进行交互。

** John works on his feature **

{% img /img/central6.png %}

在他的本地仓库中，John可以使用标准的Git提交流程进行开发，edit、stage、commit。如果你不熟悉工作目录，有一种方法可以让你指定提交的内容。这可以进行针对性的提交，即使你在本地进行了很多修改。

```
git status # View the state of the repo
git add # Stage a file
git commit # Commit a file
```

记住，因为这些命令只完成了本地的提交工作，John可以重复这个过程，随意进行提交，而不必担心远程的中央仓库发生了什么。这对于大feature而言非常有用，因为这些大feature往往需要打碎成更多更简单、更小的部分。

** Mary works on her feature **

{% img /img/central7.png %}

于此同时，Mary也在她本地进行自己的feature开发，使用同样的edit、stage、commit流程。和John一样，她不必担心远程的中央仓库正在发生什么。而且也不担心John在他的本地仓库正在干什么，因为所有的本地仓库都是私有的。

** John publishes his feature **

{% img /img/central8.png %}

一旦John完成了他的feature，他就应该把自己的本地提交发布到中央仓库，使其他团队成员可以访问。他可以通过git push来完成，

```
git push origin master
```

记住，origin是当John clone的时候Git创建的指向中央仓库的远程连接。参数master告诉git，他希望让origin的master分支和他本地的master分支一样。因为中央仓库在John clone之后还没有被更新过，所以不会导致冲突，push将会顺利完成。

** Mary tries to publish her feature **

{% img /img/central9.png %}

我们来看看当John成功发布之后，Mary试图发布的时候会发生什么。她可以使用完全一样的命令，

```
git push origin master
```

但是，因为她本地的历史与中央仓库的历史发生了偏离，Git将会拒绝她的请求。

```
error: failed to push some refs to '/path/to/repo.git'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. Merge the remote changes (e.g. 'git pull')
hint: before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

这阻止了Mary覆盖官方的提交。她需要首先将John的提交拿到本地，和本地的修改进行集成，然后重试。

** Mary rebases on top of John's commits **

{% img /img/central10.png %}

Mary可以使用git pull将上游的提交混合到本地。这个命令有点像svn update，它把上游的提交历史取到本地，并试图和本地的修改合并。

```
git pull --rebase origin master 
```

选项--rebase告诉Git在同步了中央仓库的改动之后，把Mary的改动移到master分支的顶部。如下图所示，

{% img /img/central11.png %}

如果你忘了那个选项，pull仍然可以工作，但是每次有人需要同步中央仓库的时候，都会出现很多"merge commit"。对于这种工作模式而言，最好rebase，而不要生成merge commit。

** Mary resolves a merge conflict **

{% img /img/central12.png %}

Rebase把本地的提交一次一个的放到更新过的master上。这意味着merge conflict只会发生在你的一次提交之上，而不是把你的所有提交作为整体进行合并。这可以使你的提交更有针对性，并且保持一个干净的提交历史。相应的，这使得找到bug是在哪里引进的更加容易，如果必要，可以对项目以最小代价进行回滚操作。

如果Mary和John分别工作在无关的feature上，rebase的过程不大可能会发生冲突。但是如果冲突发生了，Git将会在当前提交处暂停rebase的过程，并且输入如下信息，和一些相应的提示，

```
CONFLICT (content): Merge conflict in <some-file>
```

{% img /img/central13.png %}

Git的一个伟大之处在于，任何人都可以解决他们自己的合并冲突。在我们的例子中，Mary可以简单的执行git status看看问题发生在哪里。冲突文件将会出现在Unmerged path一节。

```
# Unmerged paths:
# (use "git reset HEAD <some-file>..." to unstage)
# (use "git add/rm <some-file>..." as appropriate to mark resolution)
#
# both modified: <some-file>
```

然后，她就可以把文件修改成自己需要的样子。一旦她修改完毕，她就可以将文件置入工作区，让git rebase继续做剩下的事情，

```
git add <some-file> 
git rebase --continue
```

这就是需要做的所有事情了。Git将会移动到下一个提交，对所有产生冲突的提交重复刚才的步骤。

如果你遇到了某种情况，而且解决不了时，不必惊慌。只需要执行下面的命令，你就可以回到执行git pull --rebase之前的状态。

```
git rebase --abort
```

** Mary successfully publishes her feature **

{% img /img/central14.png %}

当她完成与中央仓库的同步之后。Mary就可以成功发布她的改动了。

```
git push origin master
```

#Feature Branch Workflow
{% img /img/feature1.png %}

一旦你熟悉了Centralized Workflow，在你的开发过程中加入feature分支成了一种简单的促进开发者协作的方法。

Feature Branch Workflow的核心想法在于所有feature都应该在自己的分支中开发，而不是都在master分支中。这种封装使得多个开发者可以在不干扰主代码库的前提下开发自己的feature。这同时意味着master分支将永远不会包含不健全的代码，这对于不断进行的集成开发是一个非常大的好处。

这种对feature开发的封装，也使得我们可以在开发中利用pull requests，这是一种发起关于某个分支的讨论的方法。这给了其他开发者一个在代码被合并到主项目之前审查某个feature的机会。或者，当你卡在某个feature开发中时，你可以发起一个pull request向你的同事征求意见。总而言之，pull requests以一种非常简单的方式为你的团队提供了评论彼此工作的条件。

##How it works
Feature Branch Workflow仍然使用中央仓库，master分支仍然代表官方工程的历史。但是，并不是直接提交到本地的master分支，开发者每次开始一个新feature时，首先需要创建一个新的分支。Feature分支应该有描述性的名称，比如animated-menu-items或者issue-#1061.这是为了给每个分支提供一个清晰、明确的目的。

Git对于master分支和feature分支没有本质上的区分，所以开发者可以edit、stage并且commit改动到feature分支，就和在Centralized Workflow中一样。

除此之外，feature分支可以被合并到中央仓库中。这就可以在不动官方代码的前提下在开发者之间共享一个feature。因为master是唯一的特殊分支，在中央仓库中储存多个feature分支并不会带来什么问题。当然，这也是一种备份本地提交的好办法。

** Pull Requests **

除了隔离feature的开发环境之外，分支使得我们可以通过pull requests来讨论代码改动。一旦某人完成了一个feature，他们不需要立刻合并到master中。他们会合并到feature分支，然后发起一个pull request要求合并他们的改动到master中。这给了其他开发者一个在其进入主代码库之前审查代码改动的机会。

代码审查时pull requests的一个好处，但是它却是被设计做一种讨论代码的通用方法。你可以认为pull requests是针对某个分支的讨论。这意味着也可以在开发流程中更早的阶段使用它。比如说，如果一个开发者需要帮助，他就可以发起一个pull request。相关的人会被自动通知，他们就可以看到提交下面的问题了。

一旦一个pull request被接受了，发布一个feature的过程和Centralized Workflow是一样的。首先，需要确保本地的master分支和上游的master进行了同步，然后，你将feature分支合并到master，再合并到中央仓库的master中。

一些代码管理工具可以帮助我们处理pull requests，如Bitbucker或Stash。

##Example

下面的例子将演示如何将pull requests作为代码审查的一种形式，但是切记它还可以用于很多其他的目的。

** Mary begins a new feature **

{% img /img/feature2.png %}

在她开始开发一个feature之前，她需要一个隔离的分支。她可以新开一个分支，

```
git checkout -b marys-feature master
```

这检出了一个叫作marys-feature的分支，基于master。选项-b告诉Git如果这个分支不存在，就创建一个。在这个分支上，Mary edit、stage并且commit，按照通常的方式，提交多次后建立起了她的feature。

```
git status
git add <some-file>
git commit
```

** Mary goes to lunch **

{% img /img/feature3.png %}

在早上Mary为她的分支进行了若干次提交。在她去吃午饭之前，应该把她的feature分支上传到中央仓库。这相当于进行了备份，但是如果Mary和其他开发者进行协作，那这使得其他开发者可以访问她的提交内容了。

```
git push -u origin marys-feature
```

这个命令将marys-feature上传到中央仓库origin，选项-u将其添加为一个远程跟踪分支。设置好这个跟踪分支后，Mary可以直接执行git push，无需任何其他参数就可以上传feature了。

** Mary finishes her feature **

{% img /img/feature4.png %}

当Mary吃完午饭回来后，她完成了她的feature。在她将其合并入master之前，她需要发起一个pull request让组内其余的人知道她完成了。但是首先，她需要确保中央仓库已经有了她最新的提交，

```
git push
```

然后，她在她的Git GUI中发起了一个pull request，请求合并marys-feature到master，其他组员将会自动收到提醒。pull requests允许在提交的下面进行评论，所以这对于问问题、讨论来说非常简单。

** Bill receives the pull request **

{% img /img/feature5.png %}

Bill收到了pull request并且查看了marys-feature。他决定在集成到官方工程之前进行一些修改，然后他和Mary前前后后通过pull request进行了一番交流。

** Mary makes the changes **

{% img /img/feature6.png %}

Mary通过edit、stage、commits完成了修改，并且上传到了中央仓库。她所有的活动都在pull request中有所展现，Bill仍然可以进行评论。

如果他愿意，Bill可以拿到marys-feature，在自己的本地备份中工作。任何他的提交也会出现在pull request中。

** Mary publishes her feature **

一旦Bill决定要接受这个pull request了，某个人需要合并这个feature到稳定的工程中去，这个人既可以是Bill也可以使Mary。

```
git checkout master
git pull
git pull origin marys-feature
git push
```

首先，不论是谁都要检出master分支，然后确保它是最新的。然后，git pull origin marys-feature将中央仓库中的marys-feature合并到本地的master。有也可以简单的使用git merge marys-feature，但是上面的命令确保你总是拿到最新的feature branch。最后，更新过的master需要上传到origin去。

一些GUI能够自动化pull request的接受过程，仅仅需要点击Accept按钮，就可以触发一系列命令完成工作。如果你的不行，它至少也可以在合并代码之后自动关闭这个pull request。

** Meanwhile, John is doing the exact same thing **

当Mary和Bill开发marys-feature，在Mary的pull request中讨论时，John也在做同样的事情。通过隔离feature到不同的分支，所有人都可以独立的工作，如果必要和其他开发者共享代码改动也是很轻松的事情。

#Gitflow Workflow

{% img /img/gitflow1.png %}

Gitflow Workflow源自Vincent Driessen在[nvie](http://nvie.com/)的文章。

Gitflow Workflow定义了一个用于工程发布的严格的分支模型。比起Feature Branch Workflow稍微复杂一些，其为管理较大项目提供了一个可靠的框架。

这种工作模式在Feature Branch Workflow的基础上没有增加新的概念或者命令。它只是为不同的分支定义了明确的角色，并且定义它们之间何时以及如何交互。与Feature Branch Workflow相比，它为准备、维护、发布定义了自己的分支。当然，你也可以享受到所有Feature Branch Workflow拥有的好处：pull requests，隔离环境，和更加有效的协作。

##How it works
Gitflow Workflow仍然使用一个中央仓库。与其他工作模式相同，开发者可以在本地工作，然后再将分支push到中央仓库中。唯一的不同在于工程的分支结构。

** Historical Branches **

与单一master分支不同，本工作模式使用两个分支来记录工程的历史。master分支存储官方发布的历史，develop分支用于集成feature。要为master分支的每一个提交打上一个版本号作为tag也是很方便的。

{% img /img/gitflow2.png %}

本工作模式的其余部分都在考虑让这两个分支进行交互。

** Feature Branches **

每一个新feature应该在自己的分支上，这个分支可以被push到中央仓库以便备份或者协作之用。但是，feature分支把develop分支作为父分支，而不是从master分支上分裂出来。当一个feature完成后，它会被合并会develop分支。Features永远都不应该直接和master交互。

{% img /img/gitflow3.png %}

注意feature分支和develop分支就是Feature Branch Workflow。但是Gitflow Workflow并不只是这样。

** Release Branches **

{% img /img/gitflow4.png %}

当develop收集到了足够一次发布的features时，或者一个预先约定的发布日期到达时，你就从develop分裂出一个分支。这个分支的创建就代表着一个发布周期的开始，所以这个时间点之后任何新feature都不能加进来，除了修复bug，文档生成，和其他发布相关的任务才能进入这个分支。当这个分支可以发布时，这个分支将会合并到master，并且用版本号打上一个tag。除此之外，还应该合并会develop分支，因为这个分支可能包含了在创建之初还没有的进展。

使用一个特定的分支来准备发布，就可以让一个团队继续优化当前版本，而另一个团队继续为下个版本开发新feature。它同时也创建了良好定义的开发术语，比如我们可以说，“这周我们准备4.0版的发布”，实际上我们也可以在代码仓库中看到这个结构。

** Maintenance Branches **
{% img /img/gitflow5.png %}

Maintenance或者说hotfix分支用来对生产环境的版本进行快速修复。这是唯一可以直接从master分裂的分支。一旦修复完成，就应该马上被合并到master和develop分支中，而且master应该用更新版本号打上一个tag。

拥有一个特定的开发线路供修复bug使得你的团队可以在不干扰其他工作流程，也不用等待下个发布周期的前提下处理issue。你可以认为maintenance分支是一个直接和master分支交互的发布分支。

##Example

下面的例子展示了这个工作模式如何处理一个发布周期。我们假设已经创建了一个中央仓库。

** Create a develop branch **

{% img /img/gitflow6.png %}

第一步是在默认master分支的基础上补全一个develop分支。一个简单的方法是在本地创建一个空的develop分支，然后push到服务器上，

```
git branch develop
git push -u origin develop
```

这个分支包含了整个工程的完整历史，而master只包含了一个削减过的版本。其他开发者现在应该clone中央仓库，并且创建一个develop的跟踪分支。

```
git clone ssh://user@host/path/to/repo.git
git checkout -b develop origin/develop
```

现在所有人都在本地建立起了一份历史分支的备份。

** Mary and John begin new features **

{% img /img/gitflow7.png %}

我们的例子开始于John和Mary要开发不同的feature。他们都需要创建各自的分支。他们不应该以master作为基础，而应该以develop作为基础，

```
git checkout -b some-feature develop
```

他们两个都通过edit、stage、commit的方法，向各自的feature分支进行提交。

```
git status
git add <some-file>
git commit
```

** Mary finishes her feature **

{% img /img/gitflow8.png %}

在提交了若干次之后，Mary认为她的feature已经完成了。如果她的团队使用pull requests，这就是一个合适的发起一个pull request请求合并她的feature到develop的时间。否则，她可以合并到本地的develop然后push到中央仓库，

```
git pull origin develop
git checkout develop
git merge some-feature
git push
git branch -d some-feature
```

第一个命令确保本地的develop分支是最新的。注意feature始终不应该直接合并到master中。冲突的解决方法和Centralized Workflow中描述的一样。

** Mary begins to prepare a release **

{% img /img/gitflow9.png %}

当John仍然在开发他的feature时，Mary开始准备项目的第一个官方发行。跟feature开发一样，她使用一个新的分支来封装发行的准备工作。这一步也是创建发行版本号的时候，

```
git checkout -b release-0.1 develop
```

这个分支用来整理、测试、更新文档，以及为下一次发行做一切准备工作。这就是像是一个为了优化发行版本的feature分支。

一旦Mary创建了这个分支，并且push到了中央仓库，这个发行就变成feature-frozen了。任何不在develop中的功能都要推迟到下个发布周期。

** Mary finishes the release **

{% img /img/gitflow10.png %}

当release准备好上线时，Mary将其合并回master和develop，然后删除这个发行分支。合并会develop是非常重要的，因为重要的修改可能被加入了这个发行分支，他们对于新的feature而言是有用的。如果Mary的组织强调代码审查，这也是一个理想的发起pull request的位置。

```
git checkout master
git merge release-0.1
git push
git checkout develop
git merge release-0.1
git push
git branch -d release-0.1
```

release分支好像是一个feature开发版本(develop)与公开发行版本(master)之间的缓冲区。不论什么时候你合并回master，你都应该为提交打上tag，

```
git tag -a 0.1 -m "Initial public release" master
git push --tags
```

Git提供了一些脚本，可以在代码仓库发生了一些特定的事件时触发。你可以配置，使得每当有master分支或者一个tag被push到中央仓库时，自动的创建一个公开发行版本。

** End-user discovers a bug **

{% img /img/gitflow11.png %}

在发布之后，Mary回到下个版本的feature开发中。直到一个终端用户在现有版本中发现了一个bug。为了处理这个bug，Mary从master创建了一个maintenance分支，修复bug，然后直接合并回master。

```
git checkout -b issue-#001 master
# Fix the bug
git checkout master
git merge issue-#001
git push
```

和release分支一样，maintenance分支包括了重要的修改，这些修改也应该合并到develop中。然后，就可以删掉这个分支了，

```
git checkout develop
git merge issue-#001
git push
git branch -d issue-#001
```

#Forking Workflow 

Forking Workflow和本文中谈到的其他工作模式都不相同。它并不是只有一个单独的服务器端的仓库来扮演中央代码库，在这个模式中，每个开发者都有一个自己的服务器端的仓库。这意味着每一个贡献者都有两个Git的仓库，一个私有的本地的，一个共有的服务器端的。

{% img /img/forking1.png %}

Forking Workflow最大的好处在于不用所有人都push到一个中央仓库。开发者可以push自己的服务器端仓库，只有项目的维护者才能push到官方的仓库。它允许维护者维护者接受其他开发者的提交，而不给他们对官方代码库的写权限。

这就构成了一种分布式的工作模式。为大型团队安全的协作提供了一种灵活的方式。

##How it works
跟其他Git工作模式一样，Forking Workflow开始于服务器端的官方公开仓库。但是当一个新开发者想要在这个项目上工作时，他并不能呢个直接clone这个官方工程。

他需要先fork这个官方工程，在服务器上创建一份自己的备份。这个新的备份就是它的个人公开仓库，其他开发者都不允许push到这个仓库，但是他们可以pull这个仓库的改动到自己的代码中。当他们创建自己服务器端的备份后，开发者执行git clone在本地建立一个备份。这就是他们私人的开发环境，就跟别的工作模式一样。

当他们想要发布本地的提交时，他们push这个提交到他们自己的公开仓库中，而不是官方的那个。然后，他们发起一个pull request，让官方工程维护者知道有一个更新等待被集成。那个pull request也成为了一个关于提交代码讨论的地方。

有O把这个feature集成到官方代码库中，维护者pull贡献者的改到到本地，检查功能是否正常，是否打破工程，然后合并到本地master，然后push到服务器的公开仓库中。现在这个feature就是这个工程的一部分了，其他开发者也应该从官方仓库pull这些改动到本地来进行同步。

** The official repository **

在Forking Workflow中的“官方”一词只是一种习惯。从技术角度来说，Git并不认为官方的公共仓库和其他开发者的公共仓库有什么不同。实际上，唯一使得官方仓库官方的原因在于其实工程维护者的公开仓库。

** Branching in the Forking Workflow **

所有这些个人的公共仓库只是一种方便共享分支给其他开发者的方式。所有人仍然应该使用分支来隔离不同的feature，就像Feature Branch Workflow和Gitflow Workflow一样。唯一的区别是这些分支如何共享。在Forking Workflow中，他们被pull到另一个开发者的本地仓库中，而Feature Branch Workflow和Gitflow Workflow则是push到官方仓库中。

##Example

** [github](https://github.com/) **

** [gitcafe](https://gitcafe.com) **