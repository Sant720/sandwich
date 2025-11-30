---
title: "My first time using Hugo"
date: 2025-11-26T20:58:09-05:00
weight: 20
---

# Hugo on WSL: First time setup 
*A real-world log of installing Hugo, fixing errors, and getting a docs site running with hugo-book.*

---

## Introduction

This page documents my **actual terminal journey** getting Hugo installed on a WSL Ubuntu environment, configuring the **hugo-book** theme, and fixing the issues I ran into along the way. I use WSL daily and have experience with similar documentation tools (Nextra, Jekyll, Docusaurus), but this was my first time working with Hugo itself.

My goal was to go beyond a quick demo and get a **real documentation site** running: proper theme, working layouts, and a production-style deployment on Vercel. That meant dealing with package managers, theme compatibility, version mismatches, and a few low-level surprises from glibc.

This write-up includes:

- Wrong install methods I tried (snap, old apt packages, manual tarballs).
- PATH and GLIBC problems.
- The final, stable solution using **Homebrew on Linux**.
- Vercel deployment and build configuration.

> [!NOTE]
> This article is partly a self-reference log and partly a guide for anyone trying to run Hugo with hugo-book on WSL. It assumes you are comfortable with the terminal and basic Git usage.

---

## Installing Hugo on WSL

### 1. First attempts: snap and apt

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

I tried again and got the same error. This happens because **snapd is not running on WSL**, so snap is not a viable way to install Hugo there.

Next, I tried the Ubuntu package via **apt**:

```bash
sudo apt install hugo
hugo version
```

Output:

```console {hl_lines=[1]}
Hugo Static Site Generator v0.68.3/extended linux/amd64 BuildDate: 2020-03-25T06:15:45Z
```

So I had Hugo, but it was **v0.68.3** — very old compared to current releases.

I recognized this potential issue, but since this was my first time using Hugo I wanted to see where it would take me.

> [!WARNING]
> The Ubuntu repositories often ship **significantly outdated** Hugo versions. Always verify the installed version with `hugo version` before committing to a theme.

### 2. Creating the site and adding the hugo-book theme

With apt-installed Hugo, I created the site in the current directory:

```bash
hugo new site .
```

Hugo scaffolded the site and printed the usual `Just a few more steps` message.

Then I initialized Git and added the **hugo-book** theme as a Git submodule:

```bash
git init
git submodule add https://github.com/alex-shpak/hugo-book themes/hugo-book
```

> [!info]
> Adding the theme as a Git submodule makes it easy to pull upstream changes later without copying files manually. However, I later "vendored" the submodule so that I could edit SCSS more easily for this project.

### 3. Trying to run the theme: layout and `css` errors

I tried to run the site using the theme directly:

```bash
hugo server --minify --theme hugo-book
```

I immediately got:

```console {linenos=inline hl_lines=[3]}
Error: add site dependencies: load resources: loading templates:
"themes/hugo-book/layouts/_partials/docs/html-head.html:32:1":
parse failed: template: _partials/docs/html-head.html:32: function "css" not defined
```

This error (`function "css" not defined`) comes from Hugo Pipes / SCSS processing. It typically means:

- Hugo is **too old**.
- It is not the **extended** build.

In my case, I had `v0.68.3` from apt, which is far too old for the current hugo-book theme.

Still, I added some sample content to verify if the site would even load:

```bash
hugo new docs/_index.md
hugo new docs/getting-started.md
```

Then I tried:

```bash
hugo server -D
```

This time Hugo ran, but I got a series of warnings:

```text
found no layout file for "HTML" for kind "page"
found no layout file for "HTML" for kind "section"
found no layout file for "HTML" for kind "home"
```

The page was effectively blank. The root problem was still the **incompatibility between my Hugo version and the theme I had selected**.

---

## Failed upgrade attempts

### 4. Attempting manual upgrades with tarballs (GLIBC and PATH issues)

Realizing I needed a newer Hugo, I removed the apt version:

```bash
sudo apt remove hugo
```

Then I tried installing Hugo manually from GitHub releases. One example:

```bash
cd /tmp
curl -L -o hugo.tar.gz   https://github.com/gohugoio/hugo/releases/download/v0.149.0/hugo_extended_0.149.0_linux-amd64.tar.gz

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

This happened because my shell had cached the old `hugo` path (`/usr/bin/hugo`) from the apt install. After clearing the hash and continuing to experiment with tarballs (including `v0.152.2`), I still ran into problems.

At one point, `which -a hugo` showed:

```text
/usr/local/bin/hugo
/usr/bin/hugo
/bin/hugo
```

and running `hugo version` still reported the old `v0.68.3`, meaning the apt version was shadowing the new one. I reinstalled and removed the apt package multiple times during this process.

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

- Snap did not work on WSL.
- The apt package was too old.
- The prebuilt binary from GitHub did not match my glibc version.

Time for a different approach.

> [!WARNING]
> If you see GLIBC version errors, you are not dealing with a Hugo configuration problem. You are dealing with a **system compatibility issue** between your distro and the binary you downloaded.

---

## How I got a working Hugo

### 5. Installing Homebrew on WSL and letting it handle Hugo

The final (and correct) solution was to use **Homebrew on Linux**. Homebrew installs Hugo in a way that works with the local system and provides the **extended** build.

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

The installer printed a **“Next steps”** section instructing me to add `brew` to my shell. I followed it:

```bash
echo >> /home/santi/.bashrc
echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> /home/santi/.bashrc
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
```

Homebrew also recommended installing GCC, which I did:

```bash
sudo apt-get install build-essential   # Already installed, but safe.
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

- A **modern** Hugo version (`v0.152.2`).
- The **extended** build.
- A binary that actually runs correctly in WSL.

It was time to try running the server.

```bash
cd ~/code/sandwich
hugo server -D
```

This time, with the Homebrew-installed Hugo, the site built correctly.

> [!SUCCESS]
> After several failed installation paths, Homebrew provided a stable, extended Hugo build that works cleanly on WSL.

---

### 6. Prefilling content from hugo-book’s example site

To quickly get meaningful docs content and navigation, I copied the example content from the hugo-book theme:

```bash
cd ~/code/sandwich
cp -R themes/hugo-book/exampleSite/content.en/* content/
```

Then I added some finishing setup touches to `config.toml`:

```toml 
baseURL = "http://localhost:1313/" # Might change this later to support relative URLs. 
languageCode = "en-us"
title = "Sandwich"
theme = "hugo-book"

# Basic params to get started.
[params]
BookTheme = "light"
BookSearch = false # hugo-book docs state that search can be clunky, better to turn it off for this small project.
BookToC = true
BookComments = false

# Goldmark trims unsafe outputs which might prevent some shortcodes from rendering.
[markup.goldmark.renderer]
  unsafe = true
```

Then I ran the server again:

```bash
hugo server -D
```

Now my local site loaded with the **full hugo-book demo content** (sidebar, sections, sample pages, etc.), and I could start replacing it with my own docs, including this setup journey page.

> [!TIP]
> Copying the example site content is a quick way to understand how a theme expects its content tree to be structured before you rewrite it with your own documentation.

---

### 7. Vercel deployment

After all this, Vercel deployment was easy. I did it straight from Vercel's UI by connecting it to my GitHub repository.

However, I quickly ran into this build error:

```bash {linenos=inline hl_lines=[8,9]}
14:55:51.934 Running build in Washington, D.C., USA (East) – iad1
14:55:51.935 Build machine configuration: 2 cores, 8 GB
14:55:52.068 Cloning github.com/Sant720/sandwich (Branch: main, Commit: 9832fa7)
14:55:52.069 Previous build caches not available.
14:55:52.828 Cloning completed: 760.000ms
14:55:53.251 Running "vercel build"
14:55:53.675 Vercel CLI 48.11.0
14:55:54.281 Installing Hugo version 0.58.2
14:55:54.914 Error: add site dependencies: load resources: loading templates: "/vercel/path0/themes/hugo-book/layouts/_partials/docs/html-head.html:32:1": parse failed: template: _partials/docs/html-head.html:32: function "css" not defined
14:55:54.919 Error: Command "hugo --gc" exited with 255
```

It was clear Vercel was defaulting to an outdated Hugo version, similar to the one I got from `sudo apt install hugo`. Luckily, it was a simple fix and all I needed to do was add **environment variables** to the deployment:

```bash
"HUGO_VERSION": "0.124.1",
"HUGO_EXTENDED": "true"
```

After this, deployment was successful.

> [!NOTE]
> If your theme relies on Hugo Pipes or other newer features, double-check which Hugo version your hosting provider is using and override it with environment variables when necessary.

---

## What I learned from this setup

### 8. Lessons learned

A few key takeaways from this whole process:

1. **Snap is not a good choice on WSL.**  
   `snapd` is not running, so `sudo snap install hugo` fails.
2. **The Ubuntu apt package for Hugo can be very outdated.**  
   I got `v0.68.3`, which is too old for modern themes like hugo-book.
3. **Prebuilt GitHub binaries can fail on older distros due to GLIBC version mismatches.**  
   Even after fixing PATH, I could not run the downloaded binary because my system glibc was older than what Hugo was compiled against.
4. **Homebrew on Linux is a great option for WSL.**  
   Installing Hugo with `brew install hugo` gave me a modern, extended build that works cleanly on WSL.
5. **Include environment variables on Vercel deployment to avoid build errors.**  
   Vercel will also default to outdated Hugo versions.

With Hugo now running via Homebrew and the hugo-book example content copied in, I have a clean, functional documentation site and a much better understanding of how Hugo behaves on WSL.

> [!INFO]
> Most of the time spent here was not “Hugo work” but **tooling and environment work**. Once the install path was correct, the actual docs authoring experience was straightforward.

### 9. Current working setup (summary)

- **Environment:** WSL (Ubuntu).
- **Hugo install method:** Homebrew on Linux.
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

- **Vercel deployment:**

  Live at https://sandwich-webpros.vercel.app/.

From here, I can focus on writing actual documentation for the challenge and customizing the site as I please.
