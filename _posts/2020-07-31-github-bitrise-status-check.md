---
layout: post
title: GitHub, Bitrise and Status Checks
tags: [github, bitrise]
---
As we develop our software daily, we need to take care of a lot of things. Thankfully there are great tools that help us automate repetitive tasks and free us to focus on new ideas and write better codes. As our repository, we have GitHub with a lot of useful features, and Bitrise as a CI/CD tool helps us build, integrate, and deliver our apps to the world. There are many branching strategies; Martin Fowler covers them in his excellent article [Patterns for Managing Source Code Branches](https://martinfowler.com/articles/branching-patterns.html). One of them, which is popular, is [feature branching](https://martinfowler.com/articles/branching-patterns.html#feature-branching).

It is suggested then to protect the master branch or any other main branches to keep them safe and ready to release. One way to protect your branch is Status Checks. With a Status Checks, you can guarantee the feature branch won’t merge into your master branch unless it passes the required checks. Assume a case you create a feature branch, work on it, and after the work is done, you want to merge it. You will create a pull request, and your CI/CD tool runs all the tests to be sure that your new code won’t break any tests on the master branch. It only allows you to merge when all the tests passed; otherwise, you can’t put the master branch in danger.

### The Stepes

To protect your master branch in GitHub with Bitrise as the CI/CD tool, follow these steps:

* **Register a Webhook**


  When you create your Bitrise workflow, you need to register a Webhook.

  ​	![register a Webhook]({{site.baseurl}}/assets/img/github_bitrise/register_webhook.png){:height="75%" width="75%"}

* **Update Triggers**

  In the Triggers section, change the target branch of the pull request to the master branch.

  ​	![update triggers]({{site.baseurl}}/assets/img/github_bitrise/trigger_master.png){:height="75%" width="75%"}

* **Enable GitHub Checks**

  In Bitrise settings, enable GitHub checks then follow the [install our Bitrise Checks app to your GitHub repository](https://github.com/apps/bitrise-checks/installations/new).

  ​	![enable GitHub checks]({{site.baseurl}}/assets/img/github_bitrise/enable_status_check.png){:height="75%" width="75%"}

* **Create a pull request**

  In your GitHub repository, create a pull request. It triggers the Bitrise to run the workflow. Note that you still can merge the pull request no matter what would be the Bitrise result. So your branch is not safe yet.

  ​	![create a pull request]({{site.baseurl}}/assets/img/github_bitrise/create_pull_request.png){:height="75%" width="75%"}

  With doing so, GitHub can find Bitrise when you add a rule to protect your branch by requiring status checks to pass, before merging.
  It’s time to add a branch protection rule. Go to your Github repository settings, add a rule, write master as your branch name, enable Require status checks to pass before merging, and select Bitrise. And it’s done. Great, now your branch is protected.

  ​	![branch rules bitrise]({{site.baseurl}}/assets/img/github_bitrise/branch_rules_bitrise.png){:height="75%" width="75%"}

* **A protected branch**

  After the master branch is being protected, when you create a pull request before you can merge the changes to the master branch, Bitrise should check whether your tests will pass or not. Until then, the merge button is disabled.

  ​	![before status check]({{site.baseurl}}/assets/img/github_bitrise/before_status_check.png){:height="75%" width="75%"}

  

  In case the Status Checks fails, it’s not possible to merge until the failure getting fixed.

  ​	![status check rejection]({{site.baseurl}}/assets/img/github_bitrise/status_check_reject.png){:height="75%" width="75%"}

  

  And when all your Status Checks pass, you have a green light to safely merge the feature branch to the master branch.

  ​	![status check success]({{site.baseurl}}/assets/img/github_bitrise/after_status_check.png){:height="75%" width="75%"}


### Conclusion

  In our busy day full of distractions, we need to take some measures to delegate our works. One way is delegating repetitive tasks and automate them. With this automation, we also need to adopt some protection to avoid breaking our work by mistake. One of them is protecting our mainline to be healthy and ready to release. With enabling Status Checks for our mainline (the master branch in GitHub), we can be sure that we can not push our commits directly to the master branch,  and merging other branches to it is safe.
