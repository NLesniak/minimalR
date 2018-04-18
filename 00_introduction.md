---
layout: lesson
title: "Session 0: Introduction"
output: markdown_document
---




## Philosophy
I have never taken a course or workshop in using R. I've read a lot of books on how to program with R. To be honest, I'm not sure how much they helped. I learned R by taking a single script that I wrote to create a scatter plot and modifying it or "hacking it" to get it to do what I wanted. If I ran into a problem, I would either google the error message or the question I was trying to answer. As I asked around, I learned that most people learned R by hacking their way to success along with a lot of practice. That is the underlying philosophy of this series of lessons. Most programming books slowly build to something useful with silly examples along the way. The first code you will write in Lesson 1 will be the basis of every other piece of code we write in these tutorials. We will start with working code for a plot that could be published and hack it until we have a plot showing which taxa are associated with health or disease.

I suspect that you will understand the first chunk of code we write. We will strive for readable code that is easy to understand. That being said, just because you suspect that the `geom_point` function will add points to a plot, doesn't mean that you know how to use `geom_point` or that you would know how to make a bar chart. Calmly accept your ignorance and know that all will be explained eventually. Learning experts have found that we do not learn best by taking on a topic and beating it to death until we've mastered it. Rather, we learn best when we learn something partially, move on to something else that we also learn partially, but can fold in the previous knowledge to help us improve our partial knowledge of the earlier topic. It's kind of like taking steps forward in a dark room only to get to the end and see that you knew the path all the way along. This is the approach that we will be taking with these lessons. My goal is not to provide a reference on R or to necessarily document every nook and cranny of the language and its myriad packages. I will empower you to do that.

The final philosophical point I will make is that I believe it is important to eat your own dog food as an educator. Everything I teach, is how I want to code and how I want those that work for me to code. There is definitely always room for improvement, but be confident that I'm not trying to sell you on something that I do not use myself. That being said, although I don't claim that the plots we'll make are works of aRt, I do think that they're pretty close to being publication quality. Why make a crappy plot, when you could make a good one that pus your work in the best possible light?

If you notice a bug, something that is unclear, have an idea for a better approach, or want to see something added, please file an issue or, even better, a pull request at the project's [GitHub repository](href="https://github.com/riffomonas/minimalR">minimalR GitHub repository).


## Why R





## What you need to do these tutorials...
* [R](https://cloud.r-project.org/)
* Text editor (e.g. [atom]()) or [RStudio](https://www.rstudio.com/products/rstudio/download/#download)
* [Raw data files](https://github.com/riffomonas/raw_data/archive/0.1.zip). This will download a directory called `raw_data-X.X` where the "X.X" is the version number. Remove the `-X.X` and make sure the directory is uncompressed


## Set up our minimalR project...
* In your home directory or desktop create a directory called `minimalR`
* Move your decompressed `raw_data` directory into `minimalR`. There should only be one thing in `minimalR`, which is the `raw_data` directory.
* To make life easier, you should start with RStudio. Open `RStudio` and do "File->New Project->Existing Directory". Use the "Browse" button to find `minimalR`. Once you're there (you should only see `raw_data` in the directory), select open. My copy of `minimalR` is on the desktop and it lists my "Project working directory" as `~/Desktop/minimalR`. Click "Create Project"
* In the lower right corner you will see that the "Files" tab is selected. In the panel it will have a file called `minimalR.Rproj` and a directory called `raw_data`.
* Quit RStudio
* Use your finder to navigate to your `minimalR` directory
* Double click on `minimalR.Rproj`. This is probably the quickest way to have RStudio open up in your desired working directory.


## Customizing RStudio
* There are many ways to customize RStudio. You can find the options by going to the Preferences window.
* In the first tab, "General" the following items **should never be checked**. You likely don't need any of these to be checked except to be notified of RStudio:
	- Restore .RData into workspace at startup
	- Save workspace to .RData on exit (toggle should say "Never")
	- Always save history
* Click "Apply"
* Click "OK"


## Oversized calculator
On the left side there is a tab for console. This is where we will be entering most of our commands. Go ahead and type `2+2` at the `>` prompt


```r
2+2
```

```
## [1] 4
```

Now type the following at the prompt (feel free to use your own name)


```r
my_name <- "Pat Schloss"
```

Now look in the upper right panel. In the "Environment" tab you'll see that there's a new variable - `my_name` and the value you just assigned it. We'll talk more about variables later, but for now, know that you can see the variables you've defined in this pane. Go ahead and click on the "History" tab. There you'll see the last two commands we've entered.


## Working through tutorials
As you go through the tutorials you should be saving your code in a text file. Note that a Microsoft Word docx file is not a text file! We want a simple file that only contains text, no formatting. Go "File->New File->Rscript". This will open a file called "Untitled1" in the upper left panel and it will push the "Console" panel down along the left side. Save "Untitled1" as `lesson_00.R`. You should now see `lesson_00.R` listed in the "Files" tab in the lower right corner. Go ahead and enter `2+2` in `lesson_00.R`. One of the nice features of RStudio is that you can put your cursor on the line or highlight the lines you want to run in `lesson_00.R` and then press the "Run" button and it will copy, paste, and run the line(s) in the "Console" window. Alternatively, you can check the "Source on Save" button and every time you save the file, it will run the code in that file. Keep in mind that it will run every command so if you have some non-R code in the file, it will likely gag and complain. I would suggest you create a separate `lesson_XX.R` file for each lesson that we do as we work through the lessons.


## My setup
If you run `sessionInfo` at the console, you will see the version of R and the packages you have installed and attached (more about what this all means later). Here's what mine looks like. It's pretty vanilla.


```r
sessionInfo()
```

```
## R version 3.4.4 (2018-03-15)
## Platform: x86_64-apple-darwin15.6.0 (64-bit)
## Running under: macOS High Sierra 10.13.4
## 
## Matrix products: default
## BLAS: /Library/Frameworks/R.framework/Versions/3.4/Resources/lib/libRblas.0.dylib
## LAPACK: /Library/Frameworks/R.framework/Versions/3.4/Resources/lib/libRlapack.dylib
## 
## locale:
## [1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8
## 
## attached base packages:
## [1] stats     graphics  grDevices utils     datasets  methods   base     
## 
## other attached packages:
## [1] knitr_1.20.1 ezknitr_0.6 
## 
## loaded via a namespace (and not attached):
## [1] compiler_3.4.4    magrittr_1.5      tools_3.4.4       stringi_1.1.7    
## [5] R.methodsS3_1.7.1 stringr_1.3.0     R.utils_2.6.0     evaluate_0.10.1  
## [9] R.oo_1.21.0
```