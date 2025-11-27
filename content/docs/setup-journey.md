---
title: "Setting up Hugo"
date: 2025-11-26T20:58:09-05:00
draft: true
weight: 20
---

# My Hugo Setup Journey on WSL
*A real-world log of installing Hugo, fixing errors, and getting a docs site running with hugo-book*

## Introduction

This page documents my **actual terminal journey** getting Hugo installed on a WSL Ubuntu environment, configuring the **hugo-book** theme, and fixing all the issues I ran into along the way.

It includes:

- Wrong install methods I tried (snap, old apt packages, manual tarballs)
- PATH and GLIBC problems
- The final, stable solution using **Homebrew on Linux**
- How I finally got a working docs site with prefilled content

This is partly a self‑reference and partly a guide for anyone who hits the same problems on WSL.

---

## 1. First attempts: snap and apt

I started in my intended project directory:

```bash
cd ~/code/sandwich
pwd
# /home/santi/code/sandwich
```

My first attempt was to install Hugo via **snap**:

```bash
sudo snap install hugo
```

This failed with:

```text
error: cannot communicate with server: Post "http://localhost/v2/snaps/hugo": dial unix /run/snapd.socket: connect: no such file or directory
```

I tried again and got the same error. This happens because **snapd isn’t running on WSL**, so snap is not a viable way to install Hugo there.

Next, I tried the Ubuntu package via **apt**:

```bash
sudo apt install hugo
hugo version
```

Output:

```text
Hugo Static Site Generator v0.68.3/extended linux/amd64 BuildDate: 2020-03-25T06:15:45Z
```

So I had Hugo, but it was **v0.68.3** — very old compared to current releases.

---

## 2. Creating the site and adding the hugo-book theme

With apt-installed Hugo, I created the site in the current directory:

```bash
hugo new site .
```

Hugo scaffolded the site and printed the usual “Just a few more steps” message.

Then I tried to add the **hugo-book** theme as a Git submodule:

```bash
git submodule add httpsgithub.com/alex-shpak/hugo-book themes/hugo-book
```

This first failed with:

```text
fatal: not a git repository (or any of the parent directories): .git
```

I had forgotten to initialize Git, so I fixed that:

```bash
git init
git submodule add httpsgithub.com/alex-shpak/hugo-book themes/hugo-book
```

The first attempt to clone the submodule failed with an HTTP2 framing error, but a second `git submodule add` worked and fully cloned the theme.

---

## 3. Trying to run the theme: layout and `css` errors

I tried to run the site using the theme directly:

```bash
hugo server --minify --theme hugo-book
```

I immediately got:

```text
Error: add site dependencies: load resources: loading templates:
"themes/hugo-book/layouts/_partials/docs/html-head.html:32:1":
parse failed: template: _partials/docs/html-head.html:32: function "css" not defined
```

This error (`function "css" not defined`) comes from Hugo Pipes / SCSS processing. It typically means:

- Hugo is **too old**, or
- It’s not the **extended** build.

In my case, I had `v0.68.3` from apt, which is far too old for the current hugo-book theme.

I also created some content:

```bash
hugo new docs/_index.md
hugo new docs/getting-started.md
```

Then tried:

```bash
hugo server -D
```

This time Hugo ran, but I got a series of warnings:

```text
found no layout file for "HTML" for kind "page"
found no layout file for "HTML" for kind "section"
found no layout file for "HTML" for kind "home"
```

and the page was effectively blank. The root problem was still the **Hugo version / theme compatibility**.

---

## 4. Attempting manual upgrades with tarballs (GLIBC & PATH issues)

Realizing I needed a newer Hugo, I removed the apt version:

```bash
sudo apt remove hugo
```

Then I tried installing Hugo manually from GitHub releases. One example:

```bash
cd /tmp
curl -L -o hugo.tar.gz \
  https://github.com/gohugoio/hugo/releases/download/v0.149.0/hugo_extended_0.149.0_linux-amd64.tar.gz

tar -xzf hugo.tar.gz
sudo mv hugo /usr/local/bin/hugo
```

However, when I ran:

```bash
hugo version
```

I got:

```text
bash: /usr/bin/hugo: No such file or directory
```

This happened because my shell had cached the old `hugo` path (`/usr/bin/hugo`) from the apt install. After clearing the hash and continuing to experiment with tarballs (including v0.152.2), I still ran into problems.

At one point, `which -a hugo` showed:

```text
/usr/local/bin/hugo
/usr/bin/hugo
/bin/hugo
```

and running `hugo version` still reported the old v0.68.3, meaning the apt version was shadowing the new one. I re-installed and removed the apt package multiple times during this process.

When I finally cleared out the apt version and forced the system to use `/usr/local/bin/hugo`, I hit a new issue:

```bash
hugo version
```

returned:

```text
hugo: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.33' not found
hugo: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.32' not found
hugo: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.34' not found
hugo: /lib/x86_64-linux-gnu/libstdc++.so.6: version `GLIBCXX_3.4.29' not found
```

This meant:

- The Hugo binary I downloaded was compiled against **newer glibc and libstdc++** than what my WSL Ubuntu had.
- The binary simply could not run on this environment.

So:
- Snap didn’t work on WSL.
- The apt package was too old.
- The prebuilt binary from GitHub didn’t match my glibc version.

Time for a different approach.

---

## 5. Installing Homebrew on WSL and letting it handle Hugo

The final (and correct) solution was to use **Homebrew on Linux**. Homebrew installs Hugo in a way that works with the local system, and provides the **extended** build.

First, I removed the incompatible binary:

```bash
sudo rm /usr/local/bin/hugo
```

Then I made sure base tools were installed and up to date:

```bash
sudo apt update
sudo apt install -y build-essential procps curl file git
```

Next, I installed Homebrew:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

The installer printed a **“Next steps”** section instructing me to add brew to my shell. I followed it:

```bash
echo >> /home/santi/.bashrc
echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> /home/santi/.bashrc
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
```

Homebrew also recommended installing GCC, which I did:

```bash
sudo apt-get install build-essential   # already installed, but safe
brew install gcc
```

This step pulled in its own glibc and toolchain into `/home/linuxbrew/.linuxbrew`.

Then I opened a fresh shell:

```bash
exec $SHELL
```

And finally installed Hugo via Homebrew:

```bash
brew install hugo
```

Verification:

```bash
which hugo
# /home/linuxbrew/.linuxbrew/bin/hugo

hugo version
# hugo v0.152.2+extended+withdeploy linux/amd64 BuildDate=2025-10-24T15:31:49Z VendorInfo=brew
```

At this point I had:

- A **modern** Hugo version (v0.152.2)
- The **extended** build
- A binary that actually runs correctly in WSL

---

## 6. Serving the site from the correct directory

Earlier, I had accidentally tried to run Hugo from `/tmp`, which gave me:

```text
Error: command error: Unable to locate config file or config directory.
Perhaps you need to create a new site.
```

The fix was simple: run the server from the **project root** where `config.toml` lives:

```bash
cd ~/code/sandwich
hugo server -D
```

This time, with the Homebrew-installed Hugo, the site built correctly.

---

## 7. Prefilling content from hugo-book’s example site

To quickly get meaningful docs content and navigation, I copied the example content from the hugo-book theme:

```bash
cd ~/code/sandwich
cp -R themes/hugo-book/exampleSite/content.en/* content/
```

Added some finishing setup touches to `config.toml`:

```toml 
baseURL = "http://localhost:1313/" # Might change this later to support relative URLs. 
languageCode = "en-us"
title = "Sandwich"
theme = "hugo-book"

# Basic params to get started
[params]
BookTheme = "light"
BookSearch = false # hugo-book docs state that Search can be clunky, better turn it off for this small project
BookToC = true
BookComments = false

# Goldmark trims unsafe outputs which might prevent some shortcodes from rendering
[markup.goldmark.renderer]
  unsafe = true
```

Then I ran:

```bash
hugo server -D
```

Now my local site loaded with the **full hugo-book demo content** (sidebar, sections, sample pages, etc.), and I could start replacing it with my own docs, including this setup journey page.

---

## 8. Lessons learned

A few key takeaways from this whole process:

1. **Snap is not a good choice on WSL.**  
   `snapd` isn’t running, so `sudo snap install hugo` fails.

2. **The Ubuntu apt package for Hugo can be very outdated.**  
   I got `v0.68.3`, which is too old for modern themes like hugo-book.

3. **Prebuilt GitHub binaries can fail on older distros due to GLIBC version mismatches.**  
   Even after fixing PATH, I couldn’t run the downloaded binary because my system glibc was older than what Hugo was compiled against.

4. **Homebrew on Linux is a great option for WSL.**  
   Installing Hugo with `brew install hugo` gave me a modern, extended build that works cleanly on WSL.

5. **Always run `hugo server` from the project root (or use `-s`).**  
   Otherwise Hugo can’t find your config and content.

With Hugo now running via Homebrew and the hugo-book example content copied in, I have a clean, functional documentation site and a much better understanding of how Hugo behaves on WSL.

---

## 9. Current working setup (summary)

- **Environment:** WSL (Ubuntu)
- **Hugo install method:** Homebrew on Linux
- **Hugo version:**

  ```bash
  hugo version
  # hugo v0.152.2+extended+withdeploy linux/amd64 VendorInfo=brew
  ```

- **Project root:**

  ```bash
  cd ~/code/sandwich
  ```

- **Run local server:**

  ```bash
  hugo server -D
  ```

- **Prefilled content:**

  ```bash
  cp -R themes/hugo-book/exampleSite/content.en/* content/
  ```

From here, I can focus on writing actual documentation instead of fighting the toolchain.

