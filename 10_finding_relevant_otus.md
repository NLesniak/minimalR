---
layout: lesson
title: "Session 10: Finding Relevant OTUs"
output: markdown_document
---

## Learning objectives

* Kruskal-Wallis test
* Correcting for multiple hypotheses
* Plotting OTU data
* Getting help

The strategies we saw in the previous section were pretty effective at allowing us to compare the relative abundance of various taxa across different categories while allowing us to see the variation in the data. These are far superior to the widely used pie and stacked bar plots. Of course, we are rarely super interested in what's going on at the phylum level, we'd rather see what's going on at the genus or OTU level. In this section we'll develop a few methods that will allow us to select out interesting taxa for visualization.

We need to get our shared and metadata files. We need to get our shard data (i.e. `code/baxter.subsample.shared`) converted into a relative abundance table.


```r
source("code/baxter.R")
```

```
## 
## Attaching package: 'dplyr'
```

```
## The following objects are masked from 'package:stats':
## 
##     filter, lag
```

```
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

```r
shared <- read.table(file="data/baxter.subsample.shared", header=T, stringsAsFactors=F, row.names=2)
shared <- shared[,-c(1,2)]

rel_abund <- shared / apply(shared, 1, sum)
metadata <- get_meta()

stopifnot(rownames(shared) == metadata$sample)
```

Here's a plan of analysis that I like using: 1) filter the shared data to only keep those OTUs that have an average relative abundance over some threshold; 2) perform a statistical test to find OTUs that are differentially represented between treatment groups; 3) plot the relative abundances for those OTUs where there is a significant difference by diagnosis.

First, we're going to filter our shared file. We do this for several reasons. Our shared file is very sparse with a lot of OTUs only appearing in a couple of samples or are generally very rare. We would never want to base some inference on OTUs that only appear a few times. Furthermore, by filtering to those more abundant OTUs, we limit the number of hypothesis tests we make reducing the likelihood of false negatives when we correct for multiple comparisons. There's nothing magical about this threshold. In some studies we calculate the average within each treatment group or instead require an OTU to appear in a set number or percentage of samples. We'll create a vector that contains the average relative abundance for each OTU across the samples, find those OTUs that have an average relative abundance over 0.1% (i.e. an average of about 10 sequences per sample), and subset shared object to the more abundant OTUs.


```r
mean_rel_abund <- apply(rel_abund, 2, mean)
abundant <- mean_rel_abund > 0.001
rel_abund_subset <- rel_abund[, abundant]
```

The resulting `rel_abund_subset` matrix represents 130 OTUs. You might wonder what fraction of the sequences are contained within this subset. We can again use the `apply` function across the columns for each sample and then use the `range` function to find the minimum and maximum relative abundance represented for each sample


```r
total_rel_abund <- apply(rel_abund_subset, 1, sum)
range(total_rel_abund)
```

```
## [1] 0.5249763 0.9885090
```

That represents between 52 and 99% of the data. I'll leave it to you to play with this further to come up with a better way of representing the overall relative abundance without getting bogged down in super rare sequences.

In the second step we will apply a test to quantify differential effect of diagnosis on the relative abundance of our remaining OTUs. There are many options here. Some have been specifically developed for microbial ecology (e.g. metastats, lefse). Alternatively, you may choose to use a random forest approach to identify OTUs that provide the best discrimination between the groups. For demonstration purposes we'll perform a Kruskal-Wallis rank sum test using built in R functions. This works, because we have data that are not normally distributed and we have three groups. To run the test on our first OTU we would do the following


```r
dx_ordinal <- factor(metadata$dx, levels=c("normal", "adenoma", "cancer"))
otu_test <- kruskal.test(rel_abund_subset[,1] ~ dx_ordinal)
```

Looking at the output of `otu_test` we get

```

	Kruskal-Wallis rank sum test

data:  rel_abund_subset[, 1] by dx_ordinal
Kruskal-Wallis chi-squared = 0.4699, df = 2, p-value = 0.7906

```

The resulting p-value was 0.79. Let's take a closer look at `otu_test` because we'd like to find a way to extract the p-value from each of our tests. If we run `str(otu_test)` we get

```
List of 5
 $ statistic: Named num 0.47
  ..- attr(*, "names")= chr "Kruskal-Wallis chi-squared"
 $ parameter: Named int 2
  ..- attr(*, "names")= chr "df"
 $ p.value  : num 0.791
 $ method   : chr "Kruskal-Wallis rank sum test"
 $ data.name: chr "rel_abund_subset[, 1] by dx_ordinal"
 - attr(*, "class")= chr "htest"
```

We've hinted at lists before, but this is the first time that we really need to figure out what it's all about. A list is an object made of multiple types of data. The output above indicates that this list has seven objects within it. Clearly we want the value of `p.value`. To access values from the list we use the `$` like we might do to access a column from a data frame.


```r
otu_test$p.value
```

```
## [1] 0.7906116
```

Bingo. We'd like to apply a function that returns the p-value for each column of `rel_abund_subset`. This should sound familiar from our discussion in the previous session.


```r
test <- function(rabund, diagnosis){
	otu_test <- kruskal.test(rabund ~ diagnosis)
	return(otu_test$p.value)
}

p_values <- apply(rel_abund_subset, 2, test, diagnosis=dx_ordinal)
```

We can count the number of p-values that are smaller than 0.05 by calling `sum(p_values < 0.05)` to find 29 OTUs with a small p-value. Because we performed so many tests, thought, we must correct these p-value for performing multiple comparisons. We can do this using the `p.adjust` function. There are several methods for correcting the p-values, but my preference is the Benjimani-Hochberg approach. By doing `?p.adjust` you can find other methods for correcting the p-values. We can adjust our p-values like so


```r
p_values_adjusted <- p.adjust(p_values, method="BH")
```

Now if we do `sum(p_values_adjusted < 0.05)` we find that we have 5 OTUs that were differentially represented between the three groupings. We can retrieve the names of these OTUs and further subset the data as follows


```r
sig_otus <- names(p_values_adjusted[p_values_adjusted < 0.05])
sig_relabund <- rel_abund_subset[ , sig_otus]
```

Great - now we have a matrix of OTUs that are differentially represented between individual with normal colons, adenomas, and cancer. Using these data we will now make a strip plot of the data. Let's get plotting!


### Activity 1
Using the code from the previous session, generate a strip chart that represents the relative abundances of the differentially represented OTUs. Note that to draw the line segments you'll have to generate a variable called `median_sig_relabund`.

<input type="button" class="hideshow">
<div markdown="1" style="display:none;">


```r
median_sig_relabund <- aggregate(sig_relabund, by=list(dx_ordinal), median)

par(mar=c(5,8,0.5,0.5))
plot(NA, xlim=c(0,0.25), ylim=c(19,1), xlab="Relative Abundance (%)", ylab="", axes=F)

start <- 1
end <- 3

for(otu in 1:ncol(sig_relabund)){
	stripchart(sig_relabund[,otu]~dx_ordinal, at=start:end, add=T, vertical=F, method="jitter", jitter=0.3, axes=F, col=dx_color[levels(dx_ordinal)], pch=19)

	segments(y0=(start:end)-0.4, y1=(start:end)+0.4, x0=median_sig_relabund[,otu+1], lwd=4)
	start <- start + 4
	end <- end + 4
}
```

```
## Error in plot.xy(xy.coords(x, y), type = type, ...): object 'dx_color' not found
```

```r
axis(2, labels=colnames(sig_relabund), at=c(2,6,10, 14,18), tick=F, las=2)
axis(1, at=seq(0,1,0.2), label=seq(0,100, 20), las=1)
box()

legend(y=15, x=0.18, legend=c("Normal", "Adenoma", "Cancer"), col=c("gray", "blue", "red"), pch=19)
```

![plot of chunk unnamed-chunk-9](assets/images/10_finding_relevant_otus//unnamed-chunk-9-1.png)

</div>

Fantastic. There's a few things we might not especially like about this plot. First, the relative abundance values are largely compressed against the x-axis. It would be good to have some spread to the data. Second, the OTU names are dreadful and not descriptive. Instead of "Otu000058", we'd probably prefer something like "*Pasteurellaceae* (OTU 58)". Let's see if we can make this happen.

We subsampled the data to 10,530 sequences, so the smallest non-zero relative abundance is close to 0.000095. We'd never see that on a linear scale. Let's see how we can plot the data on a log scale. Using the horizontal strip charts, we can add the `log=x` argument to our initial `plot` command.


```r
median_sig_relabund <- aggregate(sig_relabund, by=list(dx_ordinal), median)

par(mar=c(5,8,0.5,0.5))
plot(NA, xlim=c(0,0.25), ylim=c(19,1), xlab="Relative Abundance (%)", ylab="", axes=F, log="x")
```

```
## Warning in plot.window(...): nonfinite axis limits [GScale(-
## inf,-0.60206,1, .); log=1]
```

```r
start <- 1
end <- 3

for(otu in 1:ncol(sig_relabund)){
	stripchart(sig_relabund[,otu]~dx_ordinal, at=start:end, add=T, vertical=F, method="jitter", jitter=0.3, axes=F, col=c("gray", "blue", "red"), pch=19)

	segments(y0=(start:end)-0.4, y1=(start:end)+0.4, x0=median_sig_relabund[,otu+1], lwd=4)
	start <- start + 4
	end <- end + 4
}

axis(2, labels=colnames(sig_relabund), at=c(2,6,10, 14,18), tick=F, las=2)
axis(1, at=seq(0,1,0.2), label=seq(0,100, 20), las=1)
box()

legend(y=15, x=0.18, legend=c("Normal", "Adenoma", "Cancer"), col=c("gray", "blue", "red"), pch=19)
```

![plot of chunk unnamed-chunk-10](assets/images/10_finding_relevant_otus//unnamed-chunk-10-1.png)

But when we do this, we get an error:

```
Warning message:
In plot.window(...) :
  nonfinite axis limits [GScale(-inf,-0.60206,1, .); log=1]
```

This happens because we have zeroes in the dataset and when you do `log(0)` you get `-Inf` or negative infinity. Not good. As a way around this, we can add a very small number to all of our data. Since our smallest number is close to `1e-4`, we could add `5e-5` to all of our values without changing our perception of the data. Notice that we also need to change our x-axis limits to go from 1e-5 to 0.25


```r
fudge_factor <- 5e-5
transformed <- sig_relabund + fudge_factor
median_transformed <- aggregate(transformed, by=list(dx_ordinal), median)

par(mar=c(5,8,0.5,0.5))
plot(NA, xlim=c(fudge_factor,0.25), ylim=c(19,1), xlab="Relative Abundance (%)", ylab="", axes=F, log="x")

start <- 1
end <- 3

for(otu in 1:ncol(sig_relabund)){
	stripchart(transformed[,otu]~dx_ordinal, at=start:end, add=T, vertical=F, method="jitter", jitter=0.3, axes=F, col=c("gray", "blue", "red"), pch=19)

	segments(y0=(start:end)-0.4, y1=(start:end)+0.4, x0=median_transformed[,otu+1], lwd=4)
	start <- start + 4
	end <- end + 4
}

axis(2, labels=colnames(sig_relabund), at=c(2,6,10, 14,18), tick=F, las=2)
axis(1, at=seq(0,1,0.2), label=seq(0,100, 20), las=1)
box()

legend(y=15, x=0.18, legend=c("Normal", "Adenoma", "Cancer"), col=c("gray", "blue", "red"), pch=19)
```

![plot of chunk unnamed-chunk-11](assets/images/10_finding_relevant_otus//unnamed-chunk-11-1.png)

That looks good, eh? Let's adjust our x-axis labels


```r
fudge_factor <- 5e-5
transformed <- sig_relabund + fudge_factor
median_transformed <- aggregate(transformed, by=list(dx_ordinal), median)

par(mar=c(5,8,0.5,0.5))
plot(NA, xlim=c(fudge_factor,1), ylim=c(19,1), xlab="Relative Abundance (%)", ylab="", axes=F, log="x")

start <- 1
end <- 3

for(otu in 1:ncol(sig_relabund)){
	stripchart(transformed[,otu]~dx_ordinal, at=start:end, add=T, vertical=F, method="jitter", jitter=0.3, axes=F, col=c("gray", "blue", "red"), pch=19)

	segments(y0=(start:end)-0.4, y1=(start:end)+0.4, x0=median_transformed[,otu+1], lwd=4)
	start <- start + 4
	end <- end + 4
}

axis(2, labels=colnames(sig_relabund), at=c(2,6,10, 14,18), tick=F, las=2)
axis(1, at=c(fudge_factor, 1e-3, 1e-2, 1e-1, 1), label=c(0, 0.1, 1, 10, 100), las=1)
box()

legend(y=17, x=0.09, legend=c("Normal", "Adenoma", "Cancer"), col=c("gray", "blue", "red"), pch=19)
```

![plot of chunk unnamed-chunk-12](assets/images/10_finding_relevant_otus//unnamed-chunk-12-1.png)

One last thing we might do is to draw a vertical line to indicate which points were below our limit of detection.


```r
fudge_factor <- 5e-5
transformed <- sig_relabund + fudge_factor
median_transformed <- aggregate(transformed, by=list(dx_ordinal), median)

par(mar=c(5,8,0.5,0.5))
plot(NA, xlim=c(fudge_factor,1), ylim=c(19,1), xlab="Relative Abundance (%)", ylab="", axes=F, log="x")

start <- 1
end <- 3

for(otu in 1:ncol(sig_relabund)){
	stripchart(transformed[,otu]~dx_ordinal, at=start:end, add=T, vertical=F, method="jitter", jitter=0.3, axes=F, col=c("gray", "blue", "red"), pch=19)

	segments(y0=(start:end)-0.4, y1=(start:end)+0.4, x0=median_transformed[,otu+1], lwd=4)
	start <- start + 4
	end <- end + 4
}

axis(2, labels=colnames(sig_relabund), at=c(2,6,10, 14,18), tick=F, las=2)
axis(1, at=c(fudge_factor, 1e-3, 1e-2, 1e-1, 1), label=c(0, 0.1, 1, 10, 100), las=1)
box()

legend(y=17, x=0.09, legend=c("Normal", "Adenoma", "Cancer"), col=c("gray", "blue", "red"), pch=19)

abline(v=0.9e-4, col="gray")
```

![plot of chunk unnamed-chunk-13](assets/images/10_finding_relevant_otus//unnamed-chunk-13-1.png)

That looks pretty good, eh? Now we want to turn our attention to the OTU labels. We'll start by reading in `data/baxter.cons.taxonomy` to an object we'll call `taxonomy` and we'll strip out the confidence scores and generate the name of the most specific taxonomic name that is not `unclassified`. We'll store this in a named vector where the values in the vector are the taxonomic names and the vector names are the OTU numbers.


```r
taxonomy <- read.table(file="data/baxter.cons.taxonomy", header=T, stringsAsFactors=F)
tax_no_confidence <- gsub(pattern="\\(\\d*\\)", replacement="", x=taxonomy$Taxonomy)

no_unclassified <- gsub(pattern="unclassified;", replacement="", tax_no_confidence)
best_taxonomy <- gsub(pattern=".*;(.*);", replacement="\\1", no_unclassified)

otu_name <- best_taxonomy
otu_name <- gsub("_", " ", otu_name)
names(otu_name) <- taxonomy$OTU
```

### Activity 2
Rewrite our `axis(2, ...)` function to use our new `otu_name` vector and italicize the names.

<input type="button" class="hideshow">
<div markdown="1" style="display:none;">

```r
fudge_factor <- 5e-5
transformed <- sig_relabund + fudge_factor
median_transformed <- aggregate(transformed, by=list(dx_ordinal), median)

par(mar=c(5,9,0.5,0.5))
plot(NA, xlim=c(fudge_factor,1), ylim=c(19,1), xlab="Relative Abundance (%)", ylab="", axes=F, log="x")

start <- 1
end <- 3

for(otu in 1:ncol(sig_relabund)){
	stripchart(transformed[,otu]~dx_ordinal, at=start:end, add=T, vertical=F, method="jitter", jitter=0.3, axes=F, col=c("gray", "blue", "red"), pch=19)

	segments(y0=(start:end)-0.4, y1=(start:end)+0.4, x0=median_transformed[,otu+1], lwd=4)
	start <- start + 4
	end <- end + 4
}

axis(2, labels=otu_name[colnames(sig_relabund)], at=c(2,6,10, 14,18), tick=F, las=2, font=3)
axis(1, at=c(fudge_factor, 1e-3, 1e-2, 1e-1, 1), label=c(0, 0.1, 1, 10, 100), las=1)
box()

legend(y=17, x=0.09, legend=c("Normal", "Adenoma", "Cancer"), col=c("gray", "blue", "red"), pch=19)

abline(v=0.9e-4, col="gray")
```

![plot of chunk unnamed-chunk-15](assets/images/10_finding_relevant_otus//unnamed-chunk-15-1.png)
</div>

From this plot we see that three of these OTUs are affiliated within the *Ruminococcaceae*. To get more specific, we'd like to tack on the OTU name inside parentheses. We will make the OTU names a bit more attractive, changing them from "Otu000058" to "OTU 58" and combine them with the taxonomic name.


### Activity 3
Using your `gsub` skills, complete the following line of code:

```
otus <- gsub(pattern="??????", replacement="?????", names(otu_name))
```

<input type="button" class="hideshow">
<div markdown="1" style="display:none;">

```r
otus <- gsub(pattern="Otu0*", replacement="OTU ", names(otu_name))
```
</div>

Now we want to past the taxonomic name with the OTU name. We can do this using the `paste` function.


```r
pretty_otus <- paste(otu_name, otus)
names(pretty_otus) <- names(otu_name)
```

Great, but we'd like to put the OTU name in parentheses. We can do that like so...


```r
pretty_otus <- paste(otu_name, "(", otus, ")")
names(pretty_otus) <- names(otu_name)
```

We're gaining on it, but we don't need a space between the parentheses and the OTU name. We can use the `sep` argument to tell the `paste` function to remove those extra spaces. Of course, we need to add a space before the opening parentheses so we don't get "Blautia(OTU 1)"


```r
pretty_otus <- paste(otu_name, " (", otus, ")", sep="")
names(pretty_otus) <- names(otu_name)
```

### Activity 4
Rewrite our `axis(2, ...)` function again to use our new `otu_name` vector and italicize the names.

<input type="button" class="hideshow">
<div markdown="1" style="display:none;">

```r
fudge_factor <- 5e-5
transformed <- sig_relabund + fudge_factor
median_transformed <- aggregate(transformed, by=list(dx_ordinal), median)

par(mar=c(5,13,0.5,0.5))
plot(NA, xlim=c(fudge_factor,1), ylim=c(19,1), xlab="Relative Abundance (%)", ylab="", axes=F, log="x")

start <- 1
end <- 3

for(otu in 1:ncol(sig_relabund)){
	stripchart(transformed[,otu]~dx_ordinal, at=start:end, add=T, vertical=F, method="jitter", jitter=0.3, axes=F, col=c("gray", "blue", "red"), pch=19)

	segments(y0=(start:end)-0.4, y1=(start:end)+0.4, x0=median_transformed[,otu+1], lwd=4)
	start <- start + 4
	end <- end + 4
}

axis(2, labels=pretty_otus[colnames(sig_relabund)], at=c(2,6,10, 14,18), tick=F, las=2, font=3)
axis(1, at=c(fudge_factor, 1e-3, 1e-2, 1e-1, 1), label=c(0, 0.1, 1, 10, 100), las=1)
box()

legend("bottomright", legend=c("Normal", "Adenoma", "Cancer"), col=c("gray", "blue", "red"), pch=19)

abline(v=0.9e-4, col="gray")
```

![plot of chunk unnamed-chunk-20](assets/images/10_finding_relevant_otus//unnamed-chunk-20-1.png)
</div>

Sweet! There's one small problem though - we want the name italicized, but not the parentheses and the OTU name. At this point I'm not really sure what to do to make it look like I want. Many people would take the plot and edit it in Illustrator or some other program. This is a pretty common problem for the work I do, so I'd prefer to figure it out. Also, If I decide to change a threshold or use a different correction for multiple comparisons, I'm going to have to re-edit the figure and that's going to get tedious. Illustrator is $$$ and I'm cheap. So, now what? google. But what do we google? What do we want to do? If you make your search too specific, you won't get anything, make it to vague and you'll get too much! So I google'd "r axis normal and italic". When I did the search, the top hit was:

* http://stackoverflow.com/questions/10216742/how-to-construct-an-axis-label-with-both-normal-italic-and-bold-font

This doesn't exactly answer our question but did get me thinking about `plotmath`. Following the advice to see `?plotmath` revealed that we can string together expressions using `~` and we can make a chunk of text italicized using `italic(...)`. I did some more googling of with variations on my original search and saw some examples using paste, and parse. Nothing really worked. At these moments, it is really best to strip down the problem to the "minimal reproducible example". In other words, can I create an example that shows my problem? Most of the time when I do this, I can figure things out. If not, then I have something simple that I can share with others to ask for help. Here's what I've got...

```
par(mar=c(5,8,1,1))
plot(NA, xlim=c(0,5), ylim=c(1,5), ylab="", xlab="", axes=F)
axis(1)

pet <- c("Dog", "Cat", "Goldfish", "Dog", "Rabbit")
number <- 1:5

formatted <- paste("italic(", pet, ")~(Pet ", number, ")", sep="")
axis(2, labels=sapply(formatted, as.expression), at=1:5, las=2)
```

You'll see that this is pretty free of distractions such as jargon that is really only relevant to us (e.g. OTU) and is pretty simple at 7 lines of actual code. As I mentioned, after a bit of googling and experimenting this was the closest I could get. You'll see that the plot includes the syntax, but it doesn't render it properly. Without anywhere else to go, I went to stackoverflow to get some help. This is a great repository of information related to computer programming. Before you post a question, be sure to search the repository to make sure that your question hasn't been answered yet and post a brief description of what you want to do with you minimal example. Many of the answers can be snarky, but the R community is pretty helpful and will generally be able to get you an example pretty quickly. So I posted this:

http://stackoverflow.com/questions/37364278/creating-axis-values-that-mix-italic-and-normal-font

As you look through the other stackoverflow questions, many of them seem a bit obscure. But frequently there will be one in there that is at your skill level. Answer it! Many people find that finding solutions to these types of problems really hones their skills. I got a bit impatient waiting for an answer so I posted a link to the question to twitter (https://twitter.com/ledflyd/status/734078158329262080) and Zachary Kurtz answered.

```
par(mar=c(5,8,1,1))
plot(NA, xlim=c(0,5), ylim=c(1,5), ylab="", xlab="", axes=F)
axis(1)

pet <- c("Dog", "Cat", "Goldfish", "Dog", "Rabbit")
number <- 1:5

formatted <- lapply(1:5, function(i) bquote(paste(italic(.(pet[i]))~"(Pet ", .(number[i]), ")", sep="")))

axis(2, labels=do.call(expression, formatted), at=1:5, las=2)
```

I'm not entirely sure what is going on here, but I think I know enough to scale back up to my "real" example.



```r
fudge_factor <- 5e-5
transformed <- sig_relabund + fudge_factor
median_transformed <- aggregate(transformed, by=list(dx_ordinal), median)

par(mar=c(5,13,0.5,0.5))
plot(NA, xlim=c(fudge_factor,1), ylim=c(19,1), xlab="Relative Abundance (%)", ylab="", axes=F, log="x")

start <- 1
end <- 3

for(otu in 1:ncol(sig_relabund)){
	stripchart(transformed[,otu]~dx_ordinal, at=start:end, add=T, vertical=F, method="jitter", jitter=0.3, axes=F, col=c("gray", "blue", "red"), pch=19)

	segments(y0=(start:end)-0.4, y1=(start:end)+0.4, x0=median_transformed[,otu+1], lwd=4)
	start <- start + 4
	end <- end + 4
}

formatted <- lapply(1:length(otu_name), function(i) bquote(paste(italic(.(otu_name[i]))~"(", .(otus[i]), ")", sep="")))
names(formatted) <- names(otu_name)

axis(2, labels=do.call(expression, formatted[colnames(sig_relabund)]), at=c(2,6,10, 14,18), tick=F, las=2)
axis(1, at=c(fudge_factor, 1e-3, 1e-2, 1e-1, 1), label=c(0, 0.1, 1, 10, 100), las=1)
box()

legend("bottomright", legend=c("Normal", "Adenoma", "Cancer"), col=c("gray", "blue", "red"), pch=19)

abline(v=0.9e-4, col="gray")
```

![plot of chunk unnamed-chunk-21](assets/images/10_finding_relevant_otus//unnamed-chunk-21-1.png)

BOOM! Once you get it working, be sure to go back to your stackoverflow question and check the answer as being correct. Looking back at the answer I noticed the `bquote` function being called. I'd never heard of that before. I googled "Rstats bquote example" and got a great example that in hindsight was exactly what I wanted: https://blog.snap.uaf.edu/2012/11/29/an-r-bquote-example/.
