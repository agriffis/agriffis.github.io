---
title: Neovim nightly rpms for Fedora
excerpt: Now available on Copr
author: Aron
tags: [fedora, neovim, copr]
layout: article
---

I've made nightly builds of [neovim git
master](https://github.com/neovim/neovim) available for Fedora:

    sudo dnf copr enable agriffis/neovim-nightly
    sudo dnf upgrade neovim python{2,3}-neovim

The builds run at 4am ET daily, if there are changes on the upstream master
branch since the previous build.

## How it works

<div class="post-image">
    <a title="Fedora Copr agriffis/neovim-nightly packages" href="https://copr.fedorainfracloud.org/coprs/agriffis/neovim-nightly/packages/">
        <img sizes="(min-width: 36em) 28em, 100vw"
            src="/img/575/neovim-nightly-packages.png"
            srcset="/img/1440/neovim-nightly-packages.png 1440w,
                    /img/1150/neovim-nightly-packages.png 1080w,
                    /img/1080/neovim-nightly-packages.png 1150w,
                    /img/720/neovim-nightly-packages.png 720w,
                    /img/575/neovim-nightly-packages.png 575w">
    </a>
</div>

[Fedora Copr](https://copr.fedorainfracloud.org/) is a build server where
anybody can build and share rpms, similar to [Ubuntu Personal Package
Archives](https://launchpad.net/ubuntu/+ppas). You can configure Copr to
use [rpkg](https://pagure.io/rpkg-util), which knows how to build an rpm
from an Internet-hosted git reposistory. You can even set Copr to build
automatically via web hook when you push to GitHub.

The tricky bit is that most upstream repos don't have a spec file. So
here's what I did:

1. Fork the upstream repo, add a copr branch
2. Set up Copr to build with rpkg via GitHub web hook
3. Add a spec file to the copr branch and push it to trigger the build
4. Cron job to merge from upstream, update the spec and rebuild

My crontab looks like this:

    MAILTO=aron@scampersand.com
    SHELL=/bin/bash
    PATH=/home/aron/bin:/home/aron/.local/bin:/usr/local/bin:/usr/bin

    # Long-running SSH agent so nightly builds can push to GitHub
    @reboot ssh-agent | grep export > .ssh/agent

    # Run the build at 4am ET
    0 4 * * * nightly-copr.bash src/copr-builds/neovim

You can find the [branch on GitHub](https://github.com/agriffis/neovim) and
the [builds in
Copr](https://copr.fedorainfracloud.org/coprs/agriffis/neovim-nightly/).
And if you want to know more about the nightly builds, you can find the
scripts in my [rpm-tools
repository](https://github.com/agriffis/rpm-tools).