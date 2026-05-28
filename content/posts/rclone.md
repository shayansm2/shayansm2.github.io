+++
date = '2026-02-05T12:11:08+03:30'
title = 'A Simple Local-First Syncing Solution for Personal Notes'
+++

I love working with Obsidian and Logseq. I use Obsidian mostly for note-taking and writing long-lived documents, and I enjoy its features like Canvas and additional plugins such as Excalidraw, which is my favorite. I also use Logseq for task management and writing short-lived logs.

One of their great benefits is that your files are stored locally, unlike Notion, which is actually why I switched from Notion to Obsidian in the first place. The only downside is that if you want to use Obsidian or Logseq on both a desktop and a mobile device, there isn’t a free built-in sync solution; you have to purchase their official sync service.

Of course, there are dozens of free and open-source sync plugins available, but each one typically depends on a specific cloud storage provider. So here is how I implemented a simple local-first solution for syncing my notes between my desktop and phone.

For this setup, I configured a remote that both my desktop and phone sync with. In this setup the contents of my Obsidian vault exist in three places: desktop, phone, and remote. The remote can be various storage providers, such as S3-compatible storage or cloud services like Dropbox, Google Drive, Microsoft OneDrive, and many others.

On my desktop I use `git` and `rclone`.

First, I initialize my Obsidian vault as a Git repository. The main reason is to check whether there have been changes locally before syncing with the remote. Technically, it could have been simpler to use a tool for generating checksums like sha256sum, but I chose Git because it also gives me version control for my notes.

Second, all I needed to do was install `rclone` and set up my desired remote using `rclone config`.

Finally, I added the following scripts for pushing and pulling changes from the remote, and for syncing automatically with a cron job.

```bash
obsidian_push() {
    rclone sync /path/to/obsidian/notes remote:/obsidian-bucket -v \
        --exclude ".git/**" --exclude ".obsidian/**"
}

obsidian_pull() {
    rclone sync remote:/obsidian-bucket /path/to/obsidian/notes -v \
        --exclude ".git/**" --exclude ".obsidian/**"
}

obsidian_sync() {
    echo "running obsidian sync"
    cd /path/to/obsidian/notes || return

    if [ "$(git status -s | grep -v ".obsidian" | wc -l)" -ne 0 ]; then
        echo "push changes"
        obsidian_push

        if [ $? -ne 0 ]; then
            return 1
        fi

        git add .
        git commit -m "update changes" > /dev/null
    else
        echo "pull changes"
        obsidian_pull
    fi
}
```

I also added a cron job to run `obsidian_sync` every hour.

```
0 * * * * obsidian-sync >> /tmp/cron.log 2>&1
```

On my phone, I use an app called [Roundsync](https://roundsync.com/), which is based on rclone. This lets me sync changes from the remote to my phone and vice versa. If I make changes on my phone and sync them to the remote, I can then pull them back to my desktop using `rclone sync remote local`.

Roundsync also has a feature called Triggers, which are similar to cron jobs. In my setup, I pull changes every hour, but I push my changes manually.

So the final setup looks like this:

- On the desktop, everything happens automatically.
- On the phone, pulling happens periodically but pushing is manual.

One thing to keep in mind: it’s a good idea to keep a backup of your notes somewhere else—like on GitHub or a dedicated S3 bucket—since syncing from multiple sources can sometimes lead to data loss.

Thanks for reading! I’d appreciate hearing how others handle syncing their notes.
