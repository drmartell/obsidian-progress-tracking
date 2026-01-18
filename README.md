# Obsidian Progress Tracking (for Tasks / Projects)
An Obsidian Dataview Script to create a simple task or project view dashboard.

I'm making this repository public in case it is of use/inspiration to others, I have not written it with anything other than personal use in mind.

## Usage
TLDR - put the Dashboard file somewhere and put individual task files as siblings in the file system or nested within subfolders.

Given that there are one or more Obsidian markdown task files in the format of the Template, the dashboard script (also in an Obsidian markdown file) will automatically parse them and create a representation.

<img width="743" height="590" alt="image" src="https://github.com/user-attachments/assets/2e489c04-833f-4161-a38b-15a63fe69b6b" />

If task files are in subfolders as compared to the Dashboard script, then those tasks will be grouped by folder as in the image above.

Tasks are ordered conceptually from most behind to least behind. Linear interpolation is used to assess behind-ness.

(and yes apparently I am obsessed with things with wheels and behind on things, the latter of which was the motivation for this)
