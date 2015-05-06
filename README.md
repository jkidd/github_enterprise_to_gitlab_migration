# Migrating from Github enterprise to Gitlab

## Background
We have been using Github enterprise for around 3 years. Initially Github Enterprise was chosen by our organization over similar offerings due to Github's clearly superior feature set. At the time Gitorious was the only other product that was close as far as functionality.  Then and even now (as of this writing) Github is still lacking a few critical features that we required. Namely better LDAP integration and JIRA integration. 

A major shortcoming for us was the lack of ability to apply permissions to Github organizations and repositories based on LDAP group membership. With many organizations containing hundreds of repositories, maintaining permissions manually would be cumbersome and time consuming. Also, due to Github licensing, we required the ability to disable accounts once folks were removed from particular groups in LDAP to free up licenses for new employees. 

Jira integration was also a requirement. Github has the ability to perform webhooks for any number of actions against any given repo. For each repo we simply define a URL for Github to post to with a JSON string describing the action. A simple webapp simply needed to parse the JSON, looking for references to Jira tickets. Once found, it would log a comment on whatever Jira ticket was referenced with the details of the referencing commit. The only thing that really needed to be done with regard to Github was automatically ensuring this webhook was defined for every single organization repository on a regular basis.

## Github Automation Requirements
* Create new users based on a particular LDAP group
* Grant permissions to Github organizations to a user based on LDAP group membership.
* Disable accounts for users no longer belonging to the Github LDAP group.
* Create ticket comments in Jira when ticket references were made in Git commit messages.
* Periodically update permissions and Jira hooks for organizations and repos.

## Github Approach
The Github appliance is "locked down" with regard to root access to dissuade folks from doing anything stupid and to make providing support somewhat more predictable. I of course was determined to do something stupid so I mounted the Github virtual machine root filesystem outside of the running Github VM to create a user and a /home/user/.ssh/authorized_keys file while also granting that user sudo access.

Since I had direct access to the Github virtual appliance, which was an Ubuntu VM with the Github stack installed on it, I decided that I was going to directly interact with the Rails stack using rails runner. It would be quicker and I'd have much more control right? This of course was a terrible idea because each time Github released updates I'd have to test/fix my extensions. Don't be "clever", use the API.

So that's what I did. I was able to crank it out very quickly and it accomplished what was needed. It was scheduled to run several times a day and occasionally broke with Github updates. We ran this way for 3 years.

## Why switch from Github to Gitlab
With Gitlab's current feature set there was no compelling reason to continue using Github other than the effort it would take to migrate away. Not only did Gitlab community edition do everything that we needed, it is free and completely open source. We can choose the flavor of Linux we install it on and we have complete control over it. Also saving the $250 per user per year licensing fees are also a nice perk. Gitlab offers standard support for $49 per user per year for their "Enterprise" version and is the closest analog to our level of Github support. Github requires purchasing licenses in increments of 20. Gitlab requires licenses purchased in increments of 100. 

The locked-down nature of Github's virtual appliance made dealing with certain issues difficult without engaging Github's support team. Well this would have been the case had I not granted myself a root login. It's frustrating to immediately be forced to contact support when the problem is often something that could otherwise easily be dealt with. For example, we had an early version of the Github VM that didn't have nearly enough space allocated for the root partition. It was a LVM volume so all that was needed was to add another virtual disk and extend the root volume. Had we not had root access, this would have been an ordeal that would have required wasting time with Github support rather than the 2 minutes it took otherwise.

Github's virtual appliance also made for issues with upgrades. It has been quite some time since I've run an upgrade so hopefully the following is no longer true. Each update package includes everything needed to upgrade Github from any prior version up to the latest. The reason for this is evident though you can imagine, this is a lot of stuff. This not only includes the software necessary to run Github but also all OS upgrade packages. The updates quickly became very large. At one point an upgrade was so large that it caused Github to throw 500 errors when transferring the package through the management interface (an issue dealt with in a later update IIRC). The transfer was actually succeeding and the upgrade process proceeding which was apparent by tailing the Chef log that would otherwise have been inaccessible had root access been unavailable.

## The Migration from Github to Gitlab
Our must-haves for migration were straightforward. All pull requests from Github would be migrated over to issues in Gitlab. The plan was, for each pull request, create an issue in Gitlab with the same pull/issue number, then aggregate the comments, commits, and commit messages into the summary body of the issue. All active users and user ssh keys would be migrated. All organizations, org repositories, and user permissions to organization repositories would be included. All current custom Github functionality would be preserved including automatic user creation based on LDAP group membership, automated permission granting to groups/organizations, and our Jira hook for referencing tickets.

It was decided that no personal user projects, forked or otherwise, would be migrated to force some spring cleaning. Wikis weren't extensively used so it was decided that for the very few, those could be manually migrated if necessary. It was also decided that any Gists folks wished to migrate, they could do so manually; More spring cleaning.

Finally we wanted all links to Github pull requests and commits made elsewhere throughout our organization to remain funcational. This was easily accomplished by pointing our Github DNS at our Gitlab server and making two simple rewrite rules in Nginx, one to rewrite github to gitlab and another to redirect github pull request URLs to Gitlab issues. We preserved all org/group names, repo names, and pull request numbers making the URL translation dead simple.

So that is what was done. The Github API and the octokit ruby gem, the Gitlab API and associated ruby gem, and NET::LDAP were all used for the migration. The code available in this repository was used for both the migration and the scheduled updates to sync LDAP/AD and Gitlab based on group membership.
