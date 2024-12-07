---
title: "Evernote to Paperless-NGX - Revisiting My Digital Home Office After a Decade"
date: 2024-12-07T11:37:19-06:00
draft: false
toc: true
---

## Introduction

I remember it clearly. It was 2012, and we were in our rental house in a new city as we searched for our permanent home. I had set up a home office in the small attic loft space, and my computer desk was surrounded by a filing cabinet and boxes of paperwork that I had moved and never unpacked. I was dreading having to haul these boxes yet again when we found our home, so I decided to go completely paperless and set up a digital home office workflow.

After doing some research, I purchased a Fujitsu ScanSnap S1300 sheet-feed scanner, a Fellows paper shredder, and subscribed to Evernote. I then set up a workflow using ScanSnap software to scan documents into a folder as PDFs, where I would then perform OCR on them and move them to a second folder, which Evernote was configured to watch and ingest. They would go into an Incoming folder in Evernote and I'd periodically triage that area, adding tags and sorting documents into Evernote folders. I spent the next few months slowly scanning what amounted to more than a thousand documents (well more than that in page count) and generating bags and bags full of confetti.

This has served me well for 12 years now, but it's time for a change. The two big motivating factors are:

1. I've never been fully comfortable having this data in a corporate cloud, and more generally, I've been slowly trying to divorce myself from "big tech" for the past couple of years.
2. Evernote as a product has really gone downhill. The big inflection point was in late 2020 when the company released Evernote 10. Pretty much everything got worse - performance, usability, feature set. I hung on to Evernote Legacy as long as I could, but then that also started to grow stale, and I eventually had to cross over. It's been a pain point since.

This fall I've spent a lot of time cleaning up and expanding my homelab, and this feels like the right time to bring this stuff in-house and offload from Evernote.

## The New Approach

There are several viable self-hosted options for migrating off of Evernote, so figuring out which way to go was a little tricky. The first factor is whether your Evernote is more of a note-taking use case or a document management use case - those turn out to be two pretty different needs, and there are great tools on both sides of that fence. My Evernote usage definitely skews document-heavy - of my 7,000 or so notes, nearly 6,000 are scanned PDFs. So, I focused on document management solutions. The one that gets suggested by far the most in this space is Paperless-NGX (which is a descendant of Paperless-NG, which is a descendant of Paperless), but there are others, including Docspell, Mayan EDMS, and Papermerge. Another viable approach is to use a more general cloud storage solution like Nextcloud.

I chose Paperless-NGX (mostly referred to simply as Paperless hereafter). I liked the feature set and interface, the OCR was good, it had good Docker install options and documentation, and it felt like something that my wife would be willing to use as well. You can give it a test drive using [the Paperless-NGX demo](https://demo.paperless-ngx.com) system. 

## Paperless-NGX Setup

I used the [docker-compose templates](https://github.com/paperless-ngx/paperless-ngx/tree/main/docker/compose) as a guide to set up the stack in Portainer, which I use to manage my docker containers. Portainer is mostly more trouble than it's worth, but I'm kinda committed at this point. To just set this up directly in a docker/docker-compose environment, use Paperless-NGX's [docker install script](https://docs.paperless-ngx.com/setup/#docker) instead. There are other install options, including building the docker image yourself and installing directly on bare metal, documented in their [documentation](https://docs.paperless-ngx.com).

Paperless requires three components (webserver, a database, and a Redis broker for managing scheduled tasks). You'll mostly worry about configuring the webserver, which needs a few data volumes and mounts to operate:

* A consumption directory, where Paperless watches for new documents to ingest
* A data directory, where Paperless keeps all of its internal data
* A media directory, where your documents and thumbnails will be stored
* An export directory, where Paperless saves exported documents

The defaults for these are generally fine, but I ended up changing the consumption and export directory to work better for our NAS-based home network. I wanted the consumption directory to be on the NAS, so that it was easy for me and my wife to drop documents into Paperless from our office desktops. And I wanted the export directory to be on the NAS so that it would just be included in my offsite backup without any additional effort. To achieve this, I created new mount points on my docker host server to mount these NAS areas, and then overrode the volume settings on the webserver container for the consumption and export mount points. You can adapt to whatever makes the most sense in your environment.

There were a number of additional settings I needed to configure on the webserver docker container via environment variables. You'll want to give a good read through Paperless's setup documentation and adapt for your needs. Here are my Paperless environment variables. Some of these are defaults, and I've put comments on the ones I changed in non-obvious ways.


* PAPERLESS_CONSUMER_POLLING=30 - Normally, Paperless uses fs notifications to know when things change. Doesn't work on network fs's, so switch it to poll every 30 seconds
* PAPERLESS_DATE_ORDER=MDY - Paperless uses [Dateparser](https://dateparser.readthedocs.io/en/latest/settings.html#date-order) to look for dates in your documents. This one tells Dateparser the expected date format in your document content. It's just a hint to help Dateparser be more successful auto-extracting dates from documents. Most things like memos and bills (at least here in the US) put the date as May 23, 2017 or 5/23/2017 so I went with MDY. It's optional and I find to be of limited help.
* PAPERLESS_DBHOST=db
* PAPERLESS_FILENAME_DATE_ORDER=YMD - If you put dates in your filenames, this one is crucial. It's another Dateparser hint. I used YYYY-MM-DD (sort of - keep reading) on my scanned docs so for me the value is YMD.
* PAPERLESS_OCR_LANGUAGE=eng
* PAPERLESS_OCR_USER_ARGS={"invalidate_digital_signatures": true, "continue_on_soft_render_error": true} - I experienced a couple of errors when doing batch imports of my PDF library. These allowed Paperless to continue on with the batch rather than just fail out.
* PAPERLESS_REDIS=redis://broker:6379
* PAPERLESS_SECRET_KEY=<PAPERLESS GENERATES THIS FOR YOU>
* PAPERLESS_TIME_ZONE=America/Chicago
* PAPERLESS_URL=https://url.or.ip.to.your.instance/ - Paperless breaks surprisingly hard if this isn't correct - I believe the container won't start at all. I've generated proper certs and such for my own instance, so this is https for me, but by default I believe it'll be http.

With these changes, I got Paperless-NGX up and running and ready to suck in all of my documents.

## Workflow Setup

## Migrating from Evernote

There were two significant pain points during my migration from Evernote. I'll discuss my workarounds to each.

### Exporting a set of PDFs from a Large Evernote Library and Preserving Tags

I'll tell you what I did, then I'll tell you what you should do instead.

Evernote makes it maddeningly hard to export the PDFs that you put in. You have a few unappealing options: you can do them one at a time, you can export an entire notebook as a giant PDF and then manually split it, or you can use the open-source `evernote-backup` tool, which will at least do all your notes at once and give you `enex` files (Evernote's export format) for each note, then use another tool to extract and reconstitute the PDFs within the `.enex` files (if you do go this last route, I've found Joplin to be helpful to get back to a folder full of PDFs).

What I did was none of these. When Evernote watches a directory to import new documents, it does NOT delete them after ingesting them (which is different behavior than Paperless-NGX, which does - but also preserves them in more accessible, easier to export ways, so don't worry). Because of this behavior combined with my obsessive nature, I still have every document I've ever ingested in my Evernote ingest directory as the original scanned, OCRed PDFs. I just decided to short-circuit Evernote altogether and dump these documents directly into Paperless-NGX.

The big downside of this is that I then lose whatever organization and metadata I created within Evernote. For me, this basically amounted to one tag and one folder per document. My bet was that redoing this manually on 6,000 documents would be faster than trying to sort through the awful Evernote export options. This turned out to be wrong, but it wasn't TOO bad - it took me a couple of nights to have Paperless ingest them in batches and then me re-tagging in the Paperless interface. Paperless makes it easy to tag big batches of related documents, and its filters made it easy to do things like just get to all of my bank statements in one view, and then tag them all at once. There was just a long tail of the more random/rare types of documents that took a while to slog through.

If I had to do it again just knowing what I knew at the time, I'd probably go the "export to ENEX and then import to Joplin to get back to the PDFs" route. The advantage there is that Joplin would not only extract the PDFs but also keep the tags and folders, which would then make it trivial to export into Paperless by tag or by folder. But, you don't have to worry about that because there's a better way.

The day after I finished this slog, I learned about [enex2paperless](https://github.com/kevinzehnder/enex2paperless). :facepalm: Just use that if it fits your needs.

### Making Sure Filenames are Compatible with Dateparser

I did another thing which inadvertantly shot myself in the foot. When I came up with a file naming scheme, I chose to prefix filenames with YYYYMMDD for the date - e.g. "20210722 Electric Bill.pdf". I love that format and use it all the time in my note-taking. You know who doesn't love that format? Dateparser. Dateparser needs some sort of separator to be able to parse dates from filenames.

So, that led me down a side rabbit hole of writing a Python script to parse my filenames and convert them from YYYYMMDD to YYYY-MM-DD. It's not that big of a deal, but it ate up a couple of hours to make sure I really got it right.

### Ingest Strategy and the Importance of INBOX Tags

If you use `enex2paperless`, this should be pretty painless, point it at your ENEX files and let 'er rip. If not, I suggest doing things in batches. Batches of related documents that are all going to get the same tag(s) would be easiest. Barring that, do it a couple hundred at a time, and then go through the Paperless UI to tag them and get them out of the way, rather than doing them all at once.

I didn't explore this, but you could also write your own import script which uses the Paperless API to import documents.

One thing that's crucial is to set up an INBOX tag in Paperless before starting your ingest. An INBOX tag is simply a tag that automatically gets put on any new scanned document. Paperless doesn't do this by default, and it can make it extremely hard to find documents that still need to be processed by a human as part of the workflow. Luckily, setting one up is trivial in the Tags interface - just look for the Inbox tag checkbox in the tag creation dialog. Then, when processing newly ingested documents, remove the INBOX tag whenever you put on the real tags. This makes it easy to determine what docs you still need to process.

## Tips, Tricks and Lessons Learned

* **Search for ingest tools before going it alone!** - Boy, I really wish I knew about `enex2paperless` a week earlier. :-/
* **Ingest copies, not the originals** - Paperless will delete the file from the consumption directory when it successfully ingests it. It does keep an unmodified original of the PDF within its own storage area, but that's a little harder to get to. For initial ingest, what I did was create a copy of all the PDFs I wanted to ingest, and ingested from my copy area. That way, the first five times when I wanted to fix something and start over, it was easy to recover by recopying whatever it was that I ingested.
* **Paperless is almost too great at not ingesting duplicates** - Related to the last point, if you import a document, then decide you want to redo it, it's not enough to just send the document to the trash in Paperless. You also have to empty the trash - otherwise Paperless will still see your second attempt as a duplicate. Even funnier, the log says something to the effect of "warning, this is a duplicate of a file in the trash", but it still doesn't ingest the duplicate. This is poor behavior in my opinion but once I was aware of it, it was easy enough to avoid.
* **Create an INBOX saved view** - In general, the Paperless documentation is pretty good. One area I found lacking was how to actually use the web UI. I stumbled around a while trying to figure out how to create a saved view on my dashboard. Assuming you've created an INBOX tag already, go to Documents and filter by your inbox tag. Then, go to the Views dropdown and choose Save As... Once you save that view, it's automatically added to your Dashboard. Maybe there's a more direct way to get things on your Dashboard, but that's what worked for me.
* **Turn off auto tagging** - Paperless has a lot of functionality that tries to automate things for you. One of those is auto-tagging. When you create a tag, you'll see an area for Matching Algorithm settings. At least for the kinds of tags I tend to choose, I found the matching algorithm to be almost universally wrong - newly ingested documents would have completely unrelated tags on them with regularity. This might be because it didn't have a lot of data to work with at first. In any case, for your initial bulk ingest, I suggest disabling the matching algorithm on every tag except your INBOX tag.
* **Paperless' UI is very sticky** - It took me a while to get used to Paperless' UI. Starting from a saved view (like everything tagged with my INBOX tag), I wanted to filter to related documents so that I could apply the same tag to say 25 cell phone bills at once. Well, Paperless remembers that filter even if I surf away and come back - in fact, Paperless makes it hard to surf away and come back, because it considers that you have modified the parameters of your saved view and probably want to save them before leaving. I found that quirky. It also remember anything you've checked/unchecked even if you leave and come back. It also doesn't always immediately refresh your view after you make a change (hard to explain without experiencing it). I have come to appreciate that these design choices were intentional and useful in everyday usage of Paperless, but they can add some friction to a large initial ingest workflow. You'll eventually learn little tricks and shortcuts to speed things up.

## Wrapup

In the end, I invested a few evenings (ok, some went REALLY late) of my time to de-platform my paperless workflow from Evernote and take greater control over my digital home office. The interface is more useful than Evernote, the performance is way better, privacy is improved, and I'm a little less beholden to corporate tech.

I need to figure out a solution for the other part of my Evernote usage, note-taking/URL clipping, and then I can completely disentangle myself and cancel my subscription. Currently, Joplin is looking promising for that use case, but I've yet to really explore it deeply. There are way more solutions for note-taking than document management, so there's a lot to choose from, and it really depends on your use case and ecosystem. I want to spend some time exploring a bit more before I commit to a new path.

I hope you found this useful and that I saved you from a pitfall or two in your Evernote to Paperless-NGX migration!
