# splicejam version 0.0.60.900

## changes to existing functions

* `groups2contrasts()` updated a bug that was not maintaining
factor order in two-way contrasts. Slightly enhanced logic
used in argument `removePairs` so that it works with single
factor contrasts (where that factor does not change.)
* `defineDetectedTx()` now uses decimal values for the mean
count and mean abundance group values, rounded to the
0.1 decimal place.

# splicejam version 0.0.59.900

## changes to existing functions

* `defineDetectedTx()` default argument value `cutoffTxTPMExpr=0.1`
which was previously `cutoffTxTPMExpr=2`. It appears that this
threshold is somewhat dependent upon the input data, and that
`2` was too stringent for the majority of datasets testes since the
Farris et al publication. Therefore, a more suitable default
value is less restrictive of low-TPM abundance values.
This change also makes the count-based threshold the dominant
filter for technical detection above background noise, which
makes some sense because count noise is somewhat more of a
hard boundary.

## bug fixes / changes to R-shiny sashimi plots

Overall, the changes allow some coverage data to be "missing",
though it will generally try twice to retrieve coverage. When
coverage is "missing" it is rendered empty, with no other
error or message. This change may not be ideal, if the coverage
URL is mis-typed for example, but it beneficial during a network
outage.

* `sashimiAppServer()` was updated to repeat the call to
`prepareSashimi()` when the returned object contained
a list element `some_null=TRUE`, which indicates that one
or more underlying elements of the data was returned
as `NULL`, indicating a failure. In this case, `prepareSashimi()`
is called in a way that does not re-use the existing cache.
Note that `prepareSashimi()` calls other functions, each
of which caches data. If those functions return `NULL`,
they also assign attribute `attr(x, "some_null") <- NULL`,
which helps cascade this failure up the chain to calling
functions.
* `import_juncs_from_bed()` was updated to use `tryCatch()` to
catch errors when the requested file cannot be retrieved. When
memoise cache is being used, it will try to clear the cache then
try again, or will try again without using memoise.
* `getGRcoverageFromBw()` was updated to enhance the
memoise cache repair logic: If coverage is zero-length,
it tries again. Zero-length coverage can happen from
a zero-length memoise cache file, or when the
`rtracklayer::import()` function returns zero-length coverage.
That tends to happen when the file is not accessible, network
is down, or the path is invalid, etc. Nonetheless, memoise
will happily store a zero-size cache file (!) with no
clear way to remove or ignore it. So we try to call
`memoise::drop_cache()` which only exists in memoise version
`1.1.0.9000`, currently on Github with no expected CRAN release
timeline. If we cannot remove the cache file, we try to get
coverage without using memoise. The overall summary:

   * If coverage is empty, try remove the empty memoise cache file.
   * If we removed the empty memoise cache file, we try again to retrieve
   coverage using memoise. The new memoise cache file will either
   be empty (if the source failed again), or non-empty (the second attempt
   was successful). If the file is non-empty, the repair worked,
   and future calls should use the memoise cache file without problem.
   * If it could not remove the empty memoise cache file, it tries to
   retrieve coverage again without using memoise. In this case, the
   empty memoise file still exists, which means all future calls will
   not use the cache file.

* `combineGRcoverage()` was modified to be tolerant to missing
coverage data. Typically when coverage is missing, the data
returned from `getGRcoverageFromBw()` returns a vector of `0`
values, however sometimes there is no data returned for a single
URL, perhaps due to network outage. Nonetheless, `combineGRcoverage()`
will now ignore any samples that are not present in the colnames
of the `GRanges` object supplied.


# splicejam version 0.0.58.900

## Ongoing issues with file caching and R-shiny efficiencies

* The `launchSashimiApp()` function which starts the R-shiny app
sometimes encounters errors when the required files are not
available, as has been the case with server or network outages.
Since each requested file is stored in a `memoise` local
file cache, a previously-accessed gene should pull from the cache
and avoid these errors. But newly accessed genes result in
corrupted (or zero-size) cache files. Splicejam tries to
detect corrupt cache files and reload from the source
(referring to `filesDF`) however this process is proving to
be imperfect, and the sort of thing that is difficult to
debug. 
* The remedy for weird errors in the R-shiny app is usually 
to remove all *_memoise subdirectories,
which forces the cache to be rebuilt.
* Some reference numbers regarding speed: In our testing, the typical
new gene request takes about 10-20 seconds to retrieve data,
assemble the ggplot object, then display the plot -- for 8 covergae
files and 8 junction files.
A cached gene takes about 3-5 seconds to display the plot,
almost all of that time is ggplot rendering its own plot object.
In principle, R-shiny has the ability to cache rendered
ggplot objects, which results in response of less than 1 second
for cached *rendered* plots.
However that process only works when the cached rendered plot has
identical dimensions to the newly requested plot.
Splicejam currently scales the plot to size of the web browser,
which means the benefit would only help each browser size.
For example, splicejam works really well on a phone web browser (!),
somewhat surprising to me, but it turns out to be great. 
I use it quite a lot when I'm on the go, between meetings,
and admittedly during some meetings.

## Updates to existing functions

* `prepareSashimi()` now positions junction labels at the maximum
edge of the junction arc, in stranded fashion. Might consider
adding text adjustment so the label is not centered at the edge.
* `plotSashimi()` now applies `scales::comma(..., accuracy=1)` by
default, which removed the decimal values from junction labels.
Some recent update to `scales::comma()` seems to have caused
decimal values to appear by default. A new argument `junc_accuracy`
is passed to `scales::comma(..., accuracy=junc_accuracy)` to
allow customization.
* `plotSashimi()` fixed issue where `junc_alpha` was only being applied
when data contained junctions without coverage, now the `junc_alpha` is
applied in all cases.
* `sashimiAppServer()` now displays the strand of the gene found
in the top search pane, which allows someone to set the coverage strand
consistent with the gene if desired.

# splicejam version 0.0.57.900

## Bug fixes

* Fixed issue with plotly rendering in R-shiny context, it
was throwing an error `"TypeError: postRenderHandlers is undefined"`.
Upon deeper inspection, the `uiOutput()` element being used
to display both ggplot and ggplotly output, was somehow not
providing the plotly javascript dependencies to include
relevant `postRenderHandlers`. The workaround was to add an empty
`plotlyOutput("plotly_blank")` to the UI, which appears
to cause the required javascript dependencies to be loaded.
Stackoverflow threads and Github issues suggest there exists a
method to load dependencies when embedding htmlwidgets
inside tagList context -- for example when embedding a custom
htmlwidget inside a table cell.
Perhaps the proper remedy is related.

# splicejam version 0.0.56.900

## Bug fixes

* Fixed rare error in `annotateGRfromGR()` with numeric columns,
which failed to initialize a new GRanges column with numeric(0)
instead of using a numeric form of NA, which was accomplished
with `c(0, NA)[2]`.

# splicejam version 0.0.55.900

## changes

* Changed `plotSashimi()` argument default
`ylab="read depth"`, previously it was `"score"`.
In future, normalized data might use `ylab="normalized read depth"`
but that would be a custom option for the analyst.
* `plotSashimi()` arguments `"xlabel"` and `"xlabel_ref"` control
the x-axis label, which by default `xlabel_ref=TRUE` will use
the reference (chromosome) as the x-axis label, which makes
sense because the axis shows chromosome coordinates. The `"xlabel"`
argument is intended to allow a fully custom label.

# splicejam version 0.0.54.900

## bug fixed

* Fixed bug in `compressPolyM()` edge case, polygons with only
two x values and y values all zero; threw an error
`Error in x[idx1, ] : only 0's may be mixed with negative subscripts`.
New rule requires at least 5 x-axis values, otherwise compression
doesn't seem necessary anyway.

## changes

* New argument `compress_introns` added to `exoncov2polygon()`
and `prepareSashimi()`, which determines whether to call
`compressPolygonM()` when creating polygon coordiantes for
coverage data. Setting `compress_introns=FALSE` saves
about 20% of the time it takes to prepare sashimi data.
* Changed default `compressPolyM()` argument from `minRatio=3`
to `minRatio=5`, the threshold of compression above which
polygon coordinates are adjusted.
* Changed `sashimiAppServer()` to create the memoise version
of `prepareSashimi()` only once, instead of re-creating it each
time. Probably no noticeable effect, but it feels better.

# splicejam version 0.0.53.900

## changes

* New README.Rmd file with a visual example and description
of a sashimi plot. New small data object `"demodata"` which
contains the minimum required to produce the Gria1 plot.
* Added splicejam hexsticker with an embedded sashimi plot.
An R package isn't an R package without a hexsticker.

# splicejam version 0.0.52.900

## enhancements

* Slight enhancement to `prepareSashimi()` to change the
`"name"` column to a factor, in order to force the drawing
order of junction arcs, so junctions are drawn in order of
most dominant, to least dominant, then shorter spans to
longer spans. Smaller score, wider junctions are drawn last,
since they are typically the most difficult to see otherwise.
"Dominant"" refers to the
`junction_rank` calculated by having the highest score
among junctions that start or end at the same coordinate:
`junction_rank=3` means the junction has the highest score
on both sides, and has the darkest color;
`junction_rank=2` has the highest score on one side but 
not both sides, and is lighter in color; and
`junction_rank=1` does not have the highest score on
either side of the junction, and has the lightest color.
The `junction_rank=1` entries are now drawn last, since
they typically have the lowest scores, and are small
and thus easily obscured.

## bug fixes

* `splicejam-sashimi-server()` was updated to handle changes
in the gene search field properly, avoiding edge cases
with un-detected genes that have no sashimi data.
* `prepareSashimi()` was updated to handle absence of splice
junction data, in a more robust way.
* `spliceGR2junctionDF()` was updated to handle missing
junction data, ultimately returning NULL which is easier
handle consistently by other downstream functions.
Also made small update to handle `sample_id` as a factor
when input data is supplied only as a GRanges object.

# splicejam version 0.0.51.900

## changes

* `bgaPlotly3d()` new argument `sampleGroups` allows data to
be re-grouped to define custom group centroids. Main driver is
when running BGA on un-grouped data, for example where each
replicate is its own group. Using `sampleGroups` allows the
proper sample group to be highlighted on the plot. This scenario
is mainly only advised when needing to run standard PCA or COA,
or when there are fewer than four groups, since BGA produces
`n-1` dimensions. Code was also slightly update to prepare
for broader use outside of BGA.
* `bgaPlotly3d()` was updated to handle the `"textposition"` argument
to plotly labels, allowing individual adjustment of each centroid
and sample label.
* `bgaPlotly3d()` new argument `geneLabels` allows replacing
gene identifier with a custom label, for example replacing
assay ID with gene symbol.
* `gene2gg()` updated to maintain a minimum y-axis range when
`labelExons=FALSE`. Previously the exons could be cropped.

## bug fixes

* `bgaPlotly3d()` bug fixed which prevented display of scaled
coordinates from the `bgaInfo` object, mainly because the scaled
coordinates are not supplied for individual samples. Instead the
scaling factor (adjustment used to convert centroid coordinates
from raw to scaled) is applied to sample coordinates to produce
equivalent scaled coordinates. That said, I recommend using
un-scaled coordinates, which keeps the x, y, and z axis scales
proportional to their relative contribution, which visually
reinforces the relative strength of separation in each dimension.

# splicejam version 0.0.50.900

## bug fixes

* Fixed regression in `stat_diagonal_wide_arc()` which calls
`ggforce::StatBezier`. Version 3.1.0 of ggforce changed the
required call to that function, breaking the dependency and
causing junction arcs to be invisible as a result. The new
interface mimics how `ggforce::stat_diagonal_wide()` calls
`ggforce::StatBezier`. I hope the interface does not change
in future.
* Added package version dependency for `ggforce` version 0.3.1,
which is the point where the API changed with `ggforce::StatBezier`.
* Fixed vignettes which assumed optional R packages were installed,
for example `org.Mm.eg.db`. Now if the package is not available,
it skips the corresponding sections.

# splicejam version 0.0.49.900

## changes

* Updated the `"Guides"` tab of the R-shiny Sashimi app.
It now also prints the R version, and the package
versions of jam packages.
* Updated `DESCRIPTION` to include higher specific
version numbers for several packages, to force an update
if needed.
* Updated vignette to remove hard dependency on the `tximportData`
package, making it optional. This change prevents the 250 MB
package from being required only for the example vignette.
In future, a random generated expression data matrix may
be created.

# splicejam version 0.0.48.900

## bug fix

* Fixed small bug in `getGRcoverageFromBw()` that used default
memoise path `"memoise_coverage"` instead of `"coverage_memoise"`
as with other functions. Only affected directly calling
`getGRcoverageFromBw()` since other functions passed the
directory as an argument, and thus probably only me.

## changes

* The default plotly does not enable highlighting. It is currently
too slow and laggy, subject of future optimization.
* Font sizing is reworked, to start with 14 point font and
add/subtract from that font size. It converts to ggplot2
convention of `"mm"` units, since it appears to ignore grid
`"pt"` units. The bast plot font size can be adjusted, affecting
everything including exon labels. The exon labels can separately
be adjusted relative to the base font size.
* The coordinate label on the R-shiny app includes the gene,
to make it clear when a new gene search results in a new
coordinate range.
* The `scale_factor` values are applied outside the memoise
cache strategy, which should allow modifying the `scale_factor`
values without invalidating the cache. It seems correct to
cache the unadjusted data, then separately apply any adjustment.
* The `"farrisdata"` package was added to the `"Guides"` tab
which lists the versions of relevant packages.
* The plot config side menu icon was changed from `"wrench"`
to `"gear"`, thanks to @DivadNojnarg of the
`"RinteRface/shinydashboardPlus"` package on Github!
* Updated R package dependencies for minimum versions for
`"shinydashboardPlus"` and `"farrisdata"`.

# splicejam version 0.0.47.900

## Changes

* In near future, `sashimiAppConstants()` may become
the standard way to prepare the dependencies for
sashimi plots, including the gene-transcript-exon
models, the filesDF `data.frame`, the flattened
exons, and color substitutions. This function will
be able to take a GTF file, and prepare all the
tx2geneDF cross-references, the exon and CDS exons,
the flattened exons, using `detectedTx` if available
for enhanced exon structures.
* Package dependency on memoise, using Github version
`1.1.0.9000` which allows invalidating a single problem
cache file. It has been available since October 2018
but not released to CRAN yet. This changes is a step
toward recovering from network outages, which currently
create empty cache files which cannot be recovered
without emptying the entire cache.

## Bug fixes

* `getGRcoverageFromBw()` now checks if memoise cached data
is `NULL`, which happens during network outage. If the data
returned from the cache is `NULL` it clears the cache for that key,
then tries again.
* `import_juncs_from_bed()` now checks memoise cached data for
`NULL` as described above.
* `sashimiAppConstants()` was refactored, to define variables
in a systematic way, by default updating global environment
for convenience, but optionally creating a new environment
inside which the required variables are populated. In future
this technique may become the default here along with downstream
functions, which may reference the environment rather than
data objects themselves.


# splicejam version 0.0.46.900

## Bug fixes

* Updated `groups2contrasts()` to detect when any factor level
contains `"-"`, and converts the `"-"` to `"."` prior to
detecting contrasts. Previously the `"-"` interfered with
proper contrast names, and resulted in some contrasts
not being returned by this function. Note that this function
avoids using `base::make.names()` because that function
aggressively converts other valid characters to `"."`.
However, if downstream functions require `base::make.names()`
compliant names, run that function prior to calling
`groups2contrasts()`.
* `sortSamples()` was updated to match `"wildtype"` as a control term.
* `groups2contrasts()` was updated to check for specific scenarios
where iFactors and iSamples may be missing, to cover a variety
of common scenarios. Error messages were updated to be specific
to the expected input.
* `flattenExonsBy()` removed the `...` dots argument, to
try to appease the memoise caching logic, which is suspected to
be comparing the `...` data when invalidating the cached files.
Ironically, this change itself will invalidate existing cached
files.
* `internal_junc_score()` was updated to handle data with no
`sampleColname` column, which fixed an issue with the example
code.
* `stackJunctions()` was updated to handle multiple sample_id
in the same operation, keeping each sample_id independent during
stacking. This update should fix the edge cases where
junctions appeared to be improperly stacked.
* `to_basic.GeomShape()` is now exported, since it was causing
problems during the call to `plotly::ggplotly()` when converting
`ggforce::geom_shape()` to plotly format. Somehow it worked for
the Sashimi coverage `ggforce::geom_shape()` but not for
gene exon `ggforce::geom_shape()`. I am not always here to
understand fully, but to make things work.


## changes to Sashimi workflow

* `prepareSashimi()` now returns list item `"df"` which contains
the merged coordinates for coverage, junctions, and labels. This
`data.frame` overrides the need for `plotSashimi()` to merge this
data on the fly, and takes some extra logic away regarding
junction rank, label positions, etc. It also returns `"ref2c"`
as an attribute to the `"df"`, to make sure it is available
even when requesting only the `"df"` format. The `"ref2c"`
contains the functions used to compress the introns on
the x-axis scale.
* `plotSashimi()` now expects the input `sashimi` object to
contain list element named `"df"` to include the full coordinate
`data.frame` used for plotting. Note that `plotSashimi()` will
now require output from `prepareSashimi()` in version 46 or higher.
Fortunately, memoise already invalidates the cache for even the
slightest change in any molecule in the world, so caching should
not be problematic.


# splicejam version 0.0.45.900

## R-shiny changes

* Made Sashimi plot the default tab. Fixed regression caused by
page load with partially initialized input values.
* Added `aboutExtra` as optional text to describe the data used
in the R-shiny app.
* Added R package versions to the Guides tab.
* Added option to show legend in plotly interactive plots, which
allows for some interesting filtering options, like hiding coverage,
or junctions, or subsets of junctions based on predominance
(.1 is minor, .2 is mixed, and .3 is major predominance.) Junction
ranks are based upon having the highest score at each junction end,
typically the predominant junction represents the predominant
transcript isoform.

## changes

* `plotSashimi()` new argument `junc_alpha` to control the alpha
transparency of junctions, to allow transparency in cases where
the intron coverage may be obscured by the junction arc.
* Increased default minimum junction arc height from 100 to 200,
the effect should be slightly higher junction arcs. In future,
will consider inspecting intervening coverage max height as perhaps
better estimate of a minimum junction arc, plus some buffer height.

# splicejam version 0.0.44.900

## bug fixes

* Fixed regression in R-shiny, updating the progress bar
using gene in the caption, but the function did not need
to know the gene.
* Improved overall handling of reactive gene, sample_id values.

# splicejam version 0.0.43.900

## changes

* `plotSashimi()` and `gene2gg()` have argument `label_coords` for
optional x-axis range to subset labels before displaying with
`ggrepel::geom_text_repel()`. This change solves the issue where
zooming the x-axis range with `coord_cartesian()` kept all labels
which were then arranged at the edges of the plot. This change
also necessitates re-creating the ggplot object, since it has to
change the label coordinate data.
* Fixed some legacy code that assigned `gene` and `sample_id` to
the global environment.
* Removed the `"Samples and Data"` tab from R-shiny for now.
* Improved level of detail in shiny progress bars, now shows each
file being loaded. Not sure it is actually better than having
less detail.
* The order of `sample_id` items in R-shiny `"Sample Selection"`
is now maintained, allowing custom sorting of samples.
* Fixed error when junctions had no intervening junctions to use
when determining the minimum height for a junction arc.
* Fixed errors when part of the Sashimi plot data did not exist,
for example no junction data or no coverage data.

# splicejam version 0.0.42.900

## changes

* `prepareSashimiData()` argument `use_memoise` enables memoise caching
of individual bigWig files per gene, and individual junction files per gene.
The file paths are defined by `memoise_coverage_path` and
`memoise_junction_path`. These options should greatly enhance the
success rate of caching, at the expense of creating more cache files.
* `getGRcoverageFromBw()` now has arguments `use_memoise` and
`memoise_coverage_path` to cache at the level of each bigWig file
and genomic range.
* R-shiny by default caches sashimi data, so repeated calls for the
same `gene` and same set of `sample_id` will retrieve the full cached
sashimi data. However, if any smaller argument changes, including
changes to any bigWig coverage or junction BED file, each individual
step is also cached to prevent retrieving the same coverage or junction
data repeatedly. For practical R-shiny usage, where `sample_id` is
frequently changed, this update should be a substantial improvement.
* Changed default R-shiny to non-interactive. One day will switch it
back, but need plotly ninja skills meanwhile.

# splicejam version 0.0.41.900

## R-shiny app updates

* Changed `sashimiAppConstants()` to improve handling of memoise
flatExonsByTx data.
* Updated the Guides tab with a description of Sashimi plots,
and a how-to for creating a Sashimi plot.

# splicejam version 0.0.40.900

## bug fixes

* Added `shinyjs` package dependency.

# splicejam version 0.0.39.900

## bug fixes

* Fixed regression in junction scale factors, not consistently applied
in `prepareSashimi()`.

# splicejam version 0.0.38.900

## additional package dependencies

* Dependency added for `shinyjqui`
to use `orderInput()` for drag-and-drop selection
and ordering of `sample_id` values.
* Dependency added for `shinycssloaders`.

## updates to the R-shiny app

* New tab "Sample Selection" which focuses on selecting and ordering
the unique sample_id entries found in `filesDF`.
* "Samples and Data" tab allows editing columns, and will
include updated `"scale_factor"` values in normalizing coverage
and junction scores.
* "Samples and Data" tab uses `color_sub` to colorize the table,
and will create colors for any undefined values in the `"sample_id"`
column. Other columns are colorized when all values match `names(color_sub)`
otherwise are not colorized. Columns are arranged so any extra columns
(such as group, subgroup, batch, etc.) would appear beside `sample_id`,
and typically would be colorized also using `color_sub`.
* Added spinning loader indicator to the plot panel, which covers the
time after data is prepared, but before ggplot has created the actual
visualization.
* The "Sample Selection" tab now allows setting the number of plot
panel columns. However, it breaks the synchrony with the gene-exon model,
since the gene panel is still full-width. Already thinking of
alternatives.
* Plot settings like panel height, font relative sizing, and exon
label size (for non-interactive) are available.

# splicejam version 0.0.37.900

## updates

* Added panel height to R-shiny UI, controlling the allocated
height for each facet panel.
* Numerous updates to the plotly highlighting methods.
* Reorganized the R-shiny UI, still in progress.
* Added R-shiny tab "Sample and Data" intended to customize and
select sample_id entries to display or hide in the Sashimi plots.
Purely aesthetic and non-functional, still testing out the many
javascript table options available.

# splicejam version 0.0.36.900

## bug fixes

* Fixed regression in use of `color_sub` to pre-define categorical
colors in sashimi plots.
* Updated plotly highlighting, increasing the success rate in most
test cases.

# splicejam version 0.0.35.900

## changes

* `grl2df()` now returns seqnames, and adjusts yBaseline
within each seqname (chromosome). To plot multi-chromosome
GRangesList, use `+facet_wrap(~seqnames)`
* R-shiny enabled some plotly highlighting features, currently
in testing phases.
* Major refactor of `plotSashimi()` to merge together the junction,
coverage, and label coordinate data.frames to enable plotly and
crosstalk to highlight features. The end result looks amazing.

# splicejam version 0.0.34.900

## changes

* R-shiny new options: minimum junctions; show by strand.
* R-shiny now defines each to the global environment if they do
not already exists: exonsByTx, cdsByTx, flatExonsByGene, flatExonsByTx,
tx2geneDF, detectedTx, detectedGenes. This mechanism is used for now,
both to help define custom input data, but also to help define the
data values needed to produce plots manually outside R-shiny.

# splicejam version 0.0.33.900

## changes

* `stackJunctions()` now also returns "junction_rank"
scored from 1 to 3, where 1=minor, 2=partial, 3=major,
based upon the rank of junctions entering and leaving
exons.
* Junctions are drawn in order from dominant to minor,
which has the effect of ensuring smaller junctions are
displayed. The `junc_fill` is converted to gradient so
a darker color is used for dominant junctions, to try
to highlight the major isoform splice junctions per sample.
Unfortunately, plotly does not honor the polygon render
order (yet).

# splicejam version 0.0.32.900

## changes/fixes

* Added `flatExonsByTx` to R-shiny app, to be able to display
transcript isoform exon structures alongside the flattened
gene-exon model.
* Added R-shiny option to display transcript isoforms alongside
the gene-exon model. Also option to show all or detected
transcripts.

# splicejam version 0.0.31.900

## changes/fixes

* Splice junctions now require at least one junction end within
the range being displayed, to avoid junctions that do not connect
with the visible gene model.
* The x-axis range of multiple plots are more consistently controlled,
to avoid one panel autoscaling to display a wider junction than
other panels.

## additions

* Plotly views contain custom tooltip text, using the
sample_id, the name of the exon or feature, and the track
(referring to the name of the coverage).

# splicejam version 0.0.30.900

## additions

* New ggplot2 geom `geom_diagonal_wide_arc()`, extends
`ggforce::geom_diagonal_wide()` by connecting the left and right halves
of a diagonal into one continuous ribbon arc.

## changes

* `plotSashimi()` now uses `geom_diagonal_wide_arc()` to display
junction arcs instead of using two `ggforce::geom_diagonal_wide()`.
For now, `prepareSashimi()` still provides coordinates to create
two polygons, which are combined in the geom. Still todo: figure
out how to add a label; and make the middle x position immune to
x-axis rescaling.

# splicejam version 0.0.29.900

* R-shiny app now properly keeps interactive plot x-axis ranges
in sync, when zooming the plot.
* Fixed bug in `prepareSashimi()` that occurred when
no junctions overlapped another, resulting in error
"invalid 'type' (S4) of argument".

# splicejam version 0.0.28.900

## changes to existing functions

* `defineDetectedTx()` has a new argument `zeroAsNA`
to handle the special case where some expression values
reported as zero should be treated as `NA` (no data obtained)
and therefore will not be included in group mean calculations.
As transcript-exon models get more "comprehensive", kmer
quantitation tools such as Salmon and Kallisto sometimes
need to assign abundance to one of several nearly identical
isoforms, and in low count scenarios all counts may be
assigned to one or another isoform, leaving zero in the
alternate position. Excluding zero ensures that group
mean values represent only the assigned quantitation.

# splicejam version 0.0.27.900

## bug fixes

* Packages now imported for `launchSashimiApp()`: `shiny`,
`shinydashboard`, `shinydashboardPlus`, `shinyWidgets`.
* `getGRcoverageFromBw()` now returns NULL when a BigWig file
is not accessible. Previously any error caused
`"Error in seqinfo(con) : UCSC library operation failed"` which
could mean the file does not exist, or any number of other validation
checks failed. Since `getGRcoverageFromBw()` used a vector of
BigWig files, any error caused the entire set to fail.
* `getFirstStrandedFromGRL()` added package prefix to the use
of `IRanges::heads()` which was not imported directly.

## changes

* R-shiny plots now set the plot height for ggplot2 or plotly,
depending upon the number of samples, and presence of
gene-exon model. Future versions will allow R-shiny user
customization.
* `bgaPlotly3d()` removed some unnecessary print functions.

# splicejam version 0.0.26.900

## changes

* This version is an incremental update, while we evaluate
some display options of Sashimi plots in the R-shiny app.
* Minor aesthetic changes to R-shiny app, including plotly
interactive options, and evaluating some UI elements in sidebar.
* Added option to display gene-exon model in R-shiny.

# splicejam version 0.0.25.900

## changes

* `makeTx2geneFromGtf()` help docs recommend installing the `R.utils`
package when `.gz` files are required for import, since this process
uses `data.table::fread()`.
* Package dependencies were added for shiny, shinydashboard.

## new functions

* `to_basic.GeomShape()` is a hidden function intended only to
provide basic support of `ggforce::geom_shape()` when converting
ggplot objects to plotly.

# splicejam version 0.0.24.900

## changes

* R-shiny Sashimi app now allows zooming into coordinate ranges,
defined per gene.

# splicejam version 0.0.23.900

## additions

* First working Sashimi R-shiny app, still has issues with
certain genes that needs debugging. Defaults to using
Farris et al data from `farrisdata` package, but can
be overridden with global environment variables.

# splicejam version 0.0.22.900

## additions

* Initial R-shiny `launchSashimiApp()` function.
* Suggests the `farrisdata` package for example data used
for the Farris et all RNA-seq mouse hippocampus manuscript.

# splicejam version 0.0.21.900

## bug fixes

* Fixed examples that did not explicitly call library() for
required libraries.
* Fixed error in `codonUsage2df()` example data file path.

## changes

* Added specific TODO items.
* Added two function categories: `"jam ALE-specific RNA-seq functions"`,
and `"jam codon-usage RNA-seq functions"` to help organize the
large list of accessory functions.


# splicejam version 0.0.20.900

## changes

* Minor fixes to examples.

## additions

* Added data `test_exon_gr`, `test_junc_gr`, `test_cov_gr` for
easy example data for exons, junctions, and coverage, respectively.
* Added "wide" variants of the above test data, with introns
about 100x larger than exons, consistent with mammalian gene
structures.
* Added examples to each data, showing how one would use the
raw data to generate different visualizations used in
Sashimi plots.
* Added sample data to the vignettes and some examples.
* Renamed `flattenExonsByGene()` to `flattenExonsBy()` to reflect
that the function handles `by="gene"` and `by="tx"`.

# splicejam version 0.0.19.900

## additions

* Added `create-a-sashimi-plot.Rmd` which walks through a full example
showing how to create a Sashimi plot.
* Added examples to `stackJunctions()` with schematics, including
example on plotting junctions by themselves.

## changes to existing functions

* Removed `compressGRgaps()` for now, since the methods now try to
keep GRanges intact, and instead transform the x-axis scale to compress
visible gaps.

# splicejam version 0.0.18.900

## changes to existing functions

* `bgaPlotly3d()` now properly handles ellipsoid colors,
previously the colors were assigned but not honored by
`plotly::add_trace()`.
* `combineGRcoverage()` determines strandedness by requiring all
values to be at or below zero, with at least one negative value.
Otherwise, data can have position and negative values and will
be considered positive stranded.
* `plotSashimi()` replaces `jamSashimi()` because it is just more
intuitive... Ah well.
* Fixed small typo in plotSashimi.Rd help that included an unmatched quote.

# splicejam version 0.0.17.900

## new functions

* `jamSashimi()` is used to plot Sashimi data prepared by
`prepareSashimi()` in order to separate the download
and preparation of Sashimi data from the visualization.

## changes to existing functions

* `prepareSashimi()` is refactored to remove the plot functions,
moving them to the new `jamSashimi()`.
* `flattenExonsByGene()` can now handle Tx data, mainly useful
to add CDS regions to existing exon models.
* `gene2gg()` is more robust to edge input cases.

# splicejam version 0.0.16.900

## new functions

* `prepareSashimi()` is a workhorse function that prepares several
types of Sashimi-like data output, including ggplot2-based
Sashimi plots. This plot uses vanilla ggplot2 with custom x-axis
scaling to compress genomic coordinates, while keeping data in
proper genomic coordinate space.
* `gene2gg()` is a lightweight function that creates gene and
transcript exons models, and returns a ggplot2 object for
plotting. It can optionally return the data.frame used by ggplot2.
* `grl2df()` is similar to `fortify()` from ggplot2, or the
`broom::tidy()` functions, that take a custom object and return
a data.frame for downstream use. Here the function currently works
with "rectangle" data (like exons, introns, peaks, etc.), and
"junction" (like junction regions to be represented by
`ggforce::geom_diagonal_wide()`). In future it may also handle
sequence coverage polygons.
* `exoncov2polygon()` converts a GRanges object containing coverage
in the form of NumericList for each exon or intron, into a data.frame
suitable for use by `geom_polygon()` or `ggforce::geom_shape()`.
* `getGRcoverageFromBw()` loads bigWig coverage data for a GRanges of
exons and introns, returning a GRanges object with columns representing
each sample_id. It calls `combineGRcoverage()` to combine coverages
by strand within each `sample_id`.
* `flattenExonsByGene()` is intended to provide a set of unique exons
per gene, using all or only detected transcript exon models. It numbers
exons by contiguous segment, then labels subsections of each exon
with a letter suffix, for example "exon1", "exon2a", "exon2b", "exon3".
Lastly, it can annotate exons as CDS or non-CDS, if given a set
of `cdsByTx` data.
* `make_ref2compressed()` takes a GRanges object of exons (or any
useful feature like ChIP-seq peaks, etc) and returns functions needed
to compress x-axis coordinates, in order to shrink gaps/introns to
a fixed width, suitable for use by ggplot2.
* `spliceGR2junctionDF()` takes junctions GRanges data, and flatExonsByGene,
and summarizes and annotates each junction by the gene_exon connected,
and combines scores when multiple junctions are within "spliceBuffer"
distance of an exon boundary, by default 3 bases. It can accept a
GRanges object that was derived from multiple sample_id, and will return
data summarized within each sample_id. It calls `stackJunctions()` to
combine junctions per replicate.
* `simplifyXY()` takes a vector of coordinates, and simplifies them to
the minimal set of points to represent the polygon. For sequence
coverage data, that process can reduce individual points by 90%,
but it works with any coordinate data. When two or more consecutive line
segments have identical angle (with non-zero distance), they are combined
using only the first and last points.
* `getGRgaps()`, `getGRLgaps()`, `addGRgaps()`, `addGRLgaps()` are functions
that take GRanges input, and return either just the gaps between
non-overlapping regions, or the original features with gaps inserted between
them. Useful to convert a set of exons, to a set of exons and introns.

## other changes

* Added package dependencies to ggplot2, ggforce, ggrepel

## TODO

* Need a way of annotating GRangesList gaps by the parent feature name.
* In future, allow samples to have replicates, optionally allow each
replicate to be independently plotted, while being part of the
parent `sample_id`.
* Allow alternative input for `prepareSashimi()`, namely
BAM alignment files, from which coverage and junctions can be
derived.
* Consider making `ref2c` chromosome-wide, however it would require
x-axis coordinates to know their context, in terms of chromosome/seqname.
* Multiple vignettes to demonstrate workflows: Sashimi plots; preparing
exon GRangesList; using `annotateGRLfromGRL()` to add annotations, etc.

# splicejam version 0.0.15.900

## new functions

* `curateVtoDF()`, `curateDFtoDF()` are data curation functions
to curate a vector, or a data.frame, into a data.frame with
consistent, usable nomenclature for downstream analysis. They use
a flexible yaml format that should help automate analysis pipelines
that start with raw data file import.
* `exoncov2polygon()` and `compressPolygonM()` are basic functions
for sashimi plots, efficiently converting exon/intron coverages to
multi-polygons, then optionally compressing introns and reducing
the information content to roughly similar resolution as uncompressed
regions.
* `spliceGR2junctionDF()` is the summary function to convert a set
of splice junction read counts to gene-annotated junctions, grouped
and summed where needed.
* `closestExonToJunctions()` called by `spliceGR2junctionDF()` is used
to annotate junction ends near compatible exon boundaries, with
some buffer distance allowed to "snap" junctions to the edge.


# splicejam version 0.0.14.900

## bug fixes

* Fixed issue where `numTxs` was not getting populated in `runDiffSplice()`,
otherwise the stats summary is not changed.

# splicejam version 0.0.13.900

## changes

* Added "openxlsx" to Suggests, for exporting Rmarkdown stats tables
to Excel format.
* `runDiffSplice()` includes examples using `groups2contrasts()`, also
allows `txColname,geneColname` to be defined and therefore custom.
* Changed all `verbose=TRUE` to `verbose=FALSE` by default.
* Changed `makeTx2geneFromGtf()` to use `data.table::fread()` ability
to uncompress .gz files, hopefully making it cross-platform.
* `groups2contrasts()` was updated to handle single-factor experiments,
and be more robust to two-factor experiments with missing groups
in the full design table.

## additions

* Added basic RNA-seq workflow to vignettes.

# splicejam version 0.0.12.900

## new functions

* `runDiffSplice()` is a wrapped around `limma::diffSplice()` intended to
capture several steps of pre- and post-processing, optionally applying
`limma::voom()`.
* `groups2contrasts()` takes a vector or data.frame of sample groups,
and determines the relevant pairwise and two-way contrasts, returning
a design matrix and contrast matrix.
* `sortSamples()` sorts biological samples so that known patterns of control
group terms are returned before non-control groups, in order to help
provide useful defaults when setting up sample group contrasts.
* `strsplitOrdered()` provides `base::strsplit()` but returns factor output
whose factor levels are influenced either by factor input, or by other
arguments.

## changes

* `limma` package was added to Imports.
* `runDiffSplice()` has an argument `spliceTest` which defines `test`
when calling `limma::topSplice()`. Only the "t" (t-test) returns fold change,
therefore hits are not filtered by fold change for "F" or "simes".


# splicejam version 0.0.11.900

## new functions

* `ale2violin()` takes output from `tx2ale()` with some arguments, and
produces a ggplot2 violin plot object, as well as the underlying data.
It allows a custom filtering function, which allows filtering gene lists
for relevant regions of expression.

# splicejam version 0.0.10.900

## enhancements

* `shrinkMatrix()` minor fix to remove the shadow `x` in the resulting
colnames.
* `tx2ale()` modified to be tolerant of NA values in the expression matrix.

# splicejam version 0.0.9.900

## new functions

* `sortGRL()` sorts GRangesList objects by chromosome and position.
* `getFirstStrandedFromGRL()` return the first GRanges feature per
GRangesList, ordered properly by strand.
* `annotateGRfromGR()` which annotates one GRanges object based upon
overlaps with another GRanges object.
* `annotateGRLfromGRL()` which annotates one GRangesList object based upon
overlaps only with the same index entries in a second GRangesList object.
* `findOverlapsGRL()` runs `GenomicRanges::findOverlaps()` for the case of
two GRangesList objects, matching at the GRanges level but restricting
overlaps to those matching the same GRangesList index.
* `assignGRLexonNames()` assigns exon names and numbers to a disjoint
set of exons per gene model.

## updates

* Added "S4Vectors" to package imports, since some functions like
`lengths()` have been moved there. Another reason using a package prefix
is clunky at best. If methods have dispatch, and if they did not
conflict between S3 and S4 methods, that would be the better strategy.
It is not a solution to hardcode package names into function bodies,
since every package maintainer has to stay updated on the source package
of all other dependent functions. Those details should be irrelevant to
other package maintainers.


# splicejam version 0.0.8.900

## enhancements

* The "arules" package was moved to "Imports" since it is required
for the `list2im()` function. A slower workaround could be written,
but ultimately the arules package is preferred.
* Configured the package to use pkgdown for function references.
* Added "#' @imports jamba" to import all jamba R functions.

# splicejam version 0.0.7.900

## new functions

* `factor2label()` will convert a factor to a factor label, with the
same order of levels as the input factor, but including summary stats
like the number of items for each factor level. Useful for ggplot2
visualizations, to include counts in the color legend for example.

# splicejam version 0.0.6.900

## enhancements

* Added reference file "Mouse_codon_usage.txt" to the "extdata"
folder, as example input both for farrisSeq.Rmd, and for other
species. Access the file path with this command:
`system.file("extdata", "Mouse_codon_usage.txt", package="farrisdata")`

## new functions

* `codonUsage2df()` imports a text codon usage file, and returns a
data.frame. Support function `dna2codon()` simply takes a character
vector and returns vector whose elements all have three characters,
e.g. all(nchar(x) == 3).
* Two geometric mean functions: `jamGeomean()` is preferred by
the Jam packages, but `geomean()` is provided for direct comparison
to the classical approach.
* `detectedTxInfo()` summarizes the data used to define detected
transcripts for a given gene, or for a given set of transcripts.

# splicejam version 0.0.5.900

## bug fixes

* Minor fix to `makeTx2geneFromGtf()` to remove requirement for
`col.names` during import.
* Several minor documentation updates.

# splicejam version 0.0.4.900

## bug fixes

* Updated the DESCRIPTION file to include proper "Remotes" entries
and depencies for jamba, colorjam, and jamma packages.
* Fixed issue where the `data.table` package required a specific
`#' @import data.table` entry in the roxygen2 entry for the
`shrinkMatrix` function. This issue prevented `defineDetectedTx()`
from working within an R package.

## new RNA-seq functions

* `defineDetectedTx()` is a core RNA-seq function to determine
which transcript isoforms are considered "detected" based upon
counts (or pseudocounts), TPM values if available, and the
relative abundance of isoforms per gene. When "everything with
over 10 counts" is not sufficient.
* `tx2ale()` takes a set of transcript exon models, and returns
a set of alternate 3'UTR regions, similar to ALE (alternative
last exon.) It differs slightly from other methods in that it
aggregates transcript quantities by shared ALE regions,
producing a matrix based upon unique ALEs. It does not use
counts in the ALE region itself, but the aggregate abundance
of all isoforms that contain each ALE. This method allows
tools like Salmon or Kallisto to quantify isoforms using the
best available kmers per isoform, without being restricted
to the ALE regions which have a very wide distribution of
lengths.
* `makeTx2geneFromGtf()` takes a GTF file, and produces a
`data.frame` with tx (transcript) to gene associations.

## new functions

* `list2im()` converts a list to an incidence matrix. It uses
the `arules` package fast methods for creating
`transactions` objects, which are sparse binary matrices with
linked data.frames used to describe rows and columns. Similar
to `SummarizedExperiment` but with optimization specifically for
list-to-matrix and matrix-to-list conversion. Their implementation
is memory efficient as well.
* `bgaPlotly3d()` takes a `"bga"` class, as produced by the
`made4::bga()` function, and produces a 3-D plotly visualization.
It adds ability to group centroids into "supergroups" which are
connected by a spline 3-D curve.
* `spline3d()` is the supporting function to draw interpolated
3-D curves through a set of coordinates.
* `shrinkMatrix()` shrinks a numeric matrix by groups of rows,
essentially a wrapper function for the `data.table` package
functions.
