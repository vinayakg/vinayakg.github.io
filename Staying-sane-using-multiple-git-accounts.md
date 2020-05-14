# Staying sane while using multiple git accounts

The other day I wanted to configure multiple git accounts on my laptop and I felt I had to jump through a lot of hoops to get that done, I understood how things work actually when you connect using ssh and the role of ssh-agent. Also had to refer a bunch of articles and connect dots from them to be able to make it work for me. So I felt I should write the steps I followed so it becomes easy for other folks looking to setup similar and save them time and possible pain :-)



If you don't need have the need to use multiple git accounts, then this article may not be for you, nevertheless if you decide to read - am sure there will be learnings for you on how things actually work.  



In case you have an existing setup and you would like to setup your git fresh, then you may go to the [Troubleshoot section](#Troubleshooting)



### Setting Up

#### Generate SSH keys


Check your current SSH keys (back if you need them before deleting or leave them). They are all located at `~/.ssh/`. View them using `ls -al ~/.ssh/`

Generate new SSH keys using below commands

```bash
ssh-keygen -t rsa -C "vg@work.com" -f "id_work_user"
```

Set the passphrase so even if you loose the device, the keys cannot be abused for any access, let's use RSA and avoid using DSA based algorithms (blog post coming soon). There are 2 files generated `id_work_user` and `id_work_user.pub` at the location `~/.ssh`.

Let's dissect the above command. 

`-t` option to specify the encryption algorithm, else DSA is used. I will write more about why DSA(default) is not a great option for ssh and the issues using that even with password enabled ssh keys

`-C` comment that will help us associate the key with its purpose, we will see the command later

`-f` file name to use for saving

Run the above command one time more for the other accounts and we are done with this step

```shell
ssh-keygen -t rsa -C "vinayakg@personal.com" -f "id_my_user"
```

All the generated keys are located at `~/.ssh` and we have 2 private keys (no extn) and 2 public keys (`.pub`)

#### Let your provider know these keys 

This will help us setup the auth when we push or pull code via git. We will use [GitHub](https://gitHub.com) for this example

Copy your public key `pbcopy < ~/.ssh/id_work_user.pub` and then log in to your personal GitHub account:

1. Go to `Settings`
2. Select `SSH and GPG keys` from the menu to the left.
3. Click on `New SSH key`, provide a suitable title, and paste the key in the box below
4. Click `Add key` — and you’re done!

Repeat the above steps for the other keys generated for other accounts in the above step



#### Setting up and usage



##### The SSH-Agent way


To use the above keys for auth we need to register with the `ssh-agent` on our machine. Ensure `ssh-agent` is running on the machine by using this command

```shell
eval "$(ssh-agent -s)"
```

Now add these agents using the below command

```bash
ssh-add ~/.ssh/id_rsa
ssh-add ~/.ssh/id_rsa_work_user1
```

###### Usage

In order to start using the ssh-agent, you need to have few things in place like useremail registered with git - git uses this for authentication, username (does not matter - could be any, please try and see)

Commands:

```bash
git config -l                                    #list the current config
git config --global user.name 'vinayakg'         #set the git username in config globally
git config --global user.email 'vg@work.com'     #set the git useremail in config globally 
```

With the config set correctly, we just need to set the ssh-agent correctly - it only accepts one active key at a time

```bash
ssh-add -D && ssh-add ~/.ssh/id_work_user
```

The above command will delete all existing set keys ssh-agent for ssh-agent and will set the one we want

And you can do a git clone on a new repository very easily and work with it by using our familiar commands

```shell
git clone git@github.com:vinayakg/repo
```

The next time you want to work on a repository linked to different account follow the above steps (`git config` && `ssh-add`). It's also possible to create handy aliases in git or in shell to save the number of keystrokes. You need not use the global command with git config if you are working on an existing repository. Its also possible to set email in the local .git/config, but this is not helpful for new repositories

##### The SSH Config way	(Preferred way)

Here we are going to use the same ssh config that we use for SSH auth while connecting to other machines over cloud. This method does not require additional steps except for changing the url while `git clone` or setting the url while adding `git remote`  and does not rely on any folder structures or other commands .

The SSH config is located at `~/.ssh/config`. If this file does not exist for you, please go ahead and create it.

```shell
touch ~/.ssh/config 
```

Now open the editor and add the below config

```bash
# Personal account
Host gh                               # used for identifying amongst other accounts
   HostName github.com                # git provider website
   User git                           # default user, no need to change
  #AddKeysToAgent yes                 # we are using multiple, so we will ignore this 
  UseKeychain yes                     # save to keychain
   IdentityFile ~/.ssh/id_work_user   # private key
   
# Work account-1
Host gh-work    
   HostName github.com
   User git
   IdentityFile ~/.ssh/id_rsa_work_user
```

 We will see below how to use these ==Host== name that we setup

######  Usage

Now we have 2 accounts configured. Let's say you want to clone code from personal account. Use the below command. This command will set the right git remotes, so we don't have to worry when we switch back after working in another account

```shell
git clone git@gh:vinayakg/repo
```



For second, we can use the similar command

```shell
git clone git@gh-work:work/repo
```



### Troubleshooting

#### Issues with existing git accounts on your machine

There is a default git account configured on the machine and that seems to take precedence no matter what and the only way is to delete them off your system. git config wont help here

Follow the below steps in order to reset your existing/previous settings

###### Step 1

```shell
$ git credential-osxkeychain erase
host=github.com
protocol=https
<press return>
<press return>
```



###### Step 2

Follow this step if step1 did not work

Go to Launcher :arrow_right: search box :arrow_right:type git username :arrow_right: then press delete :x:

###### Step 3

Now check you ssh-add listing and see the certification that are currently configured. If you see any suspects go ahead and delete them

`ssh-add -l` lists the attached credentials

`ssh-add -D` delete all the attached ones

And if you have multiple or `user.email` or `user.name` then go ahead and clear them with the following commands

```bash
git config --unset user.email && git config --unset user.name
```



#### Permission denied (publickey)

Try to debug and see why the auth is failing using the below command, user is always `git` (for more details see [here](https://help.github.com/en/github/authenticating-to-github/error-permission-denied-publickey#always-use-the-git-user)) and `gh` is the alias that we have set (`gh` translates to `github.com`) 

```shell
ssh -vT git@gh
```

Check if you have the right key on [GitHub](github.com) by using the below command

```shell
ssh-add -l -E md5
```



#### Change remotes for existing git repositories

If you have existing repository, you can easily change the remotes using the below command

```bash
git remote set-url origin git@gh-work:work/repo
```



### Additional Tips

You can also use `git config` with includeIf to set `git config` for work and personal separately, thats mostly a preference; I will leave that for now

```shell
[includeIf "gitdir:~/work/"]
		path = ~/work/.gitconfig
```

And the corresponding config too

```
[user]
    email = john.doe@company.tld
```



### Learnings & Conclusion

It was fun to learn and understand how it works, unless I experimented myself with 2 accounts I was not sure what combinations can work and what wont. Also understood the role of ssh-agent and how you don't need the agent for git auth if you use the ssh config

If there are any missing pieces, feel free to let me know



#### References:

https://stackoverflow.com/questions/4220416/can-i-specify-multiple-users-for-myself-in-gitconfig

https://help.github.com/en/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent#adding-your-ssh-key-to-the-ssh-agent

https://www.freecodecamp.org/news/manage-multiple-github-accounts-the-ssh-way-2dadc30ccaca/

https://help.github.com/en/github/authenticating-to-github/error-permission-denied-publickey



