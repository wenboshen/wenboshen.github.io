---
permalink: /
title: "Short Intro"
excerpt: "About me"
author_profile: true
redirect_from: 
  - /about/
  - /about.html
---

Dr. Wenbo Shen is a ZJU100 Young Professor at Zhejiang University. 
His research interests are software and system security, including operating system security, software supply chain security, container security, and program analysis. He has published more than 30 research papers at top-tier academic conferences and won three distinguished paper awards (NDSS 16, AsiaCCS 17, ACSAC 22).

Before joining Zhejiang University, he spent about four years at Samsung Research America in Silicon Valley (Mountain View, California, USA) as a Senior Research Scientist and the Tech Lead of Linux kernel protection. He has designed, implemented, and commercialized multiple kernel security features, including Linux kernel page table protection, control flow protection, and critical data protection.

Dr. Wenbo Shen got his Ph.D. in 2015 from the [Computer Science Department](https://www.csc.ncsu.edu/) of [North Carolina State University](https://www.ncsu.edu/), co-advised by [Dr. Peng Ning](http://discovery.csc.ncsu.edu/) and [Dr. Huaiyu Dai](https://www.ece.ncsu.edu/people/hdai/). He earned his bachelor's degree in Software Engineering in 2010 from [Harbin Institute of Technology](http://www.hit.edu.cn/), advised by [Dr. Weizhe Zhang](http://homepage.hit.edu.cn/wzzhang).

Background
------
* 2019 -  Now, Zhejiang University (Hangzhou, China), College of Computer Science and Technology, Assistant Professor
* 2015 - 2019, Samsung Research America (Silicon Vally, USA), Senior Research Scientist and the Tech Lead
* 2010 - 2015, North Carolina State University (Raleigh, NC, USA), Department of Computer Science, Ph.D
* 2006 - 2010, Harbin Institute of Technology (Harbin, China), School of Software Engineering, B.Eng

Getting started
======
1. Register a GitHub account if you don't have one and confirm your e-mail (required!)
1. Fork [this repository](https://github.com/academicpages/academicpages.github.io) by clicking the "fork" button in the top right. 
1. Go to the repository's settings (rightmost item in the tabs that start with "Code", should be below "Unwatch"). Rename the repository "[your GitHub username].github.io", which will also be your website's URL.
1. Set site-wide configuration and create content & metadata (see below -- also see [this set of diffs](http://archive.is/3TPas) showing what files were changed to set up [an example site](https://getorg-testacct.github.io) for a user with the username "getorg-testacct")
1. Upload any files (like PDFs, .zip files, etc.) to the files/ directory. They will appear at https://[your GitHub username].github.io/files/example.pdf.  
1. Check status by going to the repository settings, in the "GitHub pages" section

Site-wide configuration
------
The main configuration file for the site is in the base directory in [_config.yml](https://github.com/academicpages/academicpages.github.io/blob/master/_config.yml), which defines the content in the sidebars and other site-wide features. You will need to replace the default variables with ones about yourself and your site's github repository. The configuration file for the top menu is in [_data/navigation.yml](https://github.com/academicpages/academicpages.github.io/blob/master/_data/navigation.yml). For example, if you don't have a portfolio or blog posts, you can remove those items from that navigation.yml file to remove them from the header. 

Create content & metadata
------
For site content, there is one markdown file for each type of content, which are stored in directories like _publications, _talks, _posts, _teaching, or _pages. For example, each talk is a markdown file in the [_talks directory](https://github.com/academicpages/academicpages.github.io/tree/master/_talks). At the top of each markdown file is structured data in YAML about the talk, which the theme will parse to do lots of cool stuff. The same structured data about a talk is used to generate the list of talks on the [Talks page](https://academicpages.github.io/talks), each [individual page](https://academicpages.github.io/talks/2012-03-01-talk-1) for specific talks, the talks section for the [CV page](https://academicpages.github.io/cv), and the [map of places you've given a talk](https://academicpages.github.io/talkmap.html) (if you run this [python file](https://github.com/academicpages/academicpages.github.io/blob/master/talkmap.py) or [Jupyter notebook](https://github.com/academicpages/academicpages.github.io/blob/master/talkmap.ipynb), which creates the HTML for the map based on the contents of the _talks directory).

**Markdown generator**

I have also created [a set of Jupyter notebooks](https://github.com/academicpages/academicpages.github.io/tree/master/markdown_generator
) that converts a CSV containing structured data about talks or presentations into individual markdown files that will be properly formatted for the academicpages template. The sample CSVs in that directory are the ones I used to create my own personal website at stuartgeiger.com. My usual workflow is that I keep a spreadsheet of my publications and talks, then run the code in these notebooks to generate the markdown files, then commit and push them to the GitHub repository.

How to edit your site's GitHub repository
------
Many people use a git client to create files on their local computer and then push them to GitHub's servers. If you are not familiar with git, you can directly edit these configuration and markdown files directly in the github.com interface. Navigate to a file (like [this one](https://github.com/academicpages/academicpages.github.io/blob/master/_talks/2012-03-01-talk-1.md) and click the pencil icon in the top right of the content preview (to the right of the "Raw | Blame | History" buttons). You can delete a file by clicking the trashcan icon to the right of the pencil icon. You can also create new files or upload files by navigating to a directory and clicking the "Create new file" or "Upload files" buttons. 

Example: editing a markdown file for a talk
![Editing a markdown file for a talk](/images/editing-talk.png)

For more info
------
More info about configuring academicpages can be found in [the guide](https://academicpages.github.io/markdown/). The [guides for the Minimal Mistakes theme](https://mmistakes.github.io/minimal-mistakes/docs/configuration/) (which this theme was forked from) might also be helpful.
