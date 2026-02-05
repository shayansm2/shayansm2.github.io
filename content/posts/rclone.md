+++
date = '2026-02-05T12:11:08+03:30'
title = 'Syncing Your Local-First Notes With Rclone'
+++

I love working with Obsidian and Logseq. I use Obsidian mostly for note-taking and writing long-lived documents, and I enjoy its cool features like Canvas and additional plugins like Excalidraw, which is my favorite. I also use Logseq for task management and writing short-lived logs. One of their great benefits is that your files are stored locally, unlike Notion—which is actually why I switched from Notion to Obsidian. The only downside is that if you want to use both a desktop and a mobile client for Obsidian or Logseq, there’s no free built-in sync solution; you have to purchase their official sync service.

Of course, there are dozens of free and open-source sync plugins available, but each one typically depends on a specific cloud storage provider. At first, I tried syncing my changes via Git to my GitHub account, which had the advantage of adding version control to my documents. The drawback, however, was that I had to keep the repository private and avoid storing confidential information in my Obsidian vault as much as possible.

Another cool solution I tried is using rclone. It can sync your data with various sources, such as S3-compatible solutions or cloud storage like Dropbox, Google Drive, Microsoft OneDrive, and many more. All you need to do is install rclone, set up your desired remote using rclone config, and then sync your Obsidian vault easily with this command:

```bash
rclone sync /path/to/obsidian/vault your-remote:/path/to/store -v
```

I personally created an alias on my desktop so I can sync my notes quickly and then access them via my Google Drive app.

On my phone, I use an app called Roundsync, which is based on rclone. This lets me sync changes from the remote to my phone and vice versa. If I make changes on my phone and sync them to the remote, I can then pull them back to my desktop using rclone sync remote local.

You can even set up cron jobs to periodically sync your changes to the remote so your notes stay updated without manual effort. One thing to keep in mind: it’s a good idea to keep a backup of your notes somewhere else—like on GitHub or a dedicated S3 bucket—since syncing from two sources can sometimes lead to data loss.

Thanks for reading! I’d appreciate your comments on how you handle syncing your notes.
