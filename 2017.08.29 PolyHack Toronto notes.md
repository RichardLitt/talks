# PolyHack Toronto

### Pain

I had a problem. `https://` consistently broke for me when I tried to push. This is most likely because I had 2FA enabled, didn't have the key stored in my osx-credentials, and [wasn't using a token](http://olivierlacan.com/posts/why-is-git-https-not-working-on-github/), but I didn't know that yet.

All of this could have been avoided if I had just figured this out earlier.

### My solution

Switching the git protocol to `ssh://` solved the problem for me, but I didn't want to do that manually, each time. I work with more git repositories than most people I know, as I do documentation work fairly often. At current count, I have 367 repositories in my `src` folder, and around a thousand repositories on GitHub. Not doing that manually.

### Protocols out there

What kind of protocols are there? https://gist.github.com/grawity/4392747

#### git

- Does not add security beyond what Git itself provides. The server is not verified.
- You cannot push over it.

#### https

- HTTPS will always verify the server automatically, using certificate authorities (generally secure).
- Uses password authentication for pushing, and still allows anonymous pull.
- Downside: You have to enter your GitHub password every time you push. Git can remember passwords for a few minutes, but you need to be careful when storing the password permanently – since it can be used to change anything in your GitHub account.
- If you have two-factor authentication enabled, you will have to use a personal access token instead of your regular password.
- HTTPS works practically everywhere, even in places which block SSH and plain-Git protocols. In some cases, it can even be a little faster than SSH, especially over high-latency connections.

#### http

- Doesn't work with GitHub anymore, but is offered by some other Git hosts.
- Works practically everywhere, like HTTPS.
- But does not provide any security – the connection is plain-text.

#### ssh

Uses public-key authentication. You have to generate a keypair (or "public key"), then add it to your GitHub account.
Using keys is more secure than passwords, since you can add many to the same account (for example, a key for every computer you use GitHub from). The private keys on your computer can be protected with passphrases.
On the other hand, since you do not use the password, GitHub does not require two-factor auth codes either – so whoever obtains your private key can push to your repositories without needing the code generator device.
However, the keys only allow pushing/pulling, but not editing account details. If you lose the private key (or if it gets stolen), you can just remove it from your GitHub account.
A minor downside is that authentication is needed for all connections, so you always need a GitHub account – even to pull or clone.
You also need to carefully verify the server's fingerprint when connecting for the first time. Many people skip that and just type "yes", which is insecure.
Uses port 9418 for it's daemon. Can be blocked for Corporate stuff. 

(Note: This description is about GitHub. On personal servers, SSH can use passwords, anonymous access, or various other mechanisms.)

#### /

-  Clone from /srv/git/project.git
- This is often used if everyone on your team has access to a shared filesystem such as an NFS mount, or in the less likely case that everyone logs in to the same computer.
- Git tries to use hardlinks or directly copy the files it needs
- Everyone who has access can manipulate date, uses same file access as server
- Not necessarily faster

#### file://

- Same as above, but git uses the same transfer processes it does for across a network


### Setting these globally

You can clone everything over git://, but tell Git to push over HTTPS.

```
[url "https://github.com/"]
    pushInsteadOf = git://github.com/
```

Run this in  `~/.config/git/config` or `.gitconfig`.

### Setting locally

```
[remote "origin"]
    url = git://RichardLitt/git-remote-to-ssh.git
    pushUrl = ssh://RichardLitt/git-remote-to-ssh.git
```

### What does GitHub allow? 

From [`hosted-git-info`](https://www.npmjs.com/package/hosted-git-info):

```
github: {
  // First two are insecure and generally shouldn't be used any more, but
  // they are still supported.
  'protocols': [ 'git', 'http', 'git+ssh', 'git+https', 'ssh', 'https' ],
  'domain': 'github.com',
  'treepath': 'tree',
  'filetemplate': 'https://{auth@}raw.githubusercontent.com/{user}/{project}/{committish}/{path}',
  'bugstemplate': 'https://{domain}/{user}/{project}/issues',
  'gittemplate': 'git://{auth@}{domain}/{user}/{project}.git{#committish}',
  'tarballtemplate': 'https://{domain}/{user}/{project}/archive/{committish}.tar.gz'
}
```

### So, back to https

I want to change all of my repository remotes from `https` to `ssh`. 

```
🐕  git remote -v
origin	https://github.com/RichardLitt/git-remote-to-ssh.git (fetch)
origin	https://github.com/RichardLitt/git-remote-to-ssh.git (push)
```

To

```
🐕  git remote -v
origin	git@github.com:RichardLitt/git-remote-to-ssh.git (fetch)
origin	git@github.com:RichardLitt/git-remote-to-ssh.git (push)
```

### Shell script

I found a handy shell script online. Let's look at it.

```sh
git checkout 79c4b111ad4d372f08a60e1fa97399e885753b95
```

### Atwood's Law

> Any application that can be written in JavaScript, will eventually be written in JavaScript.

### Exec all the bash

My first thought was using `exec()`, which can be pretty evil. I decided to use [`execa()`](https://github.com/sindresorhus/execa). So, let's do a simple port.

```js
execa(`git remote -v | grep -m1 '^${remote}' | sed -Ene's#.*(https://[^[:space:]]*).*#\1#p'`)
  .then((res) => console.log(res))
  .catch((err) => console.log('Screw you everything is broken'))
```

Of course, I got this:

```
SyntaxError: Octal escape sequences are not allowed in strict mode.
```

At this point, on my third refactor, I wondered - well, you know, there's probably a better parsing library out there for git, and I probably don't need to use `exec`, or `grep`. It's a silly thing to do. 

### Parsing gitconfig

I spent a while doing fs.readFileSync, before I decided this was stupid.

```js
const parse = require('parse-git-config')
var config = parse.sync();
```

Let's go and livecode.

### Voila

The thing is done! It works. It also:

- Transforms any git url to ssh, not just https
- Works for any valid git url
- Is way easier for me to debug

### Publish

And that's it. I now have the thing I wanted, and I've learned a ton along the way. Hooray!

### Irony

I renamed the GitHub repo from `github-origin-https-to-ssh` to `git-remote-to-ssh`, and then manually edited my .gitconfig file to reflect the change. Did I accomplish anything? I've no idea.

